+++ 
author = "Bouke Stam" 
title = "RLP Encoding" 
date = "2022-01-12" 
description = "Explanation of the RLP Encoding standard" 
tags = [ "ethereum", ] 
+++

RLP Encoding is an way to serialize and deserialize lists of data. The [Ethereum Wiki](https://eth.wiki/fundamentals/rlp) contains some excellent information about this. The page talks a lot about strings, but in practice that usually means a byte array, not a text string. This was a bit confusing for me at first. The encoding of a byte array works like this:

- if it is a single byte with a value between 0x00 and 0x7f, we just append it
- if it is 0-55 bytes long, we append 0x80 + length and then the bytes
- if it is > 55 bytes, we append 0xb7 then number of bytes in the length of the bytes, the length and after that the bytes

The encoding of a list works like this:

- if the list is 0-55 bytes long, we append 0xc0 + length and then the list
- if the list is >55 bytes, we append 0xf7 + number of bytes in the length of the list, the length and then the list

Using the sample python code in the documentation it was fairly simple to create a function to RLP encode data:

```typescript
function rlpEncode (input: Buffer | Buffer[]): Buffer {
  const toBinary = (x: number): Uint8Array => x === 0 ? new Uint8Array() :
    Buffer.concat([
      toBinary(Math.floor(x / 256)), 
      new Uint8Array([x % 256])
    ]);

  const encodeLength = (length: number, offset: number): Uint8Array => {
    if (length < 56) return new Uint8Array([length + offset]);

    const binaryLength = toBinary(length);
    return Buffer.concat([
      new Uint8Array([binaryLength.length + offset + 55]), 
      binaryLength
    ]);
  };

  if (input instanceof Buffer) {
    if (input.length === 1 && input[0] < 0x80) return input;
    return Buffer.concat([encodeLength(input.length, 0x80), input]);
  }

  const output = Buffer.concat(input.map(buffer => rlpEncode(buffer)));
  return Buffer.concat([encodeLength(output.length, 0xc0), output]);
}
```

