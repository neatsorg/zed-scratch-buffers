# zed-scratch-buffers

Zed 本体（Rust/GPUI）向けの独立パッチ。一度も保存していない新規タブ
（`Untitled-1` 相当）を、内部的には永続データ領域の `.txt` として管理し、
通常の LSP（校正・変換・翻訳など）を利用できるようにする。

表示上はタブ名に `.txt` を出さず、`Untitled-1` のまま扱う。機能は既定オフで、
通常の Zed 単体でも有効化できる。

このパッチ自体は英語固定で成立させる（`docs/repository-separation-plan.md` の
合意事項）。日本語化時の「無題-1」表示は、i18n 側（zed-personal-build 統合時）の
翻訳接続で対応する統合検証項目であり、このパッチのスコープには含まない。

設計・要件は [`docs/design.md`](docs/design.md)、調査の背景は
[`../zed-writing-tools/docs/scratch-buffer-research.md`](../zed-writing-tools/docs/scratch-buffer-research.md)
を参照。

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

`scripts/prepare` / `scripts/check` は `zed-word-counter` と同じパターンを踏襲する。

## 設定

`settings.json` の `scratch_buffers_enabled`（既定 `false`）で有効化する。
`WorkspaceSettingsContent` は `SettingsContent` に `#[serde(flatten)]` で
組み込まれているため、`"workspace": { ... }` のようにネストせず、
トップレベル直下に書く。

```json
{
  "scratch_buffers_enabled": true
}
```

## 現状

パッチ本体（第一版）を実装し、`editor`・`workspace` クレート単体のテスト、
`zed` クレートでの統合テスト（新規タブ作成・LSP登録・保存フロー・分割ペインでの
番号解放）で検証済み。詳細は [`docs/verification.md`](docs/verification.md) を参照。
`zed-personal-build` 側での統合ビルド・GUI確認は未実施。

## ライセンスと公開範囲

このリポジトリのパッチ、スクリプト、文書は GPL-3.0-or-later です。[LICENSE](LICENSE)
と [NOTICE](NOTICE) を参照してください。これはZed公式配布物ではありません。
パッチ適用済みバイナリの配布には、対応する変更済みZedソースと第三者ライセンス表示が必要です。
