# loop-kit

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

[English](README.md) · [日本語](README.ja.md)

> 📖 **はじめての方へ**: 図解でやさしく説明した [非技術者向け解説](docs/explainer-ja/)（[プレビュー画像](docs/explainer-ja/preview.png)）を用意しました。

Claude Code 用の **自己検証ビルドループ**。YES/NO で判定できる合格基準を凍結し（一意の run dir に
sha256 で固定）、各反復で **実装 → L0 決定的ゲート（test/lint/typecheck）→ 敵対的 checker
（builder ≠ checker、使えるなら別系統）→ 修正 → 反復（有界・正直な停止）**。価値はマルチエージェント
の魔法ではなく、**ループを外部検証に接地すること**。品質がモデル自身の甘い自己採点に依存しない
ようにする点にある。

## なぜ

- LLM に「これで完成？」と訊くと甘く自己採点する。通るまで基準を緩める（Goodhart）。
- 対策は、モデルが言い逃れできない **決定的ゲート** をループに置くこと:
  `node --test`、`pytest`、`tsc --noEmit`、`ruff`。これが **L0 ＝ 砦**。
- L0 の上に **checker** が敵対的に検品し（仕事は承認でなく欠陥発見）、
  **builder ≠ checker** で単一モデルの盲点が通りにくくなる。
- 基準は **先に凍結**（通すために緩めない）。一意の per-run dir に書いて **sha256 で固定**する
  ので、凍結の改変は検知できる。反復は **有界**（〜3、空回りは早期停止）。
- 何にでもループを被せない: **入口ゲート**が軽微な作業を弾き、決定的チェックが無く作れもしない
  対象には「ここではループは効かない」と正直に言う。

## 仕組み

```
  /loopify <あなたの依頼>
        |
        |  入口ゲート: 軽微 or L0 を作れない -> ループしない
        v
  +----------------------------------------------------------------+
  | 凍結: 3-5 個の YES/NO 基準 -> 一意の run dir、sha256 固定      |
  +----------------------------------------------------------------+
        |
        v   各反復（本体 /loop が駆動）
  +------------+   +----------------------+   +----------------------------+
  | 実装       | > | L0（決定的）:        | > | checker（敵対的）:         |
  | （単一     |   | test / lint / typeck |   | loop-verify があれば別系統 |
  |   builder）|   +----------+-----------+   | 無ければ validator (haiku) |
  +------------+        fail > 次反復へ、     +-------------+--------------+
                        失敗出力は生のまま                  |
                        返す（checker 省略）                v
                                    両方 PASS > 停止して出力
                                    欠陥      > その点だけ修正、反復
                                    空回り    > 早期停止（同一欠陥2連続 / diff 空）
                                    3 反復    > 最良案 + 未解決欠陥（合格を偽装しない）
```

反復は Claude Code の **本体 `/loop`**（self-paced）と、任意で **`/goal`**（完成まで自走）が駆動する。
loop-kit はこれらを **同梱しない**。同梱するのはそれらが回す *recipe* のほう。

## 独立（cross-vendor）checker: より強い形

同梱の `validator` は **同族**（Claude が Claude を検品）なので、Claude が *系統的に* 誤る欠陥は
書き手も検品役も揃って見逃す。より強い形は採点を **別のモデル系統** に渡す:
[loop-verify](https://github.com/akihidem/loop-verify)（codex / GPT / Gemini）。verdict は
`validator` と **同一契約** なので、`loop-protocol` は使えるときは **drop-in** で loop-verify を
checker にし、無ければ haiku validator に fallback する。

- **既定（zero-config）:** haiku `validator`。外部アカウント不要、install した瞬間から動く。
- **opt-in 上位:** loop-verify を回す（codex / OpenAI / Gemini バックエンド要）。同族検品に対し
  独立欠陥を 4 vs 1 で多く検出（loop-verify の edge bench）。
- **完了ゲートは最強の独立 checker で**: 反復中は速い段でよいが、「完了」を宣言する前の最終検品
  は使える中で最も独立な checker（loop-verify / 別系統レビュー）に最終成果物の全体を見せる
  （コードは final の git diff、テキスト/文書は run dir の artifact.md）。

これで loop-kit は既定では依存ゼロのまま、loop-verify を配線すれば cross-vendor の独立性に届く
（MCP サーバとして起動し `independent_verify` をセッションに出す）。別系統は共有盲点を減らすが、
検品が無謬になるわけではない。

## インストール

```bash
# akihidem marketplace は claude-env-coach repo にホストされている
claude plugin marketplace add akihidem/claude-env-coach
claude plugin install loop-kit@akihidem
```

## 使い方

- **`/loopify <作りたいもの>`** → loop-ready prompt（凍結基準 + L0 + checker の recipe）を得る。
  先頭に入口ゲートがあり、軽微な作業や L0 を作れない対象には recipe でなく「ループしない」が返る。
  prompt は `/loop` に渡すか、Claude にインラインで回させる。
- **`loop-protocol`** skill は YES/NO で切れる成果物タスクで自動 surface し、
  凍結 → 実装 → L0 → checker → 判定 のサイクルを回す。
- **checker** が凍結基準への敵対検品をする。基準は per-run dir
  （`~/.claude/tmp/loop/<日時>-<slug>/criteria.md`、sha256 固定）から読む: 使えるなら
  **loop-verify**（cross-vendor）、無ければ同梱の **`validator`** agent（haiku）。
- **`templates/goal-loop-template.md`**: 「完成まで作り切る」ジョブで `/goal` を
  決定的証拠に縛る（soft な自己採点ではなく）。

### 任意: 全成果物タスクで自動提案させる
[INSTALL.md](INSTALL.md) のスニペットを `~/.claude/CLAUDE.md` に貼ると、YES/NO で切れる成果物を
投げるたびに Claude が能動的にループを提案する。`loop-protocol` skill はスニペット無しでも発火する。
スニペットは提案を自動化するだけ。

## コンポーネント

| コンポーネント | 種別 | 役割 |
|---|---|---|
| `loopify` | command | 自由文を loop-ready prompt に変換 |
| `loop-protocol` | skill | 凍結 → 実装 → L0 → checker → 判定 の手順 |
| `validator` | agent（haiku） | 凍結基準への敵対検品（同族・既定） |
| `goal-loop-template` | template | `/goal` を決定的証拠に縛る |
| `loop-verify` | 外部・任意 | cross-vendor 独立 checker。`validator` の drop-in（[別 repo](https://github.com/akihidem/loop-verify)） |

## 正直な限界

- **既定では独立した検証ではない。** 同梱 validator は同じ Claude 族なので、Claude が *系統的に*
  誤る欠陥は両者が共に見逃す。cross-vendor の独立性には **loop-verify** を checker に配線する
  （[上記](#独立cross-vendorchecker-より強い形)参照）。どちらでも盲点は減らすが消しはしない。
- **L0 が本当の砦。** コードでは決定的な test/lint/typecheck が砦で、checker は二次パス。
  L0 が無いとループは大幅に弱くなる。足すこと（ロジックを純関数に切り出してテストする）。
- **事実検証では自己検品は砦にならない。** ソース接地 + 人間承認が砦。
- **全タスクに被せない。** 1 行修正・調査・質問にループは要らない（トークン浪費）。反復は〜3 で頭打ち。
- **ループが保証するのは床で、天井ではない。** L0 + checker が証明できるのは「凍結契約が成立している」
  まで。審美・判断・「そもそもこれを作るべきか」は、凍結時と最終レビューで人間（または最強のモデル）が持つ。
- **不可逆な操作はループの外。** merge / 本番 deploy / 公開 / 対外送信はループ本体に入れない（人間ゲート）。

## 関連する先行実装

同じ形に独立収束している設計（一読の価値あり。loop-kit は作者自身の実測から作り、後からこれらと突き合わせた）:

- [ralph-wiggum](https://github.com/anthropics/claude-code/blob/main/plugins/ralph-wiggum/README.md)
  （Anthropic 公式 plugin）: 生の継続圧。prompt を完了まで再投入、1 反復 1 タスク。loop-kit は
  そこに凍結契約と外部 checker ladder を足す。
- [proof-loop](https://github.com/LeoStehlik/proof-loop): 凍結 acceptance criteria、fresh session の
  builder ≠ verifier、タスク毎の証拠 dir。最も近い設計。loop-kit は L0 先行のコスト順と
  cross-vendor の段を足す。
- [aider](https://github.com/Aider-AI/aider) の auto-test ループ: テスト出力を生のまま返し、
  再試行上限後は最良案を採る。rich feedback と正直な停止という同じ勘所。

## ライセンス

MIT © akihidem
