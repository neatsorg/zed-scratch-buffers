# zed-scratch-buffers

Zed 本体（Rust/GPUI）向けの独立パッチ。一度も保存していない新規タブ
（`Untitled-1` 相当）を、内部的には永続データ領域の `.txt` として管理する。
表示上はタブ名に `.txt` を出さず、`Untitled-1` のまま扱いつつ、
プレーンテキストに対応した LSP（[校正・変換・翻訳](https://github.com/neatsorg/zed-writing-tools/tree/main)など）を利用できるようにする。

キャッシュ用フォルダには`buffer.txt`が複数保存されるが、
「最近使ったファイル」などとしてZed 上で扱われないようにされています。

機能は既定オフで、パッチ適用後に `settings.json` に記述すると動作する。

このパッチ自体は英語固定で成立させる（`docs/repository-separation-plan.md` の
合意事項）。日本語化時の「無題-1」表示は、i18n 側（zed-personal-build 統合時）の
翻訳接続で対応する統合検証項目であり、このパッチのスコープには含まない。

## 構成

```text
README.md
upstream.toml     # 対象 Zed のコミット固定
patches/          # 正本パッチ（*.patch）
scripts/prepare   # 対象コミットの取得・パッチ適用
scripts/check     # 型検査・テスト
docs/design.md         # 設計方針・確定した詳細
docs/verification.md   # 検証記録
```

設計・要件は [`docs/design.md`](docs/design.md)、検証結果は
[`docs/verification.md`](docs/verification.md) を参照。
`scripts/prepare` / `scripts/check` は拙作 [zed-word-counter](https://github.com/neatsorg/zed-word-counter) と同じパターンを踏襲する。

現在の固定値（2026-09-20時点）: Zed `v1.20.2` @ [`7c451e6`](https://github.com/zed-industries/zed/commit/7c451e694f3c52ee0aeb01d7e28b5fa18cd0ad2f)（正本は `upstream.toml`）。
`scripts/prepare` 実行後、`.checkout/zed` で `git log -1` すれば同じコミットであることを確認できる。

## ビルドに必要な環境

- Git（対象コミットの取得に使用）
- Rust toolchain（`rustup` 推奨。対象Zedの `rust-toolchain.toml` に従って切り替えます）
- Zed本体のビルドに必要なプラットフォーム依存パッケージ
- 対象ソースとCargo依存を取得できるネットワーク、十分なディスク容量

`scripts/prepare` が `upstream.toml` のリポジトリから固定コミットを `.checkout/zed` に取得し、
このリポジトリのパッチを適用します。Zedのプラットフォーム別ビルド要件は、使用するZedの
上流ドキュメントも確認してください。

## 実用上の設定

`settings.json` の `scratch_buffers_enabled`（既定 `false`）で有効化する。
`WorkspaceSettingsContent` は `SettingsContent` に `#[serde(flatten)]` で
組み込まれているため、`"workspace": { ... }` のようにネストせず、
トップレベル直下に書いてください。

```json
{
  "scratch_buffers_enabled": true
}
```

## 現状

パッチ本体（第一版）を実装し、`editor`・`workspace` クレート単体のテスト、
`zed` クレートでの統合テスト（新規タブ作成・LSP登録・保存フロー・分割ペインでの
番号解放）で検証済み。詳細は [`docs/verification.md`](docs/verification.md) を参照。
その後、実機テストを経てremember_navigation_history_pathにキャッシュファイル情報が漏れることを
確認し、これを除去する処理を追加した。

## ライセンスと公開範囲

このリポジトリのパッチ、スクリプト、文書は GPL-3.0-or-later です。[LICENSE](LICENSE)
と [NOTICE](NOTICE) を参照してください。これはZed公式配布物ではありません。
パッチ適用済みバイナリの配布には、対応する変更済みZedソースと第三者ライセンス表示が必要です。
