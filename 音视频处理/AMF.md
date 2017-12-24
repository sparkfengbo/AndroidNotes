AMF

AMF是一个二进制格式，用来序列化对象图（object graph），例如 [ActionScript](https://en.wikipedia.org/wiki/ActionScript)对象、XML，或者用来在AdobeFlash客户端和远程服务之间传递消息。

AMF通常用来在RTMP发送媒体流时建立连接和发送控制命令。

AMF self-contained packet
参考：
[Action Message Format](https://en.wikipedia.org/wiki/Action_Message_Format)


AMF0

这个格式制定了多种数据类型用来编码数据。
 Adobe states that AMF is mainly used to represent object graphs that include named properties in the form of key-value pairs, where the keys are encoded as strings and the values can be of any data type such as strings or numbers as well as arrays and other objects. XML is supported as a native type. Each type is denoted by a single byte preceding the actual data. The values of that byte are as below (for AMF0):
 
- Number - 0x00 (Encoded as IEEE 64-bit double-precision floating point number)
- Boolean - 0x01 (Encoded as a single byte of value 0x00 or 0x01)
- String - 0x02 (16-bit integer string length with UTF-8 string)
- Object - 0x03 (Set of key/value pairs)
- Null - 0x05
- ECMA Array - 0x08 (32-bit entry count)
- Object End - 0x09 (preceded by an empty 16-bit string length)
- Strict Array - 0x0a (32-bit entry count)
- Date - 0x0b (Encoded as IEEE 64-bit double-precision floating point number with 16-bit integer timezone offset)
- Long String - 0x0c (32-bit integer string length with UTF-8 string)
- XML Document - 0x0f (32-bit integer string length with UTF-8 string)
- Typed Object - 0x10 (16-bit integer name length with UTF-8 name, followed by entries)
- Switch to AMF3 - 0x11

说明：

- AMF对象以0x03开始，接着是一组key-value对，以0x09结束（0x09之前是0x00 0x00 表示空的key entry）。
- keys are encoded as strings
- Values can be of any type including other objects and whole object graphs can be serialized in this way.
- key如果是Object或者strings的话( object keys and strings are )，前面会有**两字节**表示key的长度（字节），比如string类型的key前面包含了0x02的三字节数据。
- Null types only contain their type-definition (0x05)
- Numbers are encoded as double-precision floating point and are composed of eight bytes.

举例：
```
var person:Object = {name:'Mike', age:'30', alias:'Mike'};
var stream:ByteArray = new ByteArray();
stream.objectEncoding = ObjectEncoding.AMF0; // ByteArray defaults to AMF3
stream.writeObject(person);
```

```
03 00 04 6e 61 6d 65 02 00 04 4d 69 6b 65 00 03 61 67 65 00 40 3e 00 00 00 00 00 00 00 05 61 6c 69 61 73 02 00 04 4d 69 6b 65 00 00 09	

. . . n a m e . . . M i k e . . a g e . @ > . . . . . . . . a l i a s . . . M i k e . . .

```

- 03为AMF的开头
- 00 04表示name这个key有4字节长度，总共占两字节，所有key都是按string类型编码，所以前面没有0x02
- 6e 61 6d 65 代表 name
- 02 00 04 三个字节中02代表string类型，00 04是规定的两字节，表示value的长度为4
- 4d 69 6b 65 是Mike
- 00 03 中代表key的长度是3
- 61 67 65代表age
- 00代表Number
- number是按双精度浮点类型存储的，占用8字节，40 3e 00 00 00 00 00 00 表示30
- 00 05表示key占用5个字节
- 61 6c 69 61 73 表示alias
- 02 00 04 表示类型是string，占用4个字节
- 4d 69 6b 65表示Mike
- 最后的00 00表示空的key entry
- 09是AMF结尾

The AMF message starts with a 0x03 which denotes an RTMP packet with Header Type of 0, so 12 bytes are expected to follow. It is of Message Type 0x14, which denotes a command in the form of a string of value "_result" and two serialized objects as arguments. The message can be decoded as follows:


AMF3


参考：
[Action Message Format](https://en.wikipedia.org/wiki/Action_Message_Format)