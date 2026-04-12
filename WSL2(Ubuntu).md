# Metavision/OpenEB 環境構築 — クイック実行ガイド

## フェーズ1：Ubuntu基本設定

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv git
sudo timedatectl set-timezone Asia/Tokyo
timedatectl
```

**所要時間：5分**

## フェーズ2：プロジェクトフォルダと仮想環境の作成

```bash
mkdir -p ~/naist_event
cd ~/naist_event
python3 -m venv .venv
source .venv/bin/activate
python -c "import sys; print(sys.executable)"
```

出力が python なら成功。

**所要時間：1分**

## フェーズ3：Metavision/OpenEB のインストール

```bash
deactivate

# 事前確認（何も出力されないが正常）
python -m pip list | grep -i metavision
ls /usr/lib/python3/dist-packages | grep metavision

# Prophesee リポジトリを追加
sudo apt -y install curl software-properties-common
curl -L https://propheseeai.jfrog.io/artifactory/api/security/keypair/prophesee-gpg/public -o /tmp/propheseeai.jfrog.op.asc
sudo cp /tmp/propheseeai.jfrog.op.asc /etc/apt/trusted.gpg.d/
sudo add-apt-repository 'https://propheseeai.jfrog.io/artifactory/openeb-debian/'
sudo apt update

# Metavision をインストール
sudo apt -y install metavision-openeb
```

**所要時間：10-15分**

## フェーズ4：仮想環境での動作確認

```bash
cd ~/naist_event
source .venv/bin/activate

# Python 検索パスを拡張
case ":$PYTHONPATH:" in
    *:/usr/lib/python3/dist-packages:*) ;;
    *) export PYTHONPATH=/usr/lib/python3/dist-packages:$PYTHONPATH ;;
esac

# 所在を確認
find /usr -name "metavision_core*"

# インポート成功確認
python -c "import metavision_core; print(metavision_core.__file__)"
```

**所要時間：2分**

## フェーズ5：永続的な設定（重要）

```bash
nano ~/.bashrc
```

ファイルの最後に以下を追加：

```bash
# Metavision を仮想環境から参照可能にする
case ":$PYTHONPATH:" in
    *:/usr/lib/python3/dist-packages:*) ;;
    *) export PYTHONPATH=/usr/lib/python3/dist-packages:$PYTHONPATH ;;
esac
```

保存：`Ctrl + O` → `Enter` → `Ctrl + X`

```bash
source ~/.bashrc
```

**所要時間：2分**

## 最終確認（一括チェック）

```bash
cd ~/naist_event
source .venv/bin/activate

echo "=== 環境チェック ==="
echo "1. Python version:"
python --version

echo -e "\n2. Python path:"
python -c "import sys; print(sys.executable)"

echo -e "\n3. Metavision import:"
python -c "import metavision_core; print('OK')"

echo -e "\n4. Metavision location:"
python -c "import metavision_core; print(metavision_core.__file__)"

echo -e "\n=== すべてOKなら環境構築完了 ==="
```

## トラブルシューティング

| 症状 | 確認コマンド | 解決策 |
|------|------------|------|
| `ModuleNotFoundError: No module named 'metavision_core'`（仮想環境外） | `deactivate` → `python3 -c "import metavision_core"` | フェーズ3を再実行 |
| `apt install metavision-openeb` でパッケージが見つからない | `apt-cache search metavision` | フェーズ3 リポジトリ追加を再実行 |
| .bashrc の設定が反映されない | `echo $PYTHONPATH` | `source ~/.bashrc` を実行 |
---

このバージョンをお使いください。naist_event フォルダが既に存在するため、フェーズ2 から開始できます。
