<!--
  Copyright (C) 2026 amisuke
  SPDX-License-Identifier: Apache-2.0
-->

# Building Axion AOSP for pipa — a beginner's guide

[日本語](README.md) | **English**

A step-by-step guide for building Axion AOSP (Android 16) for the Xiaomi Pad 6 (**pipa**) yourself.
If you have used Linux before, you should be able to follow it from top to bottom and reach a finished build.

> This repository (`local_manifests`) holds the configuration that adds "pipa-specific parts" on top of the Axion source tree.
> The file that matters is `pipa.xml`, which lists where the kernel, device tree, vendor blobs and so on are fetched from.

---

## 0. Overview

1. Prepare a build machine (Ubuntu)
2. Install the required packages and the `repo` tool
3. Fetch the Axion source (`repo init`)
4. Add this `local_manifests`
5. Sync the source (`repo sync`) — takes hours
6. Build
7. Check the resulting zip

The first sync plus build takes **half a day to a full day**. Later builds only redo what changed and are much faster.

---

## 1. What you need

| Item | Guideline |
|---|---|
| OS | 64-bit Ubuntu 22.04 / 24.04 (other Linux distributions work too; on Windows use Ubuntu under WSL2) |
| RAM | 16 GB or more (32 GB+ recommended; the build gets killed if you run out) |
| Free disk | 400 GB or more (SSD recommended) |
| CPU | More cores means a faster build |
| Network | Always on. Fetching the source alone is a very large download (tens of GB or more) |

**If you use Windows (WSL2)**
- Keep the source on the Linux filesystem (somewhere under `~/`). Putting it under `/mnt/c/...` is extremely slow and often breaks the build.
- Also check the WSL2 memory limit (`memory=` in `%UserProfile%\.wslconfig`) and your free disk space.

---

## 2. Setting up the environment

### 2-1. Install the required packages

```bash
sudo apt update
sudo apt install -y \
  bc bison build-essential ccache curl flex g++-multilib gcc-multilib \
  git git-lfs gnupg gperf imagemagick lib32z1-dev libdw-dev libelf-dev \
  libgnutls28-dev libssl-dev libxml2 libxml2-utils lz4 lzop pngcrush \
  protobuf-compiler python3 python3-protobuf rsync squashfs-tools \
  xsltproc zip unzip zlib1g-dev
```

- Package names differ slightly between Ubuntu versions. If you get "Unable to locate package", remove that name from the list and run the command again.
- If the build later says a command is missing, install the package that provides it with `apt install` and continue.

Next, install `repo`, the Android source management tool.

```bash
mkdir -p ~/.local/bin
curl https://storage.googleapis.com/git-repo-downloads/repo -o ~/.local/bin/repo
chmod a+x ~/.local/bin/repo
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 2-2. Configure git

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git lfs install
```

`repo` uses git internally and needs a name and e-mail address. They are not published anywhere.

### 2-3. Speed up rebuilds (optional)

```bash
echo 'export USE_CCACHE=1' >> ~/.bashrc
echo 'export CCACHE_EXEC=/usr/bin/ccache' >> ~/.bashrc
source ~/.bashrc
ccache -M 50G
```

This makes second and later builds faster. Skip it if you are short on disk space.

---

## 3. Fetching the source

### 3-1. Create a working folder and point it at Axion

```bash
mkdir -p ~/axion
cd ~/axion
repo init -u https://github.com/AxionAOSP/android.git -b lineage-23.2 --git-lfs
```

- `-b lineage-23.2` is Axion's Android 16 branch. If a newer branch has been published, use that instead.
- If you are asked about colored output, answer `y` or `n`; either is fine.

### 3-2. Add this local_manifests

`repo init` has created `.repo/`. Clone this repository into it as `local_manifests`.

```bash
git clone -b ax-pipa-16 https://github.com/amiami5/local_manifests .repo/local_manifests
```

- The branch name `ax-pipa-16` means "Axion, Android 16, pipa".
- If you see "already exists and is not an empty directory", a `.repo/local_manifests` is already there. Check its contents and delete it if you do not need it, then retry.
- The same branch also contains `cepheus.xml` (for the Xiaomi Mi 9). It is not needed for pipa but does no harm. To fetch fewer repositories, change its extension, e.g. `mv .repo/local_manifests/cepheus.xml .repo/local_manifests/cepheus.xml.bak`.

### 3-3. Sync the source

```bash
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
```

- This takes hours. If it stops or shows a network error, **just run the same command again**; it resumes where it left off.
- `-j8` is the number of parallel downloads. Use `-j4` on a slow connection.
- When it finishes you will have folders such as `device/xiaomi/pipa` and `kernel/xiaomi/sm8250`.

**If something goes wrong**
- `error: Cannot fetch ...`: usually a temporary network problem. Run the same command again.
- A repository cannot be found: the URL in `pipa.xml` may have moved or become private. Look up the repository named in the error.

---

## 4. Building

Always run the build commands **from the top of the source tree (`~/axion`)**.

### 4-1. Load the build environment

```bash
cd ~/axion
source build/envsetup.sh
```

### 4-2. Choose the target

```bash
axion pipa userdebug gms
```

`axion` is a helper command provided by Axion. The arguments mean:

| Argument | Meaning |
|---|---|
| `pipa` | The device codename |
| `userdebug` | The build type. `userdebug` is recommended to start with (adb root and logs are available, which makes troubleshooting easier). `user` is for release builds and `eng` for development. |
| `gms` | With Google apps. Other choices are `pico` (minimal Google apps), `core`, and `va` (no Google apps). |

At this point `device/xiaomi/pipa/vendorsetup.sh` runs automatically and applies the tablet fixes (patches to `frameworks/base` and `hardware/google/pixel`). A message like "Tablet patch applied successfully" is normal. On later runs you will see something like "already applied or not applicable; skipping", which is also normal.

### 4-3. Run the build

```bash
ax -br
```

- `ax -br` runs `brunch`, which builds everything and produces the zip.
- **The first build takes several hours to over ten hours.** You can leave it running.
- It uses all your CPU, so the machine will feel slow while it runs.

If the build is killed for lack of memory, lower the number of parallel jobs:

```bash
ax -br -j6
```

### 4-4. If you closed the terminal

Reload the environment and run the build again. Work already done is reused, so it does not start from scratch.

```bash
cd ~/axion
source build/envsetup.sh
axion pipa userdebug gms
ax -br
```

---

## 5. Checking the result

After a successful build, the output is here:

```
~/axion/out/target/product/pipa/
```

The `axion-…-pipa.zip` in that folder is the file to install on the device. Images such as `boot.img` and `recovery.img` are created alongside it.

---

## 6. Installing on the device (overview)

This guide only covers building. Before installing, check carefully against the **actual state of your own device**.

- It assumes the bootloader is unlocked and a custom recovery is installed.
- The usual flow is: boot into recovery → factory reset if needed (this erases your data) → install the zip with `adb sideload`.
- **A factory reset erases all data.** Back up first.

---

## 7. Updating the source

```bash
cd ~/axion
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
source build/envsetup.sh
axion pipa userdebug gms
ax -br
```

- `vendorsetup.sh` adds one local commit to `frameworks/base`. If a sync reports a conflict there, that commit may be the cause.

---

## 8. Common problems

| Symptom | What to do |
|---|---|
| `repo: command not found` | Do the `repo` installation in 2-1 and make sure it is on your `PATH`. |
| Sync stops partway | Run the same `repo sync` again. |
| The build stops with `Killed` | Out of memory. Lower the job count, e.g. `ax -br -j6`, and set up swap. |
| `No space left on device` | Not enough disk. Make sure you have 400 GB or more free. |
| "xxx: command not found" | A package is missing. Install it with `apt install`. |
| `Cannot locate config makefile` from `lunch` / `axion` | Check that you ran `source build/envsetup.sh` at the top of the source tree. |
| The build fails partway | Read the last few dozen lines of the error. The first line starting with `FAILED:` is the cause. |

---

## 9. References

- Axion AOSP: <https://github.com/AxionAOSP/android>
- This local_manifests: <https://github.com/amiami5/local_manifests>
