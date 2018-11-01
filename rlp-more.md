## Ethereum RLP coding

> RLP (Recursive Length Prefix), which is the encoding method used in the serialization of Ethereum. RLP is mainly used for network transmission and persistent storage of data in Ethereum.

### Why do you have to rebuild wheels?

There are many methods for object serialization, such as JSON encoding, but JSON has an obvious disadvantage: the encoding result is relatively large. For example, the following structure:

```go
type Student struct{
    Name string `json:"name"`
    Sex string `json:"sex"`
}
s := Student{Name:"icattlecoder", Sex:"male"}
bs,_ := json.Marsal(&s)
print(string(bs))
// {"name":"icattlecoder","sex":"male"}
```

Variable s has serialization result `{"name":"icattlecoder","sex":"male"}`, the length of the string is 35, the actual data is valid `icattlecoder`, and `male` a total of 16 bytes, we can see that too much redundant information is introduced when serializing JSON. Assuming Ethereum uses JSON to serialize, then the original 50GB blockchain may now be 100GB, which is double in size.

Therefore, Ethereum needs to design a coding method with smaller results.

### RLP encoding definition

RLP actually only encodes the following two types of data:

1. Byte array

2. An array of byte arrays, called a list

**Rule 1** : For a single byte whose value is between [0, 127], its encoding is itself.

Example 1: `a` The encoding is `97`.

**Rule 2** : If the byte array is long `l <= 55`, the result of the encoding is the array itself, plus the `128+l` prefix.

Example 2: The empty string encoding is `128`, ie `128 = 128 + 0`.

Example 3: The `abc` result of the encoding is `131 97 98 99`, in which `131=128+len("abc")`, `97 98 99` in order `a b c`.

**Rule 3** : If the array length is greater than 55, the first result of the encoding is the length of the encoding of 183 plus the length of the array, then the encoding of the length of the array itself, and finally the encoding of the byte array.

Example 4: Encode the following string:

```text
The length of this sentence is more than 55 bytes, I know it because I pre-designed it
```

This string has a total of 86 bytes, and the encoding of 86 requires only one byte, which is its own, so the result of the encoding is as follows:

```byte
184 86 84 104 101 32 108 101 110 103 116 104 32 111 102 32 116 104 105 115 32 115 101 110 116 101 110 99 101 32 105 115 32 109 111 114 101 32 116 104 97 110 32 53 53 32 98 121 116 101 115 44 32 73 32 107 110 111 119 32 105 116 32 98 101 99 97 117 115 101 32 73 32 112 114 101 45 100 101 115 105 103 110 101 100 32 105 116
```

The first three bytes are calculated as follows:

1. `184 = 183 + 1` Because the array length is `86` encoded and only takes up one byte.
2. `86` Array length is `86`
3. `84` the `T` character

**Rule 4** : If the list length is less than 55, the first bit of the encoding result is the length of the encoding of the 192 plus list length, and then the encoding of each sublist is sequentially connected.

Note that rule 4 itself is recursively defined.
Example 6: `["abc", "def"]` The result of the encoding is `200 131 97 98 99 131 100 101 102`.
Where in `abc` the encoded `131 97 98 99`, `def` encoding is `131 100 101 102`. The total length of the two encoded sub-strings is 8, so the encoding results of the calculated one: `192 + 8 = 200`.

**Rule 5** : If the list length exceeds 55, the first digit of the encoding result is the encoding length of 247 plus the length of the list, then the encoding of the length of the list itself, and finally the encoding of each sublist is connected in turn.

Rule 5 itself is also recursively defined, similar to rule 3.

Example 7:

```text
["The length of this sentence is more than 55 bytes, ", "I know it because I pre-designed it"]
```

The coding result is:

```byte
248 88 179 84 104 101 32 108 101 110 103 116 104 32 111 102 32 116 104 105 115 32 115 101 110 116 101 110 99 101 32 105 115 32 109 111 114 101 32 116 104 97 110 32 53 53 32 98 121 116 101 115 44 32 163 73 32 107 110 111 119 32 105 116 32 98 101 99 97 117 115 101 32 73 32 112 114 101 45 100 101 115 105 103 110 101 100 32 105 116
```

The first two bytes are calculated as follows:

1. `248 = 247 +1`
2. `88 = 86 + 2` In the example of rule 3 , the length is `86`, and in this example, since there are two substrings, the encoding of the length of each substring itself occupies 1 byte each, so it occupies 2 bytes in total.

The first three bytes `179` in accordance with **Rule 2** stars `179 = 128 + 51`

The 55th byte is `163` also derived from **Rule 2** `163 = 128 + 35`

### RLP decoding

When decoding, first `f` perform the following rule judgment according to the size of the first byte of the encoding result:

1. If f∈ [0,128), then it is a byte itself.

2. If f∈[128,184), then it is a byte array with a length of 55 or less. The length of the array is `l=f-128`

3. If f∈[184,192), then it is an array of length over 55, the length of the length of the code itself `ll=f-183`, and then read the bytes of length ll from the second byte, encoded into integers according to BigEndian, l Is the length of the array.

4. If f∈(192,247], then it is a list with a total length of not more than 55, and the list length is `l=f-192`. Recursively uses rules 1~4 for decoding.

5. If f∈(247,256], then it is a list with a length greater than 55 after encoding, its length is itself encoded length `ll=f-247`, and then reads bytes of length ll from the second byte, encoded by BigEndian into integers l, l This is the length of the sublist. It is then recursively decoded according to the decoding rules.

The above explains what is called **recursive length prefix** encoding, which itself explains the encoding rules very well.

Sample python code to debug RLP encoding:

```python
import sys
import json
from termcolor import colored

def rlp_encode(input):
    if isinstance(input, str):
        if len(input) == 1 and ord(input) < 0x80:
            return input
        else:
            return encode_length(len(input), 0x80) + input
    elif isinstance(input, list):
        output = ''
        for item in input:
            output += rlp_encode(item)
        return encode_length(len(output), 0xc0) + output


def encode_length(L, offset):
    if L < 56:
        return chr(L + offset)
    elif L < 256**8:
        BL = to_binary(L)
        return chr(len(BL) + offset + 55) + BL
    else:
        raise Exception("input too long")


def to_binary(x):
    if x == 0:
        return ''
    else:
        return to_binary(int(x / 256)) + chr(x % 256)


def format_rlp_encode(input):
    output = []
    for c in input:
        ordC = ord(c)
        if ordC >= 0x80:
            output.append("0x{:02x}".format(ordC))
        else:
            output.append("'{}'".format(c))

    return "[ " + ", ".join(output) + " ]"


# run as source file
if __name__ == "__main__":

    represent = "The "
    obj_type = "string"
    if len(sys.argv) == 2:
        input = sys.argv[1]
        if input.startswith("json"):
            input = json.loads(input[4:])
            obj_type = "list"
    else:
        input = sys.argv[1:]
        obj_type = "list"

    if len(input) == 0:
        represent += "empty "
    represent += obj_type

    represent += " {} = ".format(json.dumps(input))
    # finally output
    output = rlp_encode(input)
    represent += format_rlp_encode(output)
    print(colored(represent, 'green'))
```
