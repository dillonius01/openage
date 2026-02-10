# Instructions for macOS users

## Prerequisite steps
- XCode >= Xcode 12
- Install [Homebrew](http://brew.sh). If you use some other package managers, you're on your own :)

```
brew update-reset && brew update
brew install --cask font-dejavu
brew install cmake python3 libepoxy freetype fontconfig harfbuzz opus opusfile qt6 libogg libpng toml11 eigen@3
brew install llvm flex make
pip3 install --upgrade cython numpy mako lz4 pillow pygments setuptools toml
```

**Note:** Use `eigen@3` instead of `eigen`. Homebrew's default `eigen` is now version 5.x which is
incompatible with openage's `find_package(Eigen3 3.3)`. The `eigen@3` formula provides the required 3.x version.

**Note:** If you are using Homebrew's Python (rather than a version manager like pyenv or asdf),
add `--break-system-packages` to the `pip3 install` command above.

You will also need [nyan](https://github.com/SFTtech/nyan/blob/master/doc/building.md) and its dependencies.
The `flex` and `make` packages listed above cover nyan's requirements.

Optionally, for documentation generation:

```
brew install doxygen
```

## Clone the repository

```
git clone https://github.com/SFTtech/openage
cd openage
```

## Building

We advise against using the clang version that comes with macOS (Apple Clang) as it is notoriously out of date and often causes compilation errors. Use homebrew's clang if you don't want any trouble. You can pass the path of homebrew clang to the openage `configure` script which will generate the CMake files for building.

Since both `llvm` and `eigen@3` are keg-only (not symlinked into `/opt/homebrew`), you need to
tell CMake where to find Eigen and tell the linker to use homebrew's libc++:

```
CMAKE_PREFIX_PATH="$(brew --prefix eigen@3)" ./configure \
  --compiler="$(brew --prefix llvm)/bin/clang++" \
  --download-nyan \
  --flags="-I$(brew --prefix eigen@3)/include" \
  --ldflags="-L$(brew --prefix llvm)/lib/c++ -L$(brew --prefix llvm)/lib/unwind -lunwind"
```

Without the `--ldflags`, you may see linker errors about missing `std::__1::__hash_memory` when
building the nyan dependency, because homebrew clang's headers reference symbols not present in
the macOS system libc++.

Afterwards, trigger the build using `make`:

```
make -j$(sysctl -n hw.ncpu)
```

## Testing
`make test` runs the built-in tests.


## Running
`make run` or `cd bin && ./run` launches the game. Try `./run --help` if you don't know what to do!


## To create the documentation
`make doc`
For more options and details, refer to [doc/README.md][/doc/README.md]
