<!--
  Copyright (C) 2026 amisuke
  SPDX-License-Identifier: Apache-2.0
-->

# pipa 向け Axion AOSP ビルドガイド

**日本語** | [English](README.en.md)

Xiaomi Pad 6 (**pipa**) 用の Axion AOSP (Android 16) を、自分でビルドしたい初心者向けの手順書です。
Linux を触ったことがあれば、上から順に実行するだけでビルドまで進めるように書いています。

> このリポジトリ (`local_manifests`) は、Axion 本体のソースに「pipa 用の部品」を足すための設定ファイル置き場です。
> 中身は `pipa.xml` で、カーネル・デバイスツリー・ベンダー blob などの取得先が書かれています。

---

## 0. 全体の流れ

1. ビルド用の PC を用意する (Ubuntu)
2. 必要なパッケージと `repo` コマンドを入れる
3. Axion 本体のソースを取得する (`repo init`)
4. この `local_manifests` を置く
5. ソースを同期する (`repo sync`) ← 数時間かかります
6. ビルドする
7. できあがった zip を確認する

初回は、同期とビルドを合わせて **半日〜1 日** かかります。2 回目以降は差分だけなので、ずっと短くなります。

---

## 1. 用意するもの

| 項目 | 目安 |
|---|---|
| OS | 64bit の Ubuntu 22.04 / 24.04 (他の Linux でも可。Windows は WSL2 の Ubuntu で可) |
| メモリ | 16GB 以上 (32GB 以上を推奨。足りないと途中で落ちます) |
| 空きディスク | 400GB 以上 (SSD 推奨) |
| CPU | コア数が多いほど速い |
| ネットワーク | 常時接続。ソースの取得だけで非常に大きなダウンロード (数十 GB 以上) が発生します |

**Windows (WSL2) の場合の注意**
- ソースは必ず Linux 側のディスク (`~/` の下など) に置いてください。`/mnt/c/...` の下に置くと極端に遅くなり、ビルドも失敗しやすくなります。
- WSL2 のメモリ上限 (`%UserProfile%\.wslconfig` の `memory=`) とディスクの空きも確認してください。

---

## 2. 環境構築

### 2-1. 必要なパッケージを入れる

```bash
sudo apt update
sudo apt install -y \
  bc bison build-essential ccache curl flex g++-multilib gcc-multilib \
  git git-lfs gnupg gperf imagemagick lib32z1-dev libdw-dev libelf-dev \
  libgnutls28-dev libssl-dev libxml2 libxml2-utils lz4 lzop pngcrush \
  protobuf-compiler python3 python3-protobuf rsync squashfs-tools \
  xsltproc zip unzip zlib1g-dev
```

- パッケージ名は Ubuntu のバージョンで少し違うことがあります。「Unable to locate package」と出たものは、その名前を一覧から外して、もう一度実行してください。
- ビルド中に「○○ がない」と言われたら、そのコマンドを含むパッケージを `apt install` で追加すれば進められます。

次に、Android のソース管理ツール `repo` を入れます。

```bash
mkdir -p ~/.local/bin
curl https://storage.googleapis.com/git-repo-downloads/repo -o ~/.local/bin/repo
chmod a+x ~/.local/bin/repo
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 2-2. git の初期設定

```bash
git config --global user.name  "あなたの名前"
git config --global user.email "あなたのメールアドレス"
git lfs install
```

名前とメールアドレスは、`repo` が内部で git を使うために必要です (公開されるものではありません)。

### 2-3. ビルドを速くする設定 (任意)

```bash
echo 'export USE_CCACHE=1' >> ~/.bashrc
echo 'export CCACHE_EXEC=/usr/bin/ccache' >> ~/.bashrc
source ~/.bashrc
ccache -M 50G
```

2 回目以降のビルドが速くなります。ディスクに余裕がなければ飛ばして構いません。

---

## 3. ソースを取得する

### 3-1. 作業フォルダを作って Axion 本体を指定する

```bash
mkdir -p ~/axion
cd ~/axion
repo init -u https://github.com/AxionAOSP/android.git -b lineage-23.2 --git-lfs
```

- `-b lineage-23.2` は Axion の Android 16 用ブランチです。これより新しいブランチが公開されている場合は、そちらに合わせてください。
- 「Select the color display?」のような質問が出たら、`y` か `n` のどちらでも構いません。

### 3-2. この local_manifests を置く

`repo init` で `.repo/` ができているので、その中に `local_manifests` として取得します。

```bash
git clone -b ax-pipa-16 https://github.com/amiami5/local_manifests .repo/local_manifests
```

- ブランチ名の `ax-pipa-16` は、Axion 向けの Android 16 用という意味です。
- 「already exists and is not an empty directory」と出たら、すでに `.repo/local_manifests` があります。中身を確認して、不要なら削除してからやり直してください。
- 同じブランチに `cepheus.xml` (Xiaomi Mi 9 用) も入っています。pipa のビルドには要りませんが、あっても害はありません。取得するリポジトリを減らしたいときは、`mv .repo/local_manifests/cepheus.xml .repo/local_manifests/cepheus.xml.bak` のように、拡張子を変えてください。

### 3-3. ソースを同期する

```bash
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
```

- 数時間かかります。途中で止まったり、通信エラーが出たりしたら、**同じコマンドをもう一度実行**すれば、続きから再開します。
- `-j8` は同時に取得する数です。回線が細いときは `-j4` に下げてください。
- 終わると、`device/xiaomi/pipa` や `kernel/xiaomi/sm8250` などのフォルダができます。

**うまくいかないとき**
- `error: Cannot fetch ...` が出る: ネットワークの一時的なエラーです。同じコマンドを再実行してください。
- 取得先が見つからない: `pipa.xml` に書かれている URL の先が移動・非公開になっている可能性があります。エラーに出たリポジトリ名を調べてください。

---

## 4. ビルドする

ビルドは、**必ず `~/axion` (ソースの一番上のフォルダ) の中で**実行します。

### 4-1. ビルド環境を読み込む

```bash
cd ~/axion
source build/envsetup.sh
```

### 4-2. ターゲットを選ぶ

```bash
axion pipa userdebug gms
```

`axion` は Axion が用意しているコマンドです。引数の意味は次のとおりです。

| 引数 | 意味 |
|---|---|
| `pipa` | 端末のコードネーム |
| `userdebug` | ビルドの種類。まず `userdebug` を選ぶのがおすすめです (adb root やログが使えて、不具合調査がしやすい)。`user` は配布用、`eng` は開発用です。 |
| `gms` | Google アプリ入り。ほかに `pico` (最小の Google アプリ)、`core`、`va` (Google アプリなし) があります。 |

このとき、`device/xiaomi/pipa/vendorsetup.sh` が自動で動き、タブレット用の修正 (`frameworks/base` と `hardware/google/pixel` へのパッチ) を当てます。「Tablet patch applied successfully」のような表示が出れば正常です。2 回目以降は「already applied or not applicable; skipping」のような表示になりますが、それも正常です。

### 4-3. ビルドを実行する

```bash
ax -br
```

- `ax -br` は、`brunch` (ビルドして zip まで作る) を実行します。
- **初回は数時間〜10 時間以上**かかります。途中で席を外して構いません。
- CPU を使い切るので、ビルド中の PC は重くなります。

メモリ不足で落ちる場合は、並列数を減らします。

```bash
ax -br -j6
```

### 4-4. ターミナルを閉じてしまったとき

ビルドの続きではなく、環境を読み込み直してからもう一度実行します。作りかけの部分は使い回されるので、最初からやり直しにはなりません。

```bash
cd ~/axion
source build/envsetup.sh
axion pipa userdebug gms
ax -br
```

---

## 5. できあがったものを確認する

ビルドが成功すると、次の場所にできます。

```
~/axion/out/target/product/pipa/
```

その中の `axion-…-pipa.zip` が、端末に入れるためのファイルです。同じ場所に `boot.img` や `recovery.img` なども作られます。

---

## 6. 端末に入れる (概要)

この README はビルドまでの手順です。端末への導入は、必ず**自分の端末の状態に合わせて**確認してから行ってください。

- ブートローダーのアンロックと、リカバリーの導入が済んでいることが前提です。
- 一般的な流れは、「リカバリーに入る → 必要なら初期化 (データが消えます) → `adb sideload` で zip を入れる」です。
- **初期化するとデータがすべて消えます。** 先にバックアップを取ってください。

---

## 7. ソースを更新するとき

```bash
cd ~/axion
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
source build/envsetup.sh
axion pipa userdebug gms
ax -br
```

- `frameworks/base` には、`vendorsetup.sh` が手元でコミットを 1 つ足しています。同期でコンフリクトが出たときは、そのコミットが原因のことがあります。

---

## 8. よくあるトラブル

| 症状 | 対処 |
|---|---|
| `repo: command not found` | 2-1 の `repo` の入れ方を実行し、`PATH` を通してください。 |
| 同期が途中で止まる | 同じ `repo sync` をもう一度実行してください。 |
| ビルド中に `Killed` と出て止まる | メモリ不足です。`ax -br -j6` のように並列数を下げ、スワップも用意してください。 |
| `No space left on device` | ディスクが足りません。400GB 以上の空きを確保してください。 |
| 「○○: command not found」 | 足りないパッケージです。`apt install` で追加してください。 |
| `lunch` や `axion` で「Cannot locate config makefile」 | `source build/envsetup.sh` をソースの一番上のフォルダで実行したか、確認してください。 |
| ビルドが途中で失敗した | エラーの最後の数十行を読んでください。最初に `FAILED:` と出た行が原因です。 |

---

## 9. 参考

- Axion AOSP: <https://github.com/AxionAOSP/android>
- この local_manifests: <https://github.com/amiami5/local_manifests>
