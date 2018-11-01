It is recommended to first understand the Appendix B. Recursive Length Prefix in the Yellow Book.

More detail can refer to: [RLP in detail](rlp-more.md)

From an engineering point of view, rlp is divided into two types of data definitions, and one of them can recursively contain another type:

// T consists of L or B
$$T ≡ L∪B$$
// Any member of L belongs to T (T is also composed of L or B: note recursive definition)
$$L ≡ {t:t=(t[0],t[1],...) ∧ \forall n<‖t‖ t[n]∈T}$$  
// Any member of T belongs to O
$$B ≡ {b:b=(b[0],b[1],...) ∧ \forall n<‖b‖ b[n]∈O}$$

- [\forall](https://en.wikibooks.org/wiki/LaTeX/Mathematics#Symbols) reference LaTeX syntax standard, can use Katex in visualcode to view
- O is defined as a bytes collection
- If you think of T as a tree-like data structure, then B is the leaf, which contains only the byte sequence structure; and L is the trunk, containing multiple Bs or itself.

That is, we rely on the basis of B to form a recursive definition of T and L: the whole RLP consists of T, T contains L and B, and the members of L are both T. Such recursive definitions can describe very flexible data structures

In the specific coding, only the coding space of the first byte can be used to distinguish these structural differences:

```
B coding rules: leaves
RLP_B0 [0000 0001, 0111 1111] If it is a byte other than 0 and less than 128[1000 0000], no header is needed, and the content is encoded.
RLP_B1 [1000 0000, 1011 0111] If the length of the byte content is less than 56, that is, 55[0011 0111], the length is compressed into a byte in big endian, and 128 is added to form the header, and then the actual content is connected.
RLP_B2 (1011 0111, 1100 0000) For longer content, describe the length of the content length in a space where the second bit is not 1. Its space is (192-1)[1011 1111]-(183+1)[1011 1000]=7[0111] ，That is, the length needs to be less than 2^(7*8) is a huge number that cannot be used up.
L coding rules: branches
RLP_L1 [1100 0000, 1111 0111) If it is a combination of multiple above encoded content, it is expressed by 1 in the second bit. Subsequent content length is less than 56, that is,55[0011 0111] 则先将长度压缩后放到第一个 byte Medium (plus 192 [1100 0000]), then connect to the actual content
RLP_L2 [1111 0111, 1111 1111] For longer content, the length of the content length is described in the remaining space. Its space is 255[1111 1111]-247[1111 0111]=8[1000]，That is, the length needs to be less than 2^(8*8). It is also a huge number that cannot be used up.
```

Please copy the following code into the editor with text folding for code review (Recommended Notepad++ to properly collapse all functions under the github reference format for easy global understanding)

```go
// [/rlp/encode_test.go#TestEncode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode_test.go#L272)
func TestEncode(t *testing.T) {
	runEncTests(t, func(val interface{}) ([]byte, error) {
		b := new(bytes.Buffer)
		err := Encode(b, val)
		// [/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80)
		func Encode(w io.Writer, val interface{}) error {
			if outer, ok := w.(*encbuf); ok {
				// Encode was called by some type's EncodeRLP.
				// Avoid copying by writing to the outer encbuf directly.
				return outer.encode(val)
			}
			eb := encbufPool.Get().(*encbuf)
			// [/rlp/encode.go#encbuf](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L121)
			type encbuf struct { // Stateful encoder
				str     []byte      // string data, contains everything except list headers
				lheads  []*listhead // all list headers

				// [/rlp/encode.go#listhead](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L128)
				type listhead struct {
					offset int // index of this header in string data
					size   int // total size of encoded data (including list headers)
				}

				lhsize  int         // sum of sizes of all encoded list headers
				sizebuf []byte      // 9-byte auxiliary buffer for uint encoding
			}

			defer encbufPool.Put(eb)
			eb.reset()
			if err := eb.encode(val); err != nil { // encbuf.encode As a B-encoded (internal) function, eb is a stateful encbuf
				// [/rlp/encode.go#encbuf.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L181)
				func (w *encbuf) encode(val interface{}) error {
				rval := reflect.ValueOf(val)
				ti, err := cachedTypeInfo(rval.Type(), tags{}) // Get the current type of content encoding function from the cache
				if err != nil {
					return err
				}
				return ti.writer(rval, w) // Execution function
				// [/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L392)
				func writeUint(val reflect.Value, w *encbuf) error {
					i := val.Uint()
					if i == 0 {
						w.str = append(w.str, 0x80) // Prevent all 0 exception patterns in the sense of encoding, encoding 0 as 0x80
					} else if i < 128 { // Implement RLP_B0 encoding logic
						// fits single byte
						w.str = append(w.str, byte(i))
					} else { // Implement RLP_B1 encoding logic: because uint has a byte length of only 8 and will not exceed 56
						// TODO: encode int to w.str directly
						s := putint(w.sizebuf[1:], i) // The bit with the uint high is zero, removed by the byte granularity, and returns the number of bytes after the removal.
						w.sizebuf[0] = 0x80 + byte(s) // For uint, the maximum length is 64 bit/8 byte, so compressing the length to byte[0,256) is completely sufficient.
						w.str = append(w.str, w.sizebuf[:s+1]...) // Output all available bytes in sizebuf as encoded content
					}
					return nil
				}
				// [/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L461)
				func writeString(val reflect.Value, w *encbuf) error {
					s := val.String()
					if len(s) == 1 && s[0] <= 0x7f { // 0x7f=127 implements the RLP_B0 encoding logic, paying attention to the empty string will go below else
						// fits single byte, no string header
						w.str = append(w.str, s[0])
					} else {
						w.encodeStringHeader(len(s)) // Implement RLP_B1, RLP_B2 encoding logic
						// [/rlp/encode.go#encbuf.encodeStringHeader](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L191)
						func (w *encbuf) encodeStringHeader(size int) {
							if size < 56 { // Implement RLP_B1 encoding logic
								w.str = append(w.str, 0x80+byte(size))
							} else { // Implement RLP_B2 encoding logic
								// TODO: encode to w.str directly
								sizesize := putint(w.sizebuf[1:], uint64(size)) // Remove size in big endian, remove byte high [0000 0000] and range byte by byte granularity
								w.sizebuf[0] = 0xB7 + byte(sizesize) // 0xB7[1011 0111]183 Encode the length of the header length into the first byte
								w.str = append(w.str, w.sizebuf[:sizesize+1]...) // Complete the length information encoding of the header
							}
						}
						w.str = append(w.str, s...) // Append the actual content to the head
					}
					return nil
				}
				// [/rlp/encode.go#makeStructWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L529)
				func makeStructWriter(typ reflect.Type) (writer, error) {
					fields, err := structFields(typ) // Get field information by reflection to encode one by one
					if err != nil {
						return nil, err
					}
					writer := func(val reflect.Value, w *encbuf) error {
						lh := w.list() // Create a listhead storage object
						[/rlp/encode.go#encbuf.list]// (https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L212)
						func (w *encbuf) list() *listhead {
							lh := &listhead{offset: len(w.str), size: w.lhsize} // Create a new listhead object with the current total length of the encoding as offset , the current total length of the header lhsize as size
							w.lheads = append(w.lheads, lh)
							return lh // Add the header sequence and return it to listEnd
						}
						for _, f := range fields {
							if err := f.info.writer(val.Field(f.index), w); err != nil {
								return err
							}
						}
						w.listEnd(lh) // Set the value of the listhead object
						// [/rlp/encode.go#encbuf.listEnd](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L218)
						func (w *encbuf) listEnd(lh *listhead) {
							lh.size = w.size() - lh.offset - lh.size // Is the new header size equal to the newly added code length minus?
							if lh.size < 56 {
								w.lhsize += 1 // length encoded into kind tag
							} else {
								w.lhsize += 1 + intsize(uint64(lh.size))
							}
						}
						return nil
					}
					return writer, nil
				}
			}
				return err
			}
			return eb.toWriter(w) // encbuf.toWriter As L header encoding (internal) function
			// [/rlp/encode.go#encbuf.toWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L249)
			func (w *encbuf) toWriter(out io.Writer) (err error) {
				strpos := 0
				for _, head := range w.lheads { // For the case of no lheads data, ignore the following header encoding logic
					// write string data before header
					if head.offset-strpos > 0 {
						n, err := out.Write(w.str[strpos:head.offset])
						strpos += n
						if err != nil {
							return err
						}
					}
					// write the header
					enc := head.encode(w.sizebuf)
					// [/rlp/encode.go#encbuf.listhead.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L135)
					func (head *listhead) encode(buf []byte) []byte {
						// Convert binary
						// 0xC0 192: 1100 0000
						// 0xF7 247: 1111 0111
						return buf[:puthead(buf, 0xC0, 0xF7, uint64(head.size))]
						// [/rlp/encode.go#puthead](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L150)
						func puthead(buf []byte, smalltag, largetag byte, size uint64) int {
							if size < 56 {
								buf[0] = smalltag + byte(size)
								return 1
							} else {
								sizesize := putint(buf[1:], size)
								buf[0] = largetag + byte(sizesize)
								return sizesize + 1
							}
						}
					}
					if _, err = out.Write(enc); err != nil {
						return err
					}
				}
				if strpos < len(w.str) { // Strpos is 0, which is inevitable. The content with header encoding is directly used as the final output.
					// write string data after the last list header
					_, err = out.Write(w.str[strpos:])
				}
				return err
			}
		}
		return b.Bytes(), err
	})
}
```

# Test document format appendix

After understanding the general meaning of RLP, it is recommended to start reading from the test code for
the code segment in [/rlp/encode_test.go](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode_test.go#L272)

```go
func TestEncode(t *testing.T) {
	runEncTests(t, func(val interface{}) ([]byte, error) {
		b := new(bytes.Buffer)
		err := Encode(b, val)
		return b.Bytes(), err
	})
}
```

err: = call Encode (b, val) Encode function as an encoded entry, embodied in
[/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80)

```go
func Encode(w io.Writer, val interface{}) error {
	if outer, ok := w.(*encbuf); ok {
		// Encode was called by some type's EncodeRLP.
		// Avoid copying by writing to the outer encbuf directly.
		return outer.encode(val)
	}
	eb := encbufPool.Get().(*encbuf)
	defer encbufPool.Put(eb)
	eb.reset()
	if err := eb.encode(val); err != nil { // encbuf.toWriter as header encoding (internal) function
		return err
	}
	return eb.toWriter(w) // encbuf.toWriter as header encoding (internal) function
}
```

[/rlp/encode.go#encbuf.encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L181)

```go
func (w *encbuf) encode(val interface{}) error {
	rval := reflect.ValueOf(val)
	ti, err := cachedTypeInfo(rval.Type(), tags{}) // Get the current type of content encoding function from the cache
	if err != nil {
		return err
	}
	return ti.writer(rval, w) // Execution function
}
```

We ignore the cache type and the acquisition function of the encoding function and generate the function cachedTypeInfo to move the focus directly to the content encoding function of the specific type.

Ordinary uint Coding in [/rlp/encode.go#writeUint](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L392)

```go
func writeUint(val reflect.Value, w *encbuf) error {
	i := val.Uint()
	if i == 0 {
		w.str = append(w.str, 0x80) // Prevent all 0 exception patterns in the sense of encoding, encoding 0 as 0x80
	} else if i < 128 {
		// fits single byte
		w.str = append(w.str, byte(i))
	} else {
		// TODO: encode int to w.str directly
		s := putint(w.sizebuf[1:], i) // The bit with the uint high is zero, removed by the byte granularity, and returns the number of bytes after the removal.
		w.sizebuf[0] = 0x80 + byte(s) // For uint, the maximum length is 64 bit/8 byte, so compressing the length to byte[0,256) is completely sufficient.
		w.str = append(w.str, w.sizebuf[:s+1]...) // Output all available bytes in sizebuf as encoded content
	}
	return nil
}
```

Then [/rlp/encode.go#Encode](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L80) a subsequent logic [/rlp/encode.go#encbuf.toWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L249)

```go
func (w *encbuf) toWriter(out io.Writer) (err error) {
	strpos := 0
	for _, head := range w.lheads { // For the case of no lheads data, ignore the following header encoding logic
		// write string data before header
		if head.offset-strpos > 0 {
			n, err := out.Write(w.str[strpos:head.offset])
			strpos += n
			if err != nil {
				return err
			}
		}
		// write the header
		enc := head.encode(w.sizebuf)
		if _, err = out.Write(enc); err != nil {
			return err
		}
	}
	if strpos < len(w.str) { // Strpos is 0, which is inevitable. The content with header encoding is directly used as the final output.
		// write string data after the last list header
		_, err = out.Write(w.str[strpos:])
	}
	return err
}
```

For string encoding [/rlp/encode.go#writeString](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L461) is similar to uint, does not contain the encoding information of the header. Can refer to the yellow book

```go
func writeString(val reflect.Value, w *encbuf) error {
	s := val.String()
	if len(s) == 1 && s[0] <= 0x7f {
		// fits single byte, no string header
		w.str = append(w.str, s[0])
	} else {
		w.encodeStringHeader(len(s))
		w.str = append(w.str, s...)
	}
	return nil
}
```

function contains more complicated [/rlp/encode.go#makeStructWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L529)

```go
func makeStructWriter(typ reflect.Type) (writer, error) {
	fields, err := structFields(typ)
	if err != nil {
		return nil, err
	}
	writer := func(val reflect.Value, w *encbuf) error {
		lh := w.list() // Create a listhead storage object
		for _, f := range fields {
			if err := f.info.writer(val.Field(f.index), w); err != nil {
				return err
			}
		}
		w.listEnd(lh) // Set the value of the listhead object
		return nil
	}
	return writer, nil
}
```

[/rlp/encode.go#encbuf.list/listEnd](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L212)

```go
func (w *encbuf) list() *listhead {
	lh := &listhead{offset: len(w.str), size: w.lhsize} // Create a new listhead object with the current total length of the encoding as offset , the current total length of the header lhsize as size
	w.lheads = append(w.lheads, lh)
	return lh // Add the header sequence and return it to listEnd
}

func (w *encbuf) listEnd(lh *listhead) {
	lh.size = w.size() - lh.offset - lh.size // Is the new header size equal to the newly added code length minus?
	if lh.size < 56 {
		w.lhsize += 1 // length encoded into kind tag
	} else {
		w.lhsize += 1 + intsize(uint64(lh.size))
	}
}
```

[encbuf.toWriter](https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L248) function is implemented as

```go
func (w *encbuf) toWriter(out io.Writer) (err error) {
	strpos := 0
	for _, head := range w.lheads {
		// write string data before header
		if head.offset-strpos > 0 {
			n, err := out.Write(w.str[strpos:head.offset])
			strpos += n
			if err != nil {
				return err
			}
		}
		// write the header
		enc := head.encode(w.sizebuf)
		if _, err = out.Write(enc); err != nil {
			return err
		}
	}
	if strpos < len(w.str) {
		// write string data after the last list header
		_, err = out.Write(w.str[strpos:])
	}
	return err
}
```

https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L135

```go
func (head *listhead) encode(buf []byte) []byte {
	// Convert binary
	// 0xC0 192: 1100 0000
	// 0xF7 247: 1111 0111
	return buf[:puthead(buf, 0xC0, 0xF7, uint64(head.size))]
}
```

https://github.com/ethereum/go-ethereum/blob/master/rlp/encode.go#L150

```go
func puthead(buf []byte, smalltag, largetag byte, size uint64) int {
	if size < 56 {
		buf[0] = smalltag + byte(size)
		return 1
	} else {
		sizesize := putint(buf[1:], size)
		buf[0] = largetag + byte(sizesize)
		return sizesize + 1
	}
}
```
