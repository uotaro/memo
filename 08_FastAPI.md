# FastAPI

1. [FastAPI とは](#fastapi-とは)
1. [FastAPI 導入手順](#fastapi-導入手順)
1. [はじめてのサンプル](#はじめてのサンプル)
1. [業務で使う場合によくあるフォルダ・ファイル構成](#業務で使う場合によくあるフォルダファイル構成)

## FastAPI とは
FastAPI（読み方：ファストえーぴーあい）とは、Python3.6以降でAPIを構築するためのWebフレームワーク。  
Pythonの人気なWebフレームワークにはFlaskやDjangoなどがあるが、近年では FastAPIが 非常に使いやすい事からも注目されている。  
日本語のドキュメントが充実しているため、公式の日本語ドキュメントを一通り読み込むことである程度使える。  

公式日本語ドキュメント: [https://fastapi.tiangolo.com/ja/](https://fastapi.tiangolo.com/ja/)


## FastAPI 導入手順
自分の Python 仮想環境に fastapi をインストールすればOK。  
以下のコマンドでFastAPIで使用するすべての依存関係と機能をインストールできる。  
サーバとして使用するuvicornも一括でインストールできる。

```bash
$ pip install fastapi[all]
```

## はじめてのサンプル

### 1. シンプルプログラム作成
公式のサンプル([https://fastapi.tiangolo.com/ja/#installation](https://fastapi.tiangolo.com/ja/#installation)) をそのまま作成してみよう。  
任にのディレクトリにプロジェクトフォルダを作成し、その下に `main.py` というファイルを作成する。  
main.py に以下の内容を記述する。

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def read_root():
    # JSON形式でメッセージを返す
    return {"Hello": "World"}


@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    return {"item_id": item_id, "q": q}
```

### 2. サーバー起動
自身の Python 仮想環境内で下記コマンドを実行。  
停止する場合は `Ctrl + C`

```bash
cd (main.py があるディレクトリ＝プロジェクトルートディレクトリ)
# 開発中の場合
uvicorn main:app --host 0.0.0.0 --port 好きなポート番号 --workers 1 --reload --log-level debug
```

##### ※ 本番の場合
本番環境では **--reload を外して --workers を増やす**のが基本セット。  
ワーカー数の目安 → CPUコア数 × 2 + 1（例：4コアなら 9）

```python
# 本番（例：4コアサーバー）
uvicorn main:app --host 0.0.0.0 --port ポート番号 --workers 9 --log-level warning
```

#### 各種オプションの説明

##### main:app

- main → Pythonファイル名（main.py）
- app → そのファイル内の FastAPI インスタンス変数名
- つまり「main.py の中の app = FastAPI() を起動して」という意味

##### --host 0.0.0.0

- どのIPアドレスからのアクセスも受け付ける設定
- 0.0.0.0 = 「全員ウェルカム」
- 127.0.0.1（localhost）にすると自分のPCからしかアクセスできなくなる
- 開発中は 0.0.0.0 で問題ないが、本番環境ではリバースプロキシ（Nginxなど）と組み合わせるのが一般的

##### --port 8000

- サーバーが待ち受けるポート番号
- FastAPI の慣例は 8000
- 他のアプリと被ったら 8001 や 8080 など自由に変更OK

##### --workers 1

- サーバーのプロセス数（同時に処理できる並列数）
- 開発中は 1 で十分
- 本番環境の目安 → CPUコア数 × 2 + 1（例：4コアなら 9）
- ただし --reload と併用する場合は 1 固定（後述）

##### --reload

- コードを変更・保存すると自動でサーバーが再起動される
- 開発中は非常に便利
- ⚠️ 本番環境では絶対に使わない（パフォーマンス低下・セキュリティリスク）
- --workers を2以上にすると --reload は使えない（競合するため）


#### 他に設定を検討すべきオプション
| オプション                      |         例              | 用途                                          |
|--------------------------------|-------------------------|----------------------------------------------|
| --log-level                    | --log-level debug       | ログの詳細度（debug / info / warning / error） |
| --timeout-keep-alive           | --timeout-keep-alive 5  | アイドル接続を切るまでの秒数                    |
| --ssl-keyfile / --ssl-certfile | 証明書ファイルを指定      | HTTPS化（本番向け）                            |
| --limit-concurrency            | --limit-concurrency 100 | 同時接続数の上限                                 |

## 業務で使う場合によくあるフォルダ・ファイル構成
各ディレクトリには下記ファイル以外に `__init__.py` を配置する。  

```
プロジェクトルート/
├── app/
│   │
│   ├── main.py                     # FastAPI アプリエントリーポイント
│   ├── config.py                   # 設定管理（.env 読み込み）
│   │
│   ├── routers/                    # ルーティング定義
│   │   ├── health.py               # GET /health, GET /info
│   │   └── search.py               # POST /search/paper/*
│   │
│   ├── db/                         # DB設定に関する処理を記述したファイルを配置
│   │   └── session.py              # DBセッションの生成と管理
│   │
│   ├── models/                     # データベースのテーブル定義
│   │   ├── emp.py                  # 社員テーブル（カラム(すべてTEXT型): emp_id, emp_name, mail, did, birth_date, start_date, end_date）
│   │   └── dept.py                 # 所属部署テーブル（カラム(すべてTEXT型): dept_id, dept_name）
│   │
│   ├── schemas/                    # データのスキーマ定義を行うファイルを配置
│   │   ├── base.py                 # 共通スキーマ（HitItem, SearchResponse）
│   │   ├── entry.py                # 登録リクエストスキーマ
│   │   └── search.py               # 検索リクエストスキーマ
│   │
│   ├── services/                   # アプリロジックを配置
│   │   ├── entry/
│   │   │   └── emp.py              # 社員情報登録
│   │   └── search/
│   │       └── emp.py              # 社員情報検索
│   │
│   └── utils/
│       ├── logger.py               # ログ設定
│       └── common_utils.py         # 共通ユーティリティ
│
├── tests/                          # テスト用ファイルを配置
│   ├── conftest.py
│   ├── ・・・略・・・
│   └── test_etest_searchntry.py
│
├── logrotate/ 
│   └── logrotate.conf              # ログローテート設定ファイル
│
├── logs/                           # ログ出力先
├── .env                            # 環境変数設定
└── run_fastapi_server.sh           # 起動/停止スクリプト
```

### サンプルプログラム
当サンプルは、DBを sqLite で実装したもの。  
DBファイルはプロジェクトルート直下に data.db として作成している。

[https://github.com/uotaro/fastapi_sample](https://github.com/uotaro/fastapi_sample)
