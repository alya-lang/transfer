# transfer

[![CI](https://github.com/alya-lang/transfer/actions/workflows/ci.yml/badge.svg)](https://github.com/alya-lang/transfer/actions/workflows/ci.yml)
[![License](https://img.shields.io/github/license/alya-lang/transfer?color=blue&label=License)](LICENSE)
[![Alya](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Ftransfer%2Fmain%2Falya.toml&query=%24.package.alya-version&label=Alya&color=orange&prefix=%3E%3D)](https://github.com/alya-lang/alya)
[![Package Version](https://img.shields.io/badge/dynamic/toml?url=https%3A%2F%2Fraw.githubusercontent.com%2Falya-lang%2Ftransfer%2Fmain%2Falya.toml&query=%24.package.version&label=Version&color=brightgreen)](alya.toml)

Multi-protocol file transfer toolkit for Alya: HTTP range, resumable downloads and uploads, FTP, SFTP framing, throttling and checksums

---

## 🌟 Features

- 🌐 **Multi-Protocol Support**: Seamless transfers across HTTP, HTTPS (via the `secure` feature), pure RFC 959 FTP, explicit FTPS (AUTH TLS, `secure` feature), and SFTP v3 session transfers over caller-supplied channels
- ⏯️ **Resumable Transfers**: Automatic `.transfer_meta` state tracking; resumes interrupted downloads and uploads via RFC 7233 HTTP range requests (`Range: bytes=start-end`) and FTP restart markers (`REST` / `APPE`)
- 🔁 **Retry with Backoff**: Configurable `max_retries` / `retry_delay_ms` around every engine; only retryable failures (connection drops, timeouts, HTTP 5xx, transient FTP 4xx, truncation) are retried
- ➡️ **Redirect Following**: Downloads follow up to 5 same- or cross-protocol redirects (auth headers stripped across hosts, query strings preserved); uploads reject redirects with the `Location`
- ⚡ **Bandwidth Throttling**: Token-bucket rate limiter enforcing configurable transfer speeds (`rate_limit_bps`) on downloads and uploads, HTTP and FTP
- 📦 **Binary Mode**: Byte-exact downloads (`.binary(1)`) for non-text payloads via the new `write_bytes` / `append_bytes` / `read_bytes` builtins; checksum verification needs text mode, uploads and SFTP stay text-only for now
- 🔒 **Integrity & Checksums**: Built-in CRC-32 (IEEE 802.3), MD5 (RFC 1321), and SHA-256 (FIPS 180-4) verification engines
- 📊 **Real-Time Progress Metrics**: Granular speed (B/s, KB/s, MB/s), ETA calculation, and visual ASCII progress bar rendering
- 🛠️ **Fluent Builder API**: Ergonomic `TransferBuilder` chaining timeouts, chunk sizes, custom HTTP headers, basic auth, and progress callbacks
- 🧩 **Zero External C Dependencies**: 100% pure Alya implementation with zero native toolchain compilation requirements
- 🚩 **Feature-Gated API Slices**: Optional capability slices via `[features]` in `alya.toml` (`default = ["extras"]`) and `@cfg(feature = "extras")` gating multipart chunk planners and banners

---

## 📁 Project Architecture

```
transfer/
├── .alyalint               # Linter configuration (rules, exclusions, severity overrides)
├── .editorconfig           # Uniform formatting rules across IDEs and editors
├── .gitignore              # Ecosystem standard ignore filters
├── .vscode/                # VS Code workspace settings, DAP launch configurations & tasks
├── alya.toml               # Package manifest with dependencies, [features] and metadata
├── src/
│   ├── lib.alya            # Public API facade (download, upload, ftp_download, checksums)
│   ├── types.alya          # Domain models, TransferConfig, TransferProgress, TransferResult, enums
│   ├── builder.alya        # Fluent TransferBuilder pipeline
│   ├── core/               # Low-level utility and algorithm hierarchy
│   │   ├── str_util.alya   # Internal string index, slicing, and matching utilities
│   │   ├── url.alya        # RFC 3986 URL parsing and query/auth extraction
│   │   ├── checksum.alya   # CRC-32, MD5, and SHA-256 integrity verification
│   │   ├── progress.alya   # Speed, ETA, byte size, and duration formatters
│   │   ├── throttle.alya   # Token-bucket rate limiting delay computation
│   │   ├── resume.alya     # .transfer_meta token lifecycle and verification
│   │   ├── formatter.alya  # Visual ASCII progress bar and summary formatting
│   │   └── extras.alya     # Feature-gated (`extras`) multipart planning and banners
│   ├── http/               # HTTP protocol engine
│   │   ├── range.alya      # RFC 7233 byte-range header formatting and parsing
│   │   └── client.alya     # Resumable HTTP download/upload engines, retry, redirects, HTTPS (secure feature)
│   ├── ftp/                # FTP protocol engine
│   │   ├── protocol.alya   # RFC 959 command framing, PASV parser, PORT builder
│   │   └── client.alya     # Passive/active transfers, REST/APPE resume, retry, explicit FTPS (secure feature)
│   └── sftp/               # SFTP wire protocol
│       ├── packet.alya     # Big-endian uint32/uint64 binary framing and packing
│       ├── protocol.alya   # SFTP v3 packet encoders/decoders (incl. server replies)
│       └── client.alya     # Session transfer engine over caller-supplied channels
├── examples/
│   └── demo.alya           # Comprehensive runnable walkthrough of all package capabilities
├── tests/
│   └── test_basic.alya     # Automated test suite covering all modules
└── benches/
    └── bench_basic.alya    # Micro-benchmarks measuring URL, CRC32, range, progress throughput
```

> [!NOTE]
> **Visibility & Modularity:** Symbols annotated with `pub` (`pub function`, `pub struct`, `pub enum`, `pub interface`) are exported to external consumers and re-exporting modules. Symbols without `pub` remain strictly internal to their declaring module, preventing symbol collisions and implementation leakage.

---

## 📦 Installation

Add `transfer` to the `[dependencies]` section in your `alya.toml`:

```toml
[dependencies]
transfer = { git = "https://github.com/alya-lang/transfer", branch = "main" }
```

Or install it directly using the Alya package CLI:

```bash
alya add transfer --git https://github.com/alya-lang/transfer --branch main
alya install
```

### Package Features

| Feature | Default | Description |
|:---|:---:|:---|
| `extras` | ✅ | Multipart chunk planners and transfer banners (`plan_multipart_chunks`, `format_transfer_banner`). |
| `secure` | ❌ | TLS-backed HTTPS/FTPS transfers; enables the `tls` dependency (`https_download_once`, `ftps_*`). |

### 🔒 HTTPS & FTPS (`secure` feature)

TLS transports live behind the opt-in `secure` feature (pulls the `tls` package, which needs a C toolchain). The default build stays dependency-free:

```bash
alya install --features secure
alya test --features secure
```

Then enable per transfer with the defaults (certificates verified):

```alya
let res = transfer::download("https://example.com/archive.tar.gz", "archive.tar.gz")
    .rate_limit(1048576)
    .execute() # verified TLS; use .tls_verify(0) only for local testing
```

SFTP transfers need an SSH transport, which ships outside this package. Drive the session engine with your own channel callbacks:

```alya
let res = transfer::sftp_download_file(my_send, my_recv, "/remote/data.bin", "data.bin", cfg, null)
```

---

## 🚀 Quick Start

```alya
import "transfer" as transfer

function main()
    # 1. Fluent download builder with progress and rate limiting
    let res = transfer::download("https://example.com/archive.tar.gz", "archive.tar.gz")
        .chunk_size(65536)
        .rate_limit(1048576)
        .resumable(1)
        .on_progress(function(prog)
            say f"\r{prog.render_bar(25)} {prog.speed_bps} B/s"
        end)
        .execute()

    if res.is_ok() == 1
        say f"\nDownload completed: {res.bytes_transferred} bytes"
    end

    # 2. Direct checksum verification
    let hash = transfer::sha256("alya package ecosystem")
    say f"SHA-256: {hash}"
end

main()
```

---

## 📖 API Reference

| Symbol | Visibility | Description |
|---|---|---|
| `download(url, dest_path)` | `pub function` | Initiates a fluent `TransferBuilder` targeting a destination file path. |
| `upload(url, src_path)` | `pub function` | Initiates a fluent `TransferBuilder` sourcing data from a local file path. |
| `download_file(url, dest_path, config, on_progress)` | `pub function` | Direct download executor returning a `TransferResult`. |
| `upload_file(url, src_path, config, on_progress)` | `pub function` | Direct upload executor returning a `TransferResult`. |
| `ftp_download(host, port, user, pass, remote_path, local_path, resume, on_progress)` | `pub function` | RFC 959 FTP download with passive data mode and offset restart (`REST`). |
| `ftp_upload(host, port, user, pass, local_path, remote_path, resume, on_progress)` | `pub function` | RFC 959 FTP upload with passive data mode and append mode (`APPE`). |
| `ftps_download_file(host, port, user, pass, remote_path, local_path, config, on_progress)` | `pub function` (`secure` feature) | Explicit FTPS download (AUTH TLS + encrypted control/data). |
| `ftps_upload_file(host, port, user, pass, local_path, remote_path, config, on_progress)` | `pub function` (`secure` feature) | Explicit FTPS upload (AUTH TLS + encrypted control/data). |
| `https_transfer_download(url, dest_path, config, on_progress, max_hops)` | `pub function` (`secure` feature) | Resumable HTTPS download with retry and redirect following. |
| `https_transfer_upload(url, src_path, config, on_progress)` | `pub function` (`secure` feature) | HTTPS upload with retry. |
| `sftp_download_file(send, recv, remote_path, local_path, config, on_progress)` | `pub function` | SFTP download over caller-supplied channel callbacks. |
| `sftp_upload_file(send, recv, local_path, remote_path, config, on_progress)` | `pub function` | SFTP upload over caller-supplied channel callbacks. |
| `sftp_session_open(send, recv)` | `pub function` | SFTP INIT/VERSION handshake over channel callbacks. |
| `resolve_redirect_url(base_url, location)` | `pub function` | Resolves a redirect `Location` against the requesting URL. |
| `http_is_redirect(code)` | `pub function` | Returns 1 for redirect statuses (301/302/303/307/308). |
| `http_retryable_error(msg)` / `ftp_retryable_error(msg)` | `pub function` | Classify transfer errors as retryable (1) or fatal (0). |
| `retry_max_attempts(config)` / `retry_backoff_ms(config, attempt)` | `pub function` | Attempt budget (`max_retries + 1`) and linear backoff delay. |
| `ftp_format_port(ip, port)` | `pub function` | Builds a PORT argument for FTP active mode. |
| `throttle_wait(throttle, chunk_size)` | `pub function` | Accounts a chunk and sleeps when the rate cap is exceeded. |
| `write_bytes(path, data)` / `append_bytes(path, data)` / `read_bytes(path)` | builtins | Byte-exact file I/O preserving embedded NULs (binary mode needs them). |
| `new_config(timeout_ms, chunk_size, max_retries, rate_limit_bps, resumable)` | `pub function` | Factory constructing a `TransferConfig` with sensible defaults. |
| `new_progress(bytes_transferred, total_bytes, speed_bps)` | `pub function` | Factory constructing a `TransferProgress` snapshot. |
| `crc32(data)` | `pub function` | Computes the CRC-32 IEEE 802.3 checksum string of data. |
| `md5(data)` | `pub function` | Computes the MD5 RFC 1321 hex digest of data. |
| `sha256(data)` | `pub function` | Computes the SHA-256 FIPS 180-4 hex digest of data. |
| `verify_checksum(path, expected, algo)` | `pub function` | Validates file integrity against expected hash value. |
| `plan_multipart_chunks(total_bytes, chunk_count)` | `pub function` (`extras` feature) | Partitions byte ranges for parallel multi-part transfers. |
| `format_transfer_banner(title)` | `pub function` (`extras` feature) | Renders formatted ASCII banner for CLI transfer tools. |
| `TransferBuilder` | `pub struct` | Fluent builder with chained configuration setters and `execute()`. |
| `TransferConfig` | `pub struct` | Configuration parameters (`timeout_ms`, `chunk_size`, `rate_limit_bps`, `resumable`, `max_retries`, `retry_delay_ms`, `ftp_mode`, `ftps`, `tls_verify`, `binary`, etc.). |
| `TransferProgress` | `pub struct` | Real-time transfer metrics with `render_bar()`, `summary()`, and `is_complete()`. |
| `TransferResult` | `pub struct` | Operation outcome with `is_ok()`, `is_error()`, and `summary()`. |
| `ResumeMeta` | `pub struct` | Persistent resume metadata token (`.transfer_meta`) container. |
| `FtpResponse` | `pub struct` | RFC 959 status reply container with `is_success()` and `is_error()`. |
| `SftpPacket` | `pub struct` | Binary wire frame model for SFTP v3 packets. |
| `TransferProtocol` | `pub enum` | Protocol discriminator (`Http = 0`, `Https = 1`, `Ftp = 2`, `Sftp = 3`, `File = 4`). |
| `TransferMode` | `pub enum` | Direction mode (`Download = 0`, `Upload = 1`). |
| `TransferStatus` | `pub enum` | Task lifecycle state (`Idle = 0`, `Running = 1`, `Paused = 2`, `Completed = 3`, `Failed = 4`, `Canceled = 5`). |
| `ChecksumAlgo` | `pub enum` | Checksum algorithm selector (`None = 0`, `Sha256 = 1`, `Md5 = 2`, `Crc32 = 3`). |

> [!TIP]
> **Internal Helpers & Documentation:** Public symbols are documented with `##` Markdown docstrings, enabling automatic API documentation generation via `alya doc`. Low-level protocol decoders and token-bucket computations in submodules are encapsulated.

---

## 🧪 Running Tests & Benchmarks

Run the automated test suite using `alya test`:

```bash
alya test
```

Exercise feature selection (the `extras` slice is default-on):

```bash
alya test --features extras
alya test --no-default-features
```

Exercise the TLS transports (needs a C toolchain for the `tls` dependency):

```bash
alya install --features secure
alya test --features secure
```

Generate static API documentation:

```bash
alya doc . -o docs --markdown
```

Run the benchmark suite:

```bash
alya run benches/bench_basic.alya
```

Run the example demo:

```bash
alya run examples/demo.alya
```

Check code formatting:

```bash
alya fmt . --check
```

Run static code linter:

```bash
alya lint . --check
```

---

### 💻 Developer Tooling & VS Code Integration

This package comes preconfigured with recommended workspace settings and tasks for **Visual Studio Code**:
- **LSP & Formatting**: Auto-formatting on save and real-time Language Server diagnostics via `alya-lang.vscode-alya`.
- **DAP Debugging**: Launch configurations in `.vscode/launch.json` ready for interactive step-debugging via `F5`.
- **Predefined Tasks**: Press `Ctrl+Shift+B` or run tasks (`Test`, `Lint`, `Format`, `Build Docs`) directly from the Command Palette.

---

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository and clone it locally
2. Install dependencies:
   ```bash
   alya install
   ```
3. Create your feature branch (`git checkout -b feature/my-feature`)
4. Verify tests and formatting before opening a PR:
   ```bash
   alya test
   alya fmt . --check
   ```
5. Commit your changes (`git commit -m "feat: add feature"`) and open a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.