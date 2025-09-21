# flutter 開発メモ

1. [プロジェクト作成](#create_project)
2. [パッケージの導入方法](#get_packages)

---

<div id="create_project"></div>

# 1. プロジェクト作成

## 1-1. 初回のみ実施すること

##### 1. グローバルに fvm 設定
ターミナルにて下記コマンド実施して、グローバル環境で flutter コマンド（`flutter create`コマンド）を実行できるようにする。  
flutterのバージョンはなんでも良いので、自マシンにインストールした中で最新のflutterバージョンあたりをインストールしとこう。


```bash
fvm global flutterバージョン（手持ちの中の最新バージョン指定しとこか）
# 例
fvm global 3.35.3
```

##### 2. fvm のパスを通しておく
ホームディレクトリ直下にある .zshrc (/Users/ユーザー名/.zshrc）に、下記１行を追記する。  
これにより、どのディレクトリにいても fvm コマンドが実行できるようになる。


```bash
export PATH="$HOME/fvm/default/bin:$PATH"
```

## 1-2. flutter プロジェクトの作成

##### 1. ターミナルにてプロジェクトフォルダを作成したいディレクトリを開く

##### 2. プロジェクト作成コマンド実行
手順1で移動したディレクトリにて、下記コマンドを実行して flutter プロジェクトを作成する。

```bash
flutter create ⚪︎⚪︎⚪︎⚪︎⚪︎　# ⚪︎⚪︎⚪︎⚪︎はプロジェクトフォルダ名
```

これでプロジェクト作成OK。

##### 3. VSCode で、手順２で作成したプロジェクトフォルダを開く

##### 4. VSCode 上のターミナルにて下記コマンドを実行して、使いたい flutter のバージョンを指定する。

```bash
fvn use X.XX.X # X.XX.Xは使いたいバージョン

# 例
fvn use 3.35.3
```

コマンドを実行すると以下のように聞かれるので `y` と入力する。  
Git登録除外設定ファイルである .gitignore ファイルに .fvm フォルダを追記する？って内容の問い。

```bash
You should add the fvm version directory ".fvm/" to .gitignore.
✔ Would you like to do that now? - yes/no
```

以下のように表示されればOK。

<div style="width=50%;">

![](./images/01/fvm_use_01png.png)

</div>


##### 5. VSCode 上の flutter 拡張機能設定
VSCode 上の flutter 拡張機能設定のため、プロジェクトフォルダ直下にある .vscode フォルダ内に `settings.json` を作成する。  
作成した settings.json に、下記を記述する。  
`editor.formatOnSave ` 設定は、**勝手に dart の改行調整を無効化するため**の設定。自動でフォーマットして欲しい場合は true 設定にしてください。

```python
{
  "dart.flutterSdkPath": ".fvm/versions/3.36.0",
  "[dart]": {
    "editor.formatOnSave": false
  }
}
```

これにて flutter プロジェクト作成作業完了！


---

<div id="get_packages"></div>

# 1. パッケージの導入方法

##### 1.　パッケージの追加
プロジェクトフォルダ直下にて `fvn use X.XX.X` コマンドを実行して使う flutter バージョンを指定した後、下記コマンドでパッケージを追加する。  
いずれも**プロジェクト直下でコマンド実行**すること。

```bash
flutter pub add ⚪︎⚪︎⚪︎⚪︎ # ⚪︎⚪︎⚪︎⚪︎は使いたいパッケージ名

# 例
flutter pub add go_router
```

これにて、プロジェクトフォルダ下直下にある pubspec.yaml ファイルに以下のように自動追記される。

```python
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
  go_router: ^16.2.2 # ←追加されてる

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^5.0.0
```

`dev_dependencies` セクションに追加したい場合（リリースビルドに含めたくない場合）は以下のように `--dev` オプションを付与する。

```bash
flutter pub add --dev ⚪︎⚪︎⚪︎⚪︎ # ⚪︎⚪︎⚪︎⚪︎は使いたいパッケージ名

# 例
flutter pub add --dev build_runner
```

##### 2. パッケージの導入
手順１で pubspec.yaml にパッケージ追加したら、下記コマンドを実行してパッケージを導入する。

```bash
flutter pub get
```
