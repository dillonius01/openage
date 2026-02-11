# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

openage is a free open-source game engine clone of the Genie Engine (Age of Empires, AoE II, Star Wars: Galactic Battlegrounds). It uses C++20 for the engine core (`libopenage/`) and Python 3 for scripting, asset conversion, and code generation (`openage/`). Cython provides the C++/Python glue layer. The project uses Qt6 for GUI, OpenGL for rendering, and a custom DSL called **nyan** for game data configuration.

**Current status:** Gameplay is non-functional; the engine is being rebuilt on an ECS-based architecture. Core infrastructure (rendering, event system, pathfinding, time management) is actively developed.

## Build Commands

### macOS (ARM) Setup

Homebrew's LLVM must be used (Apple Clang fails C++20 checks). Eigen 5.x is too new; use `eigen@3`. Since both are keg-only, extra flags are needed:

```bash
brew install cmake python3 libepoxy freetype fontconfig harfbuzz opus opusfile qt6 libogg libpng toml11 eigen@3 llvm flex
pip3 install cython numpy mako lz4 pillow pygments setuptools toml

CMAKE_PREFIX_PATH="$(brew --prefix eigen@3)" ./configure \
  --compiler="$(brew --prefix llvm)/bin/clang++" \
  --download-nyan \
  --flags="-I$(brew --prefix eigen@3)/include" \
  --ldflags="-L$(brew --prefix llvm)/lib/c++ -L$(brew --prefix llvm)/lib/unwind -lunwind"
```

### Common Commands

```bash
./configure --download-nyan    # Initialize build (creates bin/ symlink)
make                           # Build everything
make run                       # Launch the game
make test                      # Run all tests + fast compliance checks
make tests                     # Run only tests (C++ + Python), no compliance
bin/run test -a                # Run all tests directly
bin/run test --help            # Test runner options
```

### Compliance/Linting

```bash
make checkfast                 # Fast checks only
make checkmerge                # Pre-merge compliance (run before submitting PRs)
make checkall                  # Full compliance check
make checkpy                   # Python-only (pystyle + pylint)
make checkuncommited           # Full check on uncommitted files only
make checkchanged              # Full check on files changed since origin/master
```

### Targeted Builds

```bash
make libopenage                # Build only the C++ library
make cppgen                    # Run C++ code generation
make pxdgen                    # Generate Cython .pxd declaration files
make cythonize                 # Compile .pyx files to C++
```

### Cleanup

```bash
make clean                     # Remove build artifacts
make cleanbuilddirs            # Full clean including configure output
make mrproper                  # Above + delete converted user assets
```

## Current Engine State

Gameplay is non-functional — the engine is mid-rewrite on an ECS architecture. The `game` command (`bin/run game`) crashes during simulation startup (specifically in `EntityFactory::add_player` when creating nyan database views for players). This is expected; the game simulation path is incomplete.

### What works

- **All 31 tests** (7 Python + 24 C++): `make tests`
- **Demos** — run from the `bin/` directory (`cd bin && ./run test -d <name>`):
  - `renderer_demo` — opens a window showcasing the rendering pipeline
  - `engine_demo` — full engine demo (includes pong, presenter, interactive features)
  - `simulation_demo` — game simulation without graphics (loads only "engine" modpack)
  - `path_demo` — pathfinding system showcase
  - `renderer_stresstest` — renderer stress test
  - `curvepong` — pong game using the curve/prediction system
- **Compliance checks**: `make checkfast`, `make checkmerge`

### What doesn't work

- `bin/run game` / `make run` — crashes in the simulation thread during player creation
- The old QML GUI (`assets/qml/main.qml`) references unregistered QML types (`yay.sfttech.openage`); the presenter loads the test QML (`assets/test/qml/main.qml`) instead
- Converted game assets (AoE1/AoE2) provide sprites but don't affect stability — crashes come from incomplete game logic, not missing assets

### Running commands

- `bin/run` must be executed from the `bin/` directory (it uses `os.getcwd()` to find the generated Python package)
- `make run` handles paths automatically but doesn't support interactive stdin prompts (asset conversion)
- For interactive use: `cd bin && ./run game` (or use demo commands above)

## Architecture

### Threading Model

The presenter, simulation, and time subsystems each run in their own thread, communicating via defined interfaces. The main loop flow is:

```
renderer (window system) -> input -> event system -> simulation -> renderer -> output
```

There are no simulation ticks — everything is event-based and scheduled by time, so updates between threads are asynchronous.

### Key Subsystems (libopenage/)

- **engine/** — Main engine logic and lifecycle
- **gamestate/** — Game simulation using ECS pattern (components in `component/`, systems in `system/`, entity creation via `entity_factory/`). Activities define unit behavior state machines.
- **renderer/** — Rendering pipeline with stages, camera system, OpenGL/Vulkan backends, Qt GUI integration
- **event/** — Time-based event scheduling system (no ticks)
- **pathfinding/** — Pathfinding with flow fields
- **presenter/** — Display subsystem connecting renderer to simulation
- **input/** — Input handling (keyboard, mouse, controllers)
- **curve/** — Curve-based data interpolation (keyframes over time)
- **coord/** — Coordinate system conversions (scene, chunk, tile, pixel)
- **audio/** — Audio system with Opus codec support
- **pyinterface/** — Python/C++ integration layer

### Python Package (openage/)

- **convert/** — Asset conversion from original game formats
- **codegen/** — C++ source code generation from data definitions
- **cppinterface/** — Cython bindings to libopenage
- **testing/** — Test harness and test discovery

### Python/C++ Interface (Cython)

- C++ functions are exposed to Cython via `pxd:` annotations in header files, auto-generated into `.pxd` files by `pxdgen`
- Always declare pxd functions as `except +` unless the C++ function is `noexcept`
- C++ calls back to Python via `PyIfFunc` function pointers or `PyObj` wrappers — never raw function pointers
- All pxd interface functions, classes, and `extern` objects need the `OAAPI` macro in headers (for Windows DLL exports)
- New C++ → Python bindings must be registered in `openage/cppinterface/setup.pyx`

## Testing

### Adding Tests

All tests must be registered in `openage/testing/testlist.py`.

**C++ tests:** `void()` functions in the `openage` namespace. Use macros from `libopenage/testing/testing.h`: `TESTFAIL`, `TESTFAILMSG`, `TESTEQUALS(left, right)`, `TESTTHROWS(expr)`. Do not declare test functions in headers.

**Python tests:** Argument-less functions returning `None` or raising `openage.testing.testing.TestError`. Use `assert_value(expr, expected)` and `assert_raises(exception_type)`.

**Python doctests:** Add module name to `testlist.py`; doctests in module/function docstrings run automatically.

**Demos:** Interactive tests for development, also registered in `testlist.py`. Run individually: `bin/run test -d demo_name`.

## Code Style

### C++
- **C++20 standard**
- **Tabs for indentation, spaces for alignment** (configured in `.clang-format`)
- Formatting enforced by clang-format (SFT codestyle)
- No column limit (`ColumnLimit: 0`)
- `else`, `catch`, `while` braces on new line (`BeforeElse: true`, `BeforeCatch: true`, `BeforeWhile: true`)
- Pointer alignment: right (`int *ptr`)
- Integer types: `int`/`unsigned int` when width doesn't matter, `(u)intXX_t` when it does, `size_t` for memory management
- Namespaces are not indented

### Python
- PEP8 with 4-space indentation (stored as spaces in the repo)
- Pylint is enforced via `make checkpy`

### Commit Messages
- Format: `component: descriptive message` (e.g., `engine: fixed vomiting animation of tentacle monster`)
- Keep commits clean — squash fixup commits before merging
