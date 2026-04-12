# EVENT_CAMERA - イベントカメラ分析プロジェクト

NAIST での研究室向けイベントカメラ（動的視覚センサ）の分析・可視化プロジェクトです。

---

## ファイル構成

### 分析スクリプト・ノートブック

| ファイル | 説明 | 用途 |
|---------|------|------|
| **`minimal_voxel_analysis.py`** | Voxel Grid 変換・分析の **メイン実行スクリプト** | 分析手法の共有・本運用 |
| `minimal_voxel_analysis.ipynb` | 上記の Jupyter Notebook 版 | 対話的な分析・実験 |

#### `minimal_voxel_analysis.py` の内容
- **イベントデータ読込**：HDF5 形式のイベントカメラデータを `EventsIterator` で処理
- **Voxel Grid 変換**：イベント座標と時刻を 3D グリッド（B × H × W）に変換
  - Bilinear temporal interpolation を使用
  - NUM_BINS=10 の時間ビンに分割
- **ビン間差分計算**：ΔV[b] = V[b+1] - V[b] で動きを抽出
- **重心軌跡計算**：各差分ステップの重心（Y, X）を算出
- **可視化**：差分 Voxel と重心軌跡を PNG で出力

**実行方法：**
```bash
python minimal_voxel_analysis.py
```

**出力:**
- `minimal_diff_voxels.png` - ビン間差分の可視化（19枚タイル）
- `minimal_centroid_trajectory.png` - 重心軌跡グラフ

---

### 出力結果・図表

| ファイル | 説明 |
|---------|------|
| `minimal_diff_voxels.png` | Voxel Grid の時系列変化（4枚ごとに折り返し） |
| `minimal_centroid_trajectory.png` | 重心 Y 座標の軌跡と変化率 |

---

### セットアップガイド

| ファイル | 説明 | 対象環境 |
|---------|------|---------|
| **`WSL2(Ubuntu).md`** | Metavision/OpenEB 環境構築手順 | Windows 11 WSL2 Ubuntu |
| **`RaspberryPi.md`** | Raspberry Pi 5 + GenX320 セットアップ | Raspberry Pi OS |

#### `WSL2(Ubuntu).md` の内容
- Ubuntu の基本設定（apt update/install）
- Python 仮想環境の構築
- Metavision SDK/OpenEB のインストール
- 依存ライブラリ（numpy, matplotlib など）のセットアップ

#### `RaspberryPi.md` の内容
- Raspberry Pi 5 の初期設定手順
- イベントカメラ（GenX320）との接続
- データ撮影・変換方法
- WSL2 ホストへのデータ転送手順

---

### ドキュメント・ガイド

| ファイル | 説明 |
|---------|------|
| **`Github使用ガイド.md`** | GitHub 操作時の注意点とベストプラクティス |
| `.gitignore` | Git の追跡除外設定 |

#### `Github使用ガイド.md` の内容
- **今回の問題と原因**：ローカル `master` とリモート `main` の履歴が分岐した経緯
- **GitHub 使用時の注意点**：ブランチ作成・プッシュの正しい流れ
- **推奨ワークフロー**：チーム開発での手順（Feature Branch → Pull Request → Merge）
- **問題対処法**：トラブル時のコマンド解説
- **参考コマンド**：よく使う Git コマンド一覧

#### `.gitignore` の設定
```
# 追跡対象外
__pycache__/
.Python
.venv/

# 大容量ファイル
*.hdf5
*.h5
*.npy
*.npz
*.mp4

# データディレクトリ全体
data/
```

---

## クイックスタート

### 1. リポジトリのクローン
```bash
git clone https://github.com/Ryunoshin3150/EVENT_CAMERA.git
cd EVENT_CAMERA
```

### 2. 環境構築（WSL2 Ubuntu の場合）

```bash
# WSL2(Ubuntu).md を参照
# または以下で簡単セットアップ
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt  # 別途作成が必要
```

### 3. 分析実行

```bash
# data/ フォルダに HDF5 ファイルを配置
python minimal_voxel_analysis.py
```

### 4. 結果確認
- `minimal_diff_voxels.png` - 差分 Voxel の時系列
- `minimal_centroid_trajectory.png` - 重心軌跡

---

## データ形式

### 入力：イベントカメラデータ
- **フォーマット**：HDF5 (`.hdf5`)
- **サンプル**：`data/FALL_DOWN_*.hdf5`, `data/SIT_DOWN_*.hdf5`, `data/STAND_UP_*.hdf5`
- **フィールド**：`x`（X座標）, `y`（Y座標）, `t`（タイムスタンプ）, `p`（極性）

### 出力：画像・グラフ
- PNG 形式（解像度 dpi=100）
- 重心軌跡グラフ（matplotlib）

---

## カスタマイズ可能なパラメータ

`minimal_voxel_analysis.py` 内で調整可能：

```python
# 画像サイズ
HEIGHT = 320          # ピクセル高
WIDTH = 320           # ピクセル幅

# 時間ビン数
NUM_BINS = 10         # 時刻を10段階に分割

# 分析区間
START_TIME_S = 5.0    # 開始時刻（秒）
DURATION_S = 2.0      # 分析期間（秒）

# ファイルパス
DATA_DIR = "data"     # HDF5 ファイルの格納フォルダ
```

---

## 技術スタック

| 項目 | ツール・ライブラリ |
|------|------------------|
| **言語** | Python 3.8+ |
| **イベント処理** | Metavision SDK / OpenEB |
| **数値計算** | NumPy |
| **可視化** | Matplotlib, japanize-matplotlib |
| **ファイルI/O** | HDF5 |
| **バージョン管理** | Git / GitHub |

---

## 分析手法の概要

### Voxel Grid 変換

イベントカメラのスパース出力を密集表現に変換：

$$V[b, y, x] = \sum_{(x_i, y_i, t_i, p_i)} p_i \cdot w(b - t_i^*)$$

ここで：
- $b$：時間ビン
- $t_i^*$：正規化タイムスタンプ
- $w()$：Bilinear kernel

### ビン間差分

時間経過での変化を抽出：

$$\Delta V[b] = V[b+1] - V[b]$$

- **正値**：物体が接近（イベント増加）
- **負値**：物体が離脱（イベント減少）

### 重心軌跡

各時刻での動作の位置を追跡：

$$cy[b] = \frac{\sum_{y,x} |\Delta V[b, y, x]| \cdot y}{\sum_{y,x} |\Delta V[b, y, x]|}$$

---

## ブランチ構成

| ブランチ | 用途 | 説明 |
|---------|------|------|
| `main` | **本体** | リリース版・安定版 |
| `ryunoshin` | 開発 | 新機能開発・実験用 |

### GitHub での操作
1. 新機能は `ryunoshin` ブランチで開発
2. **Pull Request** を作成して `main` にマージ
3. 詳細は `Github使用ガイド.md` を参照

---

## 注意事項

### Git 操作時
- **`git push -f` は個人ブランチのみ使用**
- 共有ブランチ（`main`）では使用禁止
- プッシュ前に `git log` で確認

### データ管理
- `data/` フォルダは `.gitignore` で除外
- HDF5 ファイルは Git リポジトリに含まれない
- 大容量ファイルは別途配布（USB、クラウドなど）

### 環境依存性
- Metavision SDK は Linux でのみ動作（Windows は WSL2 版を使用）
- Python バージョン確認：`python --version`

---

---

**最終更新：2026年4月12日**
