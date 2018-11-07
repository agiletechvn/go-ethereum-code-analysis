Dial.go is responsible for part of the link in p2p. For example, find the node that establishes the link. Establish a link with the node. Use discover to find the address of the specified node. And other functions.

Dial.go uses a data structure of dailstate to store the intermediate state, which is the core data structure in the dial function.

```go
// dialstate schedules dials and discovery lookups.
// it get's a chance to compute new tasks on every iteration
// of the main loop in Server.run.
type dialstate struct {
	maxDynDials int						// Maximum number of dynamic node links
	ntab        discoverTable			//discoverTable Used to do node queries
	netrestrict *netutil.Netlist

	lookupRunning bool
	dialing       map[discover.NodeID]connFlag		// Linking node
	lookupBuf     []*discover.Node // current discovery lookup results
	randomNodes   []*discover.Node // filled from Table
	static        map[discover.NodeID]*dialTask  // Static node.
	hist          *dialHistory

	start     time.Time        // time when the dialer was first used
	bootnodes []*discover.Node // default dials when there are no peers
}
```

The process of creating dailstate.

```go
func newDialState(static []*discover.Node, bootnodes []*discover.Node, ntab discoverTable, maxdyn int, netrestrict *netutil.Netlist) *dialstate {
	s := &dialstate{
		maxDynDials: maxdyn,
		ntab:        ntab,
		netrestrict: netrestrict,
		static:      make(map[discover.NodeID]*dialTask),
		dialing:     make(map[discover.NodeID]connFlag),
		bootnodes:   make([]*discover.Node, len(bootnodes)),
		randomNodes: make([]*discover.Node, maxdyn/2),
		hist:        new(dialHistory),
	}
	copy(s.bootnodes, bootnodes)
	for _, n := range static {
		s.addStatic(n)
	}
	return s
}
```

The most important method of dail is the newTasks method. This method is used to generate a task. Task is an interface.

```go
type task interface {
	Do(*Server)
}

func (s *dialstate) newTasks(nRunning int, peers map[discover.NodeID]*Peer, now time.Time) []task {
	if s.start == (time.Time{}) {
		s.start = now
	}

	var newtasks []task
	// addDial is an internal method that first checks the node via checkDial. Then set the state and finally add the node to the newtasks queue.
	addDial := func(flag connFlag, n *discover.Node) bool {
		if err := s.checkDial(n, peers); err != nil {
			log.Trace("Skipping dial candidate", "id", n.ID, "addr", &net.TCPAddr{IP: n.IP, Port: int(n.TCP)}, "err", err)
			return false
		}
		s.dialing[n.ID] = flag
		newtasks = append(newtasks, &dialTask{flags: flag, dest: n})
		return true
	}

	// Compute number of dynamic dials necessary at this point.
	needDynDials := s.maxDynDials
	// First determine the type of connection that has been established. If it is a dynamic type. Then you need to establish a reduction in the number of dynamic links.
	for _, p := range peers {
		if p.rw.is(dynDialedConn) {
			needDynDials--
		}
	}
	// Then judge the link that is being established. If it is a dynamic type. Then you need to establish a reduction in the number of dynamic links.
	for _, flag := range s.dialing {
		if flag&dynDialedConn != 0 {
			needDynDials--
		}
	}

	// Expire the dial history on every invocation.
	s.hist.expire(now)

	// Create dials for static nodes if they are not connected.
	for id, t := range s.static {
		err := s.checkDial(t.dest, peers)
		switch err {
		case errNotWhitelisted, errSelf:
			log.Warn("Removing static dial candidate", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)}, "err", err)
			delete(s.static, t.dest.ID)
		case nil:
			s.dialing[id] = t.flags
			newtasks = append(newtasks, t)
		}
	}
	// If we don't have any peers whatsoever, try to dial a random bootnode. This
	// scenario is useful for the testnet (and private networks) where the discovery
	// table might be full of mostly bad peers, making it hard to find good ones.
	// If there are no links yet. And no link was created within 20 seconds (fallbackInterval). Then use bootnode to create a link.
	if len(peers) == 0 && len(s.bootnodes) > 0 && needDynDials > 0 && now.Sub(s.start) > fallbackInterval {
		bootnode := s.bootnodes[0]
		s.bootnodes = append(s.bootnodes[:0], s.bootnodes[1:]...)
		s.bootnodes = append(s.bootnodes, bootnode)

		if addDial(dynDialedConn, bootnode) {
			needDynDials--
		}
	}
	// Use random nodes from the table for half of the necessary
	// dynamic dials.
	randomCandidates := needDynDials / 2
	if randomCandidates > 0 {
		n := s.ntab.ReadRandomNodes(s.randomNodes)
		for i := 0; i < randomCandidates && i < n; i++ {
			if addDial(dynDialedConn, s.randomNodes[i]) {
				needDynDials--
			}
		}
	}
	// Create dynamic dials from random lookup results, removing tried
	// items from the result buffer.
	i := 0
	for ; i < len(s.lookupBuf) && needDynDials > 0; i++ {
		if addDial(dynDialedConn, s.lookupBuf[i]) {
			needDynDials--
		}
	}
	s.lookupBuf = s.lookupBuf[:copy(s.lookupBuf, s.lookupBuf[i:])]
	// Launch a discovery lookup if more candidates are needed.
	// If you don't, you can't create enough dynamic links. Then create a discoveryTask to find other nodes on the network. Put into lookupBuf
	if len(s.lookupBuf) < needDynDials && !s.lookupRunning {
		s.lookupRunning = true
		newtasks = append(newtasks, &discoverTask{})
	}

	// Launch a timer to wait for the next node to expire if all
	// candidates have been tried and no task is currently active.
	// This should prevent cases where the dialer logic is not ticked
	// because there are no pending events.
	if nRunning == 0 && len(newtasks) == 0 && s.hist.Len() > 0 {
		t := &waitExpireTask{s.hist.min().exp.Sub(now)}
		newtasks = append(newtasks, t)
	}
	return newtasks
}
```

The checkDial method is used to check if the task needs to create a link.

```go
func (s *dialstate) checkDial(n *discover.Node, peers map[discover.NodeID]*Peer) error {
	_, dialing := s.dialing[n.ID]
	switch {
	case dialing:
		return errAlreadyDialing
	case peers[n.ID] != nil:
		return errAlreadyConnected
	case s.ntab != nil && n.ID == s.ntab.Self().ID:
		return errSelf
	case s.netrestrict != nil && !s.netrestrict.Contains(n.IP): // network restriction. The IP address of the other party is not in the white list.
		return errNotWhitelisted
	case s.hist.contains(n.ID):	// This ID was once linked.
		return errRecentlyDialed
	}
	return nil
}
```

taskDone method. This method will be called after the task is completed. View the type of task. If it is a link task, add it to hist. And removed from the queue being linked. If it is a query task. Put the query's record in the lookupBuf.

```go
func (s *dialstate) taskDone(t task, now time.Time) {
	switch t := t.(type) {
	case *dialTask:
		s.hist.add(t.dest.ID, now.Add(dialHistoryExpiration))
		delete(s.dialing, t.dest.ID)
	case *discoverTask:
		s.lookupRunning = false
		s.lookupBuf = append(s.lookupBuf, t.results...)
	}
}
```

The dialTask.Do method has different Do methods for different tasks. dailTask ​​is primarily responsible for establishing links. If t.dest is without an ip address. Then try to query the ip address via resolve. Then call the dial method to create the link. For static nodes. If it fails for the first time, it will try to resolve the static node again. Then try dial again (because the static node's ip is configured. If the static node's ip address changes, then we try to resolve the new address of the static node and then call the link.)

```go
func (t *dialTask) Do(srv *Server) {
	if t.dest.Incomplete() {
		if !t.resolve(srv) {
			return
		}
	}
	success := t.dial(srv, t.dest)
	// Try resolving the ID of static nodes if dialing failed.
	if !success && t.flags&staticDialedConn != 0 {
		if t.resolve(srv) {
			t.dial(srv, t.dest)
		}
	}
}
```

The resolve method. This method mainly calls the Resolve method of the discover network. If it fails, then try again after the timeout.

```go
// resolve attempts to find the current endpoint for the destination
// using discovery.
//
// Resolve operations are throttled with backoff to avoid flooding the
// discovery network with useless queries for nodes that don't exist.
// The backoff delay resets when the node is found.
func (t *dialTask) resolve(srv *Server) bool {
	if srv.ntab == nil {
		log.Debug("Can't resolve node", "id", t.dest.ID, "err", "discovery is disabled")
		return false
	}
	if t.resolveDelay == 0 {
		t.resolveDelay = initialResolveDelay
	}
	if time.Since(t.lastResolved) < t.resolveDelay {
		return false
	}
	resolved := srv.ntab.Resolve(t.dest.ID)
	t.lastResolved = time.Now()
	if resolved == nil {
		t.resolveDelay *= 2
		if t.resolveDelay > maxResolveDelay {
			t.resolveDelay = maxResolveDelay
		}
		log.Debug("Resolving node failed", "id", t.dest.ID, "newdelay", t.resolveDelay)
		return false
	}
	// The node was found.
	t.resolveDelay = initialResolveDelay
	t.dest = resolved
	log.Debug("Resolved node", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)})
	return true
}
```

The dial method, which performs the actual network connection operation. This is done mainly by the srv.SetupConn method, and then analyzed later when analyzing Server.go.

```go
// dial performs the actual connection attempt.
func (t *dialTask) dial(srv *Server, dest *discover.Node) bool {
	fd, err := srv.Dialer.Dial(dest)
	if err != nil {
		log.Trace("Dial error", "task", t, "err", err)
		return false
	}
	mfd := newMeteredConn(fd, false)
	srv.SetupConn(mfd, t.flags, dest)
	return true
}
```

The Do method of discoverTask and waitExpireTask,

```go
func (t *discoverTask) Do(srv *Server) {
	// newTasks generates a lookup task whenever dynamic dials are
	// necessary. Lookups need to take some time, otherwise the
	// event loop spins too fast.
	next := srv.lastLookup.Add(lookupInterval)
	if now := time.Now(); now.Before(next) {
		time.Sleep(next.Sub(now))
	}
	srv.lastLookup = time.Now()
	var target discover.NodeID
	rand.Read(target[:])
	t.results = srv.ntab.Lookup(target)
}


func (t waitExpireTask) Do(*Server) {
	time.Sleep(t.Duration)
}
```
