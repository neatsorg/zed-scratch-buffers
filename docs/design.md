# 設計方針

パッチ本体（第一版）を実装・検証済み。詳細は [`verification.md`](verification.md) を参照。

## 目的

一度も保存していない新規タブ（`Untitled-1` 相当）を、内部的には永続データ領域の
`.txt` として管理し、実際のバッファに紐づけて通常の LSP（校正・変換・翻訳など）を
利用できるようにする。表示上はタブ名に `.txt` を出さず、`Untitled-1` / 日本語化時は
「無題-1」のまま扱う。

背景・調査の詳細は
[`../zed-writing-tools/docs/scratch-buffer-research.md`](../zed-writing-tools/docs/scratch-buffer-research.md)
を参照。Zed の LSP 実装は、一度もディスクに紐づいていないバッファを言語サーバーへ
登録しない（`register_buffer_with_language_servers`）ため、通常の Wasm 拡張だけでは
実現できず、本体パッチが必要という結論に基づく。

## 実装要件

- 表示は `Untitled-1` / 日本語化時は「無題-1」。タブ名に `.txt` は出さない。
- 内部ファイルは永続データ領域の `.txt` とし、実際のバッファに紐づけて LSP を利用する。
- 初期言語を Plain Text に固定する。
- 内部の固有 ID と表示番号を分け、番号再利用で旧文書を上書きしない。
- 通常の保存で正式保存先を尋ねる。成功時だけ通常ファイルへ移行し、キャンセル時は下書きを保つ。
- 個別タブを閉じた場合は保管して番号を解放し、アプリ終了の場合は再表示する対象として保持する。
- 既存の Hot Exit を活かす。
- 字数カウントや writing-tools を必須依存にしない。LSP 接続は通常の Zed の仕組みを使う。
- 機能は既定オフとし、通常の Zed 単体で有効化できるようにする。

## 確定した詳細（2026-09-20、ユーザー合意）

### 1. 表示番号の範囲: アプリ全体でグローバルに採番する

ウィンドウ単位ではなく、プロセス全体で単一のカウンタを使う。

理由: 内部ファイル名（実体）は要件どおりグローバル一意な ID にするため、番号を
ウィンドウ単位にしてもファイル名の衝突は起きない。しかし Zed はタブのドラッグ＆
ドロップで別ウィンドウへの移動に対応しており、番号をウィンドウ単位で管理すると
移動のたびに番号を振り直すか重複を許容するかの判断が必要になり複雑化する。
グローバル採番なら単一のカウンタで完結し、ウィンドウ移動・分割・複製のどのケースでも
番号がぶれない。

### 2. 保管期限: `paths::temp_dir()` 配下に残し、Hot Exit で復元できるようにする
（2026-09-20、レビュー指摘を受けて改訂）

当初は「プロセス起動ごとの一時ディレクトリ、再起動で消える程度でよい」としていたが、
実装レビューで次の矛盾が見つかった。

- `paths::temp_dir()`（`crates/paths/src/paths.rs`）は Linux では `$XDG_RUNTIME_DIR`
  ではなく `~/.cache/zed` 相当（`dirs::cache_dir()`）を返す。ログアウトやプロセス終了
  では消えない、恒久的に近いキャッシュ領域であり、「再起動で消える」という前提と
  実装が食い違っていた。
- 一方、実装要件「既存の Hot Exit を活かす」は、下書きが実ファイルである以上、
  正常終了後に自動で復元されることを意味する。「再起動で消える」と「Hot Exit で
  復元される」は本質的に両立しない要件だった。

改訂: 保存先は `paths::temp_dir()/scratch-buffers` のまま（変更しない）とし、要件を
「再起動で自然に消える」から「Hot Exit のために残る」へ改める。恒久的な保管を積極的に
目指すものではないが、削除しないことで Hot Exit が機能する状態を優先する。

この変更に伴い、確定事項3（番号管理）は「新規作成時の採番」だけでなく「Hot Exit 等
既存の scratch パスバッファが復元された時点での番号予約」も担う必要があると判明した
（詳細は確定事項6）。起動時の古いエントリの自動クリーンアップは、今回のスコープでは
見送る（実施する場合は「Hot Exit の復元対象からどう除外するか」を含めて別途設計する）。

### 3. 分割ペインで最後の表示を閉じた際の扱い: 最後の参照が消えたときにのみ処理する

Zed では同じバッファ（同じ下書き）を複数のペインに分割表示できる（例: 同じ
`Untitled-1` を左右のペインに並べて別々の場所を編集）。「タブを閉じる」操作は
ペインごとに独立して発生するため、片方のペインのタブを閉じても、もう片方に
同じ下書きがまだ表示されていれば、その下書きは「使用中」のままであり、番号の解放や
保管処理を走らせてはいけない。

実装では、そのバッファを表示している全ての `Item`（各ペインのタブ）が閉じられた
ことを検出してから、番号解放・保管処理を行う。片方だけ閉じた時点で早期に処理すると、
番号の重複や、もう片方に表示中の内容との食い違いにつながる。

### 4. 閉じた下書きの復元専用 UI は作らない（2026-09-20、ユーザー合意）

要件25「アプリ終了の場合は再表示する対象として保持する」の意図を確認したところ、
「間違えて閉じた下書きも、実ファイルとして残っていれば利用者が自分で探して開き直せる」
という程度の期待であり、専用の一覧 UI・コマンドパレット統合などは不要と判明した。

- アプリ終了時の再表示: 下書きは実ファイルなので、既存の Hot Exit（正常終了時に
  開いていたタブを復元する機能）がそのまま効く。追加実装は不要。
- タブを閉じた場合の「保管」: 削除処理を書いていないので、ファイルは自然に残る。
  これも追加実装は不要。
- 閉じた下書きを見つけ直す手段（ファイルマネージャ等での直接探索）を、この機能の
  範囲として作り込む必要はない。

### 5. 既定オフの設定項目: `scratch_buffers_enabled`（既定 `false`）

`WorkspaceSettingsContent`（`crates/settings_content/src/workspace.rs`）に
`scratch_buffers_enabled: Option<bool>` を追加し、`assets/settings/default.json` で
`false` を既定値にした。`Editor::new_in_workspace` の先頭でこの設定を確認し、無効なら
従来どおりファイルなしのバッファを作る分岐にフォールバックする。

**注意（2026-09-20 の統合ビルド検証で判明）**: `SettingsContent` 構造体では
`pub workspace: WorkspaceSettingsContent` に `#[serde(flatten)]` が付いており、
`WorkspaceSettingsContent` の各フィールドは `settings.json` の**トップレベル直下**に
展開される。したがって `settings.json` では `"workspace": { "scratch_buffers_enabled": true }`
ではなく `"scratch_buffers_enabled": true` と書く必要がある。誤ってネストして書くと、
Zed は未知のプロパティとして黙って無視する（パースエラーにはならない）ため、
設定が反映されないまま気づきにくい失敗をする。Rust コード側で
`SettingsContent` の値を直接組み立てる場合（例: テストヘルパー）は
`settings.workspace.scratch_buffers_enabled` のようにフィールドパスで書いてよい
（`#[serde(flatten)]` は JSON シリアライズ表現にのみ影響し、Rust 構造体の
フィールドパスには影響しない）。

### 6. 番号管理の拡張: 経路を問わない予約、保存成功時の即時解放
（2026-09-20、レビュー指摘を受けて追加）

実装レビューで、当初の番号管理（`Editor::new_in_workspace` での新規採番と
`on_release` での解放のみ）に2つの欠陥が見つかった。

- **Hot Exit 復元での番号未予約**: 確定事項2の改訂により scratch バッファは
  Hot Exit で復元されうるが、その経路（`Editor::for_buffer` 経由、
  `Editor::new_in_workspace` を通らない）では番号がプロセス内の管理状態に
  登録されていなかった。復元直後に新規作成すると、同じ番号（例: 両方とも
  `Untitled-1`）が同時に存在しうる。
- **保存成功時に番号が解放されない**: 「名前を付けて保存」で正式ファイルへ
  移行しても、`on_release`（エディタを閉じるまで発火しない）以外に解放経路が
  なく、番号が無駄に予約されたままだった。要件「成功時だけ通常ファイルへ移行」の
  時点で、その番号は次の新規下書きに使えるべき。

対処: `Editor::new_internal`（`for_buffer`／`for_multibuffer`／`clone`／Hot Exit
復元のいずれも最終的に通る）に予約処理を一元化し、`workspace::scratch_buffers` に
バッファ ID ベースの所有者管理を追加した。

- 経路を問わず、singleton バッファが scratch パスを持てば番号を予約し
  （`reserve_display_number`）、`(number, buffer_id)` を紐づける
  （`associate_display_number_with_buffer`）。
- 保存成功で `multi_buffer::Event::FileHandleChanged` が発火した際、バッファ ID
  から番号を逆引きし（`display_number_for_buffer`。新しいパスはもう scratch
  パスでないため、パスから番号を求める `scratch_display_number` は使えない）、
  scratch でなくなっていれば即座に解放する。
- 解放は必ず所有者チェック付きの `release_display_number_owned_by(number,
  buffer_id)` を使う。単純な無条件解放だと、保存成功時の即時解放と
  `on_release` の2つの解放経路が同じ番号に対して独立に存在するため、片方が
  解放した直後にその番号が新しい下書きへ再割り当てされ、もう片方が
  "自分の番号のつもり" でそれを解放してしまう（無関係な別の下書きの番号を奪う）
  事故が起こりうる。所有者が一致する場合のみ解放することで、これを防ぐ。

### 7. i18n 対応の範囲: このパッチは英語固定、翻訳接続は統合側の責務
（2026-09-20、レビュー指摘を受けて明記）

「目的」「実装要件」に書いた「日本語化時は『無題-1』」という表示は、
`docs/repository-separation-plan.md` の合意事項（新機能の翻訳接続は統合側
= zed-personal-build で管理し、機能パッチ自体は英語で成立させる）に従い、
**このパッチでは実装しない**。現状の `format!("Untitled-{number}")` は
ハードコードされた英語文字列であり、i18n 側の抽出・翻訳の仕組みを自動では通らない。

日本語化は zed-personal-build での統合時に、`compat/` での置き換えパッチか、
このパッチ側に翻訳接続用のフックを用意して i18n 側で実装するかのいずれかで
対応する（どちらを選ぶかは統合側の設計判断）。zed-scratch-buffers 単体としての
受け入れ条件には含めない。

**方針決定（2026-09-20、[redacted-host] 実機での統合ビルド確認・EmEditor 日本語版の
表記を踏まえて再検討）**: 当初は「Untitled」という表記を英語のまま残す案も
検討したが、EmEditor 日本語版が「無題-1」のように翻訳している前例を踏まえ、
翻訳する方針に決定した。

実装方式は **`compat/` での置き換えパッチ**（zed-personal-build 側）を選ぶ。
理由:
- `localization::` クレートは i18n 統合後にしか存在しないため、
  scratch_buffers 側（`crates/workspace/src/scratch_buffers.rs`・
  `crates/editor/src/items.rs`）に直接 `localization::` 呼び出しを
  埋め込むと、vanilla Zed への単独適用・単体テストというこのパッチの
  前提（zed-scratch-buffers 単体で `cargo check`/`cargo test` が通る）が
  崩れる。
- zed-i18n 側の `apply_universal.py` を調べたところ、`format!(...)` を
  伴う動的なメッセージ（今回の `Untitled-{number}` と同種）は、
  完全自動の AST 抽出ではなく手書きの個別置き換えテーブルで対応されている。
  つまり「フック」を用意しても結局どこかに手書きのルールが要る点は
  compat/ 置き換えパッチと変わらない。
- `docs/repository-separation-plan.md` に、翻訳がパッチの前提と衝突する
  場合は `zed-personal-build/compat/` に組み合わせ専用の調整を置くと
  明記されており、これがまさにこのケースにあたる。

zed-scratch-buffers 側の実装（`format!("Untitled-{number}")`）は変更しない。
compat/ 側のパッチは、i18n の `generate-runtime-bundles`/`apply-universal`
適用後のチェックアウトに対して、この文字列を `localization::format_message`
経由の呼び出しへ置き換える形で実装する（未着手、次フェーズ）。

## 未確定・今後詰める点

- **`Untitled-N` の日本語化**: 確定事項7参照。zed-personal-build の
  `compat/` に置き換えパッチを実装する（未着手）。zed-scratch-buffers
  自体には変更不要。

（上記以外、2026-09-20 時点で残っているものはなし。実装済みの内容は
[`../docs/verification.md`](verification.md) を参照。）
