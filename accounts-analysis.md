The accounts package implements the Ethereum client's wallet and account management. Ethereum's wallet offers both keyStore mode and usb wallet. At the same time, the ABI code of the Ethereum contract is also placed in the account/abi directory. The abi project seems to have nothing to do with account management. For the time being, only the interface of account management is analyzed. The implementation code for the specific keystore and usb will not be given for the time being.

Accounts are defined by data structures and interfaces.

## data structure

account number

```go
// Account represents an Ethereum account located at a specific location defined
// by the optional URL field.
type Account struct {
	Address common.Address `json:"address"` // Ethereum account address derived from the key
	URL     URL            `json:"url"`     // Optional resource locator within a backend
}

const (
	HashLength    = 32
	AddressLength = 20
)
// Address represents the 20 byte address of an Ethereum account.
type Address [AddressLength]byte
```

Wallet: The wallet should be the most important interface in it. The specific wallet also implements this interface. The wallet has a so-called layered deterministic wallet and a regular wallet.

```go
// Wallet represents a software or hardware wallet that might contain one or more
// accounts (derived from the same seed).
type Wallet interface {
	// URL retrieves the canonical path under which this wallet is reachable. It is
	// user by upper layers to define a sorting order over all wallets from multiple
	// backends.
	URL() URL

	// Status returns a textual status to aid the user in the current state of the
	// wallet. It also returns an error indicating any failure the wallet might have
	// encountered.
	Status() (string, error)

	// Open initializes access to a wallet instance. It is not meant to unlock or
	// decrypt account keys, rather simply to establish a connection to hardware
	// wallets and/or to access derivation seeds.
	// The passphrase parameter may or may not be used by the implementation of a
	// particular wallet instance. The reason there is no passwordless open method
	// is to strive towards a uniform wallet handling, oblivious to the different
	// backend providers.
	// Please note, if you open a wallet, you must close it to release any allocated
	// resources (especially important when working with hardware wallets).
	Open(passphrase string) error

	// Close releases any resources held by an open wallet instance.
	Close() error

	// Accounts retrieves the list of signing accounts the wallet is currently aware
	// of. For hierarchical deterministic wallets, the list will not be exhaustive,
	// rather only contain the accounts explicitly pinned during account derivation.
	Accounts() []Account

	// Contains returns whether an account is part of this particular wallet or not.
	Contains(account Account) bool

	// Derive attempts to explicitly derive a hierarchical deterministic account at
	// the specified derivation path. If requested, the derived account will be added
	// to the wallet's tracked account list.
	Derive(path DerivationPath, pin bool) (Account, error)

	// SelfDerive sets a base account derivation path from which the wallet attempts
	// to discover non zero accounts and automatically add them to list of tracked
	// accounts.
	// Note, self derivaton will increment the last component of the specified path
	// opposed to decending into a child path to allow discovering accounts starting
	// from non zero components.
	// You can disable automatic account discovery by calling SelfDerive with a nil
	// chain state reader.
	SelfDerive(base DerivationPath, chain ethereum.ChainStateReader)

	// SignHash requests the wallet to sign the given hash to make a signature.
	// It looks up the account specified either solely via its address contained within,
	// or optionally with the aid of any location metadata from the embedded URL field.
	// If the wallet requires additional authentication to sign the request (e.g.
	// a password to decrypt the account, or a PIN code o verify the transaction),
	// an AuthNeededError instance will be returned, containing infos for the user
	// about which fields or actions are needed. The user may retry by providing
	// the needed details via SignHashWithPassphrase, or by other means (e.g. unlock
	// the account in a keystore).
	SignHash(account Account, hash []byte) ([]byte, error)

	// SignTx requests the wallet to sign the given transaction.
	// It looks up the account specified either solely via its address contained within,
	// or optionally with the aid of any location metadata from the embedded URL field.
	//
	// If the wallet requires additional authentication to sign the request (e.g.
	// a password to decrypt the account, or a PIN code o verify the transaction),
	// an AuthNeededError instance will be returned, containing infos for the user
	// about which fields or actions are needed. The user may retry by providing
	// the needed details via SignTxWithPassphrase, or by other means (e.g. unlock
	// the account in a keystore).
	SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)

	// SignHashWithPassphrase requests the wallet to sign the given hash with the
	// given passphrase as extra authentication information.
	// It looks up the account specified either solely via its address contained within,
	// or optionally with the aid of any location metadata from the embedded URL field.
	SignHashWithPassphrase(account Account, passphrase string, hash []byte) ([]byte, error)

	// SignTxWithPassphrase requests the wallet to sign the given transaction, with the
	// given passphrase as extra authentication information.
	// It looks up the account specified either solely via its address contained within,
	// or optionally with the aid of any location metadata from the embedded URL field.
	SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
}
```

Backend

```go
// Backend is a "wallet provider" that may contain a batch of accounts they can
// sign transactions with and upon request, do so.
type Backend interface {
	// Wallets retrieves the list of wallets the backend is currently aware of.
	// The returned wallets are not opened by default. For software HD wallets this
	// means that no base seeds are decrypted, and for hardware wallets that no actual
	// connection is established.
	// The resulting wallet list will be sorted alphabetically based on its internal
	// URL assigned by the backend. Since wallets (especially hardware) may come and
	// go, the same wallet might appear at a different positions in the list during
	// subsequent retrievals.
	Wallets() []Wallet

	// Subscribe creates an async subscription to receive notifications when the
	// backend detects the arrival or departure of a wallet.
	Subscribe(sink chan<- WalletEvent) event.Subscription
}
```

## manager.go

Manager is an account management tool that contains everything. You can communicate with all Backends to sign a transaction.

data structure

```go
// Manager is an overarching account manager that can communicate with various
// backends for signing transactions.
type Manager struct {
	backends map[reflect.Type][]Backend // Index of backends currently registered
	updaters []event.Subscription       // Wallet update subscriptions for all backends
	updates  chan WalletEvent           // Subscription sink for backend wallet changes
	wallets  []Wallet                   // Cache of all wallets from all registered backends
	feed event.Feed // Wallet feed notifying of arrivals/departures
	quit chan chan error
	lock sync.RWMutex
}
```

Create Manager

```go
// NewManager creates a generic account manager to sign transaction via various
// supported backends.
func NewManager(backends ...Backend) *Manager {
	// Subscribe to wallet notifications from all backends
	updates := make(chan WalletEvent, 4*len(backends))

	subs := make([]event.Subscription, len(backends))
	for i, backend := range backends {
		subs[i] = backend.Subscribe(updates)
	}
	// Retrieve the initial list of wallets from the backends and sort by URL
	var wallets []Wallet
	for _, backend := range backends {
		wallets = merge(wallets, backend.Wallets()...)
	}
	// Assemble the account manager and return
	am := &Manager{
		backends: make(map[reflect.Type][]Backend),
		updaters: subs,
		updates:  updates,
		wallets:  wallets,
		quit:     make(chan chan error),
	}
	for _, backend := range backends {
		kind := reflect.TypeOf(backend)
		am.backends[kind] = append(am.backends[kind], backend)
	}
	go am.update()

	return am
}
```

Update method. Is a goroutine. Will monitor all update information triggered by backend. Then forward it to the feed.

```go
// update is the wallet event loop listening for notifications from the backends
// and updating the cache of wallets.
func (am *Manager) update() {
	// Close all subscriptions when the manager terminates
	defer func() {
		am.lock.Lock()
		for _, sub := range am.updaters {
			sub.Unsubscribe()
		}
		am.updaters = nil
		am.lock.Unlock()
	}()

	// Loop until termination
	for {
		select {
		case event := <-am.updates:
			// Wallet event arrived, update local cache
			am.lock.Lock()
			switch event.Kind {
			case WalletArrived:
				am.wallets = merge(am.wallets, event.Wallet)
			case WalletDropped:
				am.wallets = drop(am.wallets, event.Wallet)
			}
			am.lock.Unlock()

			// Notify any listeners of the event
			am.feed.Send(event)

		case errc := <-am.quit:
			// Manager terminating, return
			errc <- nil
			return
		}
	}
}
```

return backend

```go
// Backends retrieves the backend(s) with the given type from the account manager.
func (am *Manager) Backends(kind reflect.Type) []Backend {
	return am.backends[kind]
}
```

Subscription message

```go
// Subscribe creates an async subscription to receive notifications when the
// manager detects the arrival or departure of a wallet from any of its backends.
func (am *Manager) Subscribe(sink chan<- WalletEvent) event.Subscription {
	return am.feed.Subscribe(sink)
}
```

For node. When was the account manager created?

```go
// New creates a new P2P node, ready for protocol registration.
func New(conf *Config) (*Node, error) {
	...
	am, ephemeralKeystore, err := makeAccountManager(conf)


func makeAccountManager(conf *Config) (*accounts.Manager, string, error) {
	scryptN := keystore.StandardScryptN
	scryptP := keystore.StandardScryptP
	if conf.UseLightweightKDF {
		scryptN = keystore.LightScryptN
		scryptP = keystore.LightScryptP
	}

	var (
		keydir    string
		ephemeral string
		err       error
	)
	switch {
	case filepath.IsAbs(conf.KeyStoreDir):
		keydir = conf.KeyStoreDir
	case conf.DataDir != "":
		if conf.KeyStoreDir == "" {
			keydir = filepath.Join(conf.DataDir, datadirDefaultKeyStore)
		} else {
			keydir, err = filepath.Abs(conf.KeyStoreDir)
		}
	case conf.KeyStoreDir != "":
		keydir, err = filepath.Abs(conf.KeyStoreDir)
	default:
		// There is no datadir.
		keydir, err = ioutil.TempDir("", "go-ethereum-keystore")
		ephemeral = keydir
	}
	if err != nil {
		return nil, "", err
	}
	if err := os.MkdirAll(keydir, 0700); err != nil {
		return nil, "", err
	}
	// Assemble the account manager and supported backends
	backends := []accounts.Backend{
		keystore.NewKeyStore(keydir, scryptN, scryptP),
	}
	// If it is a USB wallet. Need to do some extra work
	if !conf.NoUSB {
		// Start a USB hub for Ledger hardware wallets
		if ledgerhub, err := usbwallet.NewLedgerHub(); err != nil {
			log.Warn(fmt.Sprintf("Failed to start Ledger hub, disabling: %v", err))
		} else {
			backends = append(backends, ledgerhub)
		}
		// Start a USB hub for Trezor hardware wallets
		// will communicate via trezor driver which has USB device connection to communicate through
		// the Trezor hardware wallet. Initializing the Trezor is a two phase operation:
		//  * The first phase is to initialize the connection and read the wallet's
		//    features. This phase is invoked is the provided passphrase is empty. The
		//    device will display the pinpad as a result and will return an appropriate
		//    error to notify the user that a second open phase is needed.
		//  * The second phase is to unlock access to the Trezor, which is done by the
		//    user actually providing a passphrase mapping a keyboard keypad to the pin
		//    number of the user (shuffled according to the pinpad displayed).
		if trezorhub, err := usbwallet.NewTrezorHub(); err != nil {
			log.Warn(fmt.Sprintf("Failed to start Trezor hub, disabling: %v", err))
		} else {
			backends = append(backends, trezorhub)
		}
	}
	return accounts.NewManager(backends...), ephemeral, nil
}
```
