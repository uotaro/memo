# Milvus Standalone v2.6 導入手順

1. [STEP 1：Docker のインストール（WSL2側）](#step-1docker-のインストールwsl2側)
1. [STEP 2：NVIDIA Container Toolkit の導入（GPU 利用する場合）](#step-2nvidia-container-toolkit-の導入gpu-利用する場合)
1. [STEP 3：Milvus v2.6 の起動](#step-3milvus-v26-の起動)
1. [STEP 4：Milvus のポート番号設定変更、動作設定変更など](#step-4milvus-のポート番号設定変更動作設定変更など)
1. [STEP 5：Python SDK のインストール](#step-5python-sdk-のインストール)

## 起動・停止コマンド
作業ディレクトリに移動してから、

| 動作 | コマンド                |
|------|------------------------|
| 起動 | `docker compose up -d` |
| 停止 | `docker compose down`  |


## 導入環境
WSL2 / Ubuntu 22.04

| 項目       | スペック・バージョンなど                               |
| ---------- | --------------------------------------------------- |
| OS         | Windows 11 + WSL2 (Ubuntu 22.04)                    |
| CPU        | AMD Ryzen 9 8940HX with Radeon Graphics (2.40 GHz)  |
| GPU        | NVIDIA GeForce RTX 5070 Laptop                      |
| メモリ     | 32.0GB                                              |
| VRAM       | 8.0GB                                              |
| ストレージ | 954GB                                               |
| Python     | 3.10.12                                             |
| CUDA:      | 12.9.1                                              |
| PyTorch    | 2.5.1                                               |
| VSCode     | ver1.111.0                                          |
| Node.js    | ver22.22.0                                          |

現在の最新安定版は v2.6.16（2025年5月時点）、さらに v3.0-beta も公開されています。  
ここでは v2.6.x を対象に、Docker Compose を使った導入手順を説明します。

## 今後 sudo 不要にする設定やってると楽
すでに以前 sudo usermod -aG docker $USER は実行済みのはずですが、念のため確認：

```bash
groups
```

出力に docker が含まれていれば設定済みです。含まれていなければ：

```
sudo usermod -aG docker $USER
newgrp docker
```

## STEP 1：Docker のインストール（WSL2側）
WSL2 の Ubuntu 22.04 上で以下を実行します。  
任意のディレクトリでOK。

```bash
# 古い Docker があれば削除
sudo apt remove docker docker-engine docker.io containerd runc

# 必要パッケージのインストール
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Docker 公式 GPG キー追加
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# リポジトリ追加
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker Engine + Compose V2 インストール
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# sudo なしで使えるようにする
sudo usermod -aG docker $USER
newgrp docker
```

##### ⚠️ WSL2 では systemctl が使えない場合があるため、Docker デーモンは手動起動します。

```bash
sudo service docker start
# または
sudo dockerd &
```

## STEP 2：NVIDIA Container Toolkit の導入（GPU 利用する場合）

```bash
# Toolkit リポジトリ追加
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Docker が GPU を使えるよう設定
sudo nvidia-ctk runtime configure --runtime=docker
sudo service docker restart

# 動作確認
docker run --rm --gpus all nvidia/cuda:12.9.0-base-ubuntu22.04 nvidia-smi
```

最後の動作確認を実行すると以下のように表示されればOK

```bash
$ docker run --rm --gpus all nvidia/cuda:12.9.0-base-ubuntu22.04 nvidia-smi
Unable to find image 'nvidia/cuda:12.9.0-base-ubuntu22.04' locally
12.9.0-base-ubuntu22.04: Pulling from nvidia/cuda
1bba15468fcc: Pull complete
5ada09cfb5af: Pull complete
215ed5a63843: Pull complete
b4d600b97743: Pull complete
01a77ecc44d6: Pull complete
Digest: sha256:499898d51d158ad11c3ae6defa7e80829b2be04951212dfa80faf99b99a37dfe
Status: Downloaded newer image for nvidia/cuda:12.9.0-base-ubuntu22.04
Sun May 17 02:01:34 2026
+-----------------------------------------------------------------------------------------+  # NVIDIA Container Toolkit 正常動作を確認
| NVIDIA-SMI 590.62                 Driver Version: 591.97         CUDA Version: 13.1     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 5070 ...    On  |   00000000:01:00.0  On |                  N/A |  # ← RTX 5070 が認識されている
| N/A   51C    P8              5W /   45W |     804MiB /   8151MiB |      0%      Default |  # VRAM 815MB を確認
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## STEP 3：Milvus v2.6 の起動

```bash
# 作業ディレクトリ作成
mkdir ~/milvus && cd ~/milvus

# Docker Compose ファイルをダウンロード（v2.6.16）
wget https://github.com/milvus-io/milvus/releases/download/v2.6.16/milvus-standalone-docker-compose.yml \
  -O docker-compose.yml

# 起動
docker compose up -d
```

Docker Compose ファイルをダウンロードコマンドを実行したら、以下のように表示されます

```bash
$ wget https://github.com/milvus-io/milvus/releases/download/v2.6.16/milvus-standalone-docker-compose.yml \
  -O docker-compose.yml
--2026-05-17 11:07:31--  https://github.com/milvus-io/milvus/releases/download/v2.6.16/milvus-standalone-docker-compose.yml
github.com (github.com) をDNSに問いあわせています... 20.27.177.113
github.com (github.com)|20.27.177.113|:443 に接続しています... 接続しました。
HTTP による接続要求を送信しました、応答を待っています... 302 Found
場所: https://release-assets.githubusercontent.com/github-production-release-asset/208728772/00bd3f1e-fd77-461b-a39d-7ad6172bf0c4?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-05-17T03%3A01%3A56Z&rscd=attachment%3B+filename%3Dmilvus-standalone-docker-compose.yml&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-05-17T02%3A01%3A22Z&ske=2026-05-17T03%3A01%3A56Z&sks=b&skv=2018-11-09&sig=WE3aAqEe7Au3GDMTIWLWwlfh%2BtxlKfJjwmWkVVx2nIo%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc3ODk4Mzk1MSwibmJmIjoxNzc4OTgzNjUxLCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.3R_hY7tmRlgsg7C4_S5mhOBbhu9cVucESZvJBigbXlw&response-content-disposition=attachment%3B%20filename%3Dmilvus-standalone-docker-compose.yml&response-content-type=application%2Foctet-stream [続く]
--2026-05-17 11:07:32--  https://release-assets.githubusercontent.com/github-production-release-asset/208728772/00bd3f1e-fd77-461b-a39d-7ad6172bf0c4?sp=r&sv=2018-11-09&sr=b&spr=https&se=2026-05-17T03%3A01%3A56Z&rscd=attachment%3B+filename%3Dmilvus-standalone-docker-compose.yml&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2026-05-17T02%3A01%3A22Z&ske=2026-05-17T03%3A01%3A56Z&sks=b&skv=2018-11-09&sig=WE3aAqEe7Au3GDMTIWLWwlfh%2BtxlKfJjwmWkVVx2nIo%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc3ODk4Mzk1MSwibmJmIjoxNzc4OTgzNjUxLCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.3R_hY7tmRlgsg7C4_S5mhOBbhu9cVucESZvJBigbXlw&response-content-disposition=attachment%3B%20filename%3Dmilvus-standalone-docker-compose.yml&response-content-type=application%2Foctet-stream
release-assets.githubusercontent.com (release-assets.githubusercontent.com) をDNSに問いあわせています... 185.199.109.133, 185.199.111.133, 185.199.108.133, ...
release-assets.githubusercontent.com (release-assets.githubusercontent.com)|185.199.109.133|:443 に接続しています... 接続しました。
HTTP による接続要求を送信しました、応答を待っています... 200 OK
長さ: 1789 (1.7K) [application/octet-stream]
‘docker-compose.yml’ に保存中
docker-compose.yml            100%[=================================================>]   1.75K  --.-KB/s    in 0s
2026-05-17 11:07:32 (38.4 MB/s) - ‘docker-compose.yml’ へ保存完了 [1789/1789]
# docker-compose.yml が正常にダウンロードできていることを確認できる
ct1485@my-wsl:~/_local_wsl/08_tools/milvus$ ll
合計 12
drwxr-xr-x 2 ct1485 ct1485 4096  5月 17 11:07 ./
drwxr-xr-x 6 ct1485 ct1485 4096  5月 17 10:53 ../
-rw-r--r-- 1 ct1485 docker 1789  5月 14 13:14 docker-compose.yml
```

起動成功したら、以下のように表示されます

```bash
$ docker compose up -d
WARN[0000] /home/ct1485/_local_wsl/08_tools/milvus/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
[+] up 40/40
 ✔ Image quay.io/coreos/etcd:v3.5.25              Pulled                                                           14.7s
 ✔ Image minio/minio:RELEASE.2024-12-18T13-15-44Z Pulled                                                           12.3s
 ✔ Image milvusdb/milvus:v2.6.16                  Pulled                                                           42.8s
 ✔ Network milvus                                 Created                                                           0.0s
 ✔ Container milvus-minio                         Started                                                           0.8s
 ✔ Container milvus-etcd                          Started                                                           0.7s
 ✔ Container milvus-standalone                    Started                                                           0.6s
ct1485@my-wsl:~/_local_wsl/08_tools/milvus$
```

`docker compose ps` コマンドで、全コンテナの状態を確認できる。

```bash
$ docker compose ps
WARN[0000] /home/ct1485/_local_wsl/08_tools/milvus/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
NAME                IMAGE                                      COMMAND                   SERVICE      CREATED         STATUS                   PORTS
milvus-etcd         quay.io/coreos/etcd:v3.5.25                "etcd -advertise-cli…"   etcd         3 minutes ago   Up 3 minutes (healthy)   2379-2380/tcp
milvus-minio        minio/minio:RELEASE.2024-12-18T13-15-44Z   "/usr/bin/docker-ent…"   minio        3 minutes ago   Up 3 minutes (healthy)   0.0.0.0:9000-9001->9000-9001/tcp, [::]:9000-9001->9000-9001/tcp
milvus-standalone   milvusdb/milvus:v2.6.16                    "/tini -- milvus run…"   standalone   3 minutes ago   Up 3 minutes (healthy)   0.0.0.0:9091->9091/tcp, [::]:9091->9091/tcp, 0.0.0.0:19530->19530/tcp, [::]:19530->19530/tcp
```

上記応答で、以下の結果が確認できたことになる

| コンテナ           | status     |
|-------------------|------------|
| milvus-etcd       | ✅ healthy |
| milvus-minio      | ✅ healthy |
| milvus-standalone | ✅ healthy |

3つ全て healthy になっており、Milvus v2.6.16 が正常稼働しています。  
ポート 19530（SDK接続用）と 9091（WebUI用）もホストに公開されています。

## STEP 4：Milvus のポート番号設定変更、動作設定変更など
### ポート番号設定変更
ポート番号の変更は user.yaml ではなく docker-compose.yml の編集で行います。  
理由は、ポートのバインドは Docker のネットワーク設定であり、Milvus アプリ内部の設定ファイルではないためです。

##### docker-compose.yml を直接編集する

```bash
nano ~/milvus/docker-compose.yml  # または好きなエディタで
```

該当箇所（standalone サービスの ports セクション）を編集します。
以下は、milvusへのアクセスをデフォルトの `19530` から `8190` に変更する場合のサンプルです。

```yaml
standalone:
    ports:
      - "8190:19530"   # ← 19530 を 8190 に変更
      - "9091:9091"    # WebUI はそのまま（必要に応じて変更可）
```

例えばホスト側を 195819031 に変えたい場合：

```yaml
- "8190:19530"   # ホスト:コンテナ
```

編集後に再起動：

```bash
docker compose down
docker compose up -d
```

#### LAN内の他マシンからアクセスできるようにするには
IPアドレスが分かっても、初期状態のWSL2は外部からの通信をブロックしてしまいます。  
他のマシンからアクセスできるようにするには、**Windows側で「ポートフォワーディング（中継設定）」を行う必要**があります。

##### ステップ1：UbuntuのIPアドレスを調べる
まず、Ubuntuのターミナルで自身の内部IPを調べます。

```bash
hostname -I
```

`172.19.132.72` のような値が返ってきます。複数でてきたら、**一番左がWindowsとUbuntuを繋いでいるメインのIPアドレス** なはず。

##### ステップ2：Windowsで中継設定（ポートフォワーディング）を行う
Windowsのスタートボタンを右クリック ＞ 「ターミナル（管理者）」または「PowerShell（管理者）」 を開きます。  
以下のコマンドの値を書き換えて実行してください

```bash
# 例：Ubuntu上の8190番ポート（Webアプリ等）にアクセスさせたい場合
netsh interface portproxy add v4tov4 listenport=8190 listenaddress=0.0.0.0 connectport=8190 connectaddress=[ステップ1で調べたUbuntuのIP]

# 例：Web UI (ブラウザで http://[ステップ1で調べたUbuntuのIP]:9091/webui を開くと Milvus の管理画面が見られる) の ポートも通しておく場合
netsh interface portproxy add v4tov4 listenport=9091 listenaddress=0.0.0.0 connectport=9091 connectaddress=[ステップ1で調べたUbuntuのIP]
```

##### 他のマシンからのアクセス方法
設定が完了したら、他のマシン（スマホや別PC）のブラウザなどから以下のようにアクセスします。

```bash
http://[WindowsのLAN内IP]:9091
```


### Milvus 内部の動作設定
`user.yaml` を docker-compose.yml と同一ディレクトリに作成して、`volumes` でマウントする形になります。

```yaml
standalone:
    volumes:
      - /home/ct1485/_local_wsl/08_tools/milvus/user.yaml:/milvus/configs/user.yaml  # ←追加（絶対パスで指定する例）
```

#### ログを指定ファイルパスに出力したい場合

##### Milvus デフォルトのログ出力先
Milvus のログ出力は、デフォルトではファイルには出力されず、標準出力（stdout）のみに出力されます。  
Docker の場合はコンテナのログとして管理されます。

```bash
# リアルタイムでログを確認
docker logs -f milvus-standalone

# 直近 100 行だけ確認
docker logs --tail 100 milvus-standalone
```

ログをファイルに出力したい場合の設定方法は以下のとおりです。

##### ログ設定_step1. user.yaml を作成
`user.yaml` を docker-compose.yml と同一ディレクトリに作成

```bash
cd <docker-compose.yml と同一ディレクトリ>
nano user.yaml
```

以下の内容を user.yaml に記述します：

```yaml
log:
  level: info
  file:
    rootPath: /milvus/logs
    maxSize: 100
    maxBackups: 10
    maxAge: 30
```

##### ログ設定_step2. 再起動して反映

```bash
cd <docker-compose.yml と同一ディレクトリ>
docker compose down
docker compose up -d
```

##### ログ設定_step3. ログの確認
ユーザー root で、パーミッション 600 で吐き出されるため、sudo で読む必要がある。

```bash
cd <docker-compose.yml と同一ディレクトリ>
sudo tail -f ./volumes/logs/standalone-0.log
```


## STEP 5：Python SDK のインストール
**python 仮想環境に入り**、下記コマンドを実行して pymilvus モジュールをインストールします。

```bash
pip install pymilvus
```


