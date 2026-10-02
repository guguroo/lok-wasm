# LOWA — LibreOffice WASM build for in-browser Office-to-PDF conversion

This branch (`lowa-fixes`) is a fork of [coparse-inc/lok-wasm](https://github.com/coparse-inc/lok-wasm)
(LibreOffice built as WebAssembly with the LibreOffice Kit), with the changes needed to convert **docx and
xlsx documents to PDF entirely inside a web browser**. It is the "LOWA" build used by
a browser-only PDF editing tool: the output (`soffice.wasm` and friends) is loaded
in a Web Worker and no document ever leaves the user's computer.

> **Unofficial.** This is not part of the official LibreOffice project and is not endorsed by The Document
> Foundation or by the upstream `lok-wasm` project. See [License](#license).

The original `lok-wasm` README is kept unchanged [below](#upstream-readme-coparse-inclok-wasm).

## Status

| Format | Result |
|---|---|
| docx → PDF | Works. Compared with Microsoft Word's own PDF output: same page count and fonts, the same gap between Japanese and Latin text, and almost all line breaks (a few differ where a line ends within a fraction of a point of the right margin). |
| xlsx → PDF | Works (multiple sheets, merged cells, number formats, print ranges, Japanese fonts). |
| pptx → PDF | **Not supported.** The runtime terminates (`std::terminate`) while importing text shapes. |
| ods → PDF | **Not supported.** The cell contents are silently missing from the PDF. |

Only the Calc and Writer import paths matter for the intended use; everything else in LibreOffice is
untouched or best-effort.

## What differs from upstream

The base is upstream `main` at commit `6d556be` (`VERSION` v0.2.3). `git diff 6d556be lowa-fixes -- . ':!README.md'`
lists every change (22 files). In short:

**Fixes for docx/xlsx → PDF**
- `desktop/source/lib/init.cxx`: documents were loaded with a hard-coded `FilterName="MS Word 2007 XML"`, so
  every file went through the Word import. The hard-coded filter is disabled so type detection runs;
  Calc is also preloaded.
- `sc/source/filter/oox/workbookfragment.cxx`: on Emscripten, sheets are imported one after another
  (the main runtime thread may not call `Atomics.wait`).
- `vcl/headless/svpinst.cxx`, `comphelper/source/misc/threadpool.cxx`: on Emscripten, the blocking waits on
  the main runtime thread are replaced by a short sleep-and-retry loop, and the shared thread pool runs tasks
  on the calling thread. Both avoid a `std::terminate`.
- `vcl/unx/generic/fontmanager/fontsubst.cxx`: CJK glyph fallback is routed to a designated Japanese font
  (default `BIZ UDPGothic`, overridable by an environment variable) instead of whatever fontconfig picks,
  which produced tofu.
- `static/CustomTarget_emscripten_fs_image.mk`: Calc's default styles (`share/calc/styles.xml`) are part of the
  virtual file system.
- `sw/source/core/text/itrform2.cxx`: the gap between Japanese and Latin text is **25%** of the font height
  (upstream: 20%), which is what Word does; with 20%, line breaks drift in lines with many Latin characters.
- `desktop/source/app/main_wasm.cxx`: sets `SAL_LOG` and `SC_NO_THREADED_CALCULATION` at start-up.
- `filter/source/config/cache/typedetection.cxx`: type-detection diagnostics, only active when the
  `LOWA_DIAG` environment variable is set.

**Build configuration** (not LibreOffice source)
- `distro-configs/CPWASM-LOKit.conf`: build Calc as well (upstream only builds Writer), `--disable-werror`.
- `config_host.mk.in`: include Impress/Draw and Canvas modules in the build (used while trying pptx).
- `scripts/build`: force a relink on every build. gbuild can miss a change to the virtual file system and
  produce a *new data + old loader* combination that does not start.
- `in-docker`: run the arguments in the container (upstream ignores them and always starts a shell).
- Compile fixes: no-op `setAuthor()` overrides in `sc/inc/docuno.hxx` and `optuno.hxx`, `(void)` casts for
  two unused variables in `sc/`.

**Leftovers from the pptx attempt** (harmless for docx/xlsx, pptx still does not work):
`sd/source/ui/annotations/annotationwindow.cxx`, `sd/source/ui/inc/unomodel.hxx`, `sd/source/ui/view/drviewsk.cxx`,
`starmath/source/edit.cxx`, `sfx2/source/view/frmload.cxx`.

## Building

You need Linux (or WSL2) with Docker; the build runs in the container defined by the `Dockerfile`
(Emscripten 3.1.73). On a 20-thread machine a clean build takes about 36 minutes and a rebuild after changing
one file about 3 minutes. Memory use was not measured. Keep the checkout on a Linux filesystem (not a
synchronised folder, not `/mnt/c`).

```bash
git clone -b lowa-fixes <this repository> lok-wasm
cd lok-wasm
./in-docker                      # opens a shell in the build container
/scripts/configure               # no argument = release build (see below)
/scripts/build                   # ends with: Generating FcCache
```

Or non-interactively (needs a terminal, e.g. inside `tmux`): `./in-docker /scripts/configure && ./in-docker /scripts/build`.

- **Use the release configuration** (`/scripts/configure` without arguments). The `dev` configuration from the
  upstream README below builds with different optimisation and produces a `soffice.wasm` about 50 MB larger.
- The parallelism is chosen by `configure` (number of logical CPUs). To lower it on a shared machine, edit
  `export PARALLELISM?=` in `libreoffice-core/config_host.mk` before building.
- After changing the configuration, run `make clean` first (in the container, in `/libreoffice-core`).
- If you copy sources in from elsewhere, **do not preserve modification times** (`rsync -t`, `cp -p`). With old
  timestamps the build does not recompile the files and links the previous objects, and still succeeds.
- Use the `soffice.mjs` and `soffice.wasm` of the same build together; the Emscripten constant table in the
  loader changes whenever the C++ `EM_ASM` blocks change.

The result is in `libreoffice-core/instdir/program/`: `soffice.wasm`, `soffice.mjs`, `soffice.data`,
`soffice.data.js.metadata`, `soffice.d.ts`, and the font and fontconfig-cache data files.

## Branches

- `main` — tracks upstream `coparse-inc/lok-wasm` unchanged.
- `lowa-fixes` — this branch: upstream plus the changes above. Do not push to `main`: the workflows inherited from
  upstream build and release from there.

## License

`lok-wasm` is licensed under the [Mozilla Public License 2.0](LICENSE). The changes on this branch are made
available under the license of the file they modify (the modified LibreOffice files are MPL 2.0 files, see their
headers). LibreOffice itself consists of files under the MPL 2.0 and other licenses (see `libreoffice-core/`); the WebAssembly output also contains third-party libraries built from
`libreoffice-core/external`. Anyone distributing the build must follow those licenses and make the source of the
modified files available — this branch is that source (`git diff 6d556be lowa-fixes`).

The changes have not been submitted upstream.

---

# Upstream README (coparse-inc/lok-wasm)

# LOK WASM

This is a public project that forks LibreOffice to provide functionality and fixes that don't fit in [the upstream project](https://github.com/LibreOffice/core).

It also provides a simple to setup environment for working on LOK as a WASM-based app. This was originally developed for [Macro](https://macro.com) and released to the public on 18 Feb 2024.

All changes are open sourced under the [Mozilla Public License 2.0](LICENSE).

This project is not a part of the official LibreOffice project, nor endorsed by the Document Foundation.

# Prerequisites

## macOS

Install [OrbStack](https://orbstack.dev/download)

Clone the repo and enter the dev environment:

```bash
git clone https://github.com/coparse-inc/lok-wasm
./in-docker
```

## Linux with Podman

Clone the repo and enter the dev environment:

```bash
git clone https://github.com/coparse-inc/lok-wasm
./in-podman
```

## Linux with Docker

Clone the repo and enter the dev environment:

```bash
git clone https://github.com/coparse-inc/lok-wasm
./in-docker
```

## Ubuntu/Debian/Pop_OS!

```bash
apt-get install -y --no-install-recommends \
  git \
  build-essential \
  zip \
  nasm \
  python3 \
  python3-dev \
  autoconf \
  gperf \
  xsltproc \
  libxml2-utils \
  bison \
  flex \
  pkg-config \
  ccache \
  openssh-server \
  cmake \
  sudo \
  locales \
  libnss3
```

Setup the repo:

```bash
git clone https://github.com/coparse-inc/lok-wasm
./scripts/setup
```

# Building

```bash
# Run configure for the initial build or any configuration changes
./scripts/configure dev
# if you're using VS Code or clangd in vim, run this
./scripts/clangd
# Run build for any code changes
./scripts/build
```

# QA Env

Make a build first, then:

```bash
./scripts/launch-qa
```

# Debugging

Make a debug build:

```bash
# Clean the existing build
(cd libreoffice-core/ && make clean)
# Run configure for debug
./scripts/configure debug
# Run build for any code changes
./scripts/build
```

If you're debugging the QA environment with `./scripts/launch-qa`, the included LOK C++ debugging extension should be included.

Otherwise use Chrome with [the C/C++ WASM debugging tools extension](https://goo.gle/wasm-debugging-extension) installed.

Use Ctrl/Cmd+P in the Dev Tools to quickly navigate to the `.cxx` file you need to debug.

You can add breakpoints as necessary in the Dev Tools, but conditional breakpoints still aren't supported.  
If you need to add a conditional breakpoint, use an if statement with a log to set a break point on inside of your code:  
```
if (myCondition == true) {
    SAL_WARN("debug", "here");
}
```


Expand the LOK logs to see the full stack trace, clicking on the `.cxx` file will jump to the source for the file.

# Docs

<!-- TIP: in neovim, you can use `gf` to go to the file linked if the cursor is between ( ) -->

- [Important Files](./important_files.md)
