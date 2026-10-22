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
* **Interactive SSH Sessions:** Direct terminal access via Upterm, restricted securely to your authorized GitHub username (`limit-access-to-users`).
* **SSH Keepalive Configuration:** Automatic `ServerAliveInterval` configuration to ensure stable SSH connections over public relays.
* **Secure Large Payload Handling:** Store and manage encrypted binary payloads (e.g., assets or build files) in a public repository using **Git LFS** and **AES-256-CBC** encryption backed by GitHub Secrets.
* **Pre-Installed Development Stack:** Automatically provisions modern developer tools and environments right upon startup:
  * **Oracle JDK 25** (configured system-wide or via environment variables)
  * **Maven** & **Go (Golang)**
  * **Python 3**
  * **ImageMagick** & **OpenSSL**
  * **Infozip** (`zip` / `unzip`)

---

## 📁 Workflow Files

| Workflow File | OS Target | Hardware Arch | Runner / Container |
| :--- | :--- | :--- | :--- |
| `linux-arm64.yml` | Linux | ARM64 (`aarch64`) | Host: `ubuntu-24.04-arm` \| Container: `ubuntu:20.04` |
| `linux-x64.yml` | Linux | x64 (`x86_64`) | Host: `ubuntu-24.04` \| Container: `ubuntu:20.04` |
| `macos-arm64.yml` | macOS | ARM64 (`aarch64`) | Host: `macos-26` |
| `macos-x64.yml` | macOS | x64 (`x86_64`) | Host: `macos-26-intel` |

---

## 🔄 Workflow Step-by-Step Breakdown

### Linux Workflows (`linux-arm64.yml` & `linux-x64.yml`)
1. **Container Initialization:** Launches job inside an `ubuntu:20.04` container environment.
2. **Pre-Checkout Bootstrap:** Installs `git`, `git-lfs`, `unzip`, and `openssl` via `apt-get` prior to code checkout so GitHub Actions and decryption steps execute cleanly.
3. **Checkout Code (`actions/checkout@v4` with LFS):** Clones the repository and pulls Git LFS tracked files.
4. **Decrypt Payload (`openssl`):** Decrypts `payload.zip.enc` using PBKDF2 key derivation and the `ZIP_PASSWORD` GitHub Secret.
5. **Extract Payload (`unzip`):** Unpacks the archive into `./extracted_files`.
6. **Install Required Dependencies:** Installs Maven, Go, Python 3, Zip, and ImageMagick.
7. **Install & Configure Oracle Java 25:** Downloads architecture-specific JDK 25 tarball (`aarch64` or `x64`), extracts it to `/opt/jdk`, and configures `update-alternatives` alongside `JAVA_HOME` and `PATH`.
8. **Configure SSH Keepalive & Start Upterm:** Configures `~/.ssh/config` for connection stability and launches an interactive SSH session.

### macOS Workflows (`macos-arm64.yml` & `macos-x64.yml`)
1. **Checkout Code (`actions/checkout@v4` with LFS):** Clones the repository and pulls Git LFS tracked files.
2. **Decrypt Payload (`openssl`):** Decrypts `payload.zip.enc` using PBKDF2 key derivation and the `ZIP_PASSWORD` GitHub Secret.
3. **Extract Payload (`unzip`):** Unpacks the archive into `./extracted_files`.
4. **Install Required Dependencies (`brew`):** Updates Homebrew and installs Maven, Go, Python 3, ImageMagick, and OpenSSL.
5. **Install & Configure Oracle Java 25:** Downloads architecture-specific macOS JDK 25 tarball (`aarch64` or `x64`), extracts it to `/Library/Java/JavaVirtualMachines/`, and updates environment variables (`JAVA_HOME`, `PATH`).
6. **Configure SSH Keepalive & Start Upterm:** Configures `~/.ssh/config` for connection stability and launches an interactive SSH session.

---

## 🛠️ Useful Commands & Cheat Sheet

### 1. Encrypting Payloads Locally (Before Pushing)
Since this repository is public to maximize free GitHub Actions minutes, any heavy assets or private binaries must be encrypted before being pushed to Git LFS. Run this command locally on your machine:

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -in payload.zip -out payload.zip.enc
```
*(Make sure to set the exact same password as your repository's `ZIP_PASSWORD` GitHub Secret).*

### 2. Retrieving Files Securely via SCP (During Active SSH Session)
While connected to the runner via Upterm, you can pull generated files back to your local machine using an optimized `scp` command:

```bash
scp -o ServerAliveInterval=30 -l 8192 <user>@uptermd.upterm.dev:/path/to/remote/file .
```

* **`-o ServerAliveInterval=30`**: Sends keep-alive pings every 30 seconds to prevent connection drops.
* **`-l 8192`**: Limits bandwidth usage to ~1 MB/s (8192 kbps) to ensure connection stability over public relays.
* **`<user>`**: Your Upterm SSH session username provided in the GitHub Actions run logs.