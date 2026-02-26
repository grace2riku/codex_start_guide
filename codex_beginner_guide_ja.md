# Codex初心者ガイド（Claude Code経験者向け）

## 1. Codexとは
Codexは、コードの作成・修正・レビュー・テスト実行までをエージェントとして自律的に進められるAIコーディングパートナーです。ターミナルやIDEだけでなく、クラウド上のタスク実行や専用アプリでの並列作業が特徴です。

## 2. 使い始め方（最短ルート）
### 2.1 どこで使うかを選ぶ
Codexは複数の入口があります。

- Codex CLI（ターミナル）
- Codex IDE拡張（VSCode/Cursor/Windsurf）
- Codex Web / Codex App（クラウドタスク管理）

### 2.2 基本の使い方（Claude Codeと似た流れ）
Claude Codeと同様に「リポジトリ内で自然言語指示」を出すのが基本です。Codexは以下が得意です。

- リポジトリ探索
- 複数ファイル修正
- テスト実行
- 変更差分の提示

Codexは各タスクが独立したサンドボックス環境で動作します。

### 2.3 Codex CLI 深掘り
#### 2.3.1 サンドボックス環境とは
サンドボックス環境とは、タスクごとに独立した実行空間を用意し、そこでコード実行や編集を行う仕組みです。Codexのクラウド実行では、各タスクが隔離されたサンドボックス内でリポジトリと実行環境を持つことが明示されています。CLIでも、モデルが実行するシェルコマンドはサンドボックスポリシーで制限できます。

代表的なポリシー:
- `read-only` / `workspace-write` / `danger-full-access` の3段階で、モデル生成コマンドの実行範囲を制御
- `--full-auto` は `--ask-for-approval on-request` と `--sandbox workspace-write` の低摩擦プリセット

#### 2.3.2 基本コマンド紹介
まず覚えるべきCLIコマンドは以下です。

- `codex`  
  対話型UIで起動。リポジトリを読み、編集し、コマンド実行しながら進めます。
- `codex exec`  
  非対話で実行。スクリプトやCIで使いたい場合に有効。
- `codex resume`  
  直前のセッションや指定IDのセッションを再開。
- `codex login` / `codex logout`  
  認証の開始・解除。
- `codex features`  
  機能フラグの一覧・有効化・無効化。
- `codex mcp`  
  Model Context Protocol (MCP) サーバー設定の管理。
- `codex sandbox -- <command...>`  
  Codexと同じサンドボックスポリシーでコマンド実行。

よく使うオプション:
- `--model, -m`  
  モデルを指定。
- `--image, -i`  
  初回プロンプトに画像を添付。
- `--ask-for-approval, -a`  
  コマンド実行前の承認タイミングを指定。
- `--sandbox, -s`  
  サンドボックスのポリシーを指定。
- `--add-dir`  
  追加で書き込み許可したいディレクトリを指定。
- `--cd, -C`  
  作業ルートを指定。
- `--output-last-message, -o`  
  最終出力をファイルに保存。
- `--json`  
  JSONイベント出力に切り替え（CI向け）。

#### 2.3.3 ソフトウェア開発の流れでの使い方
開発フェーズごとに、CLIの機能を対応付けると運用が楽になります。

- 調査/理解  
  `codex` 起動 → 仕様理解やコード読解。必要なら `/diff` で変更点を確認。
- 設計/方針整理  
  `/review` で作業ツリーをレビューして課題整理。
- 実装  
  対話型で編集し、必要なら `--full-auto` で低摩擦運用。
- テスト/検証  
  CLIはコマンド実行ができるので、通常の `make test` や `ctest` などを回して反復。
- レビュー/差分確認  
  `/diff` と `/review` を併用して確認。
- 継続運用  
  `/init` で `AGENTS.md` を生成し、リポジトリ固有ルールを残す。

#### 2.3.4 組込みソフトウェア開発者向けの使い所
組込み開発で相性が良い観点は以下です。

- 大規模C/C++コードベースの理解  
  既存ドライバやボード依存層の概要整理に有効。
- SDKやツールチェーンが別ディレクトリにある場合  
  `--add-dir` で書き込み許可を追加して作業範囲を広げられる。
- ビルドルートを限定したい場合  
  `--cd` でファームウェア部分だけを対象にできる。
- 安全性重視の作業  
  `--sandbox read-only` で閲覧中心にし、`/permissions` で承認モードを切り替えながら進める。
- レビューや差分の見直し  
  `/diff` と `/review` をルーチン化すると品質を落としにくい。

#### 2.3.5 具体的なCLIコマンド例
実務でよく使うパターンを、最小限の例としてまとめます。

```bash
# 1) リポジトリのルートで対話起動
codex

# 2) 設計書や配線図などの画像を添付して起動
codex -i docs/spec.png

# 3) モデルを指定して起動
codex -m gpt-5-codex

# 4) 作業ディレクトリを限定して起動（ファームウェアだけ触りたい時）
codex -C firmware

# 5) 追加の書き込み許可ディレクトリを指定
codex --add-dir ../shared_libs

# 6) 低摩擦モード（承認はon-request、サンドボックスはworkspace-write）
codex --full-auto

# 7) 承認ポリシーを明示
codex -a on-request
codex -a never

# 8) サンドボックスポリシーを明示
codex --sandbox workspace-write

# 9) サンドボックス下で単発コマンドを実行
codex sandbox -- make test

# 10) 進捗や最終出力を機械処理しやすくする
codex --json --output-last-message /tmp/codex_last.txt
```

#### 2.3.6 組込み向け運用テンプレート（CLI）
日々の作業を回しやすいように、テンプレート化した運用例です。

```text
目的: 既存ドライバの初期化シーケンスを理解して改善
制約: API互換は維持、公開ヘッダは変更しない
対象: firmware/drivers/*, firmware/board/*
テスト: make test, ctest, またはターゲット固有のビルドコマンド
```

推奨フロー:
- 読み取り中心: `codex -a on-request --sandbox read-only` で開始
- 変更作業: `codex --full-auto` で反復
- 差分確認: CLI内で `/diff` → `/review` を実施
- ルールの定着: `/init` で `AGENTS.md` を生成し、プロジェクト固有ルールを保存

#### 2.3.7 組込み向け運用テンプレート（Renesas RA8P1 / FreeRTOS）
RA8P1 + FreeRTOS前提で、実運用に寄せた具体例です。プロジェクト固有のビルド手順やターゲット名は読み替えてください。

```text
目的: RA8P1のタイマ割り込みで使用するFreeRTOSタスクの優先度設計を見直す
制約: 既存API互換維持、割り込みレイテンシの悪化は避ける、スタック使用量増は最小限
対象: firmware/rtos/*, firmware/board/ra8p1/*, firmware/drivers/timer/*
テスト: (例) make test / ctest / board固有のビルド＆フラッシュコマンド
```

Codexへの指示例:

```text
目的: RA8P1のタイマ割り込み経由で起床するFreeRTOSタスクの優先度とスタックサイズを整理
制約: API互換維持、割り込みレイテンシ悪化禁止、stack overflowを回避
対象: firmware/rtos/*, firmware/board/ra8p1/*, firmware/drivers/timer/*
出力: 変更点サマリとdiff提示
テスト: make test
```

推奨フロー:
- 読み取り中心: `codex -a on-request --sandbox read-only`
- 変更作業: `codex --full-auto` で反復
- タスクの妥当性確認: `/review` で優先度やリソース使用の懸念点を列挙
- 差分確認: `/diff` で実変更を確認

## 7. 質問と回答
### Q1. ステータス行の「95% left」とは何ですか？
Codex CLIのフッター（ステータス行）は `/statusline` で表示項目をカスタマイズできます。表示項目には **モデル名、コンテキスト統計、レート制限、トークンカウンタ、セッションID、作業ディレクトリ** などがあり、`95% left` はそのいずれか（主に **残りコンテキスト容量** や **レート制限の残量**）を示している可能性が高いです。citeturn14view1

確実に判定する方法:
- `/status` を実行すると、**現在のトークン使用量** と **残りコンテキスト容量** が表示されます。citeturn14view1
- `/statusline` を実行すると、ステータス行に何が表示されているかを確認・変更できます。citeturn14view1

この2つで「95% left」が何の残量かを特定できます。

### Q2. `/status` の Context window / 5h limit / Weekly limit は何ですか？
**Context window**  
モデルが一度に保持できる文脈（トークン）容量の残量です。`26.9K used / 258K` は、**現在の会話で 26.9K トークンを使用し、最大 258K まで保持できる**という意味です。`94% left` は残り割合です。トークンは入力・出力の双方で消費されます。citeturn0search1

**5h limit**  
Codexの**5時間あたりの利用上限**の残量です。タスクの規模や複雑さ、実行場所（ローカル/クラウド）で消費が変わり、`resets 12:20` のように次回リセット時刻が表示されます。citeturn0search0turn0search2

**Weekly limit**  
Codexの**週次の利用上限**の残量です。`resets 13:44 on 5 Mar` のように、週次枠のリセット日時が表示されます。消費はタスク規模や実行場所により増減します。citeturn0search0turn0search2

## 3. Claude Codeとの比較（実務視点）
### 3.1 似ている点
- どちらもターミナルから自然言語で指示できるCLIがある
- どちらもコード説明・修正・テスト支援が可能

### 3.2 Claude Codeにはなく、Codexにある強み
以下はClaude Codeには公式にない、またはCodexの方が明確に強い領域です。

- 専用アプリでのマルチエージェント並列作業
- クラウド上での長時間タスク実行
- 自動化機能（Automations）
- Slack連携・管理機能・SDK

## 4. Claude Code経験者におすすめの使い分け
- 手元で素早く質問・修正
  どちらもCLIでOK。Claude Code感覚でCodex CLIを使うとスムーズ。
- 長い作業・複数タスクを並列化したい
  Codex Appを使う価値が大きい。特に並列タスクと独立作業ツリーが強い。

## 5. Codexで失敗しにくい指示の出し方
Claude Codeと同様、以下が効きます。

- 目的 → 制約 → 出力形式 の順で書く
- 修正対象ファイルや関数名を明示
- テスト実行の有無を指定
- 期待する差分の粒度を指示

例:

```text
目的: 認証エラーの原因調査と修正
制約: 既存のAPIレスポンス形式は変更しない
出力: 変更点を要約してからパッチを提示
対象: src/auth/session.ts
テスト: npm test -- auth
```

## 6. まとめ（要点）
- CodexはCLI・IDE・クラウドアプリの3面展開
- Claude Code経験者なら、CLI利用はほぼ同じ感覚
- 並列作業・クラウド長時間タスク・自動化がCodexの強み

---

必要なら、次のどれかに合わせて「あなたの環境向けの具体的な手順」に落とし込みます。

1. Codex CLIの初期設定と認証手順
2. Codex Appの使い方と並列作業の設計
3. Claude Code→Codexの移行チェックリスト
