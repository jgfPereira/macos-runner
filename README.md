# Unix Cross-Platform Interactive Runners

A robust GitHub Actions toolkit designed to spin up secure, fully-configured interactive SSH sessions (powered by **Upterm**) across multiple Unix-based operating systems and hardware architectures.

This repository enables cross-platform build pipelines, remote debugging, and interactive execution environments on **macOS** and **Linux** runners across both **x64** and **ARM64** architectures — all while leveraging GitHub Actions free runner minutes securely.

---

## 🚀 Key Features

* **Multi-Architecture & OS Support:** Dedicated workflows for:
  * **macOS ARM64** (`macos-26`, Apple Silicon)
  * **macOS x64** (`macos-26-intel`, Intel)
  * **Linux ARM64** (`ubuntu-24.04-arm` host running an `ubuntu:20.04` Docker container)
  * **Linux x64** (`ubuntu-24.04` host running an `ubuntu:20.04` Docker container)
* **Interactive SSH Sessions:** Direct terminal access via Upterm, restricted securely to authorized GitHub users (`limit-access-to-users: 'jgfPereira'`).
* **Container Terminal Ergonomics:** Full support for terminal colors, aliases (`ll`, `la`, colored `grep`), auto-completion (`bash-completion`), and interactive Bash shell startup inside Docker containers.
* **Dynamic Software Toolchains:** Automatically provisions modern developer stacks dynamically at runtime:
  * **Oracle JDK 25** (downloaded, extracted, and registered via `update-alternatives` or system paths)
  * **Latest Go (Golang)** (dynamically fetched via `go.dev/VERSION?m=text`)
  * **Maven**, **Python 3**, **ImageMagick**, **Binutils** (`objcopy` for `jpackage`), and **OpenSSL**
* **Secure Encrypted Assets:** Decrypts Git LFS tracked payload archives (`payload.zip.enc`) using PBKDF2 key derivation and AES-256-CBC encryption backed by GitHub Secrets.

---

## 📁 Workflow Files & Environments

| Workflow File | Target OS | Arch | Runner Host | Container / Environment | Shell |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `linux-arm64.yml` | Linux | ARM64 (`aarch64`) | `ubuntu-24.04-arm` | `ubuntu:20.04` | `/bin/bash` |
| `linux-x64.yml` | Linux | x64 (`x86_64`) | `ubuntu-24.04` | `ubuntu:20.04` | `/bin/bash` |
| `macos-arm64.yml` | macOS | ARM64 (`aarch64`) | `macos-26` | Host OS | `/bin/zsh` / `bash` |
| `macos-x64.yml` | macOS | x64 (`x86_64`) | `macos-26-intel` | Host OS | `/bin/zsh` / `bash` |

---

## 🔄 Workflow Execution Flow

### Linux Workflows (`linux-arm64.yml` & `linux-x64.yml`)
1. **Container Initialization:** Boots an `ubuntu:20.04` Docker container with `SHELL: /bin/bash` exported for Upterm.
2. **Pre-Checkout Bootstrap:** Installs `git`, `git-lfs`, `unzip`, and `openssl` via `apt-get` inside the container.
3. **Checkout Code (`actions/checkout@v4`):** Clones the repository and downloads Git LFS assets.
4. **Decrypt & Extract Payload:** Decrypts `payload.zip.enc` with `ZIP_PASSWORD` and extracts files into `./extracted_files`.
5. **Install System Dependencies:** Installs `maven`, `python3`, `zip`, `imagemagick`, `curl`, `ca-certificates`, `binutils`, and `bash-completion`.
6. **Dynamic Go Installation:** Resolves latest stable Go version from `go.dev`, downloads arch-specific tarball (`amd64` or `arm64`), and updates `$GITHUB_PATH`.
7. **Oracle Java 25 Setup:** Downloads JDK 25 tarball for Linux (`x64` or `aarch64`), configures `update-alternatives` (`java`, `javac`), and exports `JAVA_HOME`.
8. **Terminal Ergonomics Configuration:** Writes a custom colorized `.bashrc` prompt (`PS1`), enables bash-completion, and links `.profile` to ensure login SSH sessions load interactive configurations.
9. **SSH Keepalive & Upterm Start:** Configures `ServerAliveInterval 30` in `~/.ssh/config` and launches Upterm SSH.

### macOS Workflows (`macos-arm64.yml` & `macos-x64.yml`)
1. **Checkout Code:** Clones repository with Git LFS assets.
2. **Decrypt & Extract Payload:** Decrypts `payload.zip.enc` and extracts into `./extracted_files`.
3. **Install Dependencies (`brew`):** Installs `maven`, `go`, `python3`, `imagemagick`, and `openssl`.
4. **Oracle Java 25 Setup:** Downloads macOS JDK 25 tarball (`x64` or `aarch64`), extracts to `/Library/Java/JavaVirtualMachines/`, and exports `JAVA_HOME`.
5. **SSH Keepalive & Upterm Start:** Configures SSH client options and starts the Upterm session.

---

## 🛠️ Useful Commands & Cheat Sheet

### 1. Encrypting Payloads Locally (Before Pushing)
Since this repository is public, sensitive artifacts or private dependencies must be encrypted before being pushed to Git LFS:

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -in payload.zip -out payload.zip.enc
```
*(Ensure the decryption key matches the `ZIP_PASSWORD` GitHub secret).*

---

### 2. File Transfer via SCP (Active SSH Session)

Because Linux workflows run inside a Docker container, file paths differ between host and container environments:
* **Container Workspace Root:** `/__w/unix-runner/unix-runner/`
* **Relative Path:** Start from the current working directory.

#### Fetching Executables to Local Machine (Windows / macOS / Linux):
```bash
# Relative Path (Recommended)
scp -o ServerAliveInterval=30 -l 8192 <user>@uptermd.upterm.dev:extracted_files/isign/target/launcher_template-linux .

# Container Absolute Path
scp -o ServerAliveInterval=30 -l 8192 <user>@uptermd.upterm.dev:/__w/unix-runner/unix-runner/extracted_files/isign/target/launcher_template-linux .
```

* **`-o ServerAliveInterval=30`**: Prevents SSH idle timeouts.
* **`-l 8192`**: Caps transfer bandwidth at 8 Mbps (~1 MB/s) to maintain SSH session stability over public relays.