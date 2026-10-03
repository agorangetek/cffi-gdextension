# Changelog
## [0.4.0](https://github.com/agorangetek/cffi-gdextension/releases/tag/v0.4.0)
### Added
- `CFFI.get_pointer` for returning the inner pointer from Strings and Packed Arrays.
  Extremely dangerous (as with anything involving raw pointers), make sure you know what you're doing!
- Built-in C types `size_t`, `ssize_t`, `intptr_t`, `uintptr_t`, `char16_t`, `char32_t` and Godot's `real_t`.
- `CFFIPointer.address` and `CFFIPointer.element_type` properties.
- `CFFIType.alignment`, `CFFIType.name` and `CFFIType.size` properties.
- `CFFISpan`, object that provides access to a finite number of contiguous FFI elements in memory.
  Use `CFFISpan.from` to create a span from a pointer + length.
  Spans are also used when indexing struct fields of array type.
- `CFFIPointerType.from` to more easily create a pointer type from the element type
- `CFFIArrayType`: used for fixed size arrays such as `int[3]`, `float[4]` and `uint8_t[0]`.
  Zero-sized arrays are supported inside structs and apply extra padding to it if necessary.
  Array types are decayed to pointers when used as function arguments.
- `CFFIOwnedArray`, analog of `CFFIOwnedValue` but for arrays.
  Inherits from `CFFISpan` instead of `CFFIPointer`.
- GitHub Actions workflow that builds every supported platform and publishes the
  binaries as a GitHub Release whenever a `v*` tag is pushed.

### Changed
- **Breaking**: `CFFI` now inherits from `Object` instead of `CFFIScope`
  + The API provided by `CFFIScope` was replicated into `CFFI`, so this change will only break if trying to cast CFFI to CFFIScope
- **Breaking**: `CFFIType.alloc_array` and `CFFIPointer.duplicate_array` now returns `CFFIOwnedArray` instead of `CFFIOwnedValue`.
- Updated libffi from v3.5.2 to [v3.8.0](https://github.com/libffi/libffi/releases/tag/v3.8.0)
- Struct field access is much faster: reading a scalar field no longer goes
  through an array-map lookup, and the object-returning reads (struct, `[N]` and
  `[0]` fields, `get_field`) now share one code path. Measured on macOS arm64,
  best of 3 runs of 300k accesses: scalar field 306.5 ns -> 35.2 ns, `[N]` field
  489.6 ns -> 205.4 ns, `[0]` field 326.2 ns -> 205.7 ns, struct field
  275.0 ns -> 205.8 ns, `get_field` 287.7 ns -> 249.9 ns.

### Removed
- `CFFIOwnedValue.get_base_address`.
  It's not really useful, since the `CFFIOwnedValue` itself is already the `CFFIPointer` for the base address.

### Fixed
- Godot 4.7 warning: `add_singleton: RefCounted singleton 'CFFI' will be disallowed soon; raw pointer will dangle when last Ref is released. Use Object singleton.`
- `CFFIPointer::to_*_array` now copies the correct number of bytes into the resulting packed array.
- Reading a fixed-size array struct field returned a `CFFISpan` of the *array* type
  instead of its element type, so the span was `N` times too long and its element
  size, `size_bytes`, stride, `get_pointer(i)` and `to_*_array()` conversions were
  all wrong. Writes are now also kept inside the field.
- A zero-sized array field (flexible array member) could not be indexed or spanned:
  the field pointer's element type was the zero-sized array, so every offset by it
  was a no-op.
- A field declared after a flexible array member reported the type and offset of
  the flexible array member.
- `CFFISpan::to_byte_array` resized and copied the span's element count as if it
  were a byte count.
- Accessing a property on a struct-typed pointer, such as `address` or
  `element_type`, raised `Unknown field: "address"` before falling through to the
  real property. `get_field` now returns null for an unknown field as documented,
  instead of erroring.
- An argument that could not be converted to the declared parameter type crashed
  the process: the converted-argument count was never checked, so a null argument
  buffer reached `ffi_call`. It now fails with an error naming the argument and its
  declared type. Passing a `float` where `void *` is declared was a reproducible
  `SIGSEGV`.
- A `CFFISpan` is now accepted where a `T *` parameter is declared.


## [0.3.0](https://github.com/gilzoide/cffi-gdextension/releases/tag/0.3.0)
### Added
- `CFFI.memcpy` and `CFFI.memmove` for copying memory from one pointer to another
- `CFFI.memset` to fill memory from a pointer with a single byte value
- `CFFI.memcmp` and `CFFI.memequal` to compare memory from two pointers
- Support for Android devices with 16KB page size

### Fixed
- macOS universal build for x86_64

### Changed
- Updated godot-cpp to branch `godot-4.5-stable`


## [0.2.0](https://github.com/gilzoide/cffi-gdextension/releases/tag/0.2.0)
### Added
- Support for accessing global variables with `CFFILibraryHandle.get_global`
- `CFFIType.alloc_array` to allocate an array of values
- `CFFIPointer.duplicate_array` to duplicate more than one element from a pointer as an `CFFIOwnedValue`
- `CFFIPointer.get_address` method to get the base address of a pointer.
  Useful to compare pointers by value.
- Support for passing `CFFIFunction`s as function pointers to struct fields or native function arguments.
- Add `CFFICallableFunction` class that wraps `Callable` as native function pointers using libffi's closure API.
  They can be created using `CFFIScope.create_function`.
- Support for passing any packed array (other than PackedStringArray) as pointer to native functions.
- `CFFIPointer.to_*_array` methods for copying data from pointers as other types of packed arrays
- `StreamPeerCFFIPointer` stream peer that handles a binary data stream from a `CFFIPointer`.


## [0.1.0](https://github.com/gilzoide/cffi-gdextension/releases/tag/0.1.0)
### Added
- `CFFI` singleton that represents the global type scope and is the entrypoint for opening native libraries
- `CFFILibraryHandle` class representing native libraries, where native functions live
- Custom ".ffilibrary" file format, which are INI files with a `[libraries]` section that defines paths to native libraries separately per platform/architecture, using the exact [same format as ".gdextension" files]((https://docs.godotengine.org/en/stable/tutorials/scripting/gdextension/gdextension_cpp_example.html#using-the-gdextension-module)).
  They are imported as [CFFILibrary](addons/cffi/cffi_library.gd) resources.
- Editor export plugin that bundles the correct native library in builds, based on the configurations from ".ffilibrary" files
- Support for built-in types like `int` and `float`, pointer types like `const char *` and struct types
- Instantiate FFI types using `CFFIType.alloc`.
  The returned `CFFIOwnedValue` is RefCounted and releases the memory automatically whenever it gets destroyed.
- Get/set struct fields by name from `CFFIPointer`s
- Construct `String`s and `PackedByteArray`s from `CFFIPointer`s with a single method call
- Use Dictionaries as literal struct values, for example when calling a function
