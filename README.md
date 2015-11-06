# wasm-emscripten

This repository contains tools to compile C/C++ to WebAssembly s-expressions, using [Emscripten](http://emscripten.org/), by translating Emscripten's asm.js output into WebAssembly, as well as a WebAssembly interpreter that can run the translated code.

More specifically, this repository contains:

 * **asm2wasm**: An asm.js-to-WebAssembly compiler, built on Emscripten's asm optimizer infrastructure. That can directly compile asm.js to WebAssembly. You can use Emscripten to build C++ into asm.js, and together the two tools let you compile C/C++ to WebAssembly.
 * **wasm.js**: A polyfill for WebAssembly support in browsers. It receives an asm.js module, parses it using `asm2wasm`, and runs the resulting WebAssembly in a WebAssembly interpreter. It provides what looks like an asm.js module, while running WebAssembly inside.
 * **wasm-shell**: A WebAssembly interpreter that can parse S-Expression format and run the spec tests.

## Building asm2wasm

```
$ ./build.sh
```

* `asm2wasm` and `wasm-shell` require a C++11 compiler. If you also want to compile C/C++ to asm.js and then to WebAssembly (and not just asm.js to WebAssembly), you'll need Emscripten (the [stable SDK (or normal manual install)](http://kripken.github.io/emscripten-site/docs/getting_started/downloads.html) is fine).
* `wasm.js` requires Emscripten, using the `incoming` branch in the `emscripten`, `emscripten-fastcomp` and `emscripten-fastcomp-clang` repos (the stable SDK, which is enough for `asm2wasm`, is not enough for `wasm.js`).
* Older versions of gcc hit [this bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=51048). You probably need gcc 5.0 or later, or a recent version of clang. If you have Emscripten, you can just use the native clang++ from that, it is recent enough.

## Running

### asm2wasm

Just run

```
bin/asm2wasm [input.asm.js file]
```

This will print out a WebAssembly module in s-expression format to the console.

For example, try

```
$ bin/asm2wasm test/hello_world.asm.js
```

That input file contains

```javascript
  function add(x, y) {
    x = x | 0;
    y = y | 0;
    return x + y | 0;
  }
```

You should see something like this:

![example output](https://raw.github.com/WebAssembly/wasm-emscripten/master/media/example.png)

On Linux and Mac you should see pretty colors as in that image. Set `COLORS=0` in the env to disable colors if you prefer that. Set `COLORS=1` in the env to force colors (useful when piping to `more`, for example).

Set `ASM2WASM_DEBUG=1` in the env to see debug info, about asm.js functions as they are parsed, etc. `2` will show even more info.

### wasm.js

Run

```
./emcc_to_wasm.js.sh [filename.c ; whatever other emcc flags you want]
```

That will call `emcc` and then emit `a.normal.js`, a normal asm.js build for comparison purposes, and `a.wasm.js`, which contains the entire polyfill (`asm2wasm` translator + `wasm.js` interpreter).

### C/C++ Source => asm2wasm

Using emcc you can generate asm.js files for direct parsing by `asm2wasm` on the commandline, for example using

```
emcc src.cpp -o a.html --separate-asm
```

That will emit `a.html`, `a.js`, and `a.asm.js`. That last file is the asm.js module, which you can pass into `asm2wasm`.

For basic tests, that command should work, but in general you need a few more arguments to emcc, see emcc's usage in `emcc_to_wasm.js.sh`, specifically

 * `ALIASING_FUNCTION_POINTERS=0` because WebAssembly does not allow aliased function pointers (there is a single table).
 * `GLOBAL_BASE=1000` because WebAssembly lacks global variables, so `asm2wasm` maps them onto addresses in memory. This requires that you have some reserved space for those variables. With that argument, we reserve the area up to `1000`.

### wasm-shell

Run

````
bin/wasm-shell file.wast [print AST before running]
````

 * Setting `WASM_SHELL_DEBUG=1` in the env will emit a lot of debugging info.

## Testing

```
./check.py
```

(or `python check.py`) will run `asm2wasm` and `wasm.js` on the testcases in `test/`, and verify their outputs.

The `check.py` script supports some options:

```
./check.py [--interpreter=/path/to/interpreter] [TEST1] [TEST2]..
```

 * If an interpreter is provided, we run the output through it, checking for parse errors.
 * If tests are provided, we run exactly those. If none are provided, we run them all.
 * `asm2wasm` tests require no dependencies. `wasm.js` tests require `emcc` and `nodejs` in the path.

## FAQ

 * How does this relate to the new WebAssembly backend which is being developed in upstream LLVM?
  * This is separate from that. This project focuses on compiling asm.js to WebAssembly, as emitted by Emscripten's asm.js backend. This is useful because while in the long term Emscripten hopes to use the new WebAssembly backend, the `asm2wasm` route is a very quick and easy way to generate WebAssembly output. It will also be useful for benchmarking the new backend as it progresses.

## License & Contributing

Same as Emscripten: MIT license.

(parts of `src/` are synced with `tools/optimizer/` in the main emscripten repo, for convenience)

## TODO

 * Waiting for switch to stablize on the spec repo; switches are Nop'ed.
 * Reference interpreter lacks module importing support; imports are Nop'ed in native builds, but enabled in emcc builds (so wasm.js works).
 * Memory section needs the right size.

