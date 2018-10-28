Vm uses the Stack of objects in stack.go as the stack of the virtual machine. Memory represents the memory object used in the virtual machine.

## stack

The simpler is to use 1024 big.Int fixed-length arrays as the storage for the stack.

structure

```go
// stack is an object for basic stack operations. Items popped to the stack are
// expected to be changed and modified. stack does not take care of adding newly
// initialised objects.
type Stack struct {
	data []*big.Int
}

func newstack() *Stack {
	return &Stack{data: make([]*big.Int, 0, 1024)}
}
```

push operation

```go
func (st *Stack) push(d *big.Int) { // Append to the end
	// NOTE push limit (1024) is checked in baseCheck
	//stackItem := new(big.Int).Set(d)
	//st.data = append(st.data, stackItem)
	st.data = append(st.data, d)
}
func (st *Stack) pushN(ds ...*big.Int) {
	st.data = append(st.data, ds...)
}
```

pop operation

```go
func (st *Stack) pop() (ret *big.Int) { // Take it out from the end.
	ret = st.data[len(st.data)-1]
	st.data = st.data[:len(st.data)-1]
	return
}
```

The value operation of the exchange element, and the operation?

```go
func (st *Stack) swap(n int) { // Swaps the value of the element at the top of the stack and the element at a distance n from the top of the stack.
	st.data[st.len()-n], st.data[st.len()-1] = st.data[st.len()-1], st.data[st.len()-n]
}
```

Dup operation like copying the value of the specified location to the top of the heap

```go
func (st *Stack) dup(pool *intPool, n int) {
	st.push(pool.get().Set(st.data[st.len()-n]))
}
```

Peek operation. Peeking at the top of the stack

```go
func (st *Stack) peek() *big.Int {
	return st.data[st.len()-1]
}
```

Back peek at the elements of the specified location

```go
// Back returns the n'th item in stack
func (st *Stack) Back(n int) *big.Int {
	return st.data[st.len()-n-1]
}
```

Require guarantees that the number of stack elements is greater than or equal to n.

```go
func (st *Stack) require(n int) error {
	if st.len() < n {
		return fmt.Errorf("stack underflow (%d <=> %d)", len(st.data), n)
	}
	return nil
}
```

## intpool

Very simple. It is a 256-sized pool of big.int used to speed up the allocation of bit.Int.

```go
var checkVal = big.NewInt(-42)

const poolLimit = 256

// intPool is a pool of big integers that
// can be reused for all big.Int operations.
type intPool struct {
	pool *Stack
}

func newIntPool() *intPool {
	return &intPool{pool: newstack()}
}

func (p *intPool) get() *big.Int {
	if p.pool.len() > 0 {
		return p.pool.pop()
	}
	return new(big.Int)
}
func (p *intPool) put(is ...*big.Int) {
	if len(p.pool.data) > poolLimit {
		return
	}

	for _, i := range is {
		// verifyPool is a build flag. Pool verification makes sure the integrity
		// of the integer pool by comparing values to a default value.
		if verifyPool {
			i.Set(checkVal)
		}

		p.pool.push(i)
	}
}
```

## memory

Construction, memory storage is byte[]. There is also a record of lastGasCost.

```go
type Memory struct {
	store       []byte
	lastGasCost uint64
}

func NewMemory() *Memory {
	return &Memory{}
}
```

use Resize to allocate space

```go
// Resize resizes the memory to size
func (m *Memory) Resize(size uint64) {
	if uint64(m.Len()) < size {
		m.store = append(m.store, make([]byte, size-uint64(m.Len()))...)
	}
}
```

Then use Set to set the value

```go
// Set sets offset + size to value
func (m *Memory) Set(offset, size uint64, value []byte) {
	// length of store may never be less than offset + size.
	// The store should be resized PRIOR to setting the memory
	if size > uint64(len(m.store)) {
		panic("INVALID memory: store empty")
	}

	// It's possible the offset is greater than 0 and size equals 0. This is because
	// the calcMemSize (common.go) could potentially return 0 when size is zero (NO-OP)
	if size > 0 {
		copy(m.store[offset:offset+size], value)
	}
}
```

Get to get the value, one is to get the copy, one is to get the pointer.

```go
// Get returns offset + size as a new slice
func (self *Memory) Get(offset, size int64) (cpy []byte) {
	if size == 0 {
		return nil
	}

	if len(self.store) > int(offset) {
		cpy = make([]byte, size)
		copy(cpy, self.store[offset:offset+size])

		return
	}

	return
}

// GetPtr returns the offset + size
func (self *Memory) GetPtr(offset, size int64) []byte {
	if size == 0 {
		return nil
	}

	if len(self.store) > int(offset) {
		return self.store[offset : offset+size]
	}

	return nil
}
```

## Some extra helper functions in stack_table.go

```go
func makeStackFunc(pop, push int) stackValidationFunc {
	return func(stack *Stack) error {
		if err := stack.require(pop); err != nil {
			return err
		}

		if stack.len()+push-pop > int(params.StackLimit) {
			return fmt.Errorf("stack limit reached %d (%d)", stack.len(), params.StackLimit)
		}
		return nil
	}
}

func makeDupStackFunc(n int) stackValidationFunc {
	return makeStackFunc(n, n+1)
}

func makeSwapStackFunc(n int) stackValidationFunc {
	return makeStackFunc(n, n)
}
```
