RLP is short for Recursive Length Prefix. In the serialization method in Ethereum, all objects in Ethereum are serialized into byte arrays using RLP methods. Here I hope to formalize the RLP method from the Yellow Book and then analyze the actual implementation through code.

## Formal definition of the Yellow Book

We define the set T. T is defined by the following formula

![image](picture/rlp_1.png)

The O in the above figure represents a collection of all bytes, then B represents all possible byte arrays, and L represents a tree structure of more than one single node (such as a structure, or a branch node of a tree node, a non-leaf node) , T represents a combination of all byte arrays and tree structures.

We use two sub-functions to define the RLP functions, which handle the two structures (L or B) mentioned above.

![image](picture/rlp_2.png)

For all B type byte arrays. We have defined the following processing rules.

- If the byte array contains only one byte, and the size of this byte is less than 128, then the data is not processed, and the result is the original data.
- If the length of the byte array is less than 56, then the result of the processing is equal to the prefix of the original data (128 + bytes of data length).
- If it is not the above two cases, then the result of the processing is equal to the big endian representation of the original data length in front of the original data, and then preceded by (183 + the length of the big end of the original data)

The following uses a formulated language to represent

![image](picture/rlp_3.png)

**Explanation of some mathematical symbols**

- ||x|| represents the length of x
- (a).(b,c).(d,e) = (a,b,c,d,e) Represents the operation of concat, which is the addition of strings. "hello "+"world" = "hello world"
- The BE(x) function actually removes the big endian mode of the leading zero. For example, the 4-byte integer 0x1234 is represented by big endian mode as 00 00 12 34.  
  Then the result returned by the BE function is actually 12 34. The extra 00 at the beginning is removed.
- The ^ symbol represents the and operator
- The equal sign of the "three" form represents the meaning of identity

For all other types (tree structures), we define the following processing rules

First we use RLP processing for each element in the tree structure, and then concatenate the results.

- If the length of the connected byte is less than 56, then we add (192 + the length of the connection) in front of the result of the connection to form the final result.
- If the length of the connected byte is greater than or equal to 56, then we add the big endian mode of the connected length before the result of the connection, and then add (247 + the length of the big endian mode of the length after the connection)

The following is expressed in a formulaic language, and it is found that the formula is clearer.

![image](picture/rlp_4.png)  
It can be seen that the above is a recursive definition, and the RLP method is called in the process of obtaining s(x), so that RLP can process the recursive data structure.

If RLP is used to process scalar data, RLP can only be used to process positive integers. RLP can only handle integers processed in big endian mode. That is to say, if it is an integer x, then use the BE(x) function to convert x to the simplest big endian mode (with the 00 at the beginning removed), and then encode the result of BE(x) as a byte array.

If you use the formula to represent it is the following figure.

![image](picture/rlp_5.png)

When parsing RLP data. If you just need to parse the shaping data, this time you encounter the leading 00, this time you need to be treated as an exception.

**Sum up**

RLP treats all data as a combination of two types of data, one is a byte array, and the other is a data structure similar to List. I understand that these two classes basically contain all the data structures. For example, use more structs. Can be seen as a list of many different types of fields

### RLP source code analysis

The source code of RLP is not a lot, mainly divided into three files.

    decode.go			Decoder, decoding RLP data into a data structure of go
    decode_tail_test.go		Decoder test code
    decode_test.go			Decoder test code
    doc.go				Document code
    encode.go			Encoder that serializes the data structure of GO to a byte array
    encode_test.go			Encoder test
    encode_example_test.go
    raw.go				Undecoded RLP data
    raw_test.go
    typecache.go			Type cache, type cache records the contents of type -> (encoder | decoder).

#### How to find the corresponding encoder and decoder typecache.go according to the type

In languages ​​such as C++ or Java that support overloading, you can override the same function name by different types to implement methods for different types of dispatch. For example, you can also use generics to implement function dispatch.

```c
string encode(int);
string encode(long);
string encode(struct test*)
```

However, the GO language itself does not support overloading and there is no generics, so the assignment of functions needs to be implemented by itself. Typecache.go is mainly for this purpose, to quickly find its own encoder function and decoder function by its own type.

Let's first look at the core data structure.

```go
var (
	// read-write lock, used to protect the typeCache map when multi-threaded
	typeCacheMutex sync.RWMutex
	typeCache      = make(map[typekey]*typeinfo) // Core data structure, save type -> codec function
)
type typeinfo struct { // Stores encoder and decoder functions
	decoder
	writer
}
type typekey struct {
	reflect.Type
	// the key must include the struct tags because they
	// might generate a different decoder.
	tags
}
```

You can see that the core data structure is the typeCache map, the key of the Map is the type, and the value is the corresponding code and decoder.

Here's how the user gets the encoder and decoder functions.

```go
func cachedTypeInfo(typ reflect.Type, tags tags) (*typeinfo, error) {
	typeCacheMutex.RLock()		// Add a lock to protect,
	info := typeCache[typekey{typ, tags}]
	typeCacheMutex.RUnlock()
	if info != nil { // If the information is successfully obtained, then it will be returned
		return info, nil
	}
	// not in the cache, need to generate info for this type.
	typeCacheMutex.Lock()  // Otherwise, add the write lock to call the cachedTypeInfo1 function to create and return. It should be noted that in a multi-threaded environment, it is possible for multiple threads to call to this place at the same time, so when you enter the cachedTypeInfo1 method, you need to determine whether it has been preceded by another thread. The creation was successful.
	defer typeCacheMutex.Unlock()
	return cachedTypeInfo1(typ, tags)
}

func cachedTypeInfo1(typ reflect.Type, tags tags) (*typeinfo, error) {
	key := typekey{typ, tags}
	info := typeCache[key]
	if info != nil {
		// Other threads may have been created successfully, then we get the information directly and then return
		return info, nil
	}
	// put a dummmy value into the cache before generating.
	// if the generator tries to lookup itself, it will get
	// the dummy value and won't call itself recursively.
	typeCache[key] = new(typeinfo)
	info, err := genTypeInfo(typ, tags)
	if err != nil {
		// remove the dummy value if the generator fails
		delete(typeCache, key)
		return nil, err
	}
	*typeCache[key] = *info
	return typeCache[key], err
}
```

genTypeInfo is a codec function that generates the corresponding type.

```go
func genTypeInfo(typ reflect.Type, tags tags) (info *typeinfo, err error) {
	info = new(typeinfo)
	if info.decoder, err = makeDecoder(typ, tags); err != nil {
		return nil, err
	}
	if info.writer, err = makeWriter(typ, tags); err != nil {
		return nil, err
	}
	return info, nil
}
```

The processing logic of makeDecoder is roughly the same as the processing logic of makeWriter. Here I only post the processing logic of makeWriter.

```go
// makeWriter creates a writer function for the given type.
func makeWriter(typ reflect.Type, ts tags) (writer, error) {
	kind := typ.Kind()
	switch {
	case typ == rawValueType:
		return writeRawValue, nil
	case typ.Implements(encoderInterface):
		return writeEncoder, nil
	case kind != reflect.Ptr && reflect.PtrTo(typ).Implements(encoderInterface):
		return writeEncoderNoPtr, nil
	case kind == reflect.Interface:
		return writeInterface, nil
	case typ.AssignableTo(reflect.PtrTo(bigInt)):
		return writeBigIntPtr, nil
	case typ.AssignableTo(bigInt):
		return writeBigIntNoPtr, nil
	case isUint(kind):
		return writeUint, nil
	case kind == reflect.Bool:
		return writeBool, nil
	case kind == reflect.String:
		return writeString, nil
	case kind == reflect.Slice && isByte(typ.Elem()):
		return writeBytes, nil
	case kind == reflect.Array && isByte(typ.Elem()):
		return writeByteArray, nil
	case kind == reflect.Slice || kind == reflect.Array:
		return makeSliceWriter(typ, ts)
	case kind == reflect.Struct:
		return makeStructWriter(typ)
	case kind == reflect.Ptr:
		return makePtrWriter(typ)
	default:
		return nil, fmt.Errorf("rlp: type %v is not RLP-serializable", typ)
	}
}
```

You can see that it is a switch case, assigning different handlers depending on the type. This processing logic is still very simple. It is very simple for the simple type, and it can be processed according to the description above in the yellow book. However, the handling of the structure type is quite interesting, and this part of the detailed processing logic can not be found in the Yellow Book.

```go
type field struct {
	index int
	info  *typeinfo
}
func makeStructWriter(typ reflect.Type) (writer, error) {
	fields, err := structFields(typ)
	if err != nil {
		return nil, err
	}
	writer := func(val reflect.Value, w *encbuf) error {
		lh := w.list()
		for _, f := range fields {
			// f is the field structure, f.info is a pointer to typeinfo, so here is actually the encoder method that calls the field.
			if err := f.info.writer(val.Field(f.index), w); err != nil {
				return err
			}
		}
		w.listEnd(lh)
		return nil
	}
	return writer, nil
}
```

This function defines the encoding of the structure. The structFields method gets the encoder for all the fields, and then returns a method that iterates over all the fields, each of which calls its encoder method.

```go
func structFields(typ reflect.Type) (fields []field, err error) {
	for i := 0; i < typ.NumField(); i++ {
		if f := typ.Field(i); f.PkgPath == "" { // exported
			tags, err := parseStructTag(typ, i)
			if err != nil {
				return nil, err
			}
			if tags.ignored {
				continue
			}
			info, err := cachedTypeInfo1(f.Type, tags)
			if err != nil {
				return nil, err
			}
			fields = append(fields, field{i, info})
		}
	}
	return fields, nil
}
```

The structFields function iterates over all the fields and then calls cachedTypeInfo1 for each field. You can see that this is a recursive calling process. One of the above code to note is that f.PkgPath == "" This is for all exported fields. The so-called exported field is the field that starts with an uppercase letter.

#### encode.go

First define the values ​​of the empty string and the empty List, which are 0x80 and 0xC0. Note that the corresponding value of the shaped 0 value is also 0x80. This is not defined above the yellow book. Then define an interface type to implement EncodeRLP for other types.

```go
var (
	// Common encoded values.
	// These are useful when implementing EncodeRLP.
	EmptyString = []byte{0x80}
	EmptyList   = []byte{0xC0}
)

// Encoder is implemented by types that require custom
// encoding rules or want to encode private fields.
type Encoder interface {
	// EncodeRLP should write the RLP encoding of its receiver to w.
	// If the implementation is a pointer method, it may also be
	// called for nil pointers.
	//
	// Implementations should generate valid RLP. The data written is
	// not verified at the moment, but a future version might. It is
	// recommended to write only a single value but writing multiple
	// values or no value at all is also permitted.
	EncodeRLP(io.Writer) error
}
```

Then define one of the most important methods, most of the EncodeRLP methods directly call this method Encode method. This method first gets an encbuf object. Then call the object's encode method. In the encode method, the object's reflection type is first obtained, its encoder is obtained according to the reflection type, and then the encoder's writer method is called. This is related to the typecache mentioned above.

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
	if err := eb.encode(val); err != nil {
		return err
	}
	return eb.toWriter(w)
}
func (w *encbuf) encode(val interface{}) error {
	rval := reflect.ValueOf(val)
	ti, err := cachedTypeInfo(rval.Type(), tags{})
	if err != nil {
		return err
	}
	return ti.writer(rval, w)
}
```

##### encbuf Introduction

Encbuf is short for encode buffer (I guess). Encbuf appears in the Encode method, and in many Writer methods. As the name implies, this acts as a buffer during the encoding process. Let's take a look at the definition of encbuf.

```go
type encbuf struct {
	str     []byte      // string data, contains everything except list headers
	lheads  []*listhead // all list headers
	lhsize  int         // sum of sizes of all encoded list headers
	sizebuf []byte      // 9-byte auxiliary buffer for uint encoding
}

type listhead struct {
	offset int // index of this header in string data
	size   int // total size of encoded data (including list headers)
}
```

As you can see from the comments, the str field contains all the content except the head of the list. The head of the list is recorded in the lheads field. The lhsize field records the length of lheads, and the sizebuf is a 9-byte auxiliary buffer designed to handle the encoding of uint. The listhead consists of two fields, the offset field records where the list data is in the str field, and the size field records the total length of the encoded data containing the list header. You can see the picture below

![image](picture/rlp_6.png)

For ordinary types, such as string, integer, bool type and other data, it is directly filled into the str field. However, for the processing of structure types, special processing methods are required. Take a look at the makeStructWriter method mentioned above.

```go
func makeStructWriter(typ reflect.Type) (writer, error) {
	fields, err := structFields(typ)
	...
	writer := func(val reflect.Value, w *encbuf) error {
		lh := w.list()
		for _, f := range fields {
			if err := f.info.writer(val.Field(f.index), w); err != nil {
				return err
			}
		}
		w.listEnd(lh)
	}
}
```

You can see that the above code reflects the special processing method for processing structure data, that is, first call the w.list() method, and then call the listEnd(lh) method after processing. The reason for adopting this method is that we do not know how long the length of the processed structure is when we first start processing the structure, because it is necessary to determine the processing method of the head according to the length of the structure (recall the structure inside the yellow book) The way the body is processed), so we record the position of str before processing, and then start processing each field. After processing, we will see how much the length of the processed structure is after looking at the data of str.

```go
func (w *encbuf) list() *listhead {
	lh := &listhead{offset: len(w.str), size: w.lhsize}
	w.lheads = append(w.lheads, lh)
	return lh
}

func (w *encbuf) listEnd(lh *listhead) {
	lh.size = w.size() - lh.offset - lh.size    // lh.size records the length of the queue header that should be occupied when the list starts. w.size() returns the length of str plus lhsize
	if lh.size < 56 {
		w.lhsize += 1 // length encoded into kind tag
	} else {
		w.lhsize += 1 + intsize(uint64(lh.size))
	}
}
func (w *encbuf) size() int {
	return len(w.str) + w.lhsize
}
```

Then we can look at the final processing logic of encbuf, process the listhead and assemble it into complete RLP data.

```go
func (w *encbuf) toBytes() []byte {
	out := make([]byte, w.size())
	strpos := 0
	pos := 0
	for _, head := range w.lheads {
		// write string data before header
		n := copy(out[pos:], w.str[strpos:head.offset])
		pos += n
		strpos += n
		// write the header
		enc := head.encode(out[pos:])
		pos += len(enc)
	}
	// copy string data after the last list header
	copy(out[pos:], w.str[strpos:])
	return out
}
```

##### writer

The rest of the process is actually quite simple. It is based on the yellow book to fill each different data into the encbuf.

```go
func writeBool(val reflect.Value, w *encbuf) error {
	if val.Bool() {
		w.str = append(w.str, 0x01)
	} else {
		w.str = append(w.str, 0x80)
	}
	return nil
}

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

#### decode.go

The general flow of the decoder is similar to that of the encoder. Understand the general flow of the encoder and know the general flow of the decoder.

```go
func (s *Stream) Decode(val interface{}) error {
	if val == nil {
		return errDecodeIntoNil
	}
	rval := reflect.ValueOf(val)
	rtyp := rval.Type()
	if rtyp.Kind() != reflect.Ptr {
		return errNoPointer
	}
	if rval.IsNil() {
		return errDecodeIntoNil
	}
	info, err := cachedTypeInfo(rtyp.Elem(), tags{})
	if err != nil {
		return err
	}
	err = info.decoder(s, rval.Elem())
	if decErr, ok := err.(*decodeError); ok && len(decErr.ctx) > 0 {
		// add decode target type to error so context has more meaning
		decErr.ctx = append(decErr.ctx, fmt.Sprint("(", rtyp.Elem(), ")"))
	}
	return err
}

func makeDecoder(typ reflect.Type, tags tags) (dec decoder, err error) {
	kind := typ.Kind()
	switch {
	case typ == rawValueType:
		return decodeRawValue, nil
	case typ.Implements(decoderInterface):
		return decodeDecoder, nil
	case kind != reflect.Ptr && reflect.PtrTo(typ).Implements(decoderInterface):
		return decodeDecoderNoPtr, nil
	case typ.AssignableTo(reflect.PtrTo(bigInt)):
		return decodeBigInt, nil
	case typ.AssignableTo(bigInt):
		return decodeBigIntNoPtr, nil
	case isUint(kind):
		return decodeUint, nil
	case kind == reflect.Bool:
		return decodeBool, nil
	case kind == reflect.String:
		return decodeString, nil
	case kind == reflect.Slice || kind == reflect.Array:
		return makeListDecoder(typ, tags)
	case kind == reflect.Struct:
		return makeStructDecoder(typ)
	case kind == reflect.Ptr:
		if tags.nilOK {
			return makeOptionalPtrDecoder(typ)
		}
		return makePtrDecoder(typ)
	case kind == reflect.Interface:
		return decodeInterface, nil
	default:
		return nil, fmt.Errorf("rlp: type %v is not RLP-serializable", typ)
	}
}
```

We also look at the specific decoding process through the decoding process of the structure type. Similar to the encoding process, first get all the fields that need to be decoded through structFields, and then decode each field. There is almost a List() and ListEnd() operation with the encoding process, but the processing flow here is not the same as the encoding process, which will be described in detail in subsequent chapters.

```go
func makeStructDecoder(typ reflect.Type) (decoder, error) {
	fields, err := structFields(typ)
	if err != nil {
		return nil, err
	}
	dec := func(s *Stream, val reflect.Value) (err error) {
		if _, err := s.List(); err != nil {
			return wrapStreamError(err, typ)
		}
		for _, f := range fields {
			err := f.info.decoder(s, val.Field(f.index))
			if err == EOL {
				return &decodeError{msg: "too few elements", typ: typ}
			} else if err != nil {
				return addErrorContext(err, "."+typ.Field(f.index).Name)
			}
		}
		return wrapStreamError(s.ListEnd(), typ)
	}
	return dec, nil
}
```

Let's look at the decoding process of strings. Because strings of different lengths have different ways of encoding, we can get the type of the string by the difference of the prefix. Here we get the type that needs to be parsed by s.Kind() method. The length, if it is a Byte type, directly returns the value of Byte. If it is a String type, it reads the value of the specified length and returns. This is the purpose of the kind() method.

```go
func (s *Stream) Bytes() ([]byte, error) {
	kind, size, err := s.Kind()
	if err != nil {
		return nil, err
	}
	switch kind {
	case Byte:
		s.kind = -1 // rearm Kind
		return []byte{s.byteval}, nil
	case String:
		b := make([]byte, size)
		if err = s.readFull(b); err != nil {
			return nil, err
		}
		if size == 1 && b[0] < 128 {
			return nil, ErrCanonSize
		}
		return b, nil
	default:
		return nil, ErrExpectedString
	}
}
```

##### Stream structure

The other code of the decoder is similar to the structure of the encoder, but there is a special structure that is not in the encoder. That is Stream. This is a helper class for reading RLP in a streaming manner. Earlier we talked about the general decoding process is to first get the type and length of the object to be decoded through the Kind () method, and then decode the data according to the length and type. So how do we deal with the structure's fields and the structure's data? Recall that when we process the structure, we first call the s.List() method, then decode each field, and finally call s.EndList(). method. The trick is in these two methods. Let's take a look at these two methods.

```go
type Stream struct {
	r ByteReader
	// number of bytes remaining to be read from r.
	remaining uint64
	limited   bool
	// auxiliary buffer for integer decoding
	uintbuf []byte
	kind    Kind   // kind of value ahead
	size    uint64 // size of value ahead
	byteval byte   // value of single byte in type tag
	kinderr error  // error from last readKind
	stack   []listpos
}
type listpos struct{ pos, size uint64 }
```

Stream's List method when calling the List method. We first call the Kind method to get the type and length. If the type doesn't match, we throw an error. Then we push a listpos object onto the stack. This object is the key. The pos field of this object records how many bytes of data the current list has read, so it must be 0 at the beginning. The size field records how many bytes of data the list object needs to read. So when I process each subsequent field, every time I read some bytes, it will increase the value of the pos field. After processing, it will compare whether the pos field and the size field are equal. If they are not equal, an exception will be thrown. .

```go
func (s *Stream) List() (size uint64, err error) {
	kind, size, err := s.Kind()
	if err != nil {
		return 0, err
	}
	if kind != List {
		return 0, ErrExpectedList
	}
	s.stack = append(s.stack, listpos{0, size})
	s.kind = -1
	s.size = 0
	return size, nil
}
```

Stream's ListEnd method, if the current number of data read pos is not equal to the declared data length size, throw an exception, and then pop the stack, if the current stack is not empty, then add pos at the top of the stack The length of the currently processed data (used to handle this situation - the structure's field is the structure, this recursive structure)

```go
func (s *Stream) ListEnd() error {
	if len(s.stack) == 0 {
		return errNotInList
	}
	tos := s.stack[len(s.stack)-1]
	if tos.pos != tos.size {
		return errNotAtEOL
	}
	s.stack = s.stack[:len(s.stack)-1] // pop
	if len(s.stack) > 0 {
		s.stack[len(s.stack)-1].pos += tos.size
	}
	s.kind = -1
	s.size = 0
	return nil
}
```
