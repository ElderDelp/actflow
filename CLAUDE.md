# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the ACT flow repository - a complete open-source EDA (Electronic Design Automation) flow for implementing asynchronous logic circuits. The flow spans from high-level behavioral specifications through physical implementation.

## Environment Setup

**Required Environment Variables:**
- `ACT_HOME`: Installation directory for all tools (e.g., `/usr/local/cad`, `/opt/async`, or `$HOME/async`)
- `VLSI_TOOLS_SRC`: Root of the source tree (typically set to the actflow directory)

**Optional Environment Variables:**
- `BOOST_ROOT`: Custom Boost library location if system version is outdated

## Build Commands

### Full Build
```bash
./build
```
Builds and installs all components in dependency order. Build logs are saved to `logs/` directory. Each number displayed during build represents 10 lines in the log file.

### Clean Build
```bash
./clean
```
Removes all build artifacts from all components.

### Testing
```bash
# Test the core ACT library
cd act && make runtest

# Individual component tests exist in various test/ directories
# Examples:
cd actsim/test && make
cd chp2prs/test && make
```

## Architecture Overview

### Core Foundation

**ACT Library** (`act/`): The asynchronous hardware description language and core tools. Supports multiple abstraction levels:
- CHP (Communicating Hardware Processes): Behavioral specification
- HSE (Hardware State Expressions): State-based specification
- PRS (Production Rules): Gate-level asynchronous circuits
- Analog circuit descriptions

**Galois Library** (`Galois/`): Parallel programming framework providing implicit parallelism abstractions. Used extensively throughout the flow for performance-critical algorithms.

### Synthesis Tools

**chp2prs** (`chp2prs/`): Core synthesis engine translating high-level CHP specifications to PRS circuits. Supports multiple synthesis approaches:
- Syntax-Directed Translation (SDT)
- Ring-based synthesis
- Dataflow synthesis

**expropt** (`expropt/`): Boolean expression optimizer integrating external synthesis tools (abc, yosys). Handles datapath optimization with delay/power/area metadata.

**dflowmap** (`dflowmap/`): Maps ACT dataflow programs to circuit implementations with performance metrics.

### Physical Design Tools

**BiPart** (`BiPart/`): Partitioning and placement for regular structures using Galois for parallelization.

**Dali** (`Dali/`): Gridded cell placement engine with OpenMP parallelization.

**PWRoute** (`PWRoute/`): Power distribution network routing for gridded layouts.

**SPRoute** (`SPRoute/`): Scalable parallel global router using Galois library. Highly optimized for large-scale designs.

**TritonRoute-WXL** (`TritonRoute-WXL/`): Detailed router based on open-source TritonRoute.

### Analysis & Simulation

**actsim** (`actsim/`): Mixed-signal simulator supporting CHP, HSE, PRS, and analog abstraction levels. Integrates with Xyce for analog simulation.

**xcell** (`xcell/`): Cell library characterizer generating Liberty (.lib) files from ACT cell definitions using SPICE simulators.

### Support Libraries

**phyDB** (`phyDB/`): Unified physical design database shared by placement and routing tools.

**lefdef** (`lefdef/`): Si2 standard parser for LEF/DEF library and design exchange formats.

**annotate** (`annotate/`): Back-annotation readers for SPEF (parasitics) and SDF (timing).

**stdlib** (`stdlib/`): ACT standard library with data types, channels, and functions.

**sky130l** (`sky130l/`): Skywater 130nm technology configuration using lambda-based rules (lambda = 75nm).

### Utilities

**utils** (`utils/`): Contains lib2act.py for converting Liberty files to ACT black-box definitions.

**interact** (`interact/`): Interactive flow management using mini-scheme scripting.

**dflow2dot** (`dflow2dot/`): Visualization tool converting ACT dataflow graphs to GraphViz format.

**fpga_proto** (`fpga_proto/`), **prs2fpga** (`prs2fpga/`): FPGA prototyping support.

## Build System Architecture

The repository uses a hierarchical build system:
1. ACT core library is built first using configure + make
2. CMake-based components (Galois, phyDB, BiPart, Dali, PWRoute, SPRoute, TritonRoute-WXL, dflowmap) use out-of-source builds in `build/` directories
3. Makefile-based tools (chp2prs, actsim, expropt, etc.) use in-source builds
4. All components install to `$ACT_HOME`

Dependencies are managed through environment variables (LEF_ROOT, DEF_ROOT, GALOIS_INCLUDE, GALOIS_LIB) and pkg-config.

## Key Design Patterns

**Parallel-First Design**: Heavy use of Galois library for irregular parallelism in EDA algorithms. Tools like SPRoute, BiPart, and Dali leverage parallel execution for performance.

**Multi-Abstraction Support**: Tools operate at different abstraction levels, allowing optimization and analysis at appropriate granularity.

**Standard Format Integration**: Uses industry-standard formats (LEF/DEF, Liberty, SDF, SPEF) for interoperability with commercial EDA tools.

**Modular Component Design**: Each tool is self-contained with clear interfaces. Components can be used standalone or as part of the complete flow.

## Common Workflows

### Adding/Modifying a Synthesis Tool
1. Changes to synthesis likely affect `chp2prs/` for CHP-to-PRS conversion
2. Expression optimization changes go in `expropt/`
3. Rebuild affected component: `cd component && make clean && make && make install`
4. Test with examples in `test/` directories

### Modifying Physical Design
1. Placement changes: `Dali/` or `BiPart/`
2. Routing changes: `SPRoute/` (global) or `TritonRoute-WXL/` (detailed)
3. Power routing: `PWRoute/`
4. Common data structure: `phyDB/`
5. Rebuild: `cd component && rm -rf build && mkdir build && cd build && cmake .. && make && make install`

### Working with Technology Files
1. Technology configurations in `sky130l/` or similar
2. Cell libraries use ACT format or Liberty format (convertible via `utils/lib2act.py`)
3. Standard library in `stdlib/` provides common definitions

## Important Notes

- Build logs are in `logs/` directory with format `<component>.log` and `<component>.err`
- The build script applies patches from `extra/` to Galois before building
- Some components (timing libraries, dflowmap_netlist, expropt_commercial) are optional and only built if present
- C++ compiler is auto-detected by `check_omp.sh` and `check_filesystem.sh` scripts
- Parallel builds use `-j 4` by default

## Local Clone Setup

This is the user's local working clone. Conventions specific to it:

- **Remotes (parent + every submodule)**: `origin` points at the user's `ElderDelp/<name>` fork on GitHub; `upstream` fetches from the canonical source (asyncvlsi for actflow and 21 submodules, IntelligentSoftwareSystems for Galois) but its **push URL is redirected to ElderDelp** as a safety net. All pushes — including from inside submodules — must land in ElderDelp; never push to asyncvlsi/Intel.
- **`.gitmodules` is intentionally unchanged**: it still records canonical asyncvlsi/Intel URLs so a fresh clone bootstraps from upstream. The ElderDelp remote routing lives only in per-clone `.git/config` and is not committed. Do not try to "fix" `.gitmodules` to point at ElderDelp.
- **Local `build` script modification** (committed as `c871fb3`): the `patched` sentinel guard is disabled and `patch -f` is passed to `patch`, so Galois patches are reattempted on every build. Already-applied hunks fail and produce `.rej` files in `Galois/` — these are **expected and harmless**, not a build failure. Do not try to clean them up as part of routine work.
- **Untracked working-tree items** (`prs2fpga/`, `timing/`, `.act_history`, `CLAUDE.md`, etc.) are local user state. Don't add them to commits or remove them.

## Dependencies

**System Requirements:**
- C++17 compliant compiler (gcc >= 7, clang >= 7.0, Intel >= 19.0.1)
- CMake >= 3.16
- Boost >= 1.71.0
- libfmt >= 4.0
- libedit (libedit-devel/libeditline-dev)
- zlib
- m4 macro processor
- Optional: yosys for enhanced expression optimization

**External Tools Integration:**
- abc (Berkeley logic synthesis) - bundled in `expropt/abc/` and `expropt/abc2/`
- yosys (optional) - for logic synthesis
- Xyce or hspice - for analog simulation in actsim/xcell

## References

If using this flow for publications, cite:
Samira Ataei, Wenmian Hua, Yihang Yang, Rajit Manohar, Yi-Shan Lu, Jiayuan He, Sepideh Maleki, Keshav Pingali, "An Open-Source EDA Flow for Asynchronous Logic," IEEE Design & Test, vol. 38, no. 2, pp. 27-37, April 2021.

Documentation: https://avlsi.csl.yale.edu/act/
