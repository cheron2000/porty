# Repository Context: cheron2000/ccextractor

> Reviewed on 2026-11-24 from the public GitHub repository metadata, full recursive tree, README, compilation/migration docs, representative C and Rust source, test harness, Cargo manifests, and selected CI workflows. The tree was complete (1,129 entries; not truncated). This was a source review only: no build, test, fuzz, or runtime command was executed.

## 1. Executive summary

`cheron2000/ccextractor` is a large C/C++-ecosystem project repository for **CCExtractor**, a command-line tool that extracts closed captions/subtitles from broadcast recordings and video containers, and converts them into subtitle formats. The README describes real-time extraction, translation, DVD caption extraction, and conversion of closed captions to subtitles.

The repository contains a mature cross-platform build and packaging system, a substantial C implementation, an ongoing Rust migration, OCR/hard-subtitle support, a sample-based C test harness, Rust unit-test workflows, and release automation.

**Repository scale observed:** GitHub reports 151,143 KB; the recursive tree contains 1,129 entries. The reviewed source inventory includes 72 C files and 59 header files under `src/lib_ccx/`, plus 85 Rust source/manifest files under `src/rust/`.

**Important ownership note:** the README identifies `CCExtractor/ccextractor` as the official upstream repository. This repository is under the personal `cheron2000` namespace. Treat it as a fork or mirror until its GitHub fork metadata and branch relationship are checked. Do not assume it is the canonical upstream or that all upstream activity is reflected here.

## 2. Verified repository metadata

| Field | Observed value |
|---|---|
| Repository | `cheron2000/ccextractor` |
| Visibility | Public |
| Default branch | `master` |
| GitHub size | 151,143 KB |
| Archived | No |
| Recursive tree | 1,129 entries; complete |
| Primary languages | C, C++, Rust, build/configuration scripting |
| Main build systems | CMake, Autotools, platform-specific scripts/projects |
| CI | GitHub Actions, plus legacy Travis configuration |
| License declared in README | GPL-2.0 |

The README points to the official CCExtractor project site and says the official repository is `CCExtractor/ccextractor`. The codebase includes substantial bundled third-party code and binary assets.

## 3. Product purpose and capabilities

CCExtractor processes video/broadcast inputs and extracts subtitle/caption data. The README lists:
- Real-time subtitle extraction.
- Translation support.
- Closed-caption extraction from DVDs.
- Conversion of closed captions into subtitle files.

The code and documentation show a wide input/demuxing surface, including transport/program streams, elementary streams, ASF/WTV, GXF/MXF, MP4, Matroska/WebM, MythTV, raw caption formats, and optional FFmpeg integration. Decoder/output support includes legacy CEA-608, CEA-708/DTVCC, teletext, DVB/DVD subtitles, and multiple output encoders.

Documented output and processing features include SRT, WebVTT, transcript and other subtitle-oriented formats, optional OCR, and hardsubx (burned-in subtitle extraction) using OCR/FFmpeg-related dependencies.

The README documents the `--timestamp-map` option for WebVTT HLS workflows; it is disabled by default for compatibility with ordinary WebVTT players.

## 4. High-level architecture

```text
CLI / platform entry point
        |
        v
Option parsing and configuration
        |
        v
Input source and stream detection
        |
        v
Demuxers / stream readers
        |
        v
Caption and subtitle decoders
        |
        v
Timing, sequencing, and shared context
        |
        v
Subtitle encoders / output writers
        |
        +--> SRT / WebVTT / SCC / transcript / other formats

Optional paths:
  OCR + image processing + FFmpeg -> burned-in subtitle extraction
  Rust modules <-> C ABI / FFI       -> migrated decoder, demux, time, net, utility logic
```

### 4.1 Executable and orchestration

`src/ccextractor.c` is the main C application entry point and contains the top-level extraction lifecycle. The inspected `start_ccx()` flow initializes library contexts, handles optional hardsubx, configures stream timing behavior, iterates input files, selects stream-specific processing, and manages process signals on non-Windows platforms.

### 4.2 Core library

`src/lib_ccx/` contains the core C library. The inspected modules show:
- `lib_ccx.c`: shared context and decoder configuration initialization.
- `ccx_demuxer.c`: demuxer open/reset/close behavior, stream detection, and C-to-Rust dispatch when Rust is enabled.
- `ccx_decoders_608.c`: CEA-608 decoding.
- `ccx_decoders_708.c`: CEA-708/DTVCC decoding integration.
- `output.c` and encoder modules: output writing and subtitle formatting.
- `ocr.c`: Tesseract/Leptonica-backed OCR support when enabled.

The repository has a broad modular C source tree, but it is still a tightly integrated native application with shared global/configuration state and multiple build-time feature combinations.

### 4.3 Rust migration

The repository is migrating selected functionality from C to Rust while retaining C compatibility.

The Rust workspace-like layout contains:
- `src/rust/src/`: main Rust static library and application-facing modules.
- `src/rust/lib_ccxr/`: an idiomatic Rust library crate.
- `src/rust/src/libccxr_exports/`: C-compatible exports.
- `src/rust/wrapper.h` and build-time bindgen: C header bindings.
- Rust modules for CEA-708 decoding, demuxing, AVC, elementary streams, encoders, hardsubx, timing, file functions, networking, and utilities.

`src/rust/src/lib.rs` documents the Rust library and exposes modules such as `decoder`, `demuxer`, `encoder`, `avc`, `es`, `parser`, and `libccxr_exports`. It includes generated bindings and test-only mocks for C globals and external functions.

The inspected `ccx_demuxer.c` uses `#ifndef DISABLE_RUST` to route demuxer operations through Rust FFI functions; the C fallback remains present. This dual implementation is important for migration and regression review.

The migration guide describes a pattern: implement idiomatic Rust logic, expose a C ABI wrapper with a `ccxr_` prefix, declare it in C, and conditionally dispatch from the legacy C function.

## 5. Build systems and platform support

The project supports several build routes:

- **CMake:** `src/CMakeLists.txt`, with options including `WITH_FFMPEG`, `WITH_OCR`, and `WITH_HARDSUBX`.
- **Autotools:** `linux/configure.ac` and platform-specific `linux/` and `mac/` scripts.
- **Windows:** Visual Studio solution/project files and MSBuild workflows, with vcpkg, GPAC, Rust, and optional FFmpeg/Tesseract dependencies.
- **Docker:** Docker build and usage documentation.
- **Packaging:** Debian, RPM, Arch, AppImage, Snap, Chocolatey, WinGet, Homebrew, and release artifacts.

The compilation guide documents Linux, macOS, and Windows build paths. It lists native dependencies such as GPAC, CMake/GCC, Tesseract/Leptonica for OCR, FFmpeg for hardsubx, and Rust for the Rust-enabled builds.

The Autotools `configure.ac` currently specifies a Rust minimum supported version of 1.87.0, while the compilation guide's prose says Rust 1.54 or above. This is a documentation inconsistency: the effective minimum for that Autotools configuration is the stricter 1.87.0 check. Verify the CMake/Windows paths separately rather than assuming all build routes enforce identical toolchain versions.

The README says the Linux Autotools path is run from `linux/` and can build Rust components when Cargo and rustc are available. The documentation also provides CMake and platform script alternatives.

## 6. Test and CI landscape

### 6.1 C test harness

The `tests/` directory contains a Check-library-based C test harness. Its README instructs contributors to run:

```bash
cd tests
make
```

The current `tests/runtest.c` registers `ccx_encoders_splitbysentence_suite`. The Makefile links against existing Linux build objects and libcheck. The visible test suite exercises sentence-buffer splitting/append behavior.

**Coverage limitation:** the inspected harness registers one suite. This does not establish that all decoders, demuxers, encoders, OCR, and format paths have comprehensive unit coverage. The full test inventory and sample-platform tests should be reviewed before making coverage claims.

### 6.2 Rust tests

The Rust CI workflow `.github/workflows/test_rust.yml` runs:
- `cargo test` in `src/rust`.
- `cargo test` in `src/rust/lib_ccxr`.

The main Rust crate contains `#[cfg(test)]` mock bindings and state for unit tests. The exact number and effectiveness of tests were not measured in this review.

### 6.3 Formatting and linting

`.github/workflows/format.yml` runs:
- clang-format checks for selected C source files.
- `cargo fmt --all -- --check` for both Rust work directories.
- `cargo clippy -- -D warnings` for both Rust work directories.

### 6.4 Build and release automation

The tree contains 15 GitHub workflow files. Inspected workflows include:
- Linux build paths for the shell build, Autotools, CMake, OCR/hardsubx, and Rust.
- Windows matrix builds for x64 and x86 using MSBuild, vcpkg, GPAC, and Rust targets.
- Release automation for Windows installers/artifacts.
- Additional workflows for macOS, Debian packages, Docker, AppImage, Snap, Chocolatey, WinGet, Homebrew, and release publishing.

The Linux build workflow includes several independent build configurations. This is valuable because feature flags and optional system dependencies create a large build matrix.

**CI status limitation:** workflow definitions were inspected, not executed. No claim is made that current CI is green or that every matrix job passes at the reviewed commit.

## 7. Repository organization and maintainability observations

Strengths:
- Mature separation between main CLI orchestration, library functions, demuxers, decoders, encoders, and optional OCR paths.
- Build documentation covers multiple operating systems and feature combinations.
- Multiple CI workflows validate native builds and Rust code.
- Contributor guidance and migration documentation exist.
- Rust migration uses a defined FFI boundary rather than requiring an all-at-once rewrite.
- Release and package workflows support multiple distribution channels.

Costs and complexity:
- The repository bundles substantial third-party code and binary assets, contributing to its large footprint.
- Multiple build systems and platform-specific scripts increase maintenance and parity risk.
- The C/Rust dual implementation creates a period where behavior can diverge between implementations.
- Native dependencies (GPAC, FFmpeg, Tesseract, Leptonica, libcurl, clang/bindgen, platform SDKs) make clean builds nontrivial.
- Shared C structs, globals, ownership, and FFI pointers make migration correctness and lifetime management especially important.
- Some documentation is stale or inconsistent, as seen in the Rust minimum-version discrepancy and the README's general language summary versus the current C/Rust implementation.

## 8. Technical risks and review priorities

These are review priorities inferred from architecture and inspected files; they are not assertions of confirmed exploitable defects.

### P0 — C/Rust FFI safety and parity
- Audit every `extern "C"` export for nullability, pointer validity, ownership, lifetime, and panic containment.
- Document which side allocates and frees each object.
- Add tests for repeated init/free, null pointers, partial initialization, and error paths.
- Keep C and Rust fallback behavior equivalent until the C implementation is retired.
- Use differential tests on the same caption samples to compare C and Rust outputs.

The inspected Rust CEA-708 FFI functions document pointer preconditions and opaque context ownership. Those contracts should be checked against every C caller and all early-return paths.

### P0 — media parser robustness
- Treat all media/container/caption input as untrusted.
- Add malformed/truncated input tests, fuzzing, and sanitizer builds around demuxers and decoders.
- Check integer overflow, packet length bounds, timestamp rollover, allocation failures, and cleanup on parse errors.
- Maintain regression samples for every parser/decoder bug.

### P1 — build reproducibility
- Align minimum Rust version across CMake, Autotools, Windows, and documentation.
- Pin or record external toolchain/dependency versions where compatibility requires it.
- Validate optional-feature combinations: OCR on/off, FFmpeg on/off, hardsubx on/off, Rust enabled/disabled.
- Prefer clean out-of-tree builds and ensure generated artifacts do not dirty tracked source directories.

### P1 — test coverage and fixtures
- Expand the visible C unit-test registration beyond the sentence-buffer suite.
- Maintain a curated sample corpus for CEA-608, CEA-708, teletext, DVB/DVD, containers, timestamp edge cases, and OCR.
- Run Rust unit tests for both crates and add integration tests through the C ABI.
- Track expected subtitle output with golden-file regression tests.
- Add sanitizer and fuzz jobs for high-risk parsers.

### P2 — modularity and documentation
- Keep C and Rust migration boundaries explicit and documented by subsystem.
- Remove stale or contradictory build instructions.
- Clarify which build system is the recommended canonical path for each OS.
- Document optional feature availability and dependencies in a single maintained matrix.
- Keep the C/Rust migration guide updated with the actual migration status and supported fallback behavior.

### P2 — release engineering
- Verify all release artifacts are reproducible from tagged source.
- Review external action versions, token permissions, artifact retention, and release upload permissions.
- Ensure package metadata, checksums, and bundled OCR language data match release versioning.

## 9. Practical contributor workflow

For a focused change:
1. Confirm the issue and current upstream status in the official CCExtractor repository before duplicating work in a personal fork.
2. Rebase/update the relevant branch from the correct upstream base.
3. Identify whether the target code is still C, already Rust-backed, or conditionally compiled through the FFI bridge.
4. Add a minimal regression fixture or unit test before changing behavior.
5. Build the narrowest relevant configuration first, then the broader platform/feature matrix as appropriate.
6. Run the relevant C tests, Rust tests, formatting, and static checks.
7. Include exact compiler/toolchain/dependency versions and commands in the PR description.
8. Avoid unrelated formatting or generated-file changes in a focused bug fix.

The contributor guide asks contributors to check existing issues/PRs, keep submissions viable, merge current `master` into their branch, use Unix line endings, and test changes.

## 10. Suggested high-value contribution areas

Subject to checking current upstream issues and open PRs:
- CEA-708 Rust migration parity and regression tests.
- C/Rust FFI ownership and error-path tests.
- Parser robustness for malformed or truncated media inputs.
- Cross-platform build reproducibility and documentation fixes.
- Sample-based regression tests for timestamp, stream switching, and subtitle output edge cases.
- OCR/hardsubx dependency detection and platform-specific test coverage.

A good first contribution is a small, reproducible bug fix with a regression fixture. For larger migration work, keep changes narrowly scoped to one decoder/demuxer subsystem and provide differential output evidence.

## 11. Validation status and limitations

- Repository metadata: inspected.
- Recursive tree: inspected; 1,129 entries, complete and not truncated.
- README: inspected.
- Compilation guide and Rust migration guide: inspected.
- Representative C entry point, library initialization, demuxer, decoder, OCR source: inspected.
- Rust Cargo manifests, module layout, CEA-708 decoder, and FFI exports: inspected.
- C test harness and sample suite: inspected.
- Selected Linux/Windows build, Rust test, format, and release workflows: inspected.
- Full source tree: not line-by-line reviewed.
- Third-party source: not audited.
- Binary/media assets: not inspected.
- Build, tests, fuzzing, sanitizers, and runtime behavior: not run.
- Current CI status and upstream synchronization: not verified.

## 12. Coding-agent handoff

Treat this as a mature native media-processing codebase, not a greenfield project. Before editing, establish the correct upstream base and identify the exact subsystem and whether its active implementation is C or Rust. Add a regression fixture, preserve C/Rust behavior parity, and validate the relevant feature/build matrix.

For memory-safety-sensitive work, prioritize FFI ownership contracts, malformed-input handling, integer/length bounds, and cleanup paths. Do not make broad refactors across C and Rust simultaneously without a staged migration plan and differential tests.
