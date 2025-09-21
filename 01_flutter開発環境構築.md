# flutter 開発メモ

1. [プロジェクト作成](#create_project)
2. [パッケージの導入方法](#get_packages)
3. [ローカライズ方法](#localization)

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

---

<div id="localization"></div>

# 3. ローカライズ方法

##### パッケージを導入
プロジェクトディレクトリ直下で下記コマンドを実行して、ローカライズに必要なパッケージを導入する。

```sh
flutter pub add flutter_localizations --sdk=flutter
flutter pub add intl:any
```

##### pubspec.yaml 設定
プロジェクトディレクトリ直下にある `pubspec.yaml` を開き、以下のように追記する。  
これは、コードジェネレータの設定のために必要なもの。

```yaml
# 省略
flutter:
  uses-material-design: true
  generate: true # ←追記
```

`pubspec.yaml` 追記後、生成されたコードをプロジェクトから参照できるように、以下のコマンドを実行する。

```sh
flutter pub get
```

##### ローカライズの構成ファイルを作成
プロジェクトディレクトリ直下に `l10n.yaml` という名前のファイルを作成する（文頭の文字は小文字のエル。２＆３番目は数字の10）

```yaml
arb-dir: lib/l10n
template-arb-file: intl_ja.arb
output-class: L10n
nullable-getter: false
```

##### arb ファイルを作成する
プロジェクトディレクトリ/lib ディレクトリ下に `l10n` ディレクトリを作成する。  
作成した `プロジェクトディレクトリ/lib ディレクトリ/l10n` ディレクトリ下に `app_ja.arb` ファイルを作成し、以下の例のように日本語定義ファイルを作成する。  
任意のキー値に対し、表示させたい日本語を定義していく。

```json
{
    "@@locale": "ja",
    "startScreenTitle": "Edit Snapアプリ",
    "@startScreenTitle": {
        "description": "アプリのタイトル"
    },
    "helloWorldOn": "こんにちは！\n今日は{date}です。",
    "@helloWorldOn": {
        "description": "挨拶と日付を表示します。",
        "placeholders": {
            "date": {
                "type": "DateTime",
                "format": "MEd"
            }
        }
    },
    "start": "開始する",
    "@start": {
        "description": "開始ボタンのラベル"
    }
}
```

##### コードジェネレイターの実行
arb ファイルを作成したら、下記コマンドでコードジェネレイターを実行する。

```sh
flutter gen-l10n
```

##### ローカライズされたメッセージを適用
コードジェネレイターを実行したら、生成されたメッセージを以下の例(main.dart)のようにコードに反映させる。

```dart
import 'package:edit_snap/l10n/app_localizations.dart'; // 生成されたコードをインポート
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      localizationsDelegates: L10n.localizationsDelegates, // 対応言語の(今回は日本語のみ)の翻訳データを渡す
      supportedLocales: L10n.supportedLocales,             // 対応言語のリストを渡す
      title: 'Edit Snap',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: StartScreen(),
    );
  }
}

// StartScreen クラス
class StartScreen extends StatelessWidget {
  const StartScreen({super.key});

  @override
  Widget build(BuildContext context) {
    final l10n = L10n.of(context); // L10nクラスのインスタンスを取得
    return Scaffold(
      appBar: AppBar(
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        title: Text(l10n.startScreenTitle), // 対応するメッセージを生成コードから取得
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              l10n.helloWorldOn(DateTime.now()), // 対応するメッセージを生成コードから取得
              textAlign: TextAlign.center,
              style: Theme.of(context).textTheme.headlineSmall,
            ),
            ElevatedButton(
              child: Text(l10n.start), // 対応するメッセージを生成コードから取得
              onPressed: () {
                // ボタンが押されたときの処理
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

##### App Store での表示言語を設定する
iOS ネイティブの対応言語を設定する。  
これは App Store に表示されるアプリの対応言語に影響する。  
`ios/Runner/Info.plist` を開き、`CFBundleLocalizations` キーの下に `ja` を追加する。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- 省略 -->
    <key>CFBundleLocalizations</key>
    <array>
        <string>ja</string>
    </array>
    <!-- 省略 -->
</dict>
</plist>
```

これで、ローカライズ作業完了！  
サンプルは日本語のみ対応しているが、必要に合わせて言語追加すること
