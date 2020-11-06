> This post was made by [ImActuallyAnna](https://www.roblox.com/users/1094977/profile) and myself as part of a joint project for a Roblox RBXM Decoder which can be found here: <br />
> [https://devforum.roblox.com/t/roblox-model-file-decoder/630292](https://devforum.roblox.com/t/roblox-model-file-decoder/630292)

# Introduction

We had a case where we wanted the ability to insert any Roblox model into a game using a server script. Roblox provides a default solution to this through [InsertService](https://developer.roblox.com/en-us/api-reference/class/InsertService), but they impose harsh restrictions on which models can be inserted with it. 

These restrictions stem from a time where InsertService was being abused in free models to load malicious and remotely controllable third-party content into unsuspecting creators' games. Roblox has since introduced `require(id)`, another feature that carries the same implications and is being abused in the same way, but seems to be cracking down on users who publish malicious code to the toolbox. At any rate, they [don't seem particularly interested in changing this behavior any time soon](https://devforum.roblox.com/t/allow-insertservice-loadasset-to-insert-free-assets/373593).

The `require(id)` function is somewhat more restrictive as it requires a ModuleScript named `MainModule` in the model. This code is then run and *could* be used to insert models. However, this won't work for the vast majority of models in the Roblox toolbox.

The goal ultimately is to allow users to insert any public-domain Roblox models on a live game server, and allow them to do so safely. I used to maintain an Insert Wars-style game on Roblox a few years ago and am no stranger to the amount of damage unchecked scripts can cause on a game, much less a game that isn't designed to take such abuse.

To summarize, we are facing the following challenges:

- We can't insert models from the toolbox, unless the model either is pasted into the game by a `MainModule` or is in the inventory of the owner of the game.
- Even if we could insert models arbitrarily, we can't screen the scripts inside them. We can't read the `BaseScript.Source` property. This means any models we insert either run unchecked code, or don't run any code at all.

This article outlines how we are going about solving these issues.

# Roblox Model Binary Format
It didn't take long for us to stumble upon the idea of reading a `*.rbxm` file and manually creating the instances serialized within it. Most of the models on the toolbox nowadays are serialized in their binary format. [We've already written some code that does exactly that](https://github.com/Kat-Digital/RBXM-Reader). 

Reverse-engineering a binary format is no easy feat, but fortunately [someone else already did a lot of the base work for us](http://www.classy-studios.com/Downloads/RobloxFileSpec.pdf). However, the Roblox file format specification has changed over the years and the format specification I've linked no longer seems to be fully up-to-date. In that light, we thought it fitting to write our own documentation on how the Roblox binary format currently works.

## Basic Concepts
The Roblox binary format builds upon a few basic concepts. Primarily, these are byte-interleaving, value transformation and LZ4 compression. Byte-interleaving and value transformations are used primarily for the purpose of improving the compression ratio when the data is LZ4-compressed. I'll give a quick description of each concept below. Note that not all values are stored in this way.

### Endianness

Roblox model files use both big-endian and little-endian values. The endianness in this case refers to the byte order. In general, non-transformed values are encoded in little-endian while transformed values are encoded in big-endian. I will point out whenever this isn't the case.

### Byte-Interleaving
Byte-interleaving is quite simple: Rather than serializing values one-by-one, values are serialized as all of their first bytes, then all of their second bytes, and so on. For example, if we have three 32-bit integers `A`, `B`, and `C`, these would be encoded as `ABC ABC ABC ABC` rather than `AAAA BBBB CCCC`. 

This improves the compression ratio particularly in cases where the values are relatively close; if all of the highest-order bytes are zero, this means 25% of the data can effectively be cut.

### Value Storage
Roblox makes heavy use of LZ4 compression in their model and place files. Because of this, Roblox also stores values in a specific way to achieve high compression ratios with LZ4. 

Roblox transforms both integers and float values. Doubles are always stored as [regular IEEE754 Double-Precision Floating Point Numbers](https://en.wikipedia.org/wiki/Double-precision_floating-point_format).

Float values are stored in their default value, except that the LSB is used as the sign rather than the MSB. This allows for better compression ratios for values with similar magnitudes, because the exponent fits squarely within the highest-order byte.

Integers are transformed to increase the number of zero bits in negative numbers. This once again allows for better compression ratios for values with similar magnitudes, because the higher-order bytes for both positive and negative values will be the same if they're within the same ballpark. For integers, the LSB is the sign (0 = positive, 1 = negative) and the magnitude of the value is stored in the upper 31 bits. Additionally, if the number is negative, the magnitude is reduced by 1 before storing the value to avoid double-defining 0.

Finally, strings are preceded by a 4-byte, little-endian integer that states the length of the string, followed by the literal string. Strings in RBXM files are not null-terminated.

### LZ4 Compression
Roblox model files use LZ4 compression to store most of the data. Fortunately, we only want to read the files for the moment, so we only have to concern ourselves with the much simpler decompression algorithm. Below is a quick description of how it works.

Each LZ4-compressed block of data is constructed of a 12-byte header, followed by the compressed data. The header is constructed as follows:

| Field             | Size    | Description                                               |
|-------------------|---------|-----------------------------------------------------------|
| Compressed Size   | 4 bytes | The size of the compressed data, not including the header |
| Uncompressed Size | 4 bytes | The size of the data when uncompressed                    |
| Zero              | 4 bytes | 4 bytes that are always set to 0                          |

Followed by this header are blocks of compressed data. All blocks contain "literal data" and "match data". Literal data is data that couldn't be compressed, and match data is data that is repeated some number of times. The last block is detected as any block where there is no match data, i.e. the compressed data stream ends immediately after the literal data.

A block is always constructed as follows:

| Field                    | Size     | Description                                                            |
|--------------------------|----------|------------------------------------------------------------------------|
| Token                    | 1 byte   | Contains the size of the literal data in the 4 higher-order bits, and the size of the match data in the 4 lower-order bits.[2] |
| Optional: Literal Length | Variable | More bytes determining the length of the literal data[1]               |
| Literal Data             | Variable | The literal data, with its length as defined above                     |
| Offset                   | 2 bytes  | The offset at which the match data begins, in little-endian byte order |
| Optional: Match Length   | Variable | More bytes determining the length of the match data[1][2]              |

[1] Since the lengths of literal and match data are both given within just 4 bit in the token byte, those values would be restricted to 15 at most. To get around this, if the length of the literal or match data in the token byte are 15, additional bytes are added to further define the length. Any additional bytes are added to the length, and the length value ends after the first byte that isn't equal to 0xFF. As an example, if the first few bytes of the block are `0xF4 0xFF 0xFF 0x23 0x61`, the length of the literal data is defined as `0x0F + 0xFF + 0xFF + 0x23 = 0x0230 (560)`, with `0x61` being the first byte of the literal data. The length of the match data is encoded in the same way.

[2] The minimum length of the match data is actually 4 bytes, so 4 is always added to the match length encoded in the block.

The offset determines where the match data begins. LZ4 'jumps back' that number of bytes in the output data stream, and copies 'match length' bytes to the end of the data stream. The match length may be greater than the offset - in such cases, the match data is just repeated several times. Partial repetitions are permitted.

As an example, the block `0xBA "Hello World" 0x05 0x00` would decompress as `Hello WorldWorldWorldWorl`. The literal data `Hello World` is copied to the output data stream as-is. Then, the decompression algorithm jumps back 5 bytes and copies `0x0A + 4 = 0xE (14)` bytes to the end of the output data stream.

As another example, the block `0xB0 "Hello World"` is decompressed as `Hello World` and marks the end of the compressed data stream.

## Model Files
A binary Roblox model file always begins with the bytes `3C 72 6F 62 6C 6F 78 21 89 FF 0D 0A 1A 0A 00 00`. These appear to be constant, and we use them as a way of throwing out invalid files very early on. Immediately following this header are two values:

| Field        | Size    | Description |
|--------------|---------|-------------|
| Unique Types | 4 bytes | The number of unique types (instance classes) contained within the Roblox model |
| Object count | 4 bytes | The number of instances contained within the Roblox model                       |

Following this, there are a series of META, INST and SSTR blocks. These form the general 'file header' and initialize some information that is later used to build the actual model. Following this is the model content, which is encoded as a series of PROP blocks. The total number of blocks in the header isn't really known, so the way we dealt with this was to exit the header reading code when we hit a PROP block.

After the PROP blocks follows a PRNT block, which define how instances are parented.

A block is encoded as a 4-byte name, followed by LZ4-encoded data containing the block information.

### META Blocks
As far as we're aware, META blocks contain general settings about the model, in the form of key/value pairs. The block begins with a 4-byte to define how many key/value pairs there are, followed by two strings for each pair. The first string defines the key, and the second string defines the value.

### SSTR Blocks
SSTR blocks contain shared strings. We haven't looked into this block type in detail yet and it doesn't seem to be needed for general instance storage. Mesh and physics data, as well as script source code, all seems to be stored in the form of strings inside of PROP blocks. Will update this when we know more.

### INST Blocks
INST blocks define the existence of instances. There is one INST block per unique class in the model. An inst block contains the following fields:

| Field          | Size or Type | Description |
|----------------|--------------|-------------|
| Index          | 4 bytes      | The index of this class; this is used later to refer back to this instance class |
| Class Name     | String       | The name of the class |
| Is Service     | 1 byte       | Is 0 if the instance is not a service, or non-zero if it is |
| Instance Count | 4 bytes      | The number of instances of this class |
| Referent List  | 4x #instances| Instance referents. This is stored as a list of transformed, interleaved 4-byte integers. |

It should be noted that the referent numbers are cumulative, i.e. the values are added together to give the value at any index. The array `0, 1, 1, 5` would actually mean the referents are `0, 1, 2, 7`. 

In our implementation, the reading of an INST block creates all instances of its type and stores them both in a table with the referent as the key, and an array containing all instances of this type. This makes it easier to handle properties and parenting down the line.

### PROP Blocks
PROP blocks define the values of instance properties. There is one of these blocks for every property on every class that exists in the model. The block defines the property name, property type and the class it belongs to. PROP blocks consist of the following fields:

| Field           | Size or Type    | Description |
|-----------------|-----------------|-------------|
| Class Index     | 4 bytes         | The index of the class this property belongs to |
| Property Name   | String          | The name of the property[1] |
| Property Type   | 1 byte          | A byte determining the type of value; see below for more details |
| Property Values | Depends on Type | The number of instances of this class |

[1] Some properties are serialized on a different name than they do in Studio, and would error because of this. In our implementation, we therefore created a dictionary of property names that need to be replaced with a different name. The two we currently know of are `size` and `Color3uint8`, which need to be replaced with `Size` and `Color`, respectively.

We currently have the following property types implemented:

| Hex Value | Property Type | Notes                                                        |
| --------- | ------------- | ------------------------------------------------------------ |
| 0x01      | String        | -                                                            |
| 0x02      | Boolean       | -                                                            |
| 0x03      | Int32         | -                                                            |
| 0x04      | Float         | -                                                            |
| 0x05      | Double        | -                                                            |
| 0x06      | UDim          | -                                                            |
| 0x07      | UDim2         | -                                                            |
| 0x08      | Ray           | -                                                            |
| 0x09      | Faces         | -                                                            |
| 0x0A      | Axis          | -                                                            |
| 0x0B      | BrickColor    | Doesn't seem to be used anymore in favour of Color3 or Color3uint8 |
| 0x0C      | Color3        | Unsure when this is used in favour of Color3uint8            |
| 0x0D      | Vector2       | -                                                            |
| 0x0E      | Vector3       | -                                                            |
| 0x10      | CFrame        | -                                                            |
| 0x11      | Quaternion    | Doesn't seem to be used & isn't properly implemented. Unsure how this works. |
| 0x12      | Enums         | -                                                            |
| 0x13      | Instance Ref. | -                                                            |
| 0x1A      | Color3uint8   | Unsure when this is used in favour of Color3                 |

The specific encodings of each property type are described below.

### PRNT Blocks
PRNT blocks handle parenting information. We currently don't know if there's only one or several, but our implementation supports a single PRNT block, and expects the file ending following it.

The header of a PRNT block contains a single non-transformed 4-byte integer that defines how many sets of parent data there are.

A PRNT block contains two lists of instance referents. The first defines a list of instances, and the second defines which instances the first set of instances are parented to. Both referent lists are cumulative, and are stored as interleaved and transformed 4-byte integers.

More concretely, a PRNT block consists of the following fields:

| Field              | Size or Type    | Description |
|--------------------|-----------------|-------------|
| Zero (0x00)        | 1 byte          | A PRNT block seems to begin with a single byte that is set to 0x00 |
| Parent Data Length | 4 bytes         | The number of items in each list |
| Instance Referents | 4x #Parent Data | A cumulative, interleaved list of transformed integers referring to instances |
| Parent Referents   | 4x #Parent Data | A cumulative, interleaved list of transformed integers referring to instances |

The instance with a particular index in the first list will be parented to the instance in the second list with the same index.

### File Footer
Any RBXM files are expected to end with the bytes: 

`45 4E 44 00 00 00 00 00 09 00 00 00 00 00 00 00 3C 2F 72 6F 62 6C 6F 78 3E`

We're unsure if these bytes have any particular meaning, but from the original documentation and our own testing, they seem to be constant.

## Property Types
This section explains how specific property types are stored in Roblox files. Each PROP block contains one value for each instance in the class it is defined for.

### String
Each string is encoded as a 4-byte, little-endian integer defining the length of the string in bytes, followed by the literal string. Strings in RBXM files are not null-terminated.

### Boolean
Booleans are always encoded as a single byte, where a `\0` means `false` and anything else means `true`.

### Int32
Int32 properties are stored as a list of interleaved, transformed integers.

### Float
Float properties are stored as a list of interleaved, transformed floats.

### Double
Doubles are stored as an interleaved list, but are not transformed.

### UDim
UDims are stored as two lists; the first is an interleaved list of transformed floats, which define the scale portions of the UDims. Following this is an interleaved list of transformed integers to define the offset portions.

### UDim2
UDim2 properties are stored similarly to UDim properties, but contain four individual lists of values.

The first two lists are lists of interleaved, transformed floats. They define the X and Y scale parameters, respectively. The last two lists are lists of interleaved, transformed integers, and they define the X and Y offset parameters.

### Ray
Ray properties are stored as 6 consecutive lists of interleaved, transformed floats. The first three define the X, Y and Z coordinates of the origin. The last three define the X, Y and Z portions of the direction.

### Faces
Faces represent the `Enum.NormalId` enum. Each value is a single byte, allocated as follows:

| Hex Value | NormalId |
|-----------|----------|
| 0x01      | Enum.NormalId.Front |
| 0x02      | Enum.NormalId.Bottom |
| 0x04      | Enum.NormalId.Left |
| 0x08      | Enum.NormalId.Back |
| 0x10      | Enum.NormalId.Top  |
| 0x20      | Enum.NormalId.Right |

### Axis
Axis values are stored as an array of bytes, with each byte defining one value. The three least significant bits define the X, Y and Z components of the axis, like so: `0000 0XYZ`.

### BrickColor
BrickColors are stored as their numerical representation. They are stored as interleaved but untransformed 4-byte integers, and in this case appear to be encoded in big-endian.

### Color3, Color3uint8
Color3s are stored as 3 consecutive interleave lists of transformed floats. They represent the R, G and B values of each color respectively, with each value ranging from 0 to 1.

Color3uint8s are stored as 3 lists of bytes, with the first list storing the R component, the second the G component, and the third the B component.

### Vector2, Vector3
Vectors are stored as consecutive lists of interleaved floats. Vector2 consists of two such lists representing the X and Y components, and Vector3s consist of three to represent the X, Y and Z components.

### CFrame
CFrames are stored as a list of angle data, followed by a list of Vector3s stored as described above. The angle data stores the orientation of the CFrame, with the Vector3s defining the positions.

The angle data begins with a single byte determining the angle type. This value represents a special angle. If the value is valid for a special angle, that angle defines the rotation matrix for the CFrame, and is followed by the angle data of the next CFrame. If the value is invalid for any special angle, this byte is followed by 9 untransformed, little-endian floats that define the rotation matrix of the CFrame.

| Hex Value | Special Angle (degrees) |
|-----------|-------------------------|
| 0x02      | 0, 0, 0 |
| 0x03      | 90, 0, 0 |
| 0x05      | 180, 0, 0 |
| 0x06      | 270, 0, 0 |
| 0x07      | 180, 0, 270 |
| 0x09      | 90, 90, 0 |
| 0x0A      | 0, 0, 90 |
| 0x0C      | 270, 270, 0 |
| 0x0D      | 270, 0, 270 |
| 0x0E      | 0, 270, 0 |
| 0x10      | 90, 0, 90 |
| 0x11      | 180, 90, 0 |
| 0x14      | 180, 0, 180 |
| 0x15      | 270, 0, 180 |
| 0x17      | 0, 0, 180 |
| 0x18      | 90, 0, 180 |
| 0x19      | 0, 0, 270 |
| 0x1B      | 90, 270, 0 |
| 0x1C      | 180, 0, 90 |
| 0x1E      | 270, 90, 0 |
| 0x1F      | 90, 0, 270 |
| 0x20      | 0, 90, 0 |
| 0x22      | 270, 0, 90 |
| 0x23      | 180, 270, 0 |

### Quaternion
This isn't implemented yet and we're not sure how this works yet, but doesn't seem to appear in any of the models we've tested. We'll investigate it if it turns out this property type is actually used.

### Enums
Enums are stored as a list of interleaved but untransformed big-endian 32-bit integers. The number is the ordinal of the enum it represents. This value can be directly written to enum properties.

### Instance Referents
Instance references are stored as an interleaved list of transformed integers. These values are cumulative, just as described in the section on INST blocks.

# Scripts
The current version of the model reader supports scripts, but requires an elevated script context (such as that of a Studio plugin or the Studio command bar) to work for these properties. For regular scripts, the `.Source` property is inaccessible and throws an error when we attempt to write to it. While we understand the reasons behind why this was done, it carries two huge implications for us:

- If we use InsertService or `require()` to insert models, we can't audit the code or control its execution. We can't sandbox the environment those scripts run on. We're stuck in an all-or-nothing situation where we either need to allow scripts unchecked access to our game, or disable all scripts and thus most likely destroying everything that makes the model what it is.

- If we use our own loader, we get the raw source string, but can't set it on scripts, and are stuck with the same inability to execute the code.

To get around this, we plan to use a Lua compiler such as the [version of Yueliang einsteinK ported for use on Roblox](https://www.roblox.com/library/410455309/Custom-Loadstring). We can modify the Lua bytecode interpreter to sandbox scripts on an instruction level, which is a tad more secure than traditional metatable-based sandbox implementations.

# Unions and MeshParts
While MeshParts have an asset ID and texture ID directly associated with them, as a scriptable property, UnionOperation instances do not. As such, we can't easily load them. We plan to work around this by reading [the MeshData](https://devforum.roblox.com/t/roblox-mesh-format/326114) ourselves and placing polygons as individual WedgeParts. This perhaps isn't the most performant approach but it is one that allows us to display unions when this normally wouldn't be possible at all.

