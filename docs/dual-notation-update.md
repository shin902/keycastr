# KeyCastr デュアル表記（macOS + Windows）実装ガイド

このドキュメントは、KeyCastr に macOS / Windows のショートカットを同時表示する機能を追加する開発手順をまとめたものです。`main` ブランチにマージ済みの実装内容と、既存コードを修正する際のポイントを整理しています。

## 1. 概要

- 目的: Default Visualizer の表示文字列を、macOS 表記と Windows 表記の 2 行（または 1 行区切り）で同時に表示できるようにする。
- アプローチ: Visualizer 設定パネルに新しいチェックボックスを追加し、`NSUserDefaults` の `default.showDualNotation` フラグにバインドしてオン/オフを切り替える。
- 対象コード: `keycastr/KCDefaultVisualizer.*`, `keycastr/KCEventTransformer.*`, `keycastr/KCVisualizerTests/KCKeystrokeConversionTests.m`。

## 2. 前提条件

1. Xcode 15 以降（ARC / macOS 12+ 対応）。
2. Git サブモジュール (`git submodule update --init --recursive`) 済み。
3. 開発マシンで KeyCastr に入力監視/アクセシビリティ権限を付与済み。

## 3. UI 変更手順

1. `KCDefaultVisualizerPreferencesView` にデュアル表記チェックボックスを追加。
   - コードベースでは nib を変更せず、`awakeFromNib` で NSButton を動的生成。
   - レイアウト定数（`kSeparatorLineY`・`kCheckboxBelowSeparatorOffset` など）で既存ラジオボタン群と整列。
2. チェックボックスを `NSUserDefaultsController` の `values.default.showDualNotation` にバインド。
3. オートレイアウト/Autoresizing をラジオボタンと同じ設定にし、Display タブの幅変更にも追従。

該当実装は `keycastr/KCDefaultVisualizer.m` 91–141 行付近。

## 4. `NSUserDefaults` キー定義

`+visualizerDefaults` に `default.showDualNotation: @NO` を追加（`KCDefaultVisualizer.m` 272 行付近）。これにより初期値が false となり、既存ユーザーはオプトイン方式で機能を有効化できます。

## 5. 既存コードの修正ポイント

### 5.1 `KCDefaultVisualizer`

- `addKeystroke:` にデュアル表記処理を追加。
  ```objc
  BOOL showDualNotation = [[NSUserDefaults standardUserDefaults] boolForKey:@"default.showDualNotation"];
  if (showDualNotation) {
      NSString *macOSNotation = [keystroke convertToString];
      KCEventTransformer *transformer = [KCEventTransformer currentTransformer];
      NSString *windowsNotation = [transformer transformedValueToWindowsNotation:keystroke];
      NSString *dualNotation = [NSString stringWithFormat:@"%@  │  %@", macOSNotation, windowsNotation];
      [self appendString:dualNotation];
  } else {
      [self appendString:[keystroke convertToString]];
  }
  ```
- 既存 `appendString:` 等の処理は変更不要。

### 5.2 `KCEventTransformer`

- `-transformedValueToWindowsNotation:` を追加し、Windows 風の修飾キー順序（Ctrl → Alt → Shift）に整形。
- `-_windowsSpecialKeys` を持たせ、矢印キー・ファンクションキー等を Windows 表記へ変換。
- `default_displayModifiedCharacters` フラグとの相互作用を考慮し、記号や死んだキー（dead keys）への対応を実装。

### 5.3 `KCDefaultVisualizerPreferencesView`

- `setupDualNotationCheckbox`, `configureDualNotationCheckbox`, `bindDualNotationCheckbox` など補助メソッドを追加して責務を整理。

## 6. `defaults` コマンドとの整合性

- デバッグビルドのバンドル ID は `io.github.keycastr`。公式配布ビルドでは別 ID の場合があるため、以下を用いて実際の ID を確認する。
  ```sh
  /usr/libexec/PlistBuddy -c 'Print CFBundleIdentifier' /Applications/KeyCastr.app/Contents/Info.plist
  ```
- 正しい ID に対して書き込み/確認する。
  ```sh
  defaults write <BundleID> default.showDualNotation -bool true
  defaults read <BundleID> default.showDualNotation
  ```

## 7. テスト

1. `KCVisualizerTests/KCKeystrokeConversionTests.m` に Windows 変換用のユニットテストを追加。
   - Cmd+C → Ctrl+C
   - Cmd+Shift+P → Ctrl+Shift+P
   - Opt+A → Alt+A
   - Cmd+Opt+I → Ctrl+Alt+I
   - 矢印キー、Function キー、Shift+Tab など特殊キーの網羅
2. Xcode から `Product > Test`、または CLI:
   ```sh
   xcodebuild test -scheme KeyCastr -destination 'platform=macOS'
   ```

## 8. ビルド & リリース手順への組み込み

1. `git pull` 後に `KeyCastr.xcodeproj` を開き、`Cmd+B` でビルド。
2. Prefs → Display タブでチェックボックスが表示されることと、オン時に macOS/Windows 両方の文字列が描画されることを目視確認。
3. `DEVELOPING.md` のリリース手順に従い、`Info.plist` のバージョンを更新して notarize。
4. `bin/` 内の `.app` zip を差し替え、タグ作成 → GitHub Release → Sparkle appcast 更新 → Homebrew cask 更新。

## 9. 既存環境への適用フロー

1. 最新 `main` を取得し、`KeyCastr.app` をビルドして配布。
2. 既存ユーザーはアプリを更新後、Preferences → Display → 「Show dual notation (macOS + Windows)」をオン。
3. GUI が存在しない古いバージョンでは、上記 `defaults write` にて手動で有効化可能（ただし UI に表示されるのは最新版のみ）。

---

このドキュメントを元にアップデートを実装することで、Mac と Windows のショートカット表記を同時に確認したいユーザーのニーズに対応できます。