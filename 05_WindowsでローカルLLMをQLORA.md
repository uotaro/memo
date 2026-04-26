# モデルのダウンロード、ROLA

1. [ファインチューニング基礎知識](#basic_knowledge)
1. [WSL/Pythonのセットアップ](#setup_wsl)
2. [必要なシステムパッケージのインストール](#install_system_packages)
3. [必要なライブラリのインストール](#install_libraries)
4. [ファインチューニング対象のモデルをダウンロード](#download_model)
1. [学習データセットの準備](#prepare_dataset)
1. [Unsloth とは](#about_unsloth)

## 環境

| 項目       | スペック・バージョンなど         |
| ---------- | -------------------------------- |
| OS         | Windows 11 + WSL2 (Ubuntu 22.04) |
| GPU        | NVIDIA GeForce RTX 5070 Laptop   |
| メモリ     | 32.0GB                           |
| VRAM       | 8.0GB                           |
| ストレージ | 954GB                            |
| Python     | 3.10.12                          |
| CUDA:      | 12.9.1                           |
| PyTorch    | 2.5.1                            |
| VSCode     | ver1.111.0                       |
| Node.js    | ver22.22.0                       |

---

<div id="basic_knowledge"></div>

## ファインチューニング基礎知識

### 1. loss（ロス：間違いの大きさ）
- 意味: AIの「解答のズレ（間違い）」を数値化したもの。  
- たとえ: テストの「失点」。  
- 見方: 学習が進むにつれて、この数値が 下がっていく のが理想。  
数値が下がる＝「ござる」という話し方のルールを正しく理解し始めている、ということ。

### 2. grad_norm（グラッド・ノーム：学習の勢い）
- 意味: AIが知識を書き換えるときの「修正の強さ」。
- たとえ: ノートに書いた間違いを直すときの「消しゴムとペンで書き直す力加減」。
- 見方: 極端に大きすぎず（暴走）、小さすぎず（停滞）、ある程度の数値で安定しているのが良い状態。  
最初の方は大きく動き、徐々に落ち着いてくることが多い。

### 3. learning_rate（ラーニング・レート：学習率）
- 意味: AIが一度にどれくらい知識を吸収するかという「歩幅」の設定。
- たとえ: 先生が指定した「一歩の大きさ」。
- 見方: 今回の設定では、最初は少しずつ歩幅を広げ（Warmup）、その後は少しずつ小さくしていく。  
最後の方で歩幅を小さくするのは、ゴール付近で微調整をして完璧に仕上げるため。

### 4. epoch（エポック：学習の進み具合）
- 意味: 用意した参考書（データセット）を、何周読んだかを示す割合。  
- たとえ: 「参考書の読破率」。
- 見方: 0.0005328 というのは、「参考書の全ページのうち、まだ 0.05% くらいしか読んでいないよ」という意味。  
これが 1.0 になると、用意した「ござるデータ」を全て1回読み終えたことになります。

---

<div id="setup_wsl"></div>

## 1. WSL/Pythonのセットアップ
下記サイト参照
[memo/03_Win11にWSLでUbuntu環境構築.md](https://github.com/uotaro/memo/blob/main/03_Win11%E3%81%ABWSL%E3%81%A7Ubuntu%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89.md)
[Windows WSL入門：インストールからUbuntu環境構築まで完全ガイド](https://zenn.dev/long910/articles/2026-02-21-wsl-ubuntu-setup)

---

<div id="install_system_packages"></div>

## 2. 必要なシステムパッケージのインストール
下記パッケージを、あらかじめターミナルでインストール


```
sudo apt-get update
sudo apt-get install libcurl4-openssl-dev libssl-dev -y
sudo apt-get install -y libcurl4-openssl-dev libssl-dev make g++
```

---

<div id="install_libraries"></div>

## 3. 必要なライブラリのインストール
Python仮想環境内で、下記ライブラリをインストール

### pytorch, torchvision（CUDA 12.x Nightly 版）
CUDA 12.8 対応の PyTorch を導入するには、公式の安定版（Stable）ではなく Nightly ビルド（開発版）をインストールする必要がある。   
現在、PyTorch の安定版は CUDA 12.4 前後までのサポートが一般的だが、最新の RTX 5070 (Blackwell 世代) は CUDA 12.8 や 12.9 以降で導入された新しいアーキテクチャ（sm_120）をフル活用するために、Nightly 版の使用が推奨されている。

```
pip3 install --pre torch torchvision --index-url https://download.pytorch.org/whl/nightly/cu128
```


### transformers ライブラリのインストール

```
pip install transformers peft bitsandbytes accelerate datasets
```

### Flash Attention 2
プリビルド（配布物）の Flash Attention は最新の RTX 5070 に対応していないことが多いため、ソースからビルドしてインストールする。  
これには ninja が必要。

```
# 準備：ビルド用ツールのインストール
pip install ninja

# Flash Attention のソースビルド（5〜15分ほどかかります）
pip install flash-attn --no-build-isolation
```

### unsloth ライブラリに必要な下記ライブラリもインストール

```
pip install unsloth_zoo
```


### unsloth ライブラリのインストール

```
pip install git+https://github.com/unslothai/unsloth.git
```


---

<div id="download_model"></div>

## 4. ファインチューニング対象のモデルをダウンロード
ダウンロード済みの「.gguf」ファイルをそのままファインチューニングの「入力」として使うことはできない。  

- 理由  
GGUFは推論（実行）に特化して固められた形式であり、学習に必要な勾配計算などの情報を保持していないから。  

- 正しい方法  
学習には Hugging Face形式（Safetensors等）のモデル を対象とする。  
精度を優先するなら、学習は 量子化していない オリジナルモデル（16bit）を対象に学習させたほうが良い。  
しかし、自分のマシンのスペックの都合で Out of Memory (OOM) エラー対策として、4bit量子化済みモデルを使う。  
学習が終わった「後」で、成果物をGGUFとして書き出すのが一般的な流れ。  

#### unsloth/xxxx-bnb-4bit モデルをファインチューニング対象としよう
Unsloth が提供している unsloth/xxxx-bnb-4bit モデルは、単なる量子化モデルではない。  
**Unslothが配布している「-bnb-4bit」モデルは、UnslothライブラリでQLoRAを行うために「最も手軽かつ最適化された」専用モデル**である。

- 動的な再量子化  
学習時には内部で必要な計算を高い精度で行うよう最適化されている。  

- 高速化  
16bit モデルを読み込んでから 4bit に変換する手間を省き、最初から Unsloth の高速カーネルに最適化された状態でロードされる。


### 1. ダウンロードしたいモデルを探す
[Huggingfaceサイト](https://huggingface.co/) で望みのモデルを探す。  
今回は [unsloth/Qwen2.5-7B-Instruct-bnb-4bit](https://huggingface.co/unsloth/Qwen2.5-3B-Instruct-bnb-4bit) をダウンロードする。  

### 2. huggingface-cli インストール
モデルは huggingface-cli(hf) コマンドでダウンロードするのが推奨されている。  
このため、まずは**Python仮想環境内で**下記コマンドで huggingface-cli をインストールする。

```
python3 -m pip install -U huggingface_hub
```

インストールできたか、下記コマンドで確認する。

```
hf --help
```

### 3. モデルのダウンロード

#### 1. HF_TOKEN 発行
HuggingFace の HF_TOKEN を発行する。  
発行した アクセストークンは、[HuggingFace の「マイページ」 → 「Access Tokens」](https://huggingface.co/settings/tokens) で一覧を確認可能。


#### 2. HF_TOKEN を環境変数にセット
セキュリティを考慮して、.bashrc にトークンはべた書きしない。
~/.mytoken みたいなファイルを作成し、そこに以下のように記述。

```bash
export HF_TOKEN="HF_TOKENトークン文字列"
```

.bashrc に以下のように記述して、上記ファイルを読みこむ。

```bash
# ========== 秘密情報は別ファイルで管理 ==========
[ -f "$HOME/.mytoken" ] && source "$HOME/.mytoken"
```

上記のように書き込んだら `source ~/.bashrc` コマンドで反映。  

念のため最後に下記コマンドで、環境変数 HF_TOKEN として認識されているかチェックする。

```
echo $HF_TOKEN
```

#### 3. モデルによっては承認が必要
Meta 社の Llama モデルシリーズなどは「Gated Model」と呼ばれ、Meta社に事前の利用申請が必要。  
所望モデルのサイトにて、必要があれば承認手続きを行う。

###### Gated Model の事前利用申請方法
HuggingFaceサイトの所望モデル（例では meta-llama/Llama-3.1-8B-Instruct ）にアクセスし、こちらの情報をいくつか入力すればOK.
数時間～数日で承認が下りるはず。  
承認の確認は、 [HuggingFace の「マイページ」 → 「Gated Repositories」](https://huggingface.co/settings/gated-repos) で一覧を確認可能。  
Repository Status が PENDING から　ACCEPT？ に変わったらOK！

##### Google AI おすすめの非 Gated モデル 

###### Qwen2.5-7B-Instruct（イチオシ）
- 特徴: 中国発ですが日本語能力も非常に高く、同クラスのモデルの中でベンチマーク性能がトップクラス。  
- Unsloth用リンク: unsloth/Qwen2.5-7B-Instruct-bnb-4bit

###### Phi-4（軽量・高知能）
- 特徴: Microsoft が開発した小型モデルで、論理的思考能力が非常に高い。VRAM 消費をさらに抑えたい場合に最適。
- Unsloth用リンク: unsloth/phi-4-bnb-4bit

###### Mistral-7B-v0.3-Instruct（定番）
- 特徴: オープンソース界隈で長く愛されている定番モデル。Llama 以外で最も広く使われている構成の一つです。
- Unsloth用リンク: unsloth/mistral-7b-instruct-v0.3-bnb-4bit 


#### 4. モデルを hf コマンドでダウンロード
下記コマンドで、任意のフォルダ（下記コマンドでは Qwen2.5-7B-Instruct-bnb-4bit ）内にダウンロード。

```
mkdir unsloth
cd unsloth
hf download unsloth/Qwen2.5-7B-Instruct-bnb-4bit \
  --local-dir Qwen2.5-7B-Instruct-bnb-4bit
```

---

<div id="prepare_dataset"></div>

### 4. 学習データセットの準備
[ござるデータセット](https://huggingface.co/datasets/bbz662bbz/databricks-dolly-15k-ja-gozaru) をダウンロードして利用する。

---

<div id="about_unsloth"></div>

### 5. Unsloth とは
Unslothは、大規模言語モデルのファインチューニングを効率化するためのオープンソースライブラリ。  

> その名前の由来はナマケモノ（Sloth）からきており、「遅くない（Un-sloth）」という意味が込められている、らしい。  

#### unsloth ライブラリの主な特徴
- 高速化: ファインチューニングの速度を約2倍に向上
- メモリ効率: **GPUメモリ使用量を最大80%削減**
- 互換性: Llama 3.3、Mistral、Phi-4、Qwen 2.5、Gemmaなど多数のモデルをサポート
- 低リソース: 通常なら40GB以上のメモリが必要なモデルを、8GBのGPUでファインチューニング可能
- 正確性: 近似手法を使わないため、精度の低下がない（0%の損失）
- 使いやすさ: 初心者フレンドリーなGoogle Colabノートブックを提供

#### 利用メモリと速度の比較

`Unsloth` と `Hugging Face + FA2（Flash Attention 2）`を比較したベンチマークは以下のとおり。  
このように、Unslothは同じハードウェアでより効率的にファインチューニングを行えるだけでなく、より長いコンテキスト長を扱えることも大きな利点。


| モデル            | VRAM | Unsloth速度 |  VRAM削減  | 長いコンテキスト |  Hugging Face + FA2 |
| ---------------- | ---- | ----------- | ---------- | -------------- | ------------------- |
| lama 3.3 (70B)   | 80GB |      2x     |     >75%   |   13倍長い      |        1x           |
| Llama 3.1 (8B)   | 80GB |      2x     |     >70%   |   12倍長い      |        1x           |


#### WSL2 上で unsloth ライブラリが正しく入っているかの確認
下記コードを実行して、が正しく入っているかの確認を行う。  
WSL 上から下記 Python プログラムを実行してみる。

```python
import torch
from unsloth import FastLanguageModel

# GPU名の確認
print(f"GPU Name: {torch.cuda.get_device_name(0)}")

# VRAM確認（プロパティから直接取得）
props = torch.cuda.get_device_properties(0)
print(f"VRAM Total (Properties): {props.total_memory / 1024**3:.2f} GB")

print("Unsloth is ready!")
```

もろもろ通知メッセージも出るが、最終的に下記が出力されればOK。

```
GPU Name: NVIDIA GeForce RTX 5070 Laptop GPU
VRAM Total (Properties): 7.96 GB
Unsloth is ready!
```

### 6. ROLA

```

```

#### 学習させるデータセットの用意
サンプルとして、有名な [ござるデータセット](https://huggingface.co/datasets/bbz662bbz/databricks-dolly-15k-ja-gozaru) を使う。
ござるデータセットのダウンロード方法は以下のいずれかで行う。

##### 1. Hugging Faceから手動でファイルをダウンロードする 
JSON形式のファイルを直接PCに保存したい場合の手順。

1. [Hugging Faceの ござるデータセットページ](https://huggingface.co/datasets/bbz662bbz/databricks-dolly-15k-ja-gozaru) にアクセスする。
2. 上部のタブから [Files and versions] をクリック
3. リストにある `databricks-dolly-15k-ja-gozaru.json` の横にあるダウンロードアイコン（`[↓]`）をクリックするとダウンロードされる。 

##### 2. Pythonライブラリ `datasets` を使って直接読み込む
プログラム内で直接データを読み込みたい場合に最適

1. Python ライブラリ `datasets` のインストール

```
pip install datasets
```

2. 下記コードを実行

```python
from datasets import load_dataset

# データセットをロード
dataset = load_dataset("bbz662bbz/databricks-dolly-15k-ja-gozaru", split="train")

# 内容の確認
print(dataset[0])
```


#### サンプルコード：LoRA(QLoRA)ファインチューニングーニング
[ござるデータセット]()に対してLoRA（QLoRA）でファインチューニングを行うサンプルコード👇  
学習後、モデルの推論テストも行い、最後にLoRAアダプターを保存するところまでを1ファイルで完結。  
LORAアダプターをベースモデルにマージして gguf形式に変換するのは別ファイルに分けた（PCスペック等により失敗してめちゃやり直したから・・・）。

📝 [https://github.com/uotaro/llamacpp_and_lora/blob/main/09_train_gozaru_ROLA.py](https://github.com/uotaro/llamacpp_and_lora/blob/main/09_train_gozaru_ROLA.py)


#### サンプルコード：gguf形式に変換
LoRAアダプターをベースモデルにマージして、gguf形式に変換するサンプルコード👇  
実行前に、09_train_gozaru_ROLA.py を実行して、LoRAアダプターが保存されている必要がある。

📝 [https://github.com/uotaro/llamacpp_and_lora/blob/main/10_convert_to_gguf.py](https://github.com/uotaro/llamacpp_and_lora/blob/main/10_convert_to_gguf.py)


#### TrainingArguments() 各引数について説明

##### 1. バッチサイズとメモリに関する設定
一度にどれだけのデータを処理し、いつ学習（更新）するかを決める。

###### per_device_train_batch_size = 1
- 意味: 1回の計算でGPUに読み込むデータの数。
- 例え: 一度に解く「問題」の数。  
値を大きくすると学習は速くなるが、GPUのメモリを大量に消費する。

###### gradient_accumulation_steps = 8
- 意味: 何回分の計算結果を溜めてから、モデルを更新するか。
- 例え: 「1問解くごとに答え合わせ」するのではなく、「8問分溜めてから一気に復習」するイメージ。  
実質的なバッチサイズは 1 × 8 = 8 になる。

##### 2. 学習スピード（学習率）の設定
モデルがどれくらい「激しく」変化するかを決める。  

###### learning_rate = 2e-4 (0.0002)
- 意味: 1回の更新でモデルのパラメータを動かす歩幅。
- 例え: 勉強の「吸収率」。  
大きすぎると知識が定着せず（発散）、小さすぎるといつまでも学習が終わらない。

###### warmup_steps = 5
意味: 最初の5回までは、非常に小さい学習率から徐々に上げていく準備期間。
例え: 「準備体操」。  
いきなり全力で学習せず、少しずつ慣らして学習を安定させる。

###### lr_scheduler_type = "linear"
- 意味: 学習が進むにつれて学習率をどう変化させるか。
- 例え: 「後半は落ち着いて勉強する」設定。  
最初は一定で、終わりに向けて徐々に歩幅を小さくし、精密に調整する。

##### 3. 学習の期間と記録
どれくらい長く勉強し、どこに結果を出すかを決める。

###### max_steps = 200
- 意味: 合計で何回更新（ステップ）を行うか。
- 例え: 「200ページ分勉強したら終了」という終わりの目標。

###### logging_steps = 1
- 意味: 何ステップごとに経過（損失など）を表示するか。
- 例え: 1問解くごとに「今の成績（進捗）」をメモして教えてくれる設定。

###### output_dir = "outputs"
- 意味: 学習したモデルや記録を保存するフォルダ名。

##### 4. 効率化と計算のコツ
GPUの力を引き出し、メモリを節約するための設定。

###### fp16 / bf16
- 意味: 計算の精度（桁数）の設定。
- 解説: データの精度を少し落とすことで、メモリを節約し、計算速度を上げる。  
最近のGPUなら bf16 がより安定する。

###### optim = "paged_adamw_8bit"
- 意味: 学習に使う「計算機（オプティマイザ）」の種類。
- 解説: メモリ消費を劇的に抑える特別な計算手法（8bit量子化など）を使っている。  
初心者が低スペックGPUで動かす際の強い味方。

###### weight_decay = 0.01
- 意味: パラメータが大きくなりすぎるのを防ぐペナルティ。
- 例え: 「丸暗記（過学習）」の防止策です。適度に力を抜くことで、新しい問題にも対応しやすくなる。

##### 5. 再現性
###### seed = 3407
意味: 乱数のシード値。
解説: これを固定することで、同じ設定でやり直したときに「全く同じ結果」が出るようになる。






