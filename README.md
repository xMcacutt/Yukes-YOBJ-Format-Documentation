# Yukes-YOBJ-Format-Documentation

The YOBJ (Yukes Object) format is a 3D model container format used by yukes in the late 2000s. It stores mesh and skeletal data with animation and textures being stored separately.

The format was modified for some games to have a unique extension and an additional 8 bytes of padding at the start of the file indicated by the string DUMY.

For this reason, the file definition below begins from after the YOBJ magic + 4bytes. All offsets should be considered rebased from 0x10 as 0x0. The section prior to 0x10 we will refer to as the pre-header.

## Pre-Header

The pre-header length depends on the file format. For some games, the YOBJ format is used with an 8 byte pre-header. In other formats such as the .ymp, the pre-header is 0x10 bytes long.

| Offset | Type    | Description                 |
| ------ | ------- | --------------------------- |
| 0x0    | Char[4] | DUMY (only in non .yobj)    |
| 0x4    | Byte[4] | Padding (only in non .yobj) |
| 0x8    | Char[4] | YOBJ Magic                  |
| 0xC    | Int32   | File Length - POF0          |

## Header

The header consists of 0x40 bytes.

| Offset | Type     | Description          |
| ------ | -------- | -------------------- |
| 0x0    | Int32    | Unk                  |
| 0x4    | Int32    | POF0 Section Ptr     |
| 0x8    | Byte[8]  | Padding              |
| 0x10   | Int32    | Mesh Count           |
| 0x14   | Int32    | Bone Count           |
| 0x18   | Int32    | Material Count       |
| 0x1C   | Int32    | Mesh Descriptors Ptr |
| 0x20   | Int32    | Bone Descriptors Ptr |
| 0x24   | Int32    | Material Names Ptr   |
| 0x28   | Int32    | Object Names Ptr     |
| 0x2C   | Int32    | Object Count         |
| 0x30   | Byte[16] | Padding              |

## Mesh Descriptors

Each Mesh Descriptor consists of 0x40 bytes. Each mesh belongs to an Object and multiple Meshes can belong to the same Object. Meshes contain submeshes which contain the Material indices so that each Mesh can use multiple Materials rather than needing a new Mesh for every material used on an object.

| Offset | Type     | Description           |
| ------ | -------- | --------------------- |
| 0x0    | Int32    | Unk                   |
| 0x4    | Int32    | SubMesh Count         |
| 0x8    | Int32    | Hashed MeshData? Ptr  |
| 0xC    | Int32    | FaceData Ptr          |
| 0x10   | Int32    | (Parent) Object Index |
| 0x14   | Int32    | Unk                   |
| 0x18   | Int32    | Vertex Data Ptr       |
| 0x1C   | Int32    | Skinning Data? Ptr    |
| 0x20   | Int32    | Unk                   |
| 0x24   | Int32    | Face Count            |
| 0x28   | Int32    | Vertex Count          |
| 0x2C   | Byte[4]  | Padding?              |
| 0x30   | Float[4] | Mesh Origin?          |

## Mesh Data

Note: The YOBJ format used a left-handed coordinate system.

The Mesh Data section contains multiple subsections with vertex positions, weights, uvs, face indices, etc. These are the sections in order of appearance:

#### Hashed Mesh Data

The Hashed Mesh data seems to provide offsets to elements of the Mesh for faster access by the engine. This is the least well understood section and may contain other data such as bone indices for skinning. More research needed.

#### Vertex Data

Vertices are defined at the Mesh level. Multiple SubMeshes may use the same Vertices for their Face Indices and Face UVs definitions.

The Vertex Data section begins with a pointer to the Vertex buffer which is usually one line below. The last two bytes of the first line of the Vertex buffer are significant. The first gives the Vertex count for the buffer and the second is the VIF Vector4f unpack command indicating that the following section contains the vertex buffer.

The Vertex buffer itself is a series of xyzw positions belonging to the mesh. Of course, the validation float w should always be equal to 1.0.

Immediately after the Vertex buffer, vertex normals are defined. The first line again ends with the count and the VIF Vector4f unpack command and then the Vertex Normals are listed similarly to the Vertex buffer.

#### Skinning

Skinning needs a significant amount more research. Until that is done, this is what we know:

The Skinning section follows on from after the Vertex Data. There is one Weight buffer for each SubMesh. The final four bytes of the first line are significant but unknown. The first byte appears to be some sort of count or alignment. The second byte might be the Bone index though this doesn't seem correct. The third byte is the number of Weights in the buffer and the fourth byte is the VIF Vector4f unpack command.

Strangely, the number of Weights is not equal to the number of Vertices. It seems it is linked to the number of faces instead.

It would seem that each Vector4f contains the Weight of each Vertex in a Face to a Bone. (Much more research required).

#### SubMesh Descriptors

The SubMesh Descriptors consist of 0xD0 (208) bytes.

Each submesh contains a number of Face Strips which contain Indices for the Face Vertices and UVs. So Faces belong to the SubMeshes but the Vertices belong to the Mesh.

| Offset | Type      | Description                   |
| ------ | --------- | ----------------------------- |
| 0x0    | Float[8]  | Unk                           |
| 0x20   | Byte[4]   | Unk                           |
| 0x24   | Int32     | Face Type ? (Almost always 7) |
| 0x28   | Int32     | Material Index                |
| 0x2C   | Byte[148] | Padding                       |
| 0xC0   | Short     | Unk                           |
| 0xC2   | Short     | Unk                           |
| 0xC4   | Int32     | Strip Count                   |
| 0xC8   | Int32     | Strip Descriptors Ptr         |
| 0xCC   | Int32     | Strip Data Ptr                |

#### Strip Descriptors

Each Strip Descriptor consists of 0x10 (16) bytes.

| Offset | Type  | Description      |
| ------ | ----- | ---------------- |
| 0x0    | Int32 | Face Vert Count  |
| 0x4    | Int32 | Unk              |
| 0x8    | Int32 | Strip Face Count |
| 0xC    | Int32 | Strip Data Ptr   |

#### Strip Data

The Strip Data may be the most complicated section of the yobj.

The first line gives information on the buffer. The final two bytes, as with the previous buffers, define the count of elements in the buffer and the VIF unpack command.

Another line then gives some more information common to PS2 VIF model formats.

At 0x30, the buffer begins. Each entry in the buffer can be thought of as 0x20 bytes long.

| Offset | Type     | Description  |
| ------ | -------- | ------------ |
| 0x0    | Float[3] | UVW          |
| 0xC    | Int32    | Vertex Index |
| 0x10   | Float[3] | Unk          |
| 0x1C   | Float    | Unk          |

The winding order for the vertices alternates with each new vertex added.

The end of the buffer is marked by 0x00000017.

## Bones

The Bone system uses node based Bones with a parent-child skeletal heirarchy. Every skeleton uses a 'null' Bone as its basis and then a 'root' Bone to determine the origin of the armature. The root Bone is always the child of the null Bone. All other Bones are then connected to root through some heirarchy.

Each bone descriptor consists of 0x50 (80) bytes.

| Offset | Type     | Description                         |
| ------ | -------- | ----------------------------------- |
| 0x0    | Char[16] | Bone Name                           |
| 0x10   | Float[4] | Local Bone Pos (XYZW)               |
| 0x20   | Float[3] | Bone Rotation (Euler XYZ - Radians) |
| 0x2C   | Byte[4]  | Padding                             |
| 0x30   | Int32    | Parent ID (0xFFFF if none)          |
| 0x34   | Float[2] | Rarely Used Unk                     |
| 0x3C   | Byte[4]  | Unk                                 |
| 0x40   | Float[3] | Global Space Bone Position ???      |
| 0x4C   | Float    | Unk - Seems Important               |

## Name Tables

There are two concatenated Name Tables in every YMP. Firstly, the Material Names are given. Each name in the table can use up to 0x10 bytes and is a null terminated string.

The index of the Material referenced by the SubMeshes is the index of the Material in the Name Table.

The same is true for the Object Table which starts immediately after the Material Table.

## POF0

The POF0 section is an encrypted string that has been reversed by others. More understanding is needed however.
When decrypted, it produces file paths pointing to the textures and other files needed for reference.
