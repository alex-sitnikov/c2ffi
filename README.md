## Version Notice

You need to use the correct branch of `c2ffi` for your version of
LLVM/Clang:

* Anything earlier: *unsupported*
* 15.0.0: branch `llvm-15.0.0` *deprecated*
* 16.0.6: branch `llvm-16.0.0` *old*
* 17.0.6: branch `llvm-17.0.0` *current*
* 18.1.x: branch `llvm-18.1.0` *current*

Development will always take place in `llvm-X.Y`, according to the
appropriate version of LLVM.  **The *master* branch has been
removed.** Check out the appropriate version for your LLVM.

**For updates:** if you want to expedite the process, and you have a newer
version than supported, posting a ticket *with build log* after having bumped
the LLVM version in `CMakeLists.txt` would be helpful; you may not even need
changed beyond that (feel free to post an PR if this is the case).

**Windows:** This apparently builds and works in Windows now as well, with MSYS2;
make sure you have the following:

```
$ pacman -S mingw-w64-x86_64-clang mingw-w64-x86_64-clang-tools-extra gcc cmake ninja
$ mkdir build; cd build
$ cmake -G Ninja ..
```

(I couldn't get make to work; nor have I tried building this with clang.)

`c2ffi` now provides tagged versions as well.  To hopefully avoid confusion,
these correlate to the appropriate version of LLVM, along with an additional
release number for c2ffi, e.g.:

* v11.1.0.0: For LLVM `11.1.0`, c2ffi release 0
* v11.1.0.1: For LLVM `11.1.0`, c2ffi release 1

# c2ffi

This is a tool for extracting definitions from C, C++, and Objective C
headers for use with foreign function call interfaces.  For instance:

```c
#define FOO (1 << 2)

const int BAR = FOO + 10;

typedef struct my_point {
    int x;
    int y;
    int odd_value[BAR + 1];
} my_point_t;

enum some_values {
    a_value,
    another_value,
    yet_another_value
};

void do_something(my_point_t *p, int x, int y);
```

Running `c2ffi` on this, we can get the following JSON output:

```json
[
{ "tag": "const", "name": "BAR", "location": "/home/rpav/test.h:3:11", "type": { "tag": ":int" }, "value": 14 },
{ "tag": "struct", "name": "my_point", "id": 0, "location": "/home/rpav/test.h:5:16", "bit-size": 544, "bit-alignment": 32, "fields": [{ "tag": "field", "name": "x", "bit-offset": 0, "bit-size": 32, "bit-alignment": 32, "type": { "tag": ":int" } }, { "tag": "field", "name": "y", "bit-offset": 32, "bit-size": 32, "bit-alignment": 32, "type": { "tag": ":int" } }, { "tag": "field", "name": "odd_value", "bit-offset": 64, "bit-size": 480, "bit-alignment": 32, "type": { "tag": ":array", "type": { "tag": ":int" }, "size": 15 } }] },
{ "tag": "typedef", "name": "my_point_t", "location": "/home/rpav/test.h:9:3", "type": { "tag": ":struct", "name": "my_point", "id": 0 } },
{ "tag": "enum", "name": "some_values", "id": 0, "location": "/home/rpav/test.h:11:6", "fields": [{ "tag": "field", "name": "a_value", "value": 0 }, { "tag": "field", "name": "another_value", "value": 1 }, { "tag": "field", "name": "yet_another_value", "value": 2 }] },
{ "tag": "function", "name": "do_something", "location": "/home/rpav/test.h:17:6", "variadic": false, "parameters": [{ "tag": "parameter", "name": "p", "type": { "tag": ":pointer", "type": { "tag": "my_point_t" } } }, { "tag": "parameter", "name": "x", "type": { "tag": ":int" } }, { "tag": "parameter", "name": "y", "type": { "tag": ":int" } }], "return-type": { "tag": ":void" } }
]
```

Because this uses [Clang](http://clang.llvm.org/) as a parser, the C,
C++, or Objective C is fully and correctly parsed.

## Building

See [the docker files](Docker/) for the latest CI scripts.

Building a `c2ffi` branch requires the appropriate version of Clang and LLVM, which
you can [obtain from the repository](http://clang.llvm.org/get_started.html),
[by download](http://llvm.org/releases/download.html), or by using the
package manager of your OS.

If building fails, then see [Notes](#notes) below.

`c2ffi` uses cmake.  This is relatively easy to use.  However, if you
built `clang++` with special options (e.g., libc++, libc++abi,
libcxxrt, etc), see [Notes](#notes) below.

```console
c2ffi/ $ mkdir build
c2ffi/ $ cd build/
build/ $ cmake ..
  :
  : Output
  :
build/ $ make
  :
  : Output
  :
build/ $ ./bin/c2ffi
Usage: c2ffi [options ...] FILE

Options:
      -I, --include        Add a "LOCAL" include path
      -i, --sys-include    Add a <system> include path
      -D, --driver         Specify an output driver (default: json)

      -o, --output         Specify an output file (default: stdout)
      -M, --macro-file     Specify a file for macro definition output

      -N, --namespace      Specify target namespace/package/etc

      -A, --arch           Specify the target triple for LLVM
                           (default: x86_64-unknown-linux-gnu)
      -x, --lang           Specify language (c, c++, objc, objc++)

Drivers: json, sexp, null
```

Now you have a working `c2ffi`.  If not, see [Notes](#notes).

### Building on Debian and derivatives

On Debian 11 (bullseye), you may need to specify `CXX` manually:

```
sudo apt-get install llvm-11 clang-11 libclang-11-dev libclang-cpp11-dev

[...]

CXX=clang++-11 cmake ../
```

### Building on Nix(OS)

You can type `nix-shell` in the project's directory, and Nix will
download and make available everything needed for building `c2ffi` in
a freshly opened nested shell. Then you can proceed with the general
build instructions.

### Notes

* Packaged clang binaries should now work.  **But**, because these
  appear to be build with *gcc*, it is not possible to build c2ffi
  with clang!  So use gcc in this case.

* You need llvm/clang dev libraries.  `libclang.so` is not enough.  If
  you don't have `libclangAST.a`, you have to install any `-dev` type
  packages in your dist.  Not all dists may package these.

* If you build with clang and get link errors about ABI functions, you
  may need to link to `-lc++abi` or similar.

* If you build with clang and get link errors about random LLVM and
  Clang functions, you need to build with gcc, because your clang was
  built with gcc.

* Building on OSX may require specifying
  `LIBCLANG_CPPFLAGS=/usr/local/include` or wherever you installed
  LLVM.  And you will have to build LLVM, because Apple's build does
  not seem to include the appropriate headers or libraries.  *(Not
  verified recently.)*

* If you're seeing compiler errors, you probably checked out the wrong
  branch.  Verify your `clang -v` vs your `git branch`.

## Usage

There are generally two steps to using `c2ffi`:

* Generate output for a particular header or file, gathering macro
  definitions (with the `-M <file>.c` parameter)

* Generate output for macro definitions by running `c2ffi` again on
  the *generated* file (without `-M`)

This is due to the preprocessor being a huge hack (see below).
However, once this is done, you should have two files with all the
necessary data for your FFI bindings.

Currently JSON is the default output.  This is in a rather wordy
hierarchical format, with each object having a "tag" field which
describes it.  All objects are contained in an array.  This should
make it fairly easy (or at least far easier than parsing C yourself)
to transform into language-specific bindings.

This format may be documented at some point, but for now, you'll have
to look at the input and the output!  I recommend a pretty-printing
reformatter for the JSON.  Patches to produce prettier output will be
accepted. `;-)`

## Errors

You may encounter errors if the code in question is not correct.
Presumably, most of the time, you will be running c2ffi on existing,
known-working code.

In this case, the most likely "error" you will encounter will look
like this:

```
Skipping invalid Decl:
FunctionDecl 0x21e05f0 </usr/include/glib-2.0/glib/gmacros.h:328:22, /usr/include/glib-2.0/glib/deprecated/gthread.h:282:65> g_cond_timed_wait 'int (GCond *, GMutex *, GTimeVal *)' extern
|-ParmVarDecl 0x21e0480 <line:280:42, col:58> cond 'GCond *'
|-ParmVarDecl 0x21e04f0 <line:281:42, col:58> mutex 'GMutex *'
`-ParmVarDecl 0x21e0560 <line:282:42, col:58> timeval 'GTimeVal *' invalid
```

This usually means that Clang didn't find a header, and it doesn't
know about one of the types referenced.  **Look at the top of your
error output**.  Missing header errors will often appear there.

You should specify any necessary additional include paths with
`-i`(for system headers, i.e. those using `<brackets>`) or `-I` (for
local headers, i.e. those using `"quotes"`).

Generally, any issue relating to an error with C, includes, or the
like is not a bug with c2ffi.  However, c2ffi should not abort or
crash; any such error is certainly a bug with c2ffi.

## Language Support

### C

C support should be fairly complete.  Formerly variadic functions and
bitfield support was incomplete.  These should now be fully-supported.

Note however that bitfield support is platform- and sometimes
compiler-specific; if your platform ABI does not provide a strict
definition, expect the layout of structs which use bitfields to be
undefined.

### C++

C++ support should be fairly complete.  This outputs everything as it
normally would for C, as well as namespace, classes, methods, and
class hierarchy (including base class offsets).

Template support is limited to *instantiated* templates (including
both classes/structs/unions and functions).  `c2ffi` can output a new
`.hpp` file using the `-T` parameter with explicit instantiations for
those it finds declared but not instantiated.  E.g.,

```c++
template<typename T>
class C { T t; };

typedef class C<int> C_int;
```

Using `c2ffi -T file.T.hpp ...`, this will produce the following,
which `#includes` the original to produce a complete definition:

```c++
#include "original.cpp"
template class C<int>;
```

**Note:** The behavior of this *has changed*.  This used to produce a file which did not include the original.  You can now use `-D null` to output only the `.T.hpp` file, and then produce full output from that.  This simpifies the process.

### ObjC

Basic support at least exists.  I am not an Objective C person and
don't really have a great way to use or test the output, or verify
that all the useful features are included.

If you send me example source along with some information about what
would be useful, I can try to accommodate.  If you write a translator
for the JSON to an ObjC bridge, let me know and I will link it below.

### ObjC++

Untested.

### Importing

Processing the JSON into a usable format is fairly straightforward.
Some care must be given to handle anonymous types (e.g., `typedef
struct { ... } type_t;`), but writing these is fairly trivial
overall.

The following language bindings exist for `c2ffi`:

* [cl-autowrap](https://github.com/rpav/cl-autowrap/): Create bindings
  in Commonn Lisp from a `.h` with `c2ffi` using a simple `(c-include
  "file.h")`

* [c2ffi-ruby](https://github.com/rpav/c2ffi-ruby): Uses the JSON
  from c2ffi to produce a nicely-formatted Ruby file for ruby-ffi.

## New Output Drivers

If you're feeling motivated, it should be fairly simple to produce a
new output driver.  Look in `src/drivers/` and you can see the source
for JSON, Sexp (lisp symbolic expressions), and possibly some others.

You will need to do the following:

* Create a new subclass of OutputDriver in `src/drivers/`; copying one of
  the existing ones is probably the easiest.

* Add this file to `src/Makefile.am`

* Add the factory function to `src/OutputDriver.cpp`.

* Write your code!

## The Preprocessor

The preprocessor handling is, as was noted, a huge hack.  This is due
entirely to the fact that `#define` macros can contain just about
anything, and thus it's not easy to tell if they are useful values or
syntax hackery.

For this, `c2ffi` uses a simple heuristic:

* If there are arithmetic operators (`+`, `-`, `*`, `<<`, etc),
  parens, numbers, and identifiers, it's treated as "useful".

* If only ints are found, it's treated as an `__int128_t`; if floats are
  found, it's treated as a `double`; if a string is found, a `char*`

Why the odd `__int128_t`?  Because without more parsing (and
technically, without context), it can't be determined as signed or
unsigned.  So this is declared with very large capacity which will
hold the entire range of signed and unsigned 64-bit ints.

If you're dealing with unsigned 128-bit int constants, you'll have to
do it yourself.  I personally haven't seen any.

## Credits

Special thanks:

* [Jarhmander](https://github.com/Jarhmander) for version updates.
* [Simon Kissane](https://github.com/skissane) for a number of features.
* [spaceotter](https://github.com/spaceotter) for implementing real clang header paths.
* [hjudt](https://github.com/hjudt) for the llvm-17 update.

## License

This is currently GPL2, but it will almost certainly be moved to
LGPL2, as I would like to make a shared library which you can load
definitions at runtime.  It may be moved to BSD or similar at some
point in the future.
