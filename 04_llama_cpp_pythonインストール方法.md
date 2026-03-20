# llama_cpp_pythonインストール方法

1. [CUDAツールキット nvcc のインストール](#install_nvcc\)
2. [llama_cpp_python のインストール](#install_llama_cpp)


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

zsh の場合は以下のように ~/.zsh に追記

```
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.zshrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.zshrc
source ~/.zshrc

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

## 2. llama_cpp_python のインストール

1. 共通：必要なビルドツールのインストール
まず、ビルドに必要なツール類を Ubuntu 側にインストールする。

```
sudo apt update
sudo apt install -y build-essential python3-dev cmake
```


2. ビルド用パッケージを最新にする
仮想環境（.venv_wsl）に入った状態で、まず以下のツール自体を最新版に上げる。

```
# 仮想環境を起動（WindowsのScriptsではなくbinを使う）
source .venv_wsl/bin/activate

# pip自体とビルドに必要なツールを最新にする
pip install --upgrade pip setuptools wheel scikit-build-core
```



3. NVIDIAドライバインストール（GPU (CUDA) を使って高速化する場合）
以前の失敗キャッシュが残っていると邪魔をするので、FORCE_CMAKE=1 を付けて、明示的に CUDA（GPU）を有効にするフラグ を渡してインストールを実施する。  
仮想環境に入った状態で実行すること。


```
# 仮想環境に入った状態で実行
# キャッシュを使わずに、CUDA有効でビルド
CMAKE_ARGS="-DGGML_CUDA=on" FORCE_CMAKE=1 pip install llama-cpp-python --no-cache-dir

```

4. 本当にGPU版としてビルドされたか確認
下記コマンドで確認する。

```
python3 -c "import llama_cpp; print(f'CUDA support: {llama_cpp.llama_supports_gpu_offload()}')"
```

5. 適当なGGUFモデルをダウンロードして、生成AIに答えさせてみよう

```
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