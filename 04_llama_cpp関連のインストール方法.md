# llama_cpp関連のインストール方法

1. [CUDAツールキット nvcc のインストール](#install_nvcc)
2. [Ubuntu用llama-cppをソースからビルド(CUDAありのため)](#install_llama_cpp)
3. [llama.cpp サーバーの起動・停止方法](#llamacpp_server_start_stop)
4. [llama.cpp サーバーにアクセスして生成](#llamacpp_server_generate)
5. [llama_cpp_python のインストール](#install_llama_cpp_python)


---

<div id="install_nvcc"></div>

## 1. CUDAツールキット(nvcc)のインストール

現在のマシンは NVIDIA製のGPU：RTX 5070 搭載かつ CUDA 12.9 対応ドライバーが認識されている状態。  


1. 古いCUDAツールキットを削除
競合を避けるため、一度古いものを消す。

```bash
sudo apt purge -y nvidia-cuda-toolkit
sudo apt autoremove -y
```


2. CUDA 12.9.1 をインストール（WSL専用版）
CUDA 12.9.1 のサイトは以下のとおり。  
https://developer.nvidia.com/cuda-12-9-1-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_network



```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-9
```

3. nvcc（CUDAコンパイラ）のパスが通す
インストールしただけでは nvcc が見つからないので、~/.bashrc に追記

```
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

bash の場合は以下のように ~/.bashrc に追記。

```
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
```

そのあと、下記コマンドで設定を反映させる。

```
source ~/.bashrc
```

下記コマンドでCUDAツールキット(nvcc)のパスが通っているか確認する。

```
which nvcc
```


4. 下記コマンドで、所望のバージョンのCUDAツールキット(nvcc)がインストールされたか確認する。

```
nvcc --version
```

---

<div id="install_llama_cpp"></div>

## 2. Ubuntu用llama-cppをソースからビルド(CUDAありのため)

1. ビルド前にCUDAツールキットの確認

```bash
nvcc --version
```

2. 任意のローカルディレクトリに、llama-cpp リポジトリをクローン

```bash
cd 任意のローカルディレクトリ
# リポジトリをクローン
git clone https://github.com/ggml-org/llama.cpp.git
```

3. CUDA ありでビルド
下記コマンドでビルドを実行。ビルドには10～15分ほどかかる。

```bash
cd llama.cpp
# CUDAありでビルド
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j$(nproc)
```

最終的に、以下のように出力されればOK!

```bash
[100%] Built target llama-cli
[100%] Built target llama-server
```

4. 動作確認
下記コマンドで動作確認する。

```bash
# バイナリが存在するか確認
ls -la build/bin/llama-server

# バージョン確認
./build/bin/llama-server --version
```

以下のように出力されればOK

```bash
$ ls -la build/bin/llama-server
-rwxr-xr-x 1 ct1485 ct1485 9173736  4月 19 17:10 build/bin/llama-server
ct1485@my-wsl:~/_local_wsl/08_tools/llama.cpp$ ./build/bin/llama-server --version
ggml_cuda_init: found 1 CUDA devices (Total VRAM: 8150 MiB):
  Device 0: NVIDIA GeForce RTX 5070 Laptop GPU, compute capability 12.0, VMM: yes, VRAM: 8150 MiB
version: 8845 (037bfe38d)
built with GNU 11.4.0 for Linux x86_64
```

6. SERVER_EXEパス
例えば `/home/ct1485/_local_wsl/08_tools/` ディレクトリ下に llama.cpp を git clone してビルドした場合、下記パスとなる。

```bash
SERVER_EXE="/home/ct1485/_local_wsl/08_tools/llama.cpp/build/bin/llama-server"
```


---

<div id="llamacpp_server_start_stop"></div>

## 3. llama.cpp サーバーの起動・停止方法
### llama.cpp サーバーの起動
下記内容の bash スクリプトを作成する。  
`05_run_api_server.bash` という名前で作成した場合 `bash 05_run_api_server.bash` で実行。

```bash
#!/bin/bash
# =======================================================================================
# 05_run_api_server.bash - Llama-cpp API Server 起動用スクリプト
# 当スクリプトの実行方法：
# 1. ターミナルで以下のコマンドを入力して実行する。
#      bash 05_run_api_server.bash
# 2. または、実行権限を付与してから直接実行する。
#      chmod +x 05_run_api_server.bash
#      ./05_run_api_server.bash
# =======================================================================================

# --- Configuration ---
# Full path to the llama-server executable
SERVER_EXE="/home/ct1485/_local_wsl/08_tools/llama.cpp/build/bin/llama-server"

# Full path to your model file
MODEL_PATH="/home/ct1485/_local_wsl/08_tools/models/gguf/Llama-3.3-8B-Instruct.Q4_K_S.gguf"

# Number of layers to offload to GPU (-1 for all)
GPU_LAYERS=-1

# Context size (maximum tokens the model handles)
CTX_SIZE=2048

# Network port to listen on
PORT=8080

# --- Log Configuration ---
LOG_DIR="$(dirname "$0")/logs"
LOG_FILE="${LOG_DIR}/llama_cpp.log"
MAX_LOG_SIZE=$((10 * 1024 * 1024))  # 10MB
MAX_LOG_COUNT=5                      # 最大5世代分保持
PID_FILE="${LOG_DIR}/llama_cpp.pid"

# --- Functions ---

# ログローテート関数
rotate_log() {
    if [ -f "${LOG_FILE}" ] && [ "$(stat -c%s "${LOG_FILE}")" -ge "${MAX_LOG_SIZE}" ]; then
        echo "[INFO] Rotating log files..."
        # 古いログを1つずつずらす
        for i in $(seq $((MAX_LOG_COUNT - 1)) -1 1); do
            [ -f "${LOG_FILE}.${i}" ] && mv "${LOG_FILE}.${i}" "${LOG_FILE}.$((i + 1))"
        done
        mv "${LOG_FILE}" "${LOG_FILE}.1"
    fi
}

# 既存プロセスチェック関数
check_already_running() {
    if [ -f "${PID_FILE}" ]; then
        OLD_PID=$(cat "${PID_FILE}")
        if kill -0 "${OLD_PID}" 2>/dev/null; then
            echo "[ERROR] Server is already running. PID: ${OLD_PID}"
            echo "[ERROR] Stop it first: bash 06_stop_api_server.bash"
            exit 1
        else
            # PIDファイルが残っているが、プロセスは死んでいる場合
            echo "[WARN] Stale PID file found. Cleaning up..."
            rm -f "${PID_FILE}"
        fi
    fi
}    

# --- Main ---

# logsディレクトリ作成
mkdir -p "${LOG_DIR}"

# 既存プロセスチェック
check_already_running

# ログローテート
rotate_log

echo "[INFO] Starting Llama-cpp API Server..."
echo "[INFO] Model Path: ${MODEL_PATH}"
echo "[INFO] API URL:    http://localhost:${PORT}/v1"
echo "[INFO] Log File:   ${LOG_FILE}"

# バックグラウンドで起動、ログをファイルに出力
"${SERVER_EXE}" \
    -m "${MODEL_PATH}" \
    --n-gpu-layers ${GPU_LAYERS} \
    --ctx-size ${CTX_SIZE} \
    --port ${PORT} \
    >> "${LOG_FILE}" 2>&1 &

SERVER_PID=$!
echo "${SERVER_PID}" > "${PID_FILE}"

echo "[INFO] Server started. PID: ${SERVER_PID}"
echo "[INFO] Check log:   tail -f ${LOG_FILE}"
echo "[INFO] Stop server: bash 06_stop_api_server.bash"
```

### llama.cpp サーバーの停止
下記内容の bash スクリプトを作成する。  
`06_stop_api_server.bash` という名前で作成した場合 `bash 06_stop_api_server.bash` で実行。

```bash
#!/bin/bash
# =======================================================================================
# 06_stop_api_server.bash - Llama-cpp API Server 停止用スクリプト
# =======================================================================================

PID_FILE="$(dirname "$0")/logs/llama_cpp.pid"

if [ ! -f "${PID_FILE}" ]; then
    echo "[ERROR] PID file not found. Server may not be running."
    exit 1
fi

PID=$(cat "${PID_FILE}")

if kill -0 "${PID}" 2>/dev/null; then
    echo "[INFO] Stopping server. PID: ${PID}"
    kill "${PID}"
    rm -f "${PID_FILE}"
    echo "[INFO] Server stopped."
else
    echo "[WARN] Process not found. Cleaning up PID file."
    rm -f "${PID_FILE}"
fi
```

---

<div id="llamacpp_server_generate"></div>

## 4. llama.cpp サーバーにアクセスして生成

### llama.cpp API利用方法
python にて、下記内容を実行すれば上記 llama-cpp サーバーから生成回答が得られるはず。  
サンプルソース：[https://github.com/uotaro/llamacpp_and_lora/blob/main/07_ask_local_api_server.py](https://github.com/uotaro/llamacpp_and_lora/blob/main/07_ask_local_api_server.py)


```python
from openai import OpenAI

# サーバーのURL（バッチファイルで設定したポート）を指定
client = OpenAI(base_url="http://localhost:8080/v1", api_key="llamacpp")

# ストリーム生成
response = client.chat.completions.create(
    model="local-model", # サーバー側で読み込んでいるため、この文字列は任意です
    messages=[{"role": "user", "content": "兵庫県姫路市でおすすめの観光スポットを5つ教えてください。"}],
    stream=True,
    temperature=0.7,
    max_tokens=512
)

# 逐次表示
for chunk in response:
    content = chunk.choices[0].delta.content
    if content:
        print(content, end="", flush=True)

print("\n")
```


---

<div id="install_llama_cpp_python"></div>

## 5. llama_cpp_python のインストール

1. 共通：必要なビルドツールのインストール
まず、ビルドに必要なツール類を Ubuntu 側にインストールする。

```bash
sudo apt update
sudo apt install -y build-essential python3-dev cmake
```


2. ビルド用パッケージを最新にする
仮想環境（.venv_wsl）に入った状態で、まず以下のツール自体を最新版に上げる。

```bash
# 仮想環境を起動（WindowsのScriptsではなくbinを使う）
source .venv_wsl/bin/activate

# pip自体とビルドに必要なツールを最新にする
pip install --upgrade pip setuptools wheel scikit-build-core
```



3. NVIDIAドライバインストール（GPU (CUDA) を使って高速化する場合）
以前の失敗キャッシュが残っていると邪魔をするので、FORCE_CMAKE=1 を付けて、明示的に CUDA（GPU）を有効にするフラグ を渡してインストールを実施する。  
仮想環境に入った状態で実行すること。


```bash
# 仮想環境に入った状態で実行
# キャッシュを使わずに、CUDA有効でビルド
CMAKE_ARGS="-DGGML_CUDA=on" FORCE_CMAKE=1 pip install llama-cpp-python --no-cache-dir

```

4. 本当にGPU版としてビルドされたか確認
下記コマンドで確認する。

```bash
python3 -c "import llama_cpp; print(f'CUDA support: {llama_cpp.llama_supports_gpu_offload()}')"
```

5. 適当なGGUFモデルをダウンロードして、生成AIに答えさせてみよう

```python
from llama_cpp import Llama

# モデルのロード
model_path = r"/home/ct1485/_local_wsl/08_tools/models/Llama-3.3-8B-Instruct.Q4_K_S.gguf"
# model_path = r"/home/ct1485/_local_wsl/08_tools/models/Llama-3.1-8B-Instruct-Q6_K.gguf"
llm = Llama(
    model_path=model_path,
    n_gpu_layers=-1, # 全てのレイヤーをGPUに転送する（VRAM不足なら数字を減らす）
    n_ctx=2048,      # 文脈サイズ（必要に応じて調整）
    verbose=False    # 起動時のログを非表示にする
)

# メッセージの設定
messages = [
    { "role": "user", "content": "兵庫県姫路市でおすすめの観光スポットを5つ教えてください。" }
]

response_stream = llm.create_chat_completion(
    messages=messages,
    stream=True,
    temperature=0.7,    # 創造性の調整
    repeat_penalty=1.2, # 繰り返しのペナルティ
    max_tokens=512      # 最大出力トークン数
)

# 逐次取得して表示
for chunk in response_stream:
    if "content" in chunk["choices"][0]["delta"]:
        content = chunk["choices"][0]["delta"]["content"]
        print(content, end="", flush=True)

print("\n") # 最後に改行
```