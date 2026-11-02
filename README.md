# HTTP2Shell / Node2Shell exposure check (CVE-2026-111497)

**Is your service exposed to HTTP2Shell?** It's a remote code execution bug in Google's Brotli decoder, under active attack. It affects Node.js, Bun, Electron, Python, .NET, Java, curl and OS `libbrotli` packages.

- **Check by hand:** [read the guide](https://mattfields.dev/http2shell-checker/) for the command and fixed version for each stack. Every version it gives is sourced in the [version references](https://mattfields.dev/http2shell-checker/references.html).
- **Check automatically:** run the read-only script in [Quickstart](#quickstart).

## Contents

- [Safety](#safety)
- [Quickstart](#quickstart)
- [Usage](#usage)
- [What it checks](#what-it-checks)
- [Verification](#verification)
- [Sources](#sources)
- [Contribution](#contribution)
- [License](#license)

## Safety

- **Read-only:** it reads files and changes nothing on your system.
- **No network:** container probes run with `--network=none` and never pull images.
- **Runs only version commands,** such as `node --version` for tools on your PATH. It never runs anything it finds under the scan directories.
- **Needs bash 3.2 or later** on macOS or Linux. No root needed.

## Quickstart

```bash
git clone https://github.com/fieldse/brotli-http2shell-checker.git
cd brotli-http2shell-checker
less scripts/vuln-check-CVE2026-111497-brotli.sh              # review before running
./scripts/vuln-check-CVE2026-111497-brotli.sh -r ~/src -q    # scan a source tree, report problems only
./scripts/vuln-check-CVE2026-111497-brotli.sh -x ~/private   # default roots, skipping one directory
./scripts/vuln-check-CVE2026-111497-brotli.sh --help         # all options
```

## Usage

### Reading the results

Each output line reads `LEVEL [NN] message`, where `NN` is the [check](#what-it-checks) that produced it.

- `FAIL`: a known-vulnerable version. Upgrade it.
- `WARN`: needs a manual look, such as an unknown version, a distro build, or a Brotli-decoding framework on an unpatched runtime.
- `PASS`, `INFO`, `SKIP`: fixed, information only, or not checked.

Exit codes:

- `0`: no `FAIL`
- `1`: any `FAIL`, or any `WARN` with `--strict`. Use this to gate CI.
- `2`: usage or environment error

See [how the script is tested](https://mattfields.dev/http2shell-checker/tests.html).

### What it scans by default

With no `-r`, the script scans:

- `$HOME`
- the Homebrew prefix (`/opt/homebrew` or `/usr/local`)
- `/Applications`, on macOS

Homebrew formulae and app bundles carry their own Node.js, Electron and Python packages, so a `$HOME`-only scan misses them.

### Options

- `-r DIR`: scan this directory instead of the defaults. Repeatable.
- `-x DIR`: never enter this directory. Repeatable.
- `-q`: print only `FAIL` and `WARN` lines, plus the summary.
- `-s`, `--strict`: exit 1 on `WARN` as well as `FAIL`.
- `-c`: also check local Docker or Podman images.
  - Inside an image it checks only the OS packages, the `node` and `bun` on its PATH, and the system Python's packages.
  - It skips virtualenvs, `node_modules`, version-manager installs, .NET runtimes, jars, bundled `libbrotlidec` copies and vendored source.
- `-i IMAGE`: check only this image. Implies `-c`.
- `-e ENGINE`: `docker` or `podman` for `-c`. Default: Docker if it responds, else Podman.
- `-d`: also search every binary inside `.app` bundles. Slow: minutes on a full `/Applications`.
- `--help`: every option, with long forms.

### Excluding a directory (`-x`)

- A symlinked directory also excludes its target.
- Tools installed inside it, such as `node`, `bun`, `brew`, `dotnet` and `curl`, show as `SKIP` and are not run.
- Python `sys.path` entries inside it are skipped.
- Matching ignores case on macOS.
- Excluding `/` is a usage error.

## What it checks

The check numbers are stable.

| No. | Check |
|---|---|
| 00 | Header |
| 01 | OS packages (dpkg, apk, rpm, pacman, Homebrew) |
| 02 | `node` and `bun` on PATH |
| 03 | Node.js installs from nvm, volta, asdf, fnm |
| 04 | .NET runtimes |
| 05 | Python dist-info and egg-info metadata |
| 06 | Python compiled extensions and Perl `IO::Compress::Brotli` |
| 07 | Python pins in requirements and constraints files, pyproject.toml, Pipfile.lock, poetry.lock, uv.lock, pdm.lock |
| 08 | Host Python packages |
| 09 | npm middleware that decodes Brotli |
| 10 | Java brotli4j |
| 11 | Ruby brotli gem |
| 12 | Go modules |
| 13 | Rust crates |
| 14 | Vendored decoder sources (`decode.c`) |
| 15 | Bundled `libbrotlidec` copies |
| 16 | curl |
| 17 | Envoy, Istio, nginx |
| 18 | Container engine preflight and images (`-c`, `-i`) |
| 19 | Other `node` binaries bundled by apps and tools, judged without running them |
| 20 | Electron apps (see [Electron](#electron)) |
| 21 | Executables and libraries inside `.app` bundles (`-d`) |

`--help` explains how each check decides between `FAIL` and `WARN`.

### Electron

Every Electron release from 42.0.0 on is fixed. Releases below 42 get `FAIL`, and Electron 46 prereleases get `WARN` until 46.0.0 ships. Electron uses Chromium's copy of Brotli, and Chromium 148, first shipped in Electron 42, carries the fix. Sources are in the [version references](https://mattfields.dev/http2shell-checker/references.html#g-electron).

<details>
<summary>How the Electron cut-off was confirmed</summary>

- **Chromium's copy decides.** Electron builds its Node.js against Chromium's Brotli (`//third_party/brotli:dec`), not Node's `deps/brotli` ([`patches/node/build_add_gn_build_files.patch`](https://github.com/electron/electron/blob/42-x-y/patches/node/build_add_gn_build_files.patch)). Electron's later Node 24.22.1 bumps don't change the decoder.
- **Fix commit.** The fix is Brotli commit [0ed4268a](https://github.com/google/brotli/commit/0ed4268a4401abcb4472096340477cf4af7d1a2b), cherry-picked into 1.0.10, 1.1.1 and 1.2.1.
- **Chromium 148.** Chromium rolled Brotli to `cfa184a` in [224140d410](https://github.com/chromium/chromium/commit/224140d410af003ca0266a3328accaabd24329f9) (`main@{#1603247}`, 2026-03-23). `cfa184a` [contains 0ed4268a](https://github.com/google/brotli/compare/0ed4268a44...cfa184a997ac19454d5a3ae97f6d600aac2445a6). Chromium 148 branched at `#1610480` ([Chromium Dash milestones](https://chromiumdash.appspot.com/fetch_milestones)).
- **Electron 42.** Electron 42.0.0 ships Chromium 148.0.7778.96 ([releases.json](https://releases.electronjs.org/releases.json)), whose [`third_party/brotli/README.chromium`](https://github.com/chromium/chromium/blob/148.0.7778.96/third_party/brotli/README.chromium) records `cfa184a`. Electron 43, 44 and 45 ship Chromium 150, 152 and 156.
- **Electron 41 and older.** Electron 41.10.7, the last 41 release, ships Chromium 146.0.7680.216, whose [Brotli revision](https://github.com/chromium/chromium/blob/146.0.7680.216/third_party/brotli/README.chromium) `9801a2c` predates the fix. The 39, 40 and 41 branches carry no Brotli patch of their own.
- **Cross-check.** Real Electron 42.4.1, 43.x and 44.x frameworks contain the `_ERROR_FORMAT_BLOCK_SWITCH` marker. Electron 30.0.1 and 39.8.7 contain only the unfixed 1.0.x error names.

</details>

## Verification

Check the script yourself against known-vulnerable containers. The four examples cover the OS `libbrotli1` package on Debian 12 and Ubuntu 22.04, the Python `Brotli` package, and the Node.js runtime.

- Vulnerable build: `FAIL`, exit 1.
- After the upgrade: `PASS`, exit 0.
- Run from the repository root. `-r /root` scans the container's real home directory, and nothing is planted.
- Podman works in place of `docker` in every example.

### Debian 12

```bash
docker run --rm -it -v "$PWD/scripts/vuln-check-CVE2026-111497-brotli.sh":/check.sh:ro debian:bookworm-slim bash
# inside the container
apt-get update && apt-get install -y libbrotli1=1.0.9-2+b6
bash /check.sh -r /root                        # FAIL [01] libbrotli1
apt-get install -y --only-upgrade libbrotli1   # 1.0.9-2+deb12u1
bash /check.sh -r /root                        # PASS [01] libbrotli1
```

### Ubuntu 22.04

```bash
docker run --rm -it -v "$PWD/scripts/vuln-check-CVE2026-111497-brotli.sh":/check.sh:ro ubuntu:22.04 bash
# inside the container
apt-get update && apt-get install -y libbrotli1=1.0.9-2build6
bash /check.sh -r /root                        # FAIL [01] libbrotli1
apt-get install -y --only-upgrade libbrotli1   # 1.0.9-2ubuntu0.1
bash /check.sh -r /root                        # PASS [01] libbrotli1
```

### Python

```bash
docker run --rm -it -v "$PWD/scripts/vuln-check-CVE2026-111497-brotli.sh":/check.sh:ro python:3.13-slim bash
# inside the container
pip install Brotli==1.2.0
bash /check.sh -r /root                        # FAIL [08] brotli
pip install Brotli==1.2.1
bash /check.sh -r /root                        # PASS [08] brotli
```

### Node.js

Node is fixed by moving to a newer image tag.

```bash
docker run --rm -v "$PWD/scripts/vuln-check-CVE2026-111497-brotli.sh":/check.sh:ro node:24.22.0-slim bash /check.sh -r /root   # FAIL [02] node
docker run --rm -v "$PWD/scripts/vuln-check-CVE2026-111497-brotli.sh":/check.sh:ro node:24.22.1-slim bash /check.sh -r /root   # PASS [02] node
```

## Sources

The affected and fixed versions come from vendor advisories, distro security trackers and upstream releases.
The [sources table](https://mattfields.dev/http2shell-checker/references.html) lists each version with the source behind it.

## Contribution

Found something wrong? Have an improvement? Open an [issue](https://github.com/fieldse/brotli-http2shell-checker/issues) or contact me on [LinkedIn](https://www.linkedin.com/in/mattfields-dev/).

If you find this useful, please let me know.

## License

[MIT](LICENSE)
