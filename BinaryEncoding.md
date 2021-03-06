# Binary Encoding

This document describes the [portable](Portability.md) binary encoding of the
[Abstract Syntax Tree](AstSemantics.md) nodes.

The binary encoding is a general representation of syntax trees and module
information that enables small files, fast decoding, and reduced memory usage.
See the [rationale document](Rationale.md#why-a-binary-encoding) for more detail.

The encoding is split into three layers:

* **Layer 0** is a simple pre-order encoding of the AST and related data structures.
  The encoding is dense and trivial to interact with, making it suitable for
  scenarios like JIT, instrumentation tools, and debugging.
* **Layer 1** provides structural compression on top of layer 0, exploiting
  specific knowledge about the nature of the syntax tree and its nodes.
  The structural compression introduces more efficient encoding of values, 
  rearranges values within the module, and prunes structurally identical
  tree nodes.
* **Layer 2** applies generic compression techniques, already available
  in browsers and other tooling. Algorithms as simple as gzip can deliver
  good results, but more sophisticated algorithms like 
  [LZHAM](https://github.com/richgel999/lzham_codec) and
  [Brotli](https://datatracker.ietf.org/doc/draft-alakuijala-brotli/) are able
  to deliver dramatically smaller files.

Most importantly, the layering approach allows development and standardization to
occur incrementally, even though production-quality implementations will need to
implement all of the layers.

# Primitives and key terminology

### varuint32
A [LEB128](https://en.wikipedia.org/wiki/LEB128) variable-length integer, limited to uint32_t payloads. Provides considerable size reduction.

### Pre-order encoding
Refers to an approach for encoding syntax trees, where each node begins with an identifier, followed by any arguments or child nodes.
Pre-order trees can be decoded iteratively or recursively. Alternative approaches include post-order trees and table representations.

* Examples
  * Given a simple AST node: `struct I32Add { AstNode *left, *right; }`
    * First write the operator of `I32Add` (1 byte)
    * Then recursively write the left and right nodes.

  * Given a call AST node: `struct Call { uint32_t callee; vector<AstNode*> args; }`
    * First write the operator of `Call` (1 byte)
    * Then write the (variable-length) integer `Call::callee` (1-5 bytes)
    * Then recursively write each arg node (arity is determined by looking up `callee` in table of signatures)

### Stream splitting
Refers to splitting the single encoded binary stream out into smaller streams, partitioned based on element type or semantic information.
Research has shown that splitting constants, names, and opcodes into their own streams increases the effectiveness of generic compression.

### Subtree deduplication / nullary macros
Identifies and prunes structurally identical nodes and trees of nodes. Most applications contain significant amounts of structural 
duplication that is not completely erased by generic compression.
**Non-nullary macros** are an extension of this technique that enables further compression at the cost of additional complexity.

### Index tables
Modules contain multiple index tables that assign indexes to key pieces of information like opcodes or data types. This enables
compatibility between implementations and allows information to be represented more efficiently.

### Sections
Modules are split up into sections with well-defined contents that can refer to each other and are identified by name.
The use of names allows new section types to be introduced in the future.

### Strings
Strings are encoded as null-terminated [UTF8](http://unicode.org/faq/utf_bom.html#UTF8).

# v8-native module structure

The following documents the current v8-native prototype format, not the binary encoding intended for standardization.

## High-level structure
A module contains (in this order):
* A stream of sections, containing for each section:
  - ```uint8```: A [section type identifier](https://github.com/v8/v8/blob/master/src/wasm/wasm-module.h#L26) for the section
  - The section body (defined below by section type)

### Memory section
* ```uint8```: The minimum size of the module heap in bytes, as a power of two
* ```uint8```: The maximum size of the module heap in bytes, as a power of two
* ```uint8```: ```1``` if the module's memory is externally visible

### Signatures section
* [```varuint32```](#varuint32): The number of function signatures in the section
* For each function signature:
  - ```uint8```: The number of parameters
  - ```uint8```: The function return type, as a [LocalType](https://github.com/v8/v8/blob/master/src/wasm/wasm-opcodes.h#L16)
  - For each parameter:
    + ```uint8```: The parameter type, as a LocalType

### Functions section
This section must be preceded by a [Signatures](#signatures-section) section.

* ```varuint32```: The number of functions in the section
* For each function:
  - ```uint8```: The [function declaration bits](https://github.com/v8/v8/blob/master/src/wasm/wasm-module.h#L39)
  - ```uint16```: The function signature (as an index into the Signatures section)
  - If the ```kDeclFunctionName``` bit is set:
    + ```uint32```: The offset of the function name in the file.
  - If the ```kDeclFunctionImport``` bit is set, **the function entry ends here**
  - If the ```kDeclFunctionLocals``` bit is set:
    + ```uint16```: The number of i32 locals
    + ```uint16```: The number of i64 locals
    + ```uint16```: The number of f32 locals
    + ```uint16```: The number of f64 locals
  - ```uint16```: The size of the function body, in bytes
  - The function body

### Globals section
* ```varuint32```: The number of global variable declarations in the section.
* For each global variable:
  - ```uint32```: The offset of the global variable name in the file.
  - ```uint8```: The type of the global, as a [MemType](https://github.com/v8/v8/blob/master/src/wasm/wasm-opcodes.h#L25)
  - ```uint8```: ```1``` if the global is exported

### Data Segments section
* ```varuint32```: The number of data segments in the section.
* For each data segment:
  - ```uint32```: The base address of the data segment in memory.
  - ```uint32```: The offset of the data segment's data in the file.
  - ```uint32```: The size of the data segment (in bytes)
  - ```uint8```: ```1``` if the segment's data should be automatically loaded into memory at module load time.

### Function Table section
This section must be preceded by a [Functions](#functions-section) section.

* ```varuint32```: The number of function table entries in the section
* For each function table entry:
  - ```uint16```: The index of the function (in the [Functions](#functions-section) section's index space)

### WLL section

* ```varuint32```: The size of the section body, in bytes
* The section body (contents currently undefined)

### End section
This indicates the end of the module's sections. Additional data can follow this section marker (for example, to store function names or data segment bodies) but it is not explicitly handled by the decoder.

# v8-native prototype format

The native prototype built for [V8](https://github.com/v8/v8/blob/master/src/wasm)
implements a binary format that embodies many of the ideas described in this document.
It is described in detail in a [public design doc](https://docs.google.com/document/d/1-G11CnMA0My20KI9D7dBR6ZCPOBCRD0oCH6SHCPFGx0/edit?usp=sharing).
