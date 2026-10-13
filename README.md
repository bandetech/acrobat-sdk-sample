# acrobat-sdk-sample

Adobe Acrobat のプラグインを **CMake** でビルドするサンプルです。Xcode / Visual Studio のサンプルプロジェクトを使わず、1つの `CMakeLists.txt` から Windows と macOS の両方の成果物を作ります。

| プラットフォーム | 成果物 | 形式 |
|---|---|---|
| Windows | `MyCustomPlugin.api` | DLL（拡張子を `.api` に変更） |
| macOS | `MyCustomPlugin.acroplugin` | バンドル（Mach-O の `bundle` 形式） |

SDK のヘッダは [acrobat-sdk-cmake](https://github.com/bandetech/acrobat-sdk-cmake) から `Acrobat_SDK` ターゲットとして受け取ります。

## このリポジトリに SDK のソースは含まれていません

プラグインの実装である `src/BasicPlugin.cpp` と `src/BasicPluginInit.cpp` は、**Adobe Acrobat SDK に含まれるサンプル BasicPlugin のソース**です。再配布できないため、このリポジトリには入っていません（`src/` は空です）。

利用するには、各自が Adobe から SDK を入手してコピーしてください（手順は後述）。`.gitignore` でこれらのファイルを無視しているので、誤ってコミットされることはありません。

このリポジトリが提供するのは、**ビルド設定そのもの**（`CMakeLists.txt` と `Info.plist.in`）です。

## 1. Acrobat SDK を入手する

Adobe の Developer サイトから Acrobat SDK を入手します。配布ページの URL は過去に何度か変わっているため、ここではリンクを張りません。「Acrobat SDK」で検索して、現在の配布ページを参照してください。入手には Adobe のライセンス条件への同意が必要です。

## 2. SDK パッケージを用意する

[acrobat-sdk-cmake](https://github.com/bandetech/acrobat-sdk-cmake) を用意し、その `Headers/API` と `Headers/SDK` に SDK のヘッダを配置します。手順は同リポジトリの README を参照してください。

このリポジトリの `CMakeLists.txt` は `FetchContent` で取得する設定になっています。ヘッダを手元に置いたクローンを使いたい場合は、`FetchContent_Declare` / `FetchContent_MakeAvailable` の2ブロックを次のように置き換えてください。

```cmake
add_subdirectory(/path/to/acrobat-sdk-cmake acrobat-sdk-cmake EXCLUDE_FROM_ALL)
```

## 3. プラグインのソースをコピーする

SDK 配布物の中から BasicPlugin サンプルを探し、次の2ファイルを `src/` にコピーします。配布物内のパスは SDK の版によって違うので、ファイル名を手がかりに探してください。

```text
acrobat-sdk-sample/
├── CMakeLists.txt
├── Info.plist.in
└── src/
    ├── BasicPlugin.cpp      ← SDK の BasicPlugin サンプルからコピー
    └── BasicPluginInit.cpp  ← 同上
```

### macOS では先に文字エンコーディングを直す

SDK のサンプルには、日本語が **Shift-JIS (CP932)** で書かれている箇所があります。MSVC は `/utf-8` を付けていなければシステムのコードページとして読むので Windows では問題になりませんが、clang はソースを UTF-8 として読むため、`u8"..."` の中に Shift-JIS のバイト列があるとエラーになります。

```text
BasicPlugin.cpp:107:29: error: illegal character encoding in string literal
```

UTF-8 に変換してください。

```bash
# 変換してから元のファイルに置き換える（iconv は同じファイルに直接書けない）
iconv -f CP932 -t UTF-8 src/BasicPlugin.cpp > src/BasicPlugin.utf8.cpp
mv src/BasicPlugin.utf8.cpp src/BasicPlugin.cpp
```

`file` でエンコーディングを確認できます。`Non-ISO extended-ASCII text` と出たファイルが変換対象です。

```bash
$ file src/*.cpp
src/BasicPlugin.cpp:     c program text, Unicode text, UTF-8 text
src/BasicPluginInit.cpp: c program text, ASCII text
```

> **注意**: 変換は必ず元の Shift-JIS ファイルから行ってください。Shift-JIS のファイルを UTF-8 として読んで保存し直すと、日本語がすべて置換文字 (U+FFFD) になり元に戻せません。ビルドは通るので気づきにくいです。

変換後、Windows 側は `/utf-8` を付けて UTF-8 のソースとして読ませる必要があります（`CMakeLists.txt` の `if(MSVC)` に入っています）。

## 4. ビルドする

### macOS（コマンドライン）

```bash
cmake -B build
cmake --build build
# → build/MyCustomPlugin.acroplugin
```

### macOS（Xcode プロジェクトを生成する）

```bash
cmake -B build-xcode -G Xcode
cmake --build build-xcode --config Release
# → build-xcode/Release/MyCustomPlugin.acroplugin
```

`xcode-select` が Command Line Tools を指していると、`-G Xcode` の configure が次のエラーで失敗します。

```text
xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory
'/Library/Developer/CommandLineTools' is a command line tools instance
```

そのコマンドの間だけ Xcode を使う場合（マシン全体の設定を変えません）:

```bash
DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer cmake -B build-xcode -G Xcode
```

現在の向き先は `xcode-select -p` で確認できます。

### Windows

```bat
cmake -B build
cmake --build build --config Release
```

> **注意**: Windows でエントリポイント `PlugInMain` をエクスポートするには、`.def` ファイルや `/EXPORT:PlugInMain` といった指定が別途必要です。SDK の `WinPltfm.h` では `ACCB1` が空に定義されていて `__declspec(dllexport)` が付かないためです。現在の `CMakeLists.txt` にはこの指定が入っていないので、Windows で使う場合は追加してください。

## 5. Acrobat に読み込ませる

### macOS

```bash
# Acrobat を終了し、プロセスが残っていないことを確認する
osascript -e 'quit app "Adobe Acrobat"'
pgrep -fl "Adobe Acrobat.app/Contents/MacOS"

# 消す対象を確認してから削除する
ls -d "/Library/Application Support/Adobe/Acrobat/DC/Plug-ins/MyCustomPlugin.acroplugin"
sudo rm -rf "/Library/Application Support/Adobe/Acrobat/DC/Plug-ins/MyCustomPlugin.acroplugin"

# 配置する（cp -R ではなく ditto で属性と署名を保つ）
sudo ditto build-xcode/Release/MyCustomPlugin.acroplugin \
  "/Library/Application Support/Adobe/Acrobat/DC/Plug-ins/MyCustomPlugin.acroplugin"

# Acrobat を起動する
open -a "/Applications/Adobe Acrobat DC/Adobe Acrobat.app"
```

> **警告**: 上の `sudo rm -rf` は、パスを打ち間違えたり末尾の `MyCustomPlugin.acroplugin` を落としたりすると **Acrobat に同梱されているプラグインごと消えます**（その場合は Acrobat の再インストールが必要です）。必ず先に `ls -d` で対象を確認してください。

製品名が Acrobat Studio などに変わっても、インストール先のパスは `Adobe Acrobat DC` / `Acrobat/DC/` のまま据え置かれています。手元のパスは `ls` で確認してください。

正しく読み込まれれば、メニューに「Acrobat SDK」→「Basic Plugin」が追加されます。

### Windows

ビルドした `MyCustomPlugin.api` を、Acrobat インストール先の `plug_ins` フォルダに置きます。

## macOS 固有の設定について

`CMakeLists.txt` の `elseif(APPLE)` で、Windows では不要だった設定を入れています。

| 設定 | 理由 |
|---|---|
| `add_library(... MODULE)` | プラグインは `dlopen` で読み込まれる `bundle` 形式。`SHARED`（`.dylib`）ではない |
| `BUNDLE TRUE` / `BUNDLE_EXTENSION "acroplugin"` | `Contents/MacOS/` を含むバンドル構成で出力する |
| `enable_language(OBJCXX)` ＋ `LANGUAGE OBJCXX` | SDK ヘッダが Objective-C++ 前提（`WLHFT.h` の `typedef NSURL* WDAppSpecifier;` など） |
| `target_precompile_headers(... <Carbon/Carbon.h> <PIHeaders.h>)` | macOS ではソースが `PIHeaders.h` を読まないので、プレフィックスヘッダとして渡す |
| `PLATFORM` / `PRODUCT` の定義 | SDK の `Environ.h` に macOS の既定値がなく、未定義だと `#error` で止まる |
| `CoreFoundation` のリンク | `PIMain.c` が `CFRetain` などを呼ぶ |
| `CMAKE_OSX_ARCHITECTURES` / `CMAKE_OSX_DEPLOYMENT_TARGET` | ユニバーサルバイナリと最低 OS の指定。**`project()` より前**に書く必要がある |

### Info.plist.in の `MainEntryPointV2` が必須です

`Info.plist.in` に次のキーが入っています。

```xml
<key>MainEntryPointV2</key>
<string>AcroPluginMain</string>
```

**このキーがないと、プラグインは読み込まれるのにエントリポイントが一度も呼ばれません**（メニューも出ません）。Acrobat 本体に同梱されているプラグインの Info.plist にも同じキーが入っています。

```bash
plutil -p "/Applications/Adobe Acrobat DC/Adobe Acrobat.app/Contents/Plugins/Checkers.acroplugin/Contents/Info.plist"
```

なお、このキーについての **Adobe 公式ドキュメントの記載は見つけられていません**。同梱プラグインとの比較と、追加したら動いたという結果からの推測です。Acrobat の版が上がると変わる可能性があります。

## トラブルシューティング

macOS で「読み込まれているのにメニューが出ない」場合、処理の段階ごとにログを置くと原因の範囲を絞れます。

| 最後に出たログ | 疑う場所 |
|---|---|
| 何も出ない | 配置場所、アーキテクチャ、署名など、読み込み以前の問題 |
| `dlopen OK` だけ | Acrobat がエントリポイントを呼んでいない（`Info.plist` の `MainEntryPointV2`） |
| `PISetupSDK` の途中で失敗 | 必要な HFT が取得できていない |
| `PIHandshake` 以降で失敗 | プラグイン側の初期化処理（メニュー作成など） |

ログの仕込み方（`os_log` を使うマクロ、`__attribute__((constructor))` での `dlopen` 検知、`log stream` での監視）は、下の関連記事の macOS 編で詳しく書いています。

## ライセンスと再配布について

- このリポジトリの `CMakeLists.txt` と `Info.plist.in` は自由に使ってください。
- `src/` に配置する `BasicPlugin.cpp` / `BasicPluginInit.cpp` は **Adobe Acrobat SDK のサンプルソース**です。Adobe の SDK ライセンスに従って利用し、再配布しないでください。このリポジトリをフォークして公開する場合も、これらを含めないよう `.gitignore` をそのまま維持してください。

## 関連記事

- [Acrobatプラグイン開発のCMake化 Windows編](https://qiita.com/azblueground/items/0064a04c320b22d91316)
- Acrobatプラグイン開発のCMake化 macOS編（公開後にリンクを追加）
