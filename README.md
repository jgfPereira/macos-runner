# Unix Cross-Platform Interactive Runners

A robust GitHub Actions toolkit designed to spin up secure, fully-configured interactive SSH sessions (powered by **Upterm**) across multiple Unix-based operating systems and architectures. 

This repository enables cross-platform build pipelines, remote debugging, and manual execution environments on **macOS**, **Linux x64**, and **Linux ARM64** runners — all while leveraging GitHub Actions free public runner minutes securely.

---

## 🚀 Key Features

* **Multi-Architecture Support:** Seamlessly run workflows on macOS (`macos-latest`), Linux x64 (`ubuntu-latest`), and Linux ARM64 (`ubuntu-24.04-arm`).
* **Interactive SSH Sessions:** Direct terminal access via Upterm, restricted securely to your authorized GitHub username (`limit-access-to-users`).
* **Secure Large Payload Handling:** Store and manage encrypted binary payloads (e.g., assets or build files) in a public repository using **Git LFS** and **AES-256-CBC** encryption backed by GitHub Secrets.
* **Pre-Installed Development Stack:** Automatically provisions modern developer tools and environments right upon startup:
  * **Oracle JDK 25** (configured system-wide via `update-alternatives`)
  * **Maven** & **Go (Golang)**
  * **Python 3**
  * **ImageMagick** & **OpenSSL**
  * **Infozip** (`zip` / `unzip`)

---

## 🔄 Workflow Step-by-Step Breakdown

Every workflow (`linux-x64.yml`, `linux-arm64.yml`, and macOS configurations) executes a structured sequence of initialization steps before dropping you into the interactive shell:

1. **Checkout Code (`actions/checkout@v4` with LFS):**
   Clones the repository and automatically pulls down large tracked assets (like your encrypted archive) using Git Large File Storage.
2. **Decrypt Payload (`openssl`):**
   Uses OpenSSL with PBKDF2 key derivation and the repository's `ZIP_PASSWORD` GitHub Secret to securely decrypt the payload zip file (`payload.zip.enc -> payload.zip`).
3. **Extract Payload (`unzip`):**
   Unpacks the decrypted archive into a working directory (`./extracted_files`) so your tools and resources are ready inside the runner.
4. **Install Required Dependencies (`apt-get`):**
   Updates package lists and installs the latest versions of Maven, Go, Python 3, zip/unzip, ImageMagick, and OpenSSL, verifying their versions in the build logs.
5. **Install & Configure Oracle Java 25:**
   Downloads the official Oracle JDK 25 distribution for the corresponding architecture, extracts it to `/opt/jdk`, sets up system alternatives with priority enforcement (`--set`), and exports environment variables (`JAVA_HOME`, `PATH`) globally and for the session.
6. **Start Secure Interactive SSH Session (`action-upterm`):**
   Initializes an Upterm server instance, outputting a secure SSH connection command in your workflow logs while locking access exclusively to your GitHub account.

---

## 🛠️ Useful Commands & Cheat Sheet

### 1. Encrypting Payloads Locally (Before Pushing)
Since this repository is public to maximize free GitHub Actions minutes, any heavy assets or private binaries must be encrypted before being pushed to Git LFS. Run this command locally on your machine:

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -in <input> -out <output>.enc
```
*(Make sure to set the exact same password as your repository's `ZIP_PASSWORD` GitHub Secret).*

### 2. Retrieving Files Securely via SCP (During Active SSH Session)
While connected to the runner via Upterm, you can pull generated files back to your local machine using an optimized `scp` command. Throttling the speed and enabling keep-alive packets ensures large transfers complete stably over public relays:

```bash
scp -o ServerAliveInterval=<seconds> -l <bandwidth_limit_kbps> <user>@uptermd.upterm.dev:/path/to/remote/file .
```

* **`<seconds>`** (e.g., `30`): Frequency of keep-alive pings to prevent timeout drops.
* **`<bandwidth_limit_kbps>`** (e.g., `8192`): Bandwidth limit in kilobits per second (e.g., `8192` = ~1 MB/s) to avoid choking the public relay server.
* **`<user>`**: Your Upterm SSH username provided in the GitHub Actions run logs.