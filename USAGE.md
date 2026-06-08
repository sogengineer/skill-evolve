# skill-evolve 使い方ガイドライン

実際に `skill-evolve` を動かして、対象スキルを改善するまでの **実務手順書**。
概念や理論は [README.md](./README.md) を参照。本書は「具体的に何をどの順で実行するか」に絞っている。

---

## 目次

- [前提条件](#前提条件)
- [人間の作業 vs 自動の境界線](#人間の作業-vs-自動の境界線)
- [Phase A: セットアップ(初回のみ)](#phase-a-セットアップ初回のみ)
- [Phase B: pilot(Step -1) — 最初の 1 回は必ず通す](#phase-b-pilotstep--1--最初の-1-回は必ず通す)
- [Phase C: 本ループの実行](#phase-c-本ループの実行)
- [Phase D: 結果の解釈と次アクション](#phase-d-結果の解釈と次アクション)
- [手作業を減らす裏技的 Tips](#手作業を減らす裏技的-tips)
- [運用ノウハウ](#運用ノウハウ)
- [トラブルシューティング](#トラブルシューティング)
- [チェックリスト(印刷用)](#チェックリスト印刷用)
- [FAQ](#faq)

---

## 前提条件

本ガイドを実施する前に、以下が揃っていること。

| 項目 | 確認方法 |
|---|---|
| Claude Code がインストール済み | `claude --version` |
| 対象スキルが `.claude/skills/<name>/SKILL.md` に存在 | `ls .claude/skills/` |
| 対象スキルを一度通常実行して、入出力の形を理解している | 手動で `/<対象skill名>` を 1〜2 回実行 |
| 対象スキルの description ↔ body の乖離は事前に手で潰してある | README の Step 0a を参照 |
| `[critical]` 要件が言語化できる | 「これが落ちたら全体失敗」の項目を 1〜3 個列挙できるか |

> [!warning] 「rubric が立てられない」と感じたら
> このスキルは適用対象外。本ガイドを進めずに対象スキル自体を見直すか、別の改善手段を検討する。

---

## 人間の作業 vs 自動の境界線

skill-evolve は「全自動」ではない。**判断責任を持つ作業は必ず人間が担う**設計になっている。線引きを明確に把握しておくと、不必要な手作業も、危険な自動化も避けられる。

### マトリクス

| Phase / Step | 人間がやるべきこと | 自動でやるべきこと | 境界の理由 |
|---|---|---|---|
| **Phase A: セットアップ** | 初回ディレクトリ作成、.gitignore 整備 | — | 1 回だけなのでスクリプト化の ROI が低い |
| **Phase B-1: test_cases 作成** | **prompt 文 / rubric の起草** | 既存ノートからのケース抽出補助 | rubric 設計は本ループの「設計図」。LLM 起草は容易に流れる |
| **Phase B-2: 手動採点** | **「直感と一致するか」の判定** | μ_f の機械的スコア計算 | 直感の言語化 = 暗黙知の形式知化。LLM では代替不可 |
| **Phase B-3: rubric 修正** | **不一致原因の特定** | 修正案の下書き提示 | 「なぜ不一致か」は対象ドメインの知識が要る |
| **Phase B-4: コーチ動作確認** | **Red flags 検出** | 編集案の生成 | 「これは合理化された逃げ」かは経験ベース |
| **Phase B-5: コスト試算** | 予算判断 | 計算 | 投資判断は人間 |
| **Phase C-1: 入力契約 YAML** | パラメータの初期値判断 | 後続イテレーションでの動的調整 | 初期値はドメイン依存 |
| **Phase C-2: 本ループ実行** | — | **全て自動**(選手 / リフレクタ / コーチ / ゲート / validation) | ここが skill-evolve の本体 |
| **Phase C-3: 実行中観察** | 介入シグナルの読み取り | メトリクスの可視化 | 「いつ止めるか」の判断 |
| **Phase C-4: 介入判断** | **本ループ停止判断** | ロールバック実行 | スキル設計疑念は人間判断 |
| **Phase D-1: changelog 読解** | **採用編集の妥当性レビュー** | 差分表示 | 局所修正の質は人間が読む |
| **Phase D-2: 編集の revert** | 「これは肥大化バイアス」の検出 | revert 実行 | パターン認識 |
| **Phase D-3: permanent 昇格** | **昇格 / 不昇格の最終判断** | 候補抽出 + クラスタリング | 恒久ルール化は重い判断 |

### 一文サマリー

> 「**設計と判断は人間**、**実行と計測は自動**」

具体的には:

- **rubric を書く / 直感判定する / 編集を読んでレビューする / 棄却を恒久化する** ← これらは人間
- **選手として走らせる / リフレクタとして診断する / コーチとして編集案を生成する / ミニバッチゲートで篩う / validation する** ← これらは自動

### 自動化してはいけないライン

以下の自動化はやめておく(過去のしくじり集として):

| 自動化案 | なぜダメか |
|---|---|
| 「rubric の不一致を LLM に検出させて自動修正」 | rubric 修正は「採点の物差し」変更。LLM に任せると基準が緩む方向に流れる |
| 「Red flags 該当を LLM に判定させて自動却下」 | Red flags は **そう見えて実は正しい** ケースがあり、人間文脈読みが要る |
| 「permanent buffer への自動昇格」 | 判断ミスが恒久ルール化される。3 回却下のシグナルは正しくない場合もある |
| 「validation スコアだけで Phase D を終わらせる」 | スコア改善でも「frontmatter を勝手に書き換えた」「肥大化した」を見落とす |

---

## Phase A: セットアップ(初回のみ)

### A-1. skill-evolve 本体を対象プロジェクトに導入

```bash
# 対象プロジェクトのリポジトリルートで実行
TARGET_PROJECT=/path/to/your/project
mkdir -p "$TARGET_PROJECT/.claude/skills/skill-evolve"
cp /Users/tenex01/project_persol/skill-evolve/SKILL.md \
   "$TARGET_PROJECT/.claude/skills/skill-evolve/SKILL.md"
```

### A-2. 対象スキルディレクトリに作業用サブディレクトリを作る

```bash
TARGET_SKILL=$TARGET_PROJECT/.claude/skills/<対象skill名>
mkdir -p "$TARGET_SKILL/.skill-opt"
cp "$TARGET_SKILL/SKILL.md" "$TARGET_SKILL/.skill-opt/baseline.md"
touch "$TARGET_SKILL/.skill-opt/changelog.md"
touch "$TARGET_SKILL/.skill-opt/rejection-buffer-permanent.yaml"
```

> `.skill-opt/` は `.git/` と同じ慣例でドット先頭にして visible ツリーから外しているため、`.gitignore` に **追加しない**(永続資産なので git 管理対象に含めるべき)。

### A-3. .gitignore の方針

```gitignore
# 含める(長期資産)
# .claude/skills/<対象skill名>/.skill-opt/baseline.md
# .claude/skills/<対象skill名>/.skill-opt/changelog.md
# .claude/skills/<対象skill名>/.skill-opt/rejection-buffer-permanent.yaml
# .claude/skills/<対象skill名>/.skill-opt/coach_meta_prompt_v*.md
# .claude/skills/<対象skill名>/.skill-opt/feedback_spec.md

# 除外(per-run データ)
.claude/skills/*/.skill-opt/rejection-buffer-run.yaml
.claude/skills/*/.skill-opt/.tmp-*/
```

---

## Phase B: pilot(Step -1) — 最初の 1 回は必ず通す

> **このフェーズを飛ばすと後段は全部無駄になる。** μ_f(採点関数)が壊れた状態で勾配を計算しても、誤った方向にスキルが歪んでいくだけ。

### B-0. pilot_mode を選ぶ

「触ってみるだけ」と「本格運用」では立ち上げ投資を変える。

| pilot_mode | test_cases | max_iter | n_minibatches | 立ち上げ目安 | 使いどころ |
|---|---|---|---|---|---|
| `minimal` | 6 (train 4 / val 2) | 2 | 2 | 1〜2 時間 | 効きそうか先に確認したい / お試し |
| `full` | 12 (train 8 / val 4) | 3 | 4 | 3〜5 時間 | 本格運用、継続改善 |

**推奨フロー**:

```
まず minimal で 1〜2 時間 → 手応えがあれば full に格上げ
                          → 手応えゼロなら手動チューニングに戻る or 別アプローチ
```

minimal でも **permanent buffer は通常通り更新する** ので、後で full に移行したときに学びが
引き継げる。「お試し、ただしお試しも長期成果に寄与させる」設計。

### B-1. test_cases を書き出す

`examples/pilot-template.md` の構造を雛形に、対象スキルディレクトリ配下に作成:

`examples/pilot-template.md` の構造を雛形に、対象スキルディレクトリ配下に作成:

```bash
mkdir -p "$TARGET_SKILL/examples"
$EDITOR "$TARGET_SKILL/examples/test_cases.md"
```

書くときの実務目安:

| 項目 | 目安 |
|---|---|
| 1 ケースの prompt サイズ | 5〜30 行(長すぎると pilot で手が動かなくなる) |
| 1 ケースの rubric 項目数 | 4〜8 個 |
| `[critical]` の数 | 1〜3 個(全部 critical はダメ。優先度が消える) |
| 成功ケース : 失敗ケース | だいたい 6:4 〜 7:3 |
| 境界ケース(rubric が分かれる微妙な入力) | 必ず 1 件は入れる |

### B-2. 手で対象スキルを実行して採点

```bash
# Claude Code で対象スキルを起動し、test_cases の各 prompt を順に投げる
# 出力を test_cases.md に「出力例」として貼り付けていく
```

### B-3. 人間直感との突合 — ここが本丸

採点結果を表にして、人間判断と並べる:

| case | μ_f 採点 | 人間判断 | 一致? | 不一致なら原因 |
|---|---|---|---|---|
| case-01 | ○ | ○ | ✅ | — |
| case-02 | × | × | ✅ | — |
| case-03 | ○ | × | ❌ | r4 が甘い → `[critical]` 化 |
| case-04 | × | ○ | ❌ | r2 が厳しすぎる → 「部分的」許容に修正 |
| ... | | | | |

**不一致が出たら必ず rubric を直す**。対象スキルが悪いのではなく rubric が悪い。

> [!tip] 一致率の目安
> - 90% 以上: 本ループへ進んで OK
> - 70〜90%: rubric を 1〜2 回修正してから再突合
> - 70% 未満: rubric が根本的に間違っている。test_cases から見直す

### B-4. コーチを手動で 1〜2 イテレーション回す

```
Claude Code セッションで:
> /skill-evolve を pilot モードで起動して、
  test_cases から train を 4 件選んで 1 イテレーションだけ回して、
  コーチの編集案を出力だけしてください(採用はしない)。
```

人間が編集案を読んで:

- [ ] 提案された編集が rubric の失敗ケースに対応している
- [ ] 過剰な抽象化や全書き換えではない
- [ ] Red flags(後述)に該当する提案が混ざっていない
- [ ] permanent buffer の `coach_instruction` 候補になりそうな「明らかに効かない」提案が混ざっていない

全て ✅ なら本ループ Go。1 つでも ❌ なら `coach_meta_prompt.md` を直して再試行。

### B-5. コスト試算

```
1 イテレーション当たりの LLM 呼び出し見積もり:
  - 選手 subagent: n_minibatches 個 × (train ÷ n_minibatches) ケース
  - リフレクタ subagent: n_minibatches 個
  - コーチ subagent: 1 個
  - minibatch ゲート: 編集案数 × train
  - validation: 編集案数 × validation
合計: 概ね 30〜80 回 / イテレーション(設定による)

想定総コスト: 上記 × max_iterations × 平均トークン × 単価
```

予算オーバーなら `n_minibatches` や `max_iterations` を下げる。

### B-6. pilot 成果物を確定

```
$TARGET_SKILL/.skill-opt/
├── feedback_spec.md            ← μ_f の最終仕様
├── coach_meta_prompt.md     ← コーチへの初期指示
└── pilot_report.md             ← 一致度 + コスト試算 + Go/No-Go 判定
```

---

## Phase C: 本ループの実行

### C-1. 入力契約 YAML を準備

```yaml
# $TARGET_SKILL/.skill-opt/run_config.yaml
target_skill_path:       /abs/path/to/.claude/skills/<対象>/SKILL.md
test_cases_path:         /abs/path/to/.claude/skills/<対象>/examples/test_cases.md
feedback_spec_path:      /abs/path/to/.claude/skills/<対象>/.skill-opt/feedback_spec.md
coach_meta_prompt_path:  /abs/path/to/.claude/skills/<対象>/.skill-opt/coach_meta_prompt.md
pilot_mode: full         # minimal で立ち上げ、後で full に格上げが推奨
max_iterations: 3        # minimal なら 2
edit_budget: 4
acceptance_margin: 5
n_minibatches: 4         # minimal なら 2
```

### C-2. Claude Code から起動

```
> /skill-evolve を以下の設定で本ループ実行してください:
  <run_config.yaml の内容をペースト>
```

または対話モードを使わずヘッドレスで:

```bash
claude -p "/skill-evolve を実行: target=$TARGET_SKILL/SKILL.md, \
  test_cases=$TARGET_SKILL/examples/test_cases.md, \
  feedback_spec=$TARGET_SKILL/.skill-opt/feedback_spec.md, \
  max_iter=3, edit_budget=4, margin=5, n_minibatches=4"
```

### C-3. 実行中に観察すべきもの

リアルタイムで以下を確認:

| シグナル | 良いサイン | 危ないサイン |
|---|---|---|
| 選手の不明瞭点・裁量補完 | 数が減っていく | 増えていく(スキルが余計に曖昧になっている) |
| tool_uses 偏り | ケース間で均等 | 1 ケースだけ突出(構造的欠陥仮説) |
| コーチの編集案の重複度 | 複数ミニバッチで一致 | 毎回バラバラ(診断が定まっていない) |
| minibatch ゲート通過率 | 50〜70% 程度 | 100%(ゲートが機能していない) or 0%(コーチが空振り) |
| validation スコア推移 | 単調増加 or 安定維持 | 振動 / 単調減少 |

### C-4. 介入が必要なシグナル

実行を止めて手動で介入すべきケース:

1. **validation スコアが baseline を下回った** → Step 8 のロールバックが発動しているはず。発動していなければバグ。手動で `baseline.md` から戻す
2. **同じ編集案が 3 イテレーション連続で却下されている** → permanent buffer に手動昇格
3. **選手レポートに「指示が矛盾している」が頻出** → 本ループを止めて Step 0a(description ↔ body 整合)を再点検

---

## Phase D: 結果の解釈と次アクション

### D-1. changelog.md を読む

実行終了後、以下を確認:

```
$TARGET_SKILL/.skill-opt/changelog.md
```

確認ポイント:

- [ ] baseline からの累積 train / validation スコア差分が正
- [ ] 採用編集 / 試行編集の比が 20〜40% 程度(高すぎ・低すぎは要因確認)
- [ ] ミニバッチ別の提案採用率に大きな偏りがない
- [ ] 主要却下パターン 3 つが理解できる失敗モードである

### D-2. 採用された編集を読む

特に確認すべき点:

- 編集が **局所修正** に留まっているか(構造を書き換える大変更は禁止)
- frontmatter に変更がないか(コーチは frontmatter を触ってはいけない)
- 過剰な抽象化や「念のため」追加が混ざっていないか(肥大化バイアス)

問題があれば該当編集を手で revert し、`rejection-buffer-permanent.yaml` の `coach_instruction` に「以降この種の編集禁止」を明記。

### D-3. 棄却バッファの昇格

実行終了時、`rejection-buffer-run.yaml` を眺めて昇格判定:

```yaml
# rejection-buffer-permanent.yaml への追記例
promoted_patterns:
  - id: perm-001
    promoted_at: 2026-06-09
    pattern_name: "全件強制読み"
    representative_edit: |
      op: add
      target: ワークフロー冒頭
      after: 必ず Read で全件確認
    evidence_count: 3
    structural_reason: |
      短中規模スキルでは tool_economy を悪化させ改善を打ち消す
    coach_instruction: |
      コーチへの恒久指示: 全件強制読み系の提案は禁止。
      Grep で絞り込む方針を優先。
```

> [!important] 昇格は手動で OK
> 自動昇格ロジックを組み込むと判断ミスが恒久ルール化する。**人間が読んで明示的に判断する** のが安全。

### D-4. 次の運用サイクル

- **頭打ちまで回した** → 1〜3 ヶ月放置。test_cases に新ケースを追加してから再実行
- **採用編集ゼロで終了** → 構造設計に問題あり。手動チューニングに戻る
- **大幅改善** → 同じ対象スキルを別モデルで実行して転移性確認

---

## 手作業を減らす裏技的 Tips

「人間がやるべき」と言っても、毎回ゼロから書くのは消耗する。**「人間が判断しやすい状態」まで自動で持っていく**のが王道。以下、実戦で効くショートカット集。

### 裏技 1: test_cases を「過去ログ」から半自動生成

ゼロから test_cases を書こうとすると筆が止まる。代わりに既に手元にある資産を流用:

```bash
# 例: Slack の過去スレッドや git log のコミットメッセージから素材を抽出
# 対象スキルが「コードレビュー系」なら、過去の PR レビューコメントが宝の山

claude -p "以下の PR コメント群から、test_cases.md の case 雛形を 8 件生成して。
            各 case には prompt(対象ファイルパスを含む)と rubric を書く。
            rubric は実際にレビュアーが指摘した観点をベースに、[critical] を 1〜3 個含めること。
            <PR コメントを貼る>"
```

ポイント: **生成されたケースを「そのまま使わず、人間が 5 分眺めて 1〜2 個修正する」** のが最速。LLM が生成したものをゼロから書き直すより、80% できているものを直す方が遥かに速い。

### 裏技 2: 「直感採点」を 2 軸スプレッドシート化

採点 vs 直感の突合は表で書くと面倒。代わりに **「OK っぽさ」を 1〜5 で先に書いて、後で μ_f と並べる**:

```markdown
| case | 第一印象 (1=明らかに失敗, 5=満点) | μ_f 採点 | 一致? |
|---|---|---|---|
| case-01 | 5 | ○ (0.95) | ✅ |
| case-02 | 2 | ○ (0.72) | ❌ ← μ_f が甘い |
```

第一印象を先に書いておくと、μ_f を見たあとの「後出しジャンケン」を防げる(これは自分の直感を歪める最大の罠)。

### 裏技 3: 採点の "ブレ" を早期検出する自動チェック

同じ出力を 3 回採点させて、スコアがブレる項目を特定:

```bash
# pilot 段階で実施
for i in 1 2 3; do
  claude -p "feedback_spec に従って次の出力を採点して、JSON で結果を返して: <出力例>" \
    > pilot_scoring_$i.json
done

# diff を取って、ブレている rubric 項目を特定
jq -s '[.[].rubric_scores] | transpose | map({item: .[0].id, scores: map(.score)})' \
  pilot_scoring_*.json
```

ブレている rubric 項目は judgemental(主観的)なので、**判定基準を二者択一化** する。例えば「OK/部分的/×」の 3 段階を「○/×」に圧縮。

### 裏技 4: コーチへのメタプロンプトを「却下バッファから自動再生成」

`coach_meta_prompt.md` を手で書き直すのは面倒。代わりに **却下バッファを LLM に読ませて差分追記** させる:

```bash
claude -p "以下の coach_meta_prompt_v_previous.md と
            rejection-buffer-permanent.yaml を読んで、
            「コーチが繰り返しがちな失敗パターン」を 3 つ抽出して、
            メタプロンプトの『過去の学び』セクションに追記する形で
            coach_meta_prompt_v_next.md を出力して。
            ただし既存セクションは絶対に書き換えないこと。" \
  < /dev/null > .skill-opt/coach_meta_prompt_v_next.md

# 出力を眺めて 30 秒で OK/NG を判断
```

**重要**: 出力を読まずに上書きしないこと。LLM は容易に既存ルールを「整理」と称して書き換える。

### 裏技 5: 「Red flags 検出」を pre-commit hook で半自動化

採用編集の妥当性チェックを毎回手作業でやるのは続かない。`.skill-opt/changelog.md` への追記時に **静的チェックを走らせる**:

```bash
# scripts/check_skill_diff.sh
#!/bin/bash
DIFF=$(git diff HEAD~1 -- .claude/skills/*/SKILL.md)

# Red flag パターン
echo "$DIFF" | grep -E "^\+.*(frontmatter|description:|allowed-tools:)" && {
  echo "🚨 frontmatter 改変を検出"; exit 1
}

# 行数増加が +50 行を超える編集は人間レビュー必須
ADDED=$(echo "$DIFF" | grep -c "^+")
DELETED=$(echo "$DIFF" | grep -c "^-")
NET=$((ADDED - DELETED))
[ $NET -gt 50 ] && { echo "🚨 肥大化警報: +$NET 行"; exit 1; }

echo "✅ 静的 Red flags パスしました"
```

これで毎回の changelog レビューが「警告が出たときだけ人間が見る」に変わる。

### 裏技 6: 「並列実行 + 通知 + 別作業」で待ち時間を消す

本ループは 1〜2 時間かかる。これを観察し続けるのは時間の無駄:

```bash
# バックグラウンド起動 + 終了通知
claude -p "/skill-evolve <設定>" > /tmp/skill-evolve-$$.log 2>&1 &
PID=$!

# 完了したら通知(macOS)
(wait $PID && osascript -e 'display notification "skill-evolve 完了" with title "Claude"') &

# 別作業に戻る
```

Claude Code から起動するなら `run_in_background: true` で同じことができる。**観察するな、通知を待て**。

### 裏技 7: pilot を「過去の本ループログ」から複製

2 個目以降の対象スキルで pilot をゼロから書くのは消耗。**既に成功している pilot を雛形にコピーして diff 修正**:

```bash
# 既存 pilot を雛形に
NEW_SKILL=review-fooo
cp -r .claude/skills/review-bar/.skill-opt/feedback_spec.md \
      .claude/skills/$NEW_SKILL/.skill-opt/feedback_spec.md

# 雛形 vs 新スキルの差分を AI に拾わせる
claude -p "次のスキル間の差分を踏まえて、feedback_spec.md を新スキル用に書き換えて。
            ただし [critical] の数と weight 構造は維持すること:
            旧: <.claude/skills/review-bar/SKILL.md>
            新: <.claude/skills/$NEW_SKILL/SKILL.md>"
```

### 裏技 8: permanent buffer の「派生スキルへの自動展開」

ドメインが似ている対象スキル間で permanent buffer は共有できる。手作業コピーは漏れるので **共通バッファを `symlink` で配る**:

```bash
# プロジェクト共通バッファ
mkdir -p .claude/shared/skill-opt-permanent

# 各対象スキルへ symlink で配布
for SKILL in review-error review-tx review-batch; do
  ln -sf ../../../shared/skill-opt-permanent/$SKILL-domain.yaml \
        .claude/skills/$SKILL/.skill-opt/rejection-buffer-permanent.yaml
done
```

注意: symlink を採用すると 1 つのスキルが書き込んだ permanent ルールが全スキルに波及する。**ドメインが本当に同じか** を確認してから使うこと。

### 裏技 9: ヘッドレス実行でナイトリー回す

頭打ち到達後の「3 ヶ月放置 → 再実行」を忘れがちなら cron で:

```bash
# 月次で全対象スキルに skill-evolve を回す
# crontab -e
0 3 1 * * cd /path/to/project && \
  for SKILL in $(ls .claude/skills/ | grep -v skill-evolve); do \
    claude -p "/skill-evolve target=.claude/skills/$SKILL/SKILL.md ..." \
      >> .skill-opt/monthly-$(date +%Y-%m).log 2>&1; \
  done
```

ただし **採用結果のレビュー(Phase D)は人間が朝イチで実施**。レビューなしの自動マージは厳禁。

### 裏技 10: 「失敗例 vs 成功例」を 1 ファイルにまとめて素材化

複数回実行すると、changelog に「採用 / 却下 / ロールバック」が混在する。これを **クラスタリングして再利用素材化**:

```bash
claude -p "次の changelog.md と rejection-buffer-permanent.yaml を読んで、
            このスキルに固有の『効く編集の特徴』と『効かない編集の特徴』を
            それぞれ 3 パターンずつ抽出して。
            出力先: .skill-opt/patterns_summary.md" \
  < .skill-opt/changelog.md
```

これは次の対象スキルの初期メタプロンプト材料になる。**1 つのスキルでの学びを次に転用する** ための再資産化。

### 裏技 11: 「セッション復帰」を transcript ファイルで担保

1〜2 時間の実行は接続切れリスクがある。Claude Code の `transcript` を有効化:

```bash
# settings.json
{
  "saveTranscript": true,
  "transcriptDir": "~/.claude/transcripts/skill-evolve"
}
```

接続切れ後は最新 transcript から `/resume <transcript-id>` で再開。**「最初からやり直し」を防ぐ最低限の保険**。

### Tips を全部使うと

理論値で **手作業 7〜8 割減**。特に効くのは:

- 裏技 1(test_cases 半自動生成) — 最初の壁を 1/3 に圧縮
- 裏技 5(Red flags 静的チェック) — 毎回のレビューが 10 分 → 1 分
- 裏技 6(並列実行 + 通知) — 観察コスト ゼロ

逆に **絶対に手作業を残すべき** は:

- 裏技 2(第一印象の事前記録) ← ここを LLM 化すると突合の意味が消える
- Phase D-3(permanent 昇格判定) ← ここの自動化が一番事故る

---

## 運用ノウハウ

### ノウハウ 1: test_cases は「過去の本物の失敗」を貯める

理想は「ユーザーから上がった実際の不具合報告」を 1 件 1 ケース化すること。合成データはどうしても rubric が甘くなりがちで、本番との乖離が出る。

### ノウハウ 2: edit_budget は安易に上げない

「もっと一気に直したい」と思っても、edit_budget=4 がアブレーションでの最適値。5 以上にするなら明確な根拠を pilot_report に記録。

### ノウハウ 3: permanent buffer は「効かないパターン辞典」として育てる

このバッファ自体が長期資産。3 ヶ月運用すると、対象ドメインに固有の「LLM が考えがちだが実際には効かない提案」が分類されてくる。これは別チーム / 別プロジェクトに移植する際の最初の入力になる。

### ノウハウ 4: モデル切替時は必ず再 pilot

sonnet → opus 等でモデルを切り替えると、同じスキルでも μ_f の挙動が微妙にズレる。Phase B を簡略版でいいので必ず通す。

### ノウハウ 5: 複数スキルへ展開するときは 1 つずつ

「20 個の対象スキル全てに一気に skill-evolve をかけたい」となりがちだが、まず 1 つで Phase B〜D を一通り通して感覚を掴む。permanent buffer のクラスタリング感覚は実際にやらないと身につかない。

---

## トラブルシューティング

### Q. minibatch ゲートで全提案が落ちる

**症状**: Step 4 で σ' ≤ σ が連発、Step 5 まで到達しない。

**典型原因と対処**:

| 原因 | 対処 |
|---|---|
| μ_f が変動的(同じ出力でも採点がブレる) | feedback_spec を見直し、判定基準を二者択一化 |
| ミニバッチサイズが小さすぎてノイズ支配 | `n_minibatches` を減らす(4→2)or train を増やす |
| コーチが train 過学習編集を出している | `coach_meta_prompt_v<N>.md` に「train 特定ケースを名指しした編集禁止」を追記 |

### Q. validation で baseline を下回り続ける

**症状**: Step 8 のロールバックが毎イテレーション発動。

**典型原因と対処**:

| 原因 | 対処 |
|---|---|
| validation ケースが train と質的に違う | データ分割を見直し、性質が近いペアにする |
| acceptance_margin が緩すぎてノイズ採用 | margin を 5→8 程度に上げる |
| スキル設計に構造的欠陥 | 本ループを止めて手動チューニングに戻る |

### Q. コーチが同じ提案を繰り返す

**症状**: 異なるイテレーションで同一 target / op の提案が再出現。

**典型原因と対処**:

- per-run buffer がリセットされていない → Step 0b の「先頭で空に初期化」が動いているか確認
- permanent buffer に昇格させていない → Phase D-3 の手動昇格を実施

### Q. 選手レポートの不明瞭点が増えていく

**症状**: イテレーションを重ねるごとに「指示が矛盾している」「どちらに従えばいいか不明」が増える。

**典型原因と対処**:

- 採用編集が **追加方向に偏っている**(肥大化バイアス) → coach_meta_prompt に「`add` と `delete` のバランスを取る」「冗長セクションは積極的に `delete`」を追記
- frontmatter description と body の乖離が広がっている → 本ループを止めて Step 0a を再実施

### Q. 全体的にコストが想定の 2〜3 倍になる

**典型原因と対処**:

- subagent のレポート構造が膨らんで context が肥大 → 「ツール戻り値要約」「推論メモ」を箇条書きに圧縮するよう subagent 起動契約を更新
- minibatch ゲートで通過案が多すぎる(篩い落としが効いていない) → σ' > σ の比較を「σ' >= σ + 1」程度に厳格化

---

## チェックリスト(印刷用)

### Phase A: セットアップ

- [ ] SKILL.md を `.claude/skills/skill-evolve/` にコピーした
- [ ] 対象スキルディレクトリに `.skill-opt/` を作成した
- [ ] baseline.md を保存した
- [ ] permanent buffer ファイルを git 管理対象にした
- [ ] per-run buffer を .gitignore に追加した

### Phase B: pilot

- [ ] test_cases を最低 8 件用意した(`[critical]` 1〜3 個含む)
- [ ] 境界ケースを 1 件以上含めた
- [ ] 人間直感との一致率が 90% 以上
- [ ] コーチ手動起動で編集案が妥当
- [ ] コスト試算が予算内
- [ ] feedback_spec.md / coach_meta_prompt.md / pilot_report.md を確定

### Phase C: 本ループ

- [ ] run_config.yaml を作成
- [ ] 実行中シグナル(不明瞭点 / tool_uses 偏り / ゲート通過率 / validation 推移)を観察
- [ ] 介入シグナルが出たら即停止

### Phase D: 結果解釈

- [ ] changelog.md を読んだ
- [ ] 採用編集が局所修正に留まっているか確認
- [ ] frontmatter に変更がないか確認
- [ ] 棄却バッファの昇格判定(人間判断)を実施
- [ ] 次サイクルの予定を立てた

---

## FAQ

### Q1. 対象スキルが対話型(プロンプトを段階的に投げる)の場合は?

選手 subagent の起動契約に「複数ターン会話」を含める形に拡張する。ただし pilot で再現性を確保するのが難しくなるため、可能なら「1 回の dispatch で完結する」形に対象スキル側を再設計する方が安全。

### Q2. test_cases に機密情報が含まれる場合は?

- `.gitignore` に `test_cases.md` を追加(対象プロジェクト内に閉じる)
- もしくは fixtures をマスキング済みデータで作る
- `examples/` ディレクトリごと別の private リポジトリに切り出す運用も可

### Q3. 1 回の実行にどれくらい時間がかかる?

ざっくり、max_iterations=3 / n_minibatches=4 / train 8 ケース / validation 4 ケースで:

- 各 subagent 平均 30 秒〜2 分
- 1 イテレーション = 約 15〜40 分
- 全体 = 約 45 分〜2 時間

`run_in_background` で投げて別作業しながら待つのが現実的。

### Q4. 複数の対象スキルに対して並列実行できる?

技術的には可能だが、**推奨しない**。Phase D の人間判断(編集レビュー、昇格判定)が並列処理しきれない。1 つずつ Phase A→D を回す方が長期的には早い。

### Q5. permanent buffer が大きくなりすぎたら?

50 件を超えたあたりからコーチの context 消費が無視できなくなる。クラスタリングで代表案 1 件に統合する、もしくは「最後の評価から N 日以上経過していて再現していないパターンをアーカイブする」運用を入れる。

### Q6. 改善の頭打ちはどう判定する?

- 連続 2 イテレーションで採用ゼロ → Step 7 で自動停止
- これとは別に、月次で「baseline からの累積改善幅」をプロットして、傾きが寝てきたらインターバルを開ける(週次 → 月次 → 四半期)

### Q7. 別モデル(GPT, Gemini 等)で動かせる?

skill-evolve 本体は「subagent dispatch ができる + ファイル I/O ができる」エージェント環境であれば移植可能。ただし subagent 起動契約のプロンプト構造はモデル個性に依存するため、Phase B の pilot を必ず再実施する。

---

## 関連ドキュメント

- [README.md](./README.md) — 概念 / 処理フロー / 詳細説明
- [SKILL.md](./SKILL.md) — スキル本体(`.claude/skills/skill-evolve/` に配置するファイル)
- [examples/pilot-template.md](./examples/pilot-template.md) — pilot 起動キットの汎用テンプレート
