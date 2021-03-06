---
tagline: single-executable app deployment
---

## What is

Bundle is a small framework for bundling together LuaJIT, Lua modules,
Lua/C modules, DynASM/Lua modules, C libraries, and other static assets
into a single fat executable. In its default configuration, it assumes
luapower's [toolchain][building] and [directory layout][get-involved] 
(read: you have to place your own code in the luapower directory) and 
it works on Windows, Linux and OSX, x86 and x64.

## Usage

	mgit bundle options...

	  -o  --output FILE                  Output executable (required)

	  -m  --modules "FILE1 ..."|--all|-- Lua (or other) modules to bundle [1]
	  -a  --alibs "LIB1 ..."|--all|--    Static libs to bundle            [2]
	  -d  --dlibs "LIB1 ..."|--          Dynamic libs to link against     [3]
	  -f  --frameworks "FRM1 ..."        Frameworks to link against (OSX) [4]
	  -b  --bin-modules "FILE1 ..."      Files to force bundling as blobs

	  -M  --main MODULE                  Module to run on start-up

	  -m32                               Compile for 32bit (OSX)
	  -z  --compress                     Compress the executable (needs UPX)
	  -w  --no-console                   Hide console (Windows)
	  -w  --no-console                   Make app bundle (OSX)
	  -i  --icon FILE.ico                Set icon (Windows)
	  -i  --icon FILE.png                Set icon (OSX; requires -w)

	  -ll --list-lua-modules             List Lua modules
	  -la --list-alibs                   List static libs (.a files)

	  -C  --clean                        Ignore the object cache

	  -v  --verbose                      Be verbose
	  -h  --help                         Show this screen

	 Passing -- clears the list of args for that option, including implicit args.

	 [1] .lua, .c and .dasl are compiled, other files are added as blobs.

	 [2] implicit static libs:           luajit
	 [3] implicit dynamic libs:
	 [4] implicit frameworks:            ApplicationServices


### Examples

	# full bundle: all Lua modules plus all static libraries
	mgit bundle -a --all -m --all -M main -o fat.exe

	# minimal bundle: two Lua modules, one static lib, one blob
	mgit bundle -a sha2 -m 'sha2 main media/bmp/bg.bmp' -M main -o lean.exe

	# luajit frontend with built-in luasocket support, no main module
	mgit bundle -a 'socket_core mime_core' -m 'socket mime ltn12 socket/*.lua' -o luajit.exe

	# run the unit tests
	mgit bundle-test


## How it works

The core of it is a slightly modifed LuaJIT frontend which adds two
additional loaders at the end of the `package.loaders` table, enabling
`require()` to load modules embedded in the executable when they are
not found externally. `ffi.load()` is also modified to return `ffi.C` if
the requested library is not found, allowing embedded C symbols to be used
instead. Assets can be loaded with `bundle.load(filename)` (see below),
subject to the same policy: load the embedded asset if the corresponding
file is not present in the filesystem.

This allows mixed deployments where some modules and assets are bundled
inside the exe and some are left outside, with no changes to the code and no
rebundling needed. External modules always take precedence over embedded ones,
allowing partial upgrades to the original executable without the need for a
rebuild. Finally, one of the modules (embedded or not) can be specified
to run instead of the usual REPL, effectively enabling single-executable
app deployment for pure Lua apps with no glue C code needed.

### Components

#### .mgit/bundle.sh

The bundler script: compiles and links modules to create a fat executable.

> The reason the script is hidden inside the .mgit dir is to allow you to
use the same command `mgit bundle` on all platforms. In particular, mgit
will drive the script using Git bash on Windows, if git is in your PATH.
You can run the script directly without mgit of course but always run it
from the root directory like this: `.mgit/bundle.sh` or move it there.

#### csrc/bundle/luajit.c

The standard LuaJIT frontend, slightly modified to run stuff from `bundle.c`.

#### csrc/bundle/bundle.c

The bundle loader (C part):

  * installs require() loaders on startup for loading embedded Lua
  and C modules
  * fills `_G.arg` with command-line args
  * sets `_G.arg[-1]` to the name of the main script (`-M` option)
  * calls `require'bundle_loader'`
    * (which means bundle_loader itself can be upgraded without a rebuild)

#### bundle_loader.lua

The bundle loader (Lua part):

  * sets `package.path` and `package.cpath` to load modules relative
  to the exe's dir
  * overrides `ffi.load` to return `ffi.C` when a library is not found
  * loads the main module, if any, per `arg[-1]`
  * falls back to LuaJIT REPL if there's no main module

#### bundle.lua

Optional module for loading embedded binary files: contains the function
`bundle.load(filename) -> string`.

## Search paths

External files are looked for relative to the executable directory,
regardless of the current directory, as follows:

  * shared library dependencies (either link-time or ffi.load-time) are
  searched for in $exedir
  * Lua modules are searched for in $exedir
  * Lua/C modules are searched for in $exedir/clib
  * static assets are searched for in $exedir


## A note on compression

Compressed executables cannot be mmapped, so they have to stay in RAM
fully and always. If the bundled assets are large and compressible,
better results can be acheived by compressing them individually or not
compressing them at all, instead of compressing the entire exe.
Compression also adds up to the exe's loading time.


## LGPL compliance

Some libraries in luapower are LGPL (check the package table on the homepage
to see which). LGPL does not normally allow static linking on closed-source
projects, but because a bundled executable will always load the dynamic
version of a bundled library if one is found in the directory of the exe,
this behavior complies with the requirement of LGPL to provide a way for
the end-user to use the app with a different version of the LGPL library.
