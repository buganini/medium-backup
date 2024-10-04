# My favorite data packing method in 2024
Zstandard(MessagePack(stream)) with Python/Django examples

## Introduction
Zstandard is one of the best compression algorithms available, it is also one of the supported compression methods in PostgreSQL 16, checkout the benchmark in https://www.cybertec-postgresql.com/en/lz4-zstd-pg_dump-compression-postgresql-16/

MessagePack is a more efficient alternative to JSON, particularly for resource-constrained environments like microcontrollers (MCUs). While Zstandard compression may not be feasible on MCUs due to its resource requirements, MessagePack offers several advantages:

* Chunk processing: Unlike JSON, which typically requires parsing the entire string at once, MessagePack can be processed in small chunks. This is especially beneficial for MCUs with limited memory.
* Compact representation: MessagePack provides a more compact binary format compared to JSON, reducing data size and transmission time.
* Wider range of data types and preserves them more accurately than JSON, offering significant advantages in various applications. A key example is MessagePack's native support for binary data, which JSON lacks. This feature allows for more flexible and efficient data handling, especially when dealing with raw bytes or non-textual information. The binary data support eliminates the need for encoding schemes like Base64, resulting in more compact representations and faster processing.
* Finer granularity in its numerical type support. It distinguishes between different sizes of integers (8-bit, 16-bit, 32-bit, and 64-bit) and floating-point numbers (32-bit float and 64-bit double). This precise type preservation allows for more efficient storage and transmission of numerical data, as smaller numbers can be represented more compactly. It also helps maintain the full precision of larger integers and floating-point values, which can be crucial in scientific, financial, or other data-sensitive applications.
* Multi-language support: MessagePack is available for many programming languages, making it versatile for various projects.

For C implementations on MCUs, the mpack library (https://github.com/ludocode/mpack) is my choice.

For more information on MessagePack, including supported languages and implementations, visit the official website: https://msgpack.org.

The following examples demonstrate basic JSON style usage and the use of stream-based compressors and decompressors to minimize memory usage and enhance efficiency when handling large datasets.

## Dependencies
```
pip install zstandard
pip install msgpack
```

## MessagePack as a JSON alternative
While JSON cannot accommodate the binary object in the last element of objs, MessagePack is capable of containing it.
``` python
import msgpack

objs = [
    "1",
    2,
    [3,4],
    {"5":6},
    b"7"
]

mp = msgpack.packb(objs)
print(mp) # b'\x95\xa11\x02\x92\x03\x04\x81\xa15\x06\xc4\x017'
len(mp) # 14

msgpack.unpackb(mp) #['1', 2, [3, 4], {'5': 6}, b'7']

######

import json

json.dumps(objs)
"""
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/opt/homebrew/Cellar/python@3.12/3.12.5/Frameworks/Python.framework/Versions/3.12/lib/python3.12/json/__init__.py", line 231, in dumps
    return _default_encoder.encode(obj)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/opt/homebrew/Cellar/python@3.12/3.12.5/Frameworks/Python.framework/Versions/3.12/lib/python3.12/json/encoder.py", line 200, in encode
    chunks = self.iterencode(o, _one_shot=True)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/opt/homebrew/Cellar/python@3.12/3.12.5/Frameworks/Python.framework/Versions/3.12/lib/python3.12/json/encoder.py", line 258, in iterencode
    return _iterencode(o, 0)
           ^^^^^^^^^^^^^^^^^
  File "/opt/homebrew/Cellar/python@3.12/3.12.5/Frameworks/Python.framework/Versions/3.12/lib/python3.12/json/encoder.py", line 180, in default
    raise TypeError(f'Object of type {o.__class__.__name__} '
TypeError: Object of type bytes is not JSON serializable
"""

jd = json.dumps(objs[:-1])
print(jd) # '["1", 2, [3, 4], {"5": 6}]'
len(jd) # 26
json.loads(jd) #['1', 2, [3, 4], {'5': 6}]
```

## MessagePack+Zstd: Packing to File
``` python
import zstandard
import msgpack

cctx = zstandard.ZstdCompressor(level=3)
# up to zstandard.MAX_COMPRESSION_LEVEL

cobj = cctx.compressobj()

packer = msgpack.Packer()

objs = [
    "1",
    2,
    [3,4],
    {"5":6},
    b"7"
]

with open("xxx.mpz", "wb") as f:
    # f.write(b"HEADER")

    for obj in objs:
        f.write(cobj.compress(packer.pack(obj)))

    f.write(cobj.flush())
```

## MessagePack+Zstd: Unpacking from File
``` python
import zstandard
import msgpack

with open("xxx.mpz", "rb") as f:
    # assert f.read(6) == b"HEADER"

    dctx = zstandard.ZstdDecompressor()
    with dctx.stream_reader(f) as reader:
        reader = dctx.stream_reader(f)

        unpacker = msgpack.Unpacker(reader)

        # obj = next(unpacker)
        # print("First block", repr(obj))

        for obj in unpacker:
            print(repr(obj))
```

Output
```
First block '1'
2
[3, 4]
{'5': 6}
b'7'
```

## MessagePack+Zstd: Packing to Django StreamingHttpResponse
``` python
import zstandard
import msgpack
from django.http.response import StreamingHttpResponse

def get(self, request):
    objs = [
        "1",
        2,
        [3,4],
        {"5":6},
        b"7"
    ]

    def output():
        cctx = zstandard.ZstdCompressor(level=3)
        cobj = cctx.compressobj()
    
        packer = msgpack.Packer()
        for obj in objs:
            yield cobj.compress(packer.pack(obj))
        yield cobj.flush()
  
    response = StreamingHttpResponse(output())
    response['Content-Type'] = 'application/octet-stream'
    return response
```

## MessagePack+Zstd: Unpacking from requests.get(stream=True)
``` python
import zstandard
import msgpack
import requests

CHUNK_SIZE = 4096

dctx = zstandard.ZstdDecompressor()
dobj = dctx.decompressobj()
unpacker = msgpack.Unpacker()

r = requests.get(url, stream=True)
for chunk in r.iter_content(chunk_size=CHUNK_SIZE):
    d = dobj.decompress(chunk)
    if d:
        unpacker.feed(d)
        for data in unpacker:
            print(repr(data))
```

## MessagePack+Zstd: Packing to requests.post()
``` python
import zstandard
import msgpack
import requests

objs = [
    "1",
    2,
    [3,4],
    {"5":6},
    b"7"
]

def output():
    cctx = zstandard.ZstdCompressor(level=3)
    cobj = cctx.compressobj()

    packer = msgpack.Packer()
    for obj in objs:
        yield cobj.compress(packer.pack(obj))
    yield cobj.flush()

requests.post(url, data=output())
```

## MessagePack+Zstd: Unpacking in browser with JavaScript
``` html
<script src="https://cdn.jsdelivr.net/npm/axios@1.7.7/dist/axios.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/fzstd/umd/index.js"></script>
<script src="https://rawgit.com/kawanet/msgpack-lite/master/dist/msgpack.min.js"></script>
<script>
  var unpacker = new msgpack.Decoder();
  unpacker.on("data", (data)=>{
      console.log(data); // output
  });
  
  const dctx = new fzstd.Decompress((chunk, isLast) => {
      unpacker.decode(chunk);
      if (isLast) {
          unpacker.end();
      }
  });
  
  const url = "https://......"
  
  // Haven't found a good way to do this in stream fetching
  const response = await axios.get(url, {responseType: 'arraybuffer'});
  const data = new Uint8Array(response.data);
  dctx.push(data, true);
</script>
```