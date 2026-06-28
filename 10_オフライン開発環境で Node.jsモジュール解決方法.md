# オフライン開発環境で Node.jsモジュール解決方法
社内開発環境がオフラインのスパコン上である場合など、ネットからNode.jsモジュールを展開できない。  
その場合の解決方法について記す。

1. [実現方法は２とおり](#実現方法は２とおり)
1. [必要なもの](#必要なもの)
1. [Yarn Berry (v2+) Zero-Installs 実現手順](#yarn-berry-v2-zero-installs-実現手順)
1. [VSCode用のJavaScript補完設定を入れる](#vscode用のjavascript補完設定を入れる)

## 実現方法は２とおり
今から実装するなら、2 の Yarn Berry 手法がよいかも。  

1. .yarnrc の yarn-offline-mirror で .tgz を保存 --- Yarn Classic (v1)
2. デフォルトで .yarn/cache に .zip を保存（Zero-Installs思想） --- Yarn Berry (v2+)

#### Yarn Berry（v2+）と Yarn v1 (Classic) の違い
うちの会社で Yarnオフラインインストールを導入した当時は Yarn v1 (Classic) しかなかったため、今も Yarn v1 (Classic)...  
今から導入するなら、将来的な互換性トラブルを考えると Yarn Berry が良いと思う。  
ただし[参考URLの記事](https://konux.jp/blog/yarn-classic-vs-berry/)にもあるとおり、node_modules 前提の既存プロジェクトは動かない場合もあるから注意が必要。

| 項目                    | Yarn v1 (Classic)   | Yarn Berry (v2+) |
|-------------------------|---------------------|------------------|
| オフライン化の手間      | .yarnrc を書いてミラー設定が必要                                                                | 標準でプロジェクト内に .zip が集まる（設定不要）                            |
| ファイル転送のしやすさ  | 展開後の node_modules は数万ファイルになり、SPCへの転送が激重。 .tgz のままでも管理が少し煩雑。 | 数百個の .zip ファイルにスマートにまとまるため、SPCへの転送が圧倒的に速い。 |
| Node.js v24 への対応    | メンテナンスモード（古い仕様）。最新Nodeでの動作にたまに怪しい挙動がある。                      | 現役で開発中。最新の Node.js v24 や Vue 3.5 に完全対応。                    |
| VSCodeとの相性          | 特別な設定なしで動く。                                                                          | PnP（Zip読み込み）の仕組み上、VSCode側に「SDK」の導入が必須（後述）。       |


#### 参考URL:
- [Yarnのクラシック版とBerry（v2以降）の違いと注意点](https://konux.jp/blog/yarn-classic-vs-berry/)
- [Next.jsをYarn 2のPnP/Zero-Installs + Dockerで動かす](https://zenn.dev/ryo511/articles/4eef0fc13fedcc)


Yarn Berry の標準機能である 「Zero-Installs（ゼロ・インストール）」 という仕組みを使う。  
Zero-Installs を利用するとパッケージの実体（.zip ファイル）がすべてプロジェクト内に保存されるため、 yarn install コマンドすら叩く必要がなくなり、そのまま動かすことができる。

## 必要なもの
ネットに接続でき、かつ社内開発環境のスパコンに接続できるマシン。  
そのマシンが Windows なら、スパコンが Linux だと思うので [WSL2 の Ubuntu あたりを導入](https://github.com/uotaro/memo/blob/main/03_Win11%E3%81%ABWSL%E3%81%A7Ubuntu%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89.md) しておくこと



## Yarn Berry (v2+) Zero-Installs 実現手順

### 1. Python仮想環境の外へ出る
Python の仮想環境（conda や venv など）の中ではなく、WSL（Ubuntu）の通常のターミナル画面で Node.js と Yarn を設定する。

```bash
# python仮想環境外で行う
$ deactivate
```

### 2. yarn がインストールされているか確認
下記コマンドで確認。  
yaran が入っていない場合、応答で sudo apt install cmdtest と表示されることがあるが、絶対に実行しないこと（同名のまったく違う古いツールがインストールされてしまうらしい）。
yaran が入っていない場合は、インターネットに繋がっている WSL 側で、以下の手順を順番に実行して yarn を使えるようにする。

```bash
# yarn がインストールされているか確認
# ※ 以下の応答でオススメされる「sudo apt install cmdtest」は絶対使わないこと。同名の古いツールがインストールされてしまう。
$ yarn --version
コマンド 'yarn' が見つかりません。次の方法でインストールできます:
sudo apt install cmdtest
```

### 3. Node.js の標準機能である corepack を有効にする
Node.js v24 には、Yarnなどのパッケージマネージャーを管理する「Corepack」が標準搭載されている。  
これを有効化する。

```bash
# node.js が入っていること・パス確認
$ node --version
v24.14.1
$ which node
/home/ct1485/.nvm/versions/node/v24.14.1/bin/node
# 現在の Node.js の標準機能である corepack を有効にする
corepack enable
```

### 4. yarn がインストールされていない場合はインストール

```bash
# yarn のバージョンが返ってくるか確認
$ yarn -v
! Corepack is about to download https://registry.yarnpkg.com/yarn/-/yarn-1.22.22.tgz
? Do you want to continue? [Y/n] Y
# 「CorepackがインターネットからYarn v1.22.22（Classic）をダウンロードしてもいいですか？」と確認されるので、Y（Yes）と答えてインストール
1.22.22
```

### 5. Yarn Berry（v2+）へ切り替える
**プロジェクトのディレクトリ（package.jsonがあるディレクトリ）に移動してから【←重要！】**、
現在のディレクトリ内だけで Yarn Berry を使うよう設定する。

```bash
# プロジェクトのディレクトリ（package.jsonがあるディレクトリ）へ移動！ ★重要！
cd プロジェクトのディレクトリ（package.jsonがあるディレクトリ）
# Yarn Berry（v2+）へ切り替える
$ yarn set version berry
! Corepack is about to download https://repo.yarnpkg.com/4.17.0/packages/yarnpkg-cli/bin/yarn.js
? Do you want to continue? [Y/n] Y

➤ YN0000: Done in 0s 2ms
# バージョンが変わったか確認する
$ yarn -v
4.17.0
```

### 6. Zero-Installs 用の .gitignore 設定（重要）
Yarn Berry で「キャッシュ（Zip）をSPCに持ち込む」ために、一般的にはGit管理除外にするキャッシュフォルダを、あえて除外しない（SPCに連れていく）設定にする。  
プロジェクトルートに .gitignore を作成し、以下を記述。  
すでに .gitignore が存在する場合は追記。

```bash
# Yarn Berry Zero-Installs 用の設定
.yarn/*
!.yarn/cache
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/sdks
!.yarn/versions

# 物理的なnode_modulesは不要なので除外
.pnp.*
```

### 7. Node.js パッケージ群のダウンロードを実行
`yarn install` を実行する。  
インターネットから必要なライブラリをすべて一括ダウンロードし、Zipファイルとして固める。

```bash
# プロジェクトのディレクトリ（package.jsonがあるディレクトリ）に移動
cd  /path/to/your/your_project
# .yarnrc.yml を、以下の内容で作成
cat << EOF > .yarnrc.yml
nodeLinker: pnp

enableGlobalCache: false

cacheFolder: "./.yarn/cache"
EOF
# Node.js パッケージ群のダウンロード
yarn install
```

以下の説明書きを完成させてくれますか？

`yarn install` コマンドによりNode.js パッケージ群のダウンロードができたら、
プロジェクトフォルダ下に以下のようなフォルダ・ファイルができているはず。

```
プロジェクトフォルダ/
  ├─ package.json            --- これは、もともとある（依存パッケージの定義書）
  ├─ .yarn/                  --- パッケージのzip実体や管理データが入る
  │    ├─ cache/             --- Node.jsパッケージのzipファイル群が入る（オフライン実行の核）
  │    │    ├─ .gitignore 
  │    │    ├─ @babel-code-frame-npm-7.29.7-cc910f9962-169fc20801.zip
  │    │    ├─ @babel-compat-data-npm-7.29.7-6487d724ba-47913f05e0.zip
  │    │    ├─ @babel-generator-npm-7.29.7-b512136a1f-9bf72b01b5.zip
  │    │    ├─ ・・・略・・・
  │    │    ├─ yaml-npm-2.9.0-0cdd9bc0bc-f340718df4.zip
  │    │    └─ yup-npm-1.7.1-ba72b33527-76b8c7fc2b.zip
  │    ├─ unplugged/         --- C++等のネイティブビルドが必要な一部パッケージが解凍されて配置されるフォルダ
  │    └─ install-state.gz   --- Yarnが次回のインストールを高速化するために使用する、現在の配置状態のキャッシュデータ
  ├─ .pnp.cjs                --- node_modulesの代わりとなる、パッケージの格納先（zip内）をNode.jsに教えるための依存関係マップ（CommonJS用）
  ├─ .pnp.loader.mjs         --- ESM（ECMAScript Modules）形式のコードでもPnPの仕組みを動かすための実行用ローダー
  └─ yarn.lock               --- インストールされた全パッケージの正確なバージョンと依存ツリーを固定する記録ファイル
```

## VSCode用のJavaScript補完設定を入れる
Yarn Berryは node_modules を作らないため、そのままではVSCodeが「jquery」や「vue」のコードを赤線エラーにしてしまう。  
それを解決する拡張設定をコマンドを入れる。  
このコマンドは Yarnの PnP（Plug'n'Play）モード 使用時に VSCodeでライブラリの型補完やエラー検出を正しく動作させるためのコマンド。  
プロジェクトのルートディレクトリでこれを実行すると、VSCode 用の SDK が自動設定され、快適な開発環境が構築できる。

```bash
yarn dlx @yarnpkg/sdks vscode
```

