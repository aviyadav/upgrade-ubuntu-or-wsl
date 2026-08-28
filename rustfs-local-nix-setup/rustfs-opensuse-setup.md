# Setting up RustFS on openSUSE Tumbleweed for Local Development

RustFS is an S3-compatible distributed object storage system written in Rust. This
guide walks through getting it running locally on Tumbleweed for development work.

## 1. Install system dependencies

Tumbleweed uses `zypper`. You'll need a C/C++ toolchain, OpenSSL headers, and a few
extras that Rust crates commonly depend on:

```bash
sudo zypper refresh
sudo zypper install -t pattern devel_C_C++
sudo zypper install \
  git \
  clang \
  clang-devel \
  libopenssl-devel \
  pkg-config \
  make \
  cmake \
  protobuf-devel
```

Why these:

- `devel_C_C++` pattern — pulls in `gcc`, `g++`, and base build tools.
- `clang-devel` / `clang` — needed by `bindgen`-based crates that generate FFI
  bindings at build time.
- `libopenssl-devel` + `pkg-config` — the `openssl-sys` crate links against
  system OpenSSL.
- `cmake` — some transitive native deps use it.
- `protobuf-devel` — provides the `protoc` compiler required by `prost-build`
  (used by the `pulsar` crate at build time). Without it, the build fails with
  `Could not find protoc`.

## 2. Install the Rust toolchain

Tumbleweed ships Rust in its repos, but for development you generally want
`rustup` so you can pin toolchain versions and add components easily:

```bash
# Option A (recommended): rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustup default stable

# Option B: distro package (may lag behind latest stable)
# sudo zypper install rust cargo
```

Verify:

```bash
rustc --version
cargo --version
```

## 3. Clone and build RustFS

```bash
git clone https://github.com/rustfs/rustfs.git
cd rustfs

# IMPORTANT: download the console UI assets BEFORE building.
# The rustfs/static/ dir is gitignored (only .gitkeep is committed) and the
# console frontend is embedded at compile time via rust-embed. A bare
# `cargo build` produces a binary with NO console assets, so
# http://localhost:9001 returns a RustFS-branded 404.
./scripts/static.sh        # downloads rustfs-console-latest.zip into rustfs/static/

# Debug build (faster compile, good for dev iteration)
cargo build

# or release build if you want to benchmark
cargo build --release
```

> Offline / firewall? Manually download
> `https://dl.rustfs.com/artifacts/console/rustfs-console-latest.zip`
> and unzip it into `rustfs/static/` before building.

The binary lands at `target/debug/rustfs` (or `target/release/rustfs`).

> Alternatively, `./build-rustfs.sh` (release) or `./build-rustfs.sh --dev`
> (debug) handles both the asset download and the build in one step.

> Always check the repo's `README.md` and `CONTRIBUTING.md` after cloning —
> RustFS is actively developed and build flags, workspace layout, or required
> env vars can change.

## 4. Run it locally

Pick a data directory and start the server:

```bash
mkdir -p ~/rustfs-data

# from the rustfs repo root
./target/debug/rustfs server ~/rustfs-data --console-address ":9001"
```

Typical endpoints once it's up:

- **S3 API:** `http://localhost:9000`
- **Web console:** `http://localhost:9001/rustfs/console/`
  - The console is served at the `/rustfs/console/` subpath, not the root of
    9001. Visiting `http://localhost:9001/` in a browser should auto-redirect
    (302) to `/rustfs/console/`. If you instead see a RustFS-branded `404 Not
    found` page, the console assets weren't embedded at build time — run
    `./scripts/static.sh` and rebuild (see step 3).

Credentials are usually set via env vars (e.g. `RUSTFS_ACCESS_KEY` /
`RUSTFS_SECRET_KEY`) or passed as flags — confirm the exact mechanism in the
repo's docs, since this varies between versions. If env vars are used:

```bash
export RUSTFS_ACCESS_KEY=rustfsadmin
export RUSTFS_SECRET_KEY=rustfsadmin
./target/debug/rustfs server ~/rustfs-data --console-address ":9001"
```

## 5. Test it with an S3 client

Install `aws-cli` and point it at your local instance:

```bash
sudo zypper install aws-cli

export AWS_ACCESS_KEY_ID=rustfsadmin
export AWS_SECRET_ACCESS_KEY=rustfsadmin
export AWS_DEFAULT_REGION=us-east-1

aws --endpoint-url http://localhost:9000 s3 mb s3://test-bucket
aws --endpoint-url http://localhost:9000 s3 ls
echo "hello rustfs" > /tmp/hello.txt
aws --endpoint-url http://localhost:9000 s3 cp /tmp/hello.txt s3://test-bucket/
aws --endpoint-url http://localhost:9000 s3 cp s3://test-bucket/hello.txt -
```

## 6. Day-to-day dev workflow

```bash
# fast incremental rebuild + run
cargo run -- server ~/rustfs-data --console-address ":9001"

# run tests
cargo test

# check without producing artifacts (quick)
cargo check

# format and lint
cargo fmt
cargo clippy
```

For a tight edit-test loop, consider `cargo-watch`:

```bash
cargo install cargo-watch
cargo watch -x run
```

## Common pitfalls on Tumbleweed

- **Linker errors mentioning OpenSSL** → `libopenssl-devel` or `pkg-config`
  missing. Alternatively, some Rust projects let you use the vendored feature:
  `cargo build --features vendored` or set `OPENSSL_DIR`.
- **`bindgen` / `libclang` errors** → install `clang-devel`.
- **`Could not find protoc`** → install `protobuf-devel` (provides the `protoc`
  binary that `prost-build` invokes via the `pulsar` crate). As a fallback, set
  `export PROTOC=/usr/bin/protoc`.
- **Stale distro Rust** → if you used `zypper install rust` and hit
  "unsupported toolchain" errors during build, switch to `rustup`.
- **Port 9000/9001 already in use** → another service (or a previous rustfs
  process) is bound. Use `--address` / `--console-address` with different ports.
- **`404 Not found` (RustFS-branded) at `:9001`** → the console UI assets were
  not embedded at build time. `rustfs/static/` is gitignored and must be
  populated first: run `./scripts/static.sh` (or `./build-rustfs.sh --dev`)
  before `cargo build`. Then open `/rustfs/console/`, not the root path.

## Quick Docker alternative (for sanity-checking only)

If you just want to verify your client code against RustFS without a full build:

```bash
sudo zypper install docker
sudo systemctl enable --now docker

docker run -d --name rustfs \
  -p 9000:9000 -p 9001:9001 \
  -v "$HOME/rustfs-data:/data" \
  rustfs/rustfs:latest server /data --console-address ":9001"
```

---

> **Sourcing note:** Command-line specifics (flags, env var names, default
> ports) follow common conventions but may have shifted. After cloning, skim
> `README.md` and `CONTRIBUTING.md` in the repo to confirm exact flag names and
> any project-specific build requirements. See
> [github.com/rustfs/rustfs](https://github.com/rustfs/rustfs) for the latest.
