# WSL2（Ubuntu）でのMetavision/OpenEB環境構築ガイド

## フェーズ1：Ubuntu初期設定

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv git
sudo timedatectl set-timezone Asia/Tokyo
timedatectl
```

## フェーズ2：Python開発環境構築

```bash
cd ~/naist_event
python3 -m venv .venv
source .venv/bin/activate
python -c "import sys; print(sys.executable)"
```

出力が python なら成功。

## フェーズ3：Metavision/OpenEBのインストール

```bash
deactivate
sudo apt -y install curl software-properties-common
curl -L https://propheseeai.jfrog.io/artifactory/api/security/keypair/prophesee-gpg/public -o /tmp/propheseeai.jfrog.op.asc
sudo cp /tmp/propheseeai.jfrog.op.asc /etc/apt/trusted.gpg.d/
sudo add-apt-repository 'https://propheseeai.jfrog.io/artifactory/openeb-debian/'
sudo apt update
sudo apt -y install metavision-openeb
```

## フェーズ4：仮想環境での動作確認と永続設定

```bash
source .venv/bin/activate
export PYTHONPATH=/usr/lib/python3/dist-packages:$PYTHONPATH
python -c "import metavision_core; print(metavision_core.__file__)"
```

出力が `/usr/lib/python3/dist-packages/metavision_core.so` なら成功。

### 永続設定（.bashrc への追加）

```bash
nano ~/.bashrc
```

ファイルの最後に追加：

```bash
# Metavision を仮想環境から参照可能にする
if [ -z "$VIRTUAL_ENV" ]; then
    export PYTHONPATH=/usr/lib/python3/dist-packages:$PYTHONPATH
fi
```

保存：`Ctrl + O` → `Enter` → `Ctrl + X`

```bash
source ~/.bashrc
```

## 確認チェックリスト

```bash
lsb_release -a                                          # Ubuntuバージョン確認
python --version                                        # Python 3.8以上
python -c "import metavision_core"                      # 仮想環境外で確認
source .venv/bin/activate
python -c "import metavision_core"                      # 仮想環境内で確認
```

すべて成功すれば完了です。

---
