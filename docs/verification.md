# 検証記録

## 2026-09-20: パッチ本体（第一版）の実装と検証

対象コミット: v1.20.2（`7c451e694f3c52ee0aeb01d7e28b5fa18cd0ad2f`）。
作業用チェックアウト: `.checkout/zed`（`scripts/prepare --source <zed-i18n の clean-extract>`）。

### 実装した内容

1. **`crates/workspace/src/scratch_buffers.rs`（新規モジュール）**
   - `scratch_root_dir()`: `paths::temp_dir()/scratch-buffers`。再起動で消えて構わない領域
     （docs/design.md 確定事項2）。
   - `is_scratch_path(path)`: ファイル名が `buffer.txt` かつ `scratch_root_dir()` 配下、
     という2条件で判定。単純な文字列プレフィックス一致ではなくコンポーネント境界で比較。
   - `scratch_display_number(path)` / `new_scratch_buffer_path()`: パス形式
     `scratch_root_dir()/<display_number>-<uuid>/buffer.txt` から表示番号を読み書き。
   - `allocate_display_number()` / `release_display_number()`: プロセスグローバルな
     `BTreeSet<usize>` で採番・解放（確定事項1）。`Mutex` は poison 耐性あり
     （`unwrap_or_else(PoisonError::into_inner)`）。
   - `is_display_number_in_use()` / `test_synchronization_lock()`:
     `#[cfg(any(test, feature = "test-support"))]` のテスト支援 API。
   - 単体テスト3件（パスの往復変換、誤判定しないこと、番号の再利用）。
   - `crates/workspace/Cargo.toml` に `paths` 依存を追加。

2. **`crates/editor/src/editor.rs` の `Editor::new_in_workspace`**
   - 変更前: `project.create_buffer(None, true, cx)` でファイルなしバッファを作成。
   - 変更後: `scratch_buffers::new_scratch_buffer_path()` でパスと番号を決め、
     `fs.create_dir` → `fs.create_file` → `project.open_local_buffer(path, cx)` の順で、
     実ファイルに紐づいたバッファとして開く（`register_buffer_with_language_servers` が
     `buffer.file()` を要求するため。`open_local_buffer` は内部で
     `find_or_create_worktree(path, visible=false, ...)` を使うので、非表示 worktree
     経由になりプロジェクトパネルには出ない）。
   - ファイル作成・オープンに失敗した場合は `release_display_number` で番号を戻す。
   - `set_content_language_detection_enabled(false)` で初期言語を Plain Text に固定
     （確定事項3とは別件、実装要件「初期言語を Plain Text に固定する」に対応）。
   - バッファの `cx.on_release`（＝そのバッファへの最後の強参照が消えたとき、分割ペインで
     複数表示されていても発火しない）で `release_display_number` を呼ぶ
     （確定事項3の実装）。

3. **`crates/editor/src/items.rs`**
   - `scratch_buffer_tab_label(buffer, cx)`（`pub(crate)`）: singleton バッファが
     scratch パスを指していれば `Untitled-{number}` を返す。
   - `tab_content_text` / `tab_tooltip_text` / `suggested_filename` の先頭でこれを使い、
     実パスの代わりに `Untitled-N` を出す。`suggested_filename` は `Untitled-N.txt`。
   - `can_save`: scratch バッファなら `false` を返す（後述の重大な問題の修正）。

4. **`crates/editor/src/editor.rs` の `Editor::title`**
   - 同様に `items::scratch_buffer_tab_label` を先頭でチェックし、ウィンドウタイトル等
     でも `Untitled-N` を一貫して出す。

5. **既定オフの設定項目 `workspace.scratch_buffers_enabled`（確定事項5）**
   - `crates/settings_content/src/workspace.rs`: `WorkspaceSettingsContent` に
     `scratch_buffers_enabled: Option<bool>` を追加。
   - `assets/settings/default.json`: 既定値 `false` を追加。
   - `crates/workspace/src/workspace_settings.rs`: `WorkspaceSettings` に対応フィールドを
     追加し `from_settings` で解決。
   - `crates/settings/src/vscode_import.rs`: VS Code 設定インポートの
     `WorkspaceSettingsContent` 初期化に `scratch_buffers_enabled: None` を追加（対応する
     VS Code 設定はないため常に `None`）。
   - `Editor::new_in_workspace` の先頭で `WorkspaceSettings::get_global(cx)
     .scratch_buffers_enabled` を確認し、`false`（既定）なら変更前と同じ
     `project.create_buffer(None, true, cx)` の分岐にフォールバックする。

   最初の実装ではこの設定項目自体を作り忘れており、常に有効な状態になっていた
   （実装要件「機能は既定オフとし、通常の Zed 単体で有効化できるようにする」に反する）。
   README 更新時の見直しで気づき、追加した。

### 実装中に見つかった重大な問題と対処

**保存時に確認なしで内部ファイルへ上書きされる問題（P1）**: `Buffer` の
`project::ProjectItem::project_path` は、`file()` があれば（`DiskState::Historic` でない限り）
常に `Some` を返す。scratch バッファは要件どおり実ファイルを持つため、`Editor::can_save` の
既定実装（`project_path(cx).is_some()` を見る）が `true` を返し、`Pane::save_item` が
「新しい保存先を尋ねる」分岐ではなく「そのまま保存」分岐に入ってしまい、ユーザーに確認なく
下書き用の内部ファイルへ上書き保存されてしまっていた。実装要件「通常の保存で正式保存先を
尋ねる」に反する。

対処: `can_save` を scratch バッファの場合だけ `false` にし、`can_save_as` は
（singleton なので）`true` のまま残す。`Pane::save_item` は `can_save` が `false` でも
`can_save_as` が `true` なら「新しいパスを尋ねる」分岐（`prompt_for_new_path`、
`suggested_filename` の値を初期値として使う）に入る。既存テスト
`test_open_and_save_new_file` の保存フロー全体（パス選択ダイアログ→保存完了→言語判定→
再編集→再保存）で確認済み。

### 既存テストへの影響と修正

- `test_open_and_save_new_file`／`test_setting_language_when_saving_as_single_file_worktree`
  （`crates/zed/src/zed.rs`）: `NewFile` 直後の `editor.title(cx)` の期待値を、厳密な
  `"untitled"` から `"Untitled-"` プレフィックス検証に変更（表示番号はプロセスグローバルな
  状態のため、他のテストとの実行順序に依存させないため。「厳密に何番か」はこの2テストの
  関心事ではない）。`file()` を持つこと（LSP登録の前提）の検証を追加。
  機能が既定オフになったため、両テストとも冒頭で新設のテストヘルパー
  `enable_scratch_buffers(cx)`（`settings.workspace.scratch_buffers_enabled = Some(true)`）
  を呼んで明示的に有効化している。
- 既定オフのままの既存テスト（例 `test_new_empty_workspace`、`Editor::new_file` を直接
  呼ぶ）は、`scratch_buffers_enabled` を有効化していないため変更前と同じ経路
  （ファイルなしバッファ）を通り、無修正のまま成功することを確認済み。

### 新規テスト: `test_scratch_buffer_number_released_only_after_last_split_closes`

docs/design.md 確定事項3（分割ペインで最後の表示を閉じた際の扱い）を直接検証する。

1. `NewFile` → `scratch_display_number` で番号を取得、使用中であることを確認。
2. `split_and_clone` で同じバッファを2つ目のペインに複製（`clone_on_split` は
   `Editor::clone` 経由で同じ `Entity<MultiBuffer>`／`Entity<Buffer>` を共有することを
   `assert_eq!` で確認）。
3. 片方のペイン（`original_pane`）を閉じても、番号がまだ使用中のままであることを確認。
4. 最後のペイン（`new_pane`）も閉じたら、番号が解放されることを確認
   （`weak_buffer.assert_released()` と `is_display_number_in_use` の両方で）。

**デバッグで判明した非自明な点（実装のバグではなく、テストの書き方の問題だった）**:

- テストコード自身が `editor`（`Entity<Editor>`）を変数として保持し続けると、それ自体が
  強参照になり Editor が解放されない。`drop(editor)` が必要（実アプリではペインが唯一の
  保持者なので問題にならない）。
- `gpui` の `Subscription` を返す `cx.on_release` は `.detach()` しないと登録した瞬間に
  解除されてしまう（`Subscription::drop` が unsubscribe を呼ぶため）。
- **`Workspace::handle_pane_event` の `added_to_pane` は、アイテムがペインに追加される
  たび `enqueue_item_serialization` でセッション永続化用の
  `UnboundedSender<Box<dyn SerializableItemHandle>>` にアイテムのクローンを送る。**
  受信側 `Workspace::serialize_items` はバックグラウンドタスクとして
  `cx.background_executor().timer(SERIALIZATION_THROTTLE_TIME)`（200ms）で
  スロットルしながらチャンク処理するため、`cx.run_until_parked()` だけではこの
  タイマー待ちの分がフラッシュされず、送られたアイテム（Entity への強参照を含む）が
  チャネルのバッファ内に残り続け、`Editor`／`Buffer` が解放されない。
  `cx.executor().advance_clock(SERIALIZATION_THROTTLE_TIME)` を挟む必要がある
  （既存テストにも同じパターンが1件あった、`crates/workspace/src/workspace.rs:19431`）。
  この経路の特定には `gpui` の leak-detection 機能
  （`Entity::downgrade().assert_released()`、`LEAK_BACKTRACE=1`）を使った。
  デバッグ用に `crates/gpui/src/app/entity_map.rs` のバックトレースフィルタを一時的に
  無効化する変更を加えたが、原因特定後は元に戻した（パッチには含まれない）。
- 上記のいずれも「本当のリーク」ではなく、gpui のエフェクトサイクル／非同期タスクの
  完了待ちが足りなかっただけ。実装（番号解放ロジック）自体に問題はなかった。

### テスト結果

- `cargo test -p workspace`: 272 件成功（`scratch_buffers` の単体テスト3件含む）。
- `cargo test -p editor`: 1011 件成功（既存の editor クレート全体、regression なし）。
- `cargo test -p project_panel`: 125 件成功。
- `cargo test -p command_palette`: 21 件成功。
- `cargo test -p zed --bin zed`: 93 件成功（新規1件＋既存2件の修正含む）。
  8回の連続実行のうち2回、`test_multi_workspace_session_restore` が失敗したが、
  これは今回追加・変更したテストを `--skip` で除外しても同程度の頻度
  （5回中3回）で再現する、**今回のパッチと無関係な既存の flaky テスト**と確認した
  （単独実行では常に成功する）。
- 既定オフ設定の追加後、`workspace`（272件）・`editor`（1011件）・`zed --bin zed`
  （93件、5回連続実行すべて成功、`test_multi_workspace_session_restore` の
  flakiness も同程度の頻度で再現）を再実行し、regression がないことを再確認した。

### パッチ化

`git diff --cached`（`Cargo.lock` の依存追加分含む全変更）を
`patches/0001-scratch-buffers.patch` として書き出した。クリーンな
`7c451e694f3c52ee0aeb01d7e28b5fa18cd0ad2f` チェックアウトに対し `scripts/prepare`
で単独適用できることを確認し、適用後に `scripts/check`（`cargo check`／
`cargo test --lib` を `editor`・`workspace` の両クレートに対して実行）が
成功することも確認した。

## 2026-09-20: レビュー対応

統合検証に進む前に受けたコードレビューで、3件の問題と2件の改善点が指摘された。
検証したところ、指摘のうち番号管理に関する2件（最優先・優先）はコードを直接確認し
事実と判明したため対応した。

### 1. 再起動後の番号衝突（最優先、対応済み）

- **`paths::temp_dir()` の実態**: `crates/paths/src/paths.rs` を確認したところ、
  Linux では `$XDG_RUNTIME_DIR` ではなく `dirs::cache_dir()`（`~/.cache/zed` 相当）を
  返しており、再起動やログアウトでは消えない。design.md の「再起動で消える」という
  前提と実装が食い違っていた。
  - 対応: 保存先ディレクトリ自体は変更せず、docs/design.md 確定事項2を
    「Hot Exit のために意図的に残す」前提に改訂した（詳細は同ファイル参照）。
- **Hot Exit 復元時に番号が未予約**: `allocate_display_number` の呼び出し元が
  `new_scratch_buffer_path`（新規作成時のみ）に限られており、Hot Exit 等で
  `Editor::for_buffer` 経由で scratch パスのバッファが復元されても、番号が
  プロセス内の管理状態に登録されないことをコードで確認した。
  - 対応: `Editor::new_internal`（`for_buffer`／`for_multibuffer`／`clone`／Hot Exit
    復元のいずれも最終的に通る共通コンストラクタ）に
    `reserve_scratch_buffer_number_if_applicable` を追加し、経路を問わず
    singleton バッファが scratch パスを持てば番号を予約するよう一元化した
    （docs/design.md 確定事項6）。

### 2. 保存後も番号が解放されない（対応済み）

`release_display_number` の呼び出し箇所を全て確認したところ、`on_release`
（バッファ完全 drop 時）と作成失敗時の2経路しかなく、保存成功による即時解放が
存在しないことを確認した。

- 対応: `multi_buffer::Event::FileHandleChanged`（保存成功時に発火）で
  `release_scratch_buffer_number_if_no_longer_scratch` を呼ぶようにした。
- 実装の過程で、単純な無条件解放だと「保存成功時の即時解放」と
  「`on_release`」という2つの独立した解放経路が同じ番号に対して存在するため、
  片方が解放した直後にその番号が新しい下書きへ再割り当てされ、もう片方が
  無関係な別の下書きの番号を誤って解放してしまう事故が起こりうると判断した。
  `workspace::scratch_buffers` にバッファ ID ベースの所有者管理
  （`associate_display_number_with_buffer`／`display_number_for_buffer`／
  `release_display_number_owned_by`）を追加し、所有者が一致する場合のみ解放する
  ことでこれを防いだ（`workspace` クレートに `text` 依存を追加、`BufferId` を使う
  ため）。単体テスト
  `owned_release_ignores_numbers_reassigned_to_another_buffer` でこの事故が
  起きないことを検証済み。

### 3. i18n 未対応（対応不要と確認）

`docs/repository-separation-plan.md` の合意事項（新機能の翻訳接続は統合側
= zed-personal-build で管理し、機能パッチ自体は英語で成立させる）に従っている
ため、zed-scratch-buffers 単体パッチとしては設計違反ではないと確認した。
`format!("Untitled-{number}")` が実際に i18n 側の抽出・翻訳を通るかどうかは、
zed-personal-build での統合検証項目として `sources.lock.toml` に記録した
（このパッチ側での対応は不要）。

### 5. 補足的な改善点（対応済み）

- `is_scratch_path` が親ディレクトリ名の UUID 形式を検証していなかった点を修正。
  `scratch_display_number` 内で `uuid::Uuid::parse_str` により厳密に検証し、
  `is_scratch_path` もこれを使うようにした。単体テストを追加。
- パッチファイルの trailing whitespace: `git diff --cached --check` で確認したが、
  今回作成した差分には該当箇所がなかった（レビュー時点の差分に含まれていた
  箇所は、今回の追加修正で書き換えられ解消された）。
- 保存後に残る旧い `buffer.txt` の残骸蓄積、起動時クリーンアップについては、
  docs/design.md 確定事項2の改訂に伴い「今回のスコープでは見送り」と記録した。

### 追加したテスト

- `test_reopened_scratch_buffer_reserves_its_number`（`crates/zed/src/zed.rs`）:
  `SerializableItem::deserialize` の abs_path 分岐と同じ経路
  （`project.open_local_buffer` → `Editor::for_buffer`）で既存の scratch ファイルを
  開き直し、番号が予約されること、その状態で新規作成した下書きと番号が衝突しない
  ことを検証。
- `test_open_and_save_new_file` に、保存完了直後（`Editor` を閉じる前）に番号が
  解放されていることの検証を追加。
- `scratch_buffers.rs` に `owned_release_ignores_numbers_reassigned_to_another_buffer`・
  `reserve_display_number_is_idempotent_and_blocks_reallocation` を追加。

### 並列実行での新たな flaky 化と対処

上記のテストを追加した直後、`zed --bin zed` をフルスイートで繰り返し実行すると
`test_open_and_save_new_file` が時々失敗した。原因は、追加した
「保存直後に番号が解放されていること」のアサーションが、並列実行される他のテストの
番号採番と競合していたため（`enable_scratch_buffers` を呼ぶ4つのテストのうち、
`test_synchronization_lock` を取っていないものがあった）。

対処: `enable_scratch_buffers` ヘルパー自体が `test_synchronization_lock` を取得して
返すように変更し（`#[must_use]`、呼び出し元は `let _scratch_buffers_lock =
enable_scratch_buffers(cx);` で束縛）、scratch buffers を有効化する4つのテスト
全てを自動的に直列化した。

### テスト結果（レビュー対応後）

- `cargo test -p workspace`: 274 件成功（新規2件含む）。
- `cargo test -p editor`: 1011 件成功。
- `cargo test -p project_panel`: 125 件成功。
- `cargo test -p command_palette`: 21 件成功。
- `cargo test -p zed --bin zed`: 94 件成功（新規1件含む）。10回の連続実行で
  私が触れた4テストは一度も失敗せず、既存の無関係な flaky テスト
  （`test_multi_workspace_session_restore`・
  `test_restored_project_groups_survive_workspace_key_change`、いずれも
  "session restore" 系）は、対象テストを `--skip` で除外しても同程度の頻度
  （5回中1〜3回）で再現することを確認した。
- パッチを `patches/0001-scratch-buffers.patch` として再生成し、クリーンな
  `7c451e694f3c52ee0aeb01d7e28b5fa18cd0ad2f` への単独適用・`scripts/check`
  （editor 1011件・workspace 274件）の成功を再確認した。

### 未実施（次フェーズ）

- `cargo check`／`cargo test` レベルの検証のみ。`zed-personal-build` 側での
  `apply-universal` 後のチェックアウトへの適用検証、統合ビルド、GUI 起動・動作確認は未実施。
- `docs/scratch-buffer-research.md` にある「保存済みファイルの未保存の編集」と
  「一度も保存していない新規タブ」の区別が、この実装によって解消されたことの
  実機確認。
- i18n 側で `format!("Untitled-{number}")` が抽出・翻訳可能かの確認
  （統合検証項目、上記「3. i18n 未対応」参照）。
- 起動時の古い scratch エントリのクリーンアップ（今回は見送り、
  docs/design.md 確定事項2参照）。

## 2026-09-20: レビュー再確認（1件）

前回のレビュー対応後、再レビューで1件の指摘を受けた。

### `reserve_scratch_buffer_number_if_applicable` が `is_scratch_path` を経由していない

`scratch_display_number(path)` は親ディレクトリ名の形式
（`<number>-<uuid>`、UUID の妥当性含む）しか確認せず、`scratch_root_dir()` 配下に
あるかどうかは見ていない。この判定は `is_scratch_path` にしか含まれていないが、
`crates/editor/src/editor.rs` の `reserve_scratch_buffer_number_if_applicable` は
`scratch_display_number` だけを呼び、`is_scratch_path` を経由していなかった。
そのため、`scratch_root_dir()` の外にある、たまたま同じ命名パターン
（`<number>-<uuid>/buffer.txt`）を持つだけの通常のユーザーファイルを開くと、
表示上は壊れない（タブ表示側は `is_scratch_path` を正しく使っている）ものの、
番号だけを消費してしまうことをコードで確認した。

対処: `is_scratch_path(&abs_path)` のチェックを `scratch_display_number` の前に
追加した。修正の妥当性を検証するため、一時的にこのチェックを外して新規テスト
`test_opening_lookalike_file_does_not_reserve_a_scratch_number` を実行し、
確かに失敗すること（バグを検出できること）を確認してから、チェックを復元した。

`git diff --cached --check` と `git apply --check --whitespace=error`
（クリーンな `7c451e694f3c52ee0aeb01d7e28b5fa18cd0ad2f` に対して）の両方で
trailing whitespace が無いことを確認した。

editor 1011件・workspace 274件・zed --bin zed 95件（新規1件含む、3回連続実行で
対象テストは一度も失敗せず、既存の無関係な flaky テストのみ発生）すべて成功。
パッチを再生成し、クリーンな `7c451e694f3c52ee0aeb01d7e28b5fa18cd0ad2f` への
単独適用・`scripts/check` の成功を再確認した。
