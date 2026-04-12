# Raspberry Pi 5 + Event Camera (GenX320) セットアップ手順

## 概要
本ドキュメントでは、Raspberry Pi 5 にイベントカメラ（GenX320）を接続し、撮影・データ変換・転送までの手順をまとめる。

---

## 基本情報

- Raspberry Pi IPアドレス: `172.20.10.7`
- ユーザー名: `eventcamera`
- パスワード: 箱の上に記載

---

## 1. Raspberry Pi OS の準備（準備済み）

### Imager ダウンロード
https://www.raspberrypi.com/software/

例: `imager_2.0.6.exe`

### 手順
- Raspberry Pi Imager を起動
- OS を選択
- SDカードに書き込み
- Raspberry Pi 5 に挿入

---

## 2. 初期セットアップ

### パスワード変更

```bash
sudo passwd
```

現在のパスワードを入力し、新しいパスワードを設定。

---

## 3. カメラ接続後の初期設定

```bash
cd ~/rpi-sensor-drivers

# カメラドライバ読み込み
sudo dtoverlay genx320

# カメラ設定
./rp5_setup_v4l.sh

# 環境変数
export PSEE_VAR_V4L2_BSIZE=1
export V4L2_HEAP=vidbuf_cached

# カメラ起動
metavision_viewer
```

---

## 4. 動作確認（コピペ用）

```bash
cd ~/rpi-sensor-drivers

sudo dtoverlay genx320,cam0
sudo dtoverlay genx320,cam1

dmesg | grep genx

./rp5_setup_v4l.sh

export PSEE_VAR_V4L2_BSIZE=1
export V4L2_HEAP=vidbuf_cached

metavision_viewer
```

---

## 5. 撮影

```bash
# ライブカメラを開く（何も指定しない）
metavision_viewer

# 録画（ファイル名指定可能）
metavision_viewer -o ファイル名.raw

# 再生
metavision_viewer -i ファイル名.raw
```

---

## 6. データ変換

### RAW → HDF5

```bash
metavision_file_to_hdf5 -i ファイル名.raw -o ファイル名.hdf5
```

### RAW → 動画（MP4）

```bash
cd ~

# ステップ1：RAW → AVI に変換
metavision_file_to_video -i ファイル名.raw -o ファイル名.avi

# ステップ2：AVI → MP4 に変換
ffmpeg -i ファイル名.avi ファイル名.mp4
```

---

## 7. Bias（カメラ設定）の切り替え

```bash
# デフォルト設定で録画
metavision_viewer -o ファイル名.raw

# 各bias設定で録画
metavision_viewer -b "/home/eventcamera/ダウンロード/genX320_lowlight.bias" -o ファイル名.raw
metavision_viewer -b "/home/eventcamera/ダウンロード/genX320_LED.bias" -o ファイル名.raw
metavision_viewer -b "/home/eventcamera/ダウンロード/genX320_PSM.bias" -o ファイル名.raw
metavision_viewer -b "/home/eventcamera/ダウンロード/genX320_AM.bias" -o ファイル名.raw
```

Bias設定ドキュメント: https://docs.prophesee.ai/stable/hw/manuals/biases.html

---

## 8. カメラの切り替え

```bash
metavision_viewer -s "genx320 10-003c"
metavision_viewer -s "genx320 11-003c"
```

---

## 9. オプション一覧

| オプション | 内容 |
|---------|------|
| `-i` | 保存済みRAWファイルを読み込み（再生） |
| `-o` | 録画データの保存先指定 |
| `-b` | bias設定ファイル指定 |
| `-j` | JSON設定ファイル読み込み |
| `-r` | ROI（撮影範囲）指定 |
| `-d` | Subsampling（間引き）指定 |
| なし | ライブカメラを開く |

---

## 10. 動画ファイルの確認・再生

```bash
# ファイルサイズ確認
ls -lh ファイル名.mp4

# 再生
vlc ファイル名.mp4
```

---

## 11. PCへの転送

```bash
scp eventcamera@172.20.10.7:~/rpi-sensor-drivers/ファイル名.mp4 .
scp eventcamera@172.20.10.7:~/rpi-sensor-drivers/ファイル名.raw .
scp eventcamera@172.20.10.7:~/rpi-sensor-drivers/ファイル名.hdf5 .
```

---

## 12. トラブルシューティング

### ❌ カメラが認識されない

```bash
dmesg | grep genx
sudo dtoverlay genx320,cam0
sudo dtoverlay genx320,cam1
./rp5_setup_v4l.sh
export PSEE_VAR_V4L2_BSIZE=1
export V4L2_HEAP=vidbuf_cached
metavision_viewer
```

### ❌ ffmpeg が見つからない

```bash
sudo apt install -y ffmpeg
ffmpeg -i ファイル名.avi ファイル名.mp4
```

### ❌ 環境変数が反映されない

```bash
export PSEE_VAR_V4L2_BSIZE=1
export V4L2_HEAP=vidbuf_cached
echo $V4L2_HEAP
```

---

## 13. バックアップ

**Windows（推奨：Win32DiskImager）**

1. https://sourceforge.net/projects/win32diskimager/ からダウンロード
2. Device: `\\.\PhysicalDriveX` （SDカード番号）
3. Image File: `backup.img` を指定
4. Read ボタン → 完了まで待機（15〜30分）

復元時は Write ボタン使用。

**Mac/Linux**

```bash
sudo dd if=/dev/sdX of=backup.img bs=4M status=progress
sudo dd if=backup.img of=/dev/sdX bs=4M status=progress
```

---

## 参考資料

- Bias設定: https://docs.prophesee.ai/stable/hw/manuals/biases.html
- Raspberry Pi Imager: https://www.raspberrypi.com/software/
```
