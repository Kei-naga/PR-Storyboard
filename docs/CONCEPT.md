# PR-Storyboard コンセプト v0.3

> 検討結果を反映した版。「決定」は合意済み、「提案」は未合意の推奨、「未決」は今後判断が必要な事項。
>
> v0.3 の変更: 配布形態を「Claude Code プラグイン」と「セルフホストサーバ」の 2 形態に。LLM 利用の規約整理（§7）を追加。

---

## 0. 決定事項サマリ

| 論点 | 決定 |
|---|---|
| 主な読み手 | レビュアー（エンジニア） |
| 中核の約束 | 変更の**概要・方針・内容・理由**を効率よく把握させ、レビューをボトルネックにしない。コードを読むかどうかは利用者に委ねる（中立） |
| 表示面 | Web ページ（ボード） |
| 配布形態 | **2 形態**: ① Claude Code プラグイン（手元で `/storyboard`）② セルフホストサーバ（Webhook で自動生成し PR にリンク） |
| 共通化 | 決定的処理・スキーマ・描画を **Core（ライブラリ + CLI）** に集約し、両形態で共有 |
| アーキテクチャ方針 | OSS。インフラ依存を排したシンプル構成（サーバは単一コンテナ・DB なし） |
| SCM | MVP は GitHub のみ。SCM アダプタ層は切っておく |
| 生成タイミング | サーバ形態では利用者設定で可変 |
| LLM | サブスク利用の本命はプラグイン形態。サーバ形態はチーム利用なら API キー推奨、個人利用なら公式 CLI も可（§7） |
| 静的解析 | 言語非依存・浅く |
| 閲覧認証 | サーバ形態で設定により選択（none / 署名 URL / GitHub OAuth） |

---

## 1. プロダクト定義

- **名称**: PR-Storyboard（暫定。§13 参照）
- **タグライン案**: 「コードを読むかは、あなたが決める。」
  - サブ: 「PR の理由・方針・影響を 1 ページで。手元の Claude Code でも、1 コンテナのサーバでも。」
- **一言定義**: PR ごとに「なぜ・どういう方針で・何を変え・どこに影響し・何が危ないか」を 1 枚の Web ページ（ボード）にまとめるツール。手元で呼び出すプラグインとしても、PR に自動でリンクを届けるセルフホストサーバとしても動く。

### 1.1 Why

- AI によるコード生成で PR の量とサイズが増え、**人間の理解がボトルネック**になっている。
- ボトルネックの正体は「1 行ずつ読むこと」そのものより、**読む前の全体把握**（何のための変更か、どこを読めば良いか、何が危ないか）にかかる時間。
- 本プロダクトはこの「読む前」の時間を削る。精読するか、ざっと見るか、任せるかは**チームと個人の流儀に委ねる**。レビューを省略させる道具でも、強制する道具でもない。

---

## 2. 設計原則

1. **中立**: 「承認してよい」「レビュー不要」といった判定は出さない。出すのは事実とシグナルまで。
2. **事実と AI 推定を分離する**: 依存追加・マイグレーション・設定変更・ノイズ判定などは**決定的処理**で出す。LLM は解釈と文章化のみを担う。UI 上でも「検出（事実）」と「AI 推定」を視覚的に区別し、各主張に根拠（ファイル・hunk）へのリンクを付ける。
3. **読まなくていいものを明示する**: 除外したファイルは隠さず「lockfile 3・スナップショット 12・生成コード 5 を除外」のように件数と理由を見せる。
4. **鮮度を明示する**: ボードはどのコミット（head SHA）に対するものかを常に表示し、古くなったら目立たせる。
5. **シンプルな運用**: DB なし・外部キューなし。状態はファイルだけ。
6. **認証情報に触れない**: LLM の認証は公式ツールか利用者の API キーに任せる。本プロダクトがサブスクの認証情報を読む・保存する・中継することはしない。

---

## 3. 配布形態

同じ Core を 2 つの入口から使う。

```
                   ┌──────────────── Core（ライブラリ + CLI）────────────────┐
                   │ collect → analyze（決定的） → [narrative] → validate → render │
                   │ board.json スキーマ / HTML テンプレート / 規則セット          │
                   └────────────────────────────────────────────────────────┘
                        ▲                                   ▲
      narrative を書くのは│                                   │narrative を書くのは
      Claude Code 自身    │                                   │LLM バックエンド
                        │                                   │
   ① Claude Code プラグイン                      ② セルフホストサーバ
   手元で /storyboard 123                        Webhook/polling → 自動生成
   → ローカルにボード HTML                        → PR にリンク付き sticky コメント
```

| | ① プラグイン | ② サーバ |
|---|---|---|
| 主な利用者 | レビュアー個人（自分が読む PR をその場で可視化） | チーム（PR ごとに自動でボードが届く） |
| 起動 | 手動（`/storyboard`） | 自動（イベント・コマンド・ラベル） |
| LLM | 利用者の Claude Code セッションそのもの（サブスク可） | API キー推奨。個人利用なら公式 CLI も可 |
| 出力 | ローカルの自己完結 HTML をブラウザで開く | サーバがボードを配信し PR にリンク |
| インフラ | なし | 単一コンテナ |
| 規約上の位置付け | Claude Code の通常利用そのもの（最も安全） | 条件付き（§7） |

---

## 4. ボード構成（両形態共通）

1 画面目で「理由・方針・影響・リスク」が掴めることを目標とする。

| # | セクション | 内容 | 出所 |
|---|---|---|---|
| H | ヘッダ | PR タイトル、base…head SHA、生成日時、生成手段、**古さ表示** | 事実 |
| 1 | **サマリ** | Why（変更理由）/ 方針（どういうアプローチを選んだか）/ What（何が変わったか）/ 利用者・クライアント影響の有無 | AI 推定 |
| 1b | **PR 説明との乖離** | PR 本文に書かれていない変更、書かれているのに実装されていない変更 | AI 推定（AI 生成 PR で特に効く） |
| 2 | **変更マップ** | モジュール依存グラフ上で変更ノードを色付け（変更前後が分かる差分図）。処理フロー／シーケンス図は該当する場合のみ | グラフ＝事実、フロー図＝AI 推定 |
| 3 | **読むべき箇所** | 本質的な変更を含む上位ファイル（2〜5 件）と選定理由。全ファイルの 1 行要約（折りたたみ）。除外ファイルの件数と理由 | 分類＝事実、要約・順位＝AI 推定 |
| 4 | **リスク・副作用シグナル** | 下表 | 混在（ラベル付き） |

### 4.1 リスク・副作用シグナル

| シグナル | 検出方法 |
|---|---|
| 新規／メジャーアップデートの依存パッケージ | manifest/lockfile の差分解析（決定的） |
| DB スキーマ／マイグレーション変更 | パスパターン＋ファイル種別（決定的） |
| 環境変数・設定ファイル・フィーチャーフラグ | `process.env` / `os.Getenv` 等のパターン、設定ファイルのパス（決定的） |
| CI／デプロイ設定の変更 | `.github/workflows` / Dockerfile / IaC パス（決定的） |
| 認証・権限まわりのパスへの変更 | パスとキーワードのヒューリスティック（決定的） |
| 公開 API／export の追加・削除・シグネチャ変更 | tree-sitter による浅い抽出（決定的） |
| テストの削除・変更規模に対してテストが少ない | 統計（決定的） |
| シークレットらしき文字列 | パターンマッチ（決定的） |
| 過剰実装の兆候 | 実装が 1 つしかない新規抽象、未使用の export、既存依存と機能が重なる新規依存、意図に対して不釣り合いな変更量（事実＋AI 推定） |
| 振る舞い変更なのにテストが追従していない | AI 推定 |

---

## 5. Core

### 5.1 パイプライン

| 段 | 役割 | 実装 |
|---|---|---|
| `collect` | PR 情報・diff・本文・コミットを取得 | GitHub API（SCM アダプタ）またはローカル git |
| `analyze` | 決定的解析（§6）→ `facts.json` | Core |
| narrative | 要約・方針・順位付け・乖離検出 → `narrative.json` | **形態ごとに異なる**（プラグイン＝Claude Code、サーバ＝LLM バックエンド） |
| `validate` | `narrative.json` を JSON スキーマで検証し、根拠リンクが実在するか確認 | Core |
| `render` | `facts` + `narrative` → `board.json` → HTML | Core（テンプレートは決定的） |

- **LLM の出力は信用しない**: スキーマ検証を必須とし、HTML には必ずエスケープして埋め込む。LLM 由来の Mermaid は構文検証とサニタイズを通す（`securityLevel: 'strict'`）。
- **巨大 PR**: ノイズ除外後にモジュール単位で分割して要約し、統合する（map-reduce）。上限超過時は縮退版（事実のみ）かスキップかを選べる。

### 5.2 CLI（提案）

```
pr-storyboard collect  --pr 123 [--repo owner/name]  → .storyboard/<pr>-<sha>/pr.json
pr-storyboard analyze  → .storyboard/<pr>-<sha>/facts.json
pr-storyboard validate .storyboard/<pr>-<sha>/narrative.json
pr-storyboard render   → .storyboard/<pr>-<sha>/board.html（自己完結）
pr-storyboard serve    （サーバ形態）
```

### 5.3 技術スタック（提案）

- **TypeScript / Node.js**。Claude Code・Codex・Gemini の各 CLI が Node 系であり、プラグインからも `npx` で呼べる。Octokit・web-tree-sitter（WASM）・Mermaid とも相性が良い。
- 配布: npm パッケージ（Core/CLI）、Claude Code プラグイン（マーケットプレイス）、Docker イメージ（サーバ）。

---

## 6. 決定的解析（言語非依存・浅く）

- **import 抽出**: web-tree-sitter と主要言語向けクエリで行う。クエリがない言語は正規表現にフォールバックする。モジュール（ディレクトリ）単位の依存グラフを作る。
- **manifest 差分**: package.json / lockfile、go.mod、requirements・pyproject、Cargo.toml、Gemfile、pom/gradle など。
- **ノイズ分類**: `.gitattributes` の `linguist-generated` / `linguist-vendored`、`Code generated ... DO NOT EDIT` などのマーカー、lockfile、スナップショット、フィクスチャ、モックのパスパターン。
- **パス規則**: マイグレーション、CI、IaC、設定ファイル、認証関連。
- 利用者は `.pr-storyboard.yml` で規則を追加・上書きできる（§10）。

---

## 7. LLM 利用と規約

> 公開情報に基づく整理で、法的助言ではない。各社のポリシーは短期間で変わるため、実装時と公開前に再確認する。

### 7.1 前提（Anthropic、2026-09 時点）

- サードパーティ製品が Claude.ai ログインを提供すること、利用者の代わりにサブスクの認証情報でリクエストを流すこと、認証情報やセッショントークンを収集・保存・中継することは**禁止**。
- 一方で、利用者が**改変していない Claude Code** に**自分のサブスク**でログインして使うことは、他製品の中で Claude Code が動く場合も含めて妨げない、と明記されている。
- Pro/Max の利用上限は**通常の個人利用**を前提にしている。
- 2026-06-15 に予定されていた「`claude -p`・Agent SDK は別枠クレジットから消費」という変更は**保留中**。現状はサブスクの枠から消費される。将来変わる前提で設計する。
- 出典: [Claude Code Docs: Legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)、[Claude Help Center: Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)

### 7.2 形態ごとの扱い

| 形態 | LLM | 条件 |
|---|---|---|
| ① プラグイン | 利用者の Claude Code セッション | 利用者自身の通常利用。本プロダクトは認証に関与しない |
| ② サーバ（チーム） | **API キー**（Anthropic / OpenAI / Gemini、将来 Bedrock・Vertex・Azure） | キー所有者の契約で課金。再販・仲介をしない |
| ② サーバ（個人） | 公式 CLI（`claude -p` / `codex exec` / `gemini`）も可 | 下記の条件をすべて守る |

公式 CLI バックエンドの条件:

- CLI は公式配布物を**改変せず**に使う。
- ログインは利用者がコンテナ内で **CLI 自身の公式フロー**で行う。認証情報は CLI が自分で管理するボリュームに置き、本プロダクトのコードからは読まない。トークンを本プロダクトの設定ファイルに貼らせる方式は採らない。
- **1 人のサブスクでチーム全員の PR を処理する構成は非推奨**とドキュメントに明記し、チームには API キーを案内する。
- 製品名・ロゴに「Claude Code」「Anthropic」を使わない。「Claude Code で動く」と文章で書くのは可。
- Codex CLI（ChatGPT プラン）と Gemini CLI には、それぞれ別の規約が適用される。個別に確認する。

---

## 8. ① Claude Code プラグイン形態

### 8.1 フロー

```
/storyboard 123            （PR 番号・URL・省略時は現在ブランチ vs base）
  1. pr-storyboard collect + analyze   → facts.json
  2. 対象 head を読み取り専用 worktree に展開（git fetch origin pull/123/head）
  3. Claude が facts.json と周辺コードを読み、narrative.json をスキーマどおりに書く
  4. pr-storyboard validate            → 失敗したら修正して再検証
  5. pr-storyboard render              → board.html をブラウザで開く
```

- 利用者はレビュアー本人。自分が読む PR をその場で可視化する（個人の通常利用）。
- Claude Code がエージェントとして周辺コードを読めるため、diff だけから推定するより設計意図の推定精度が高い。
- 構成: スキル（手順と JSON スキーマの指示）＋ `/storyboard` コマンド。GitHub 認証は `gh` CLI か `GITHUB_TOKEN` を使う。
- オプション（既定 OFF）: `--comment` で、サマリ数行を利用者自身のアカウントで PR にコメントする。

### 8.2 セキュリティ

- PR の中身はプロンプトインジェクションの入力になり得る。スキルの `allowed-tools` を Read / Grep / Glob と `pr-storyboard` CLI に限定し、書き込み・ネットワーク・任意シェル実行をさせない。
- 作業は使い捨ての worktree で行い、利用者の作業ツリーに触れない。

---

## 9. ② セルフホストサーバ形態

### 9.1 全体像

```
                ┌──────────────── PR-Storyboard（単一コンテナ）────────────────┐
GitHub ─webhook▶│ Receiver ─▶ Job Runner（プロセス内キュー・同時実行数制限）       │
  ▲   (or poll) │               ├─ Core: collect → analyze                    │
  │             │               ├─ Workspace: bare mirror + 読み取り専用 worktree │
  │             │               ├─ Narrator: LLM バックエンド（§7.2）           │
  │             │               ├─ Core: validate → render                    │
  │             │               └─ Store: /data/{repo}/{pr}/{sha}/board.json   │
  └─comment─────│ Commenter（sticky コメント更新）                              │
                │ Viewer：board.json を HTML テンプレートで描画 + 認証（§9.4）   │
                └─────────────────────────────────────────────────────────────┘
```

### 9.2 PR コメント（sticky・1 つを上書き更新）

```
📘 PR-Storyboard  (head: a1b2c3d)
認証トークンの保存先を Cookie に移行し…（1〜2 行）
⚠ DB migration  ⚠ 新規依存 2  ・ 読むべき 3 ファイル
→ ボードを開く
```

### 9.3 要点

- **状態は `board.json` だけ**: HTML は閲覧時にテンプレートで描画する。テンプレートを改善すれば過去のボードにも反映される。単一 HTML としてのダウンロードも可能にする。
- **DB なし**: 一覧はファイルシステムの走査で足りる規模を前提にする。ストレージはアダプタ化し、将来 S3 互換にも対応可能にする。
- **再起動に耐える最小限**: キューはメモリ上に持つ。再起動で失われたジョブは、次のイベントかコマンドで再生成される。
- **連続 push 対策**: 同じ PR の古いジョブは新しい push でキャンセルする（debounce）。
- **GitHub App 方式**: 権限は `pull_requests: write`・`contents: read`・`metadata: read` に最小化する。App Manifest フローで「ボタン 1 つで App を作成」できるようにする。
- **fork PR も扱える**: シークレットは CI ではなくサーバ側にあるため、GitHub Actions の fork 制約を受けない。
- **受信経路**: webhook（公開 URL が必要）と **polling モード**（社内ネットワークなど外から到達できない環境向け）を選べる。
- **エージェント実行の隔離**: CLI バックエンドを使う場合、読み取り専用ツールのみを許可し（Claude Code は `--allowedTools`、Codex は read-only sandbox）、ネットワーク・書き込みを禁止し、環境変数にシークレットを渡さない。

### 9.4 閲覧認証（設定で選択）

| モード | 用途 | 備考 |
|---|---|---|
| `signed-url`（**デフォルト**） | 一般用途 | HMAC 署名付きの推測困難な URL。有効期限は任意 |
| `github-oauth` | 機密性の高いリポジトリ | 閲覧者がリポジトリの読み取り権限を持つかを GitHub API で確認する |
| `none` | VPN 内・ローカル | サーバの置き場所で守る前提 |

「外部 SaaS にコードを預けない」は正確には「**利用者が選んだ LLM プロバイダ以外の第三者に渡らない**」と表記する。

---

## 10. 設定

- サーバ設定（環境変数またはファイル）: 閲覧認証・LLM バックエンド・ストレージ。
- リポジトリ設定（`.pr-storyboard.yml`）: 生成ポリシーと規則。プラグイン形態でも規則（noise / risk / language）は同じファイルを読む。

```yaml
# .pr-storyboard.yml（例）
language: ja
triggers:                     # サーバ形態のみ
  events: [opened, ready_for_review, synchronize]
  command: /storyboard
  label: storyboard
  debounce_seconds: 120
skip:
  draft: true
  max_changed_lines: 20000
  paths_only: ["docs/**", "**/*.md"]
backend:                      # サーバ形態のみ
  prefer: [anthropic-api, claude-cli]
  on_rate_limit: fallback     # wait | fallback | skip
noise:
  extra: ["**/__generated__/**"]
risk:
  auth_paths: ["src/auth/**", "src/middleware/session*"]
```

---

## 11. ポジショニング

CodeRabbit も要約・ファイル表・シーケンス図を含む walkthrough コメントを出している。したがって「図があること」は差別化にならない。差別化の軸は次のとおり。

| 軸 | PR-Storyboard | CodeRabbit / PR-Agent 等 |
|---|---|---|
| 目的 | レビュー**前**の全体把握（中立） | レビューそのもの（指摘・提案） |
| 出力 | 専用 Web ページ（折りたたみ・フィルタ・差分図などコメント欄の制約を受けない） | PR コメント／インライン指摘 |
| 信頼性設計 | 事実と AI 推定を分離し、根拠へリンク | 主に LLM 出力 |
| 運用 | プラグイン（インフラなし）またはセルフホスト 1 コンテナ | SaaS 中心 |
| コスト | 個人は手持ちの Claude サブスク、チームは自前 API キー | 席課金 or API 従量 |

- 競合というより**共存**を想定する。インライン指摘は既存ツールに任せ、本プロダクトは「読む前」に特化する。

---

## 12. スコープ

### MVP（段階的に）

1. **Core + プラグイン**（先行）: `collect` / `analyze` / `validate` / `render`、ボード全セクション、スキル＋`/storyboard`。サーバなしで価値を検証できる。
2. **サーバ**: GitHub App（webhook / polling）、単一コンテナ、ファイルストア、sticky コメント、`/storyboard` コマンド、`signed-url` / `none` 認証、API バックエンド 1 種以上 + `claude-cli`。

### 次段階

- `github-oauth` 認証、S3 互換ストア、API バックエンド拡充（Bedrock・Vertex・Azure）、`codex-cli` / `gemini-cli`
- 処理フロー／シーケンス図、過剰実装シグナルの拡充
- push 間差分（「前回のボードから何が変わったか」）
- 各主張への 👍/👎 フィードバック（JSONL に追記。精度計測用）
- プラグインで生成したボードをサーバへアップロードして共有
- 他エージェント（Codex・Gemini CLI）向けの拡張
- GitLab アダプタ

### やらないこと

- インラインのコード指摘、自動修正、承認・マージの自動化、ユーザ管理・課金基盤、ホスティング型 SaaS の運営

---

## 13. 未決事項・リスク

| # | 項目 | メモ |
|---|---|---|
| 1 | 名称 | 比較対象の Storybook と紛らわしい。「Glance」系は 30 秒の約束と直結する。いずれにせよ「Claude」を名前に含めない |
| 2 | 規約変更リスク | サブスク周りのポリシーは短期間で変わる。サーバ形態は API キーへ即切り替えられる設計を保つ |
| 3 | 生成品質の検証 | 実際の PR 20〜30 件で「サマリの正確さ」「読むべきファイルの妥当性」を人手評価してから公開する |
| 4 | 生成時間 | エージェント型は API より遅い（数分）。サーバ形態で PR 作成直後に「生成中」コメントを出すか |
| 5 | コメントの粒度 | サーバ形態の PR コメントにどこまで載せるか。現案は 1〜2 行＋バッジ＋リンク |
| 6 | 成功指標 | PR 作成から最初のレビューまでの時間、ボード閲覧率（アクセスログ）、フィードバックの正答率 |
