---
name: skill-evolve
description: "既存スキル(SKILL.md / プロンプト / マニュアル)を SkillOpt 方式で進化させる汎用メタスキル。pilot で feedback function を実証 → 並列ミニバッチで試走・リフレクション → 提案統合 → minibatch ゲート → held-out validation → 改善のみ採用、というループを subagent で回す。学習率(edit_budget)と2層棄却バッファ(per-run + permanent)で暴走と長期失敗反復を防ぐ。Use when user says 'スキル最適化', 'skill optimize', 'プロンプト最適化', 'skill-evolve'."
context: fork
model: sonnet
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent
---

## Instructions

任意の SKILL.md(あるいはプロンプト/マニュアル)を対象に最適化ループを回す **汎用メタスキル**。

役割分離の原則:選手(対象スキルで試走)・リフレクタ(診断)・コーチ(編集案)を **3 subagent に分ける**。同一 subagent に役割を兼ねさせない(自己評価バイアスの構造的回避)。

---

### いつ使うか

- 頻繁に使われる対象スキルを堅牢化したいとき
- 偽陽性 / 偽陰性が増えてきて再チューニングしたいとき
- モデル切替(例: sonnet → opus 等)で挙動が変わったとき

**使わない**場面:

- 1 回限りのアドホックなプロンプト改善(評価コストが割に合わない)
- rubric が立てられないタスク(採点不能のため自動最適化は機能しない)
- orchestrator スキル(単体 SKILL 最適化とは別問題)

詳細な位置づけや適用範囲は README.md を参照。

### 入力契約

呼び出し時、ユーザー(または上位 skill)は以下を指定:

```yaml
target_skill_path:       <改善対象 SKILL.md の絶対パス>
test_cases_path:         <テストケース md の絶対パス>
feedback_spec_path:      <Step -1 で確定した μ_f 仕様書のパス>
coach_meta_prompt_path:  <examples/coach_meta_prompt_template.md を複製したパス>
pilot_mode: full | minimal   # 立ち上げを軽くしたい場合は minimal
max_iterations: 3            # minimal mode では 2 が推奨
edit_budget: 4               # 学習率。アブレーションでの最適値
acceptance_margin: 5         # validation 改善がこのポイント以上で採用
n_minibatches: 4             # minimal mode では 2 が推奨
```

### pilot_mode のプリセット

| pilot_mode | test_cases | max_iter | n_minibatches | 立ち上げ目安 | 想定用途 |
|---|---|---|---|---|---|
| `full` | 12 (train 8 / val 4) | 3 | 4 | 3〜5 時間 | 本格運用 |
| `minimal` | 6 (train 4 / val 2) | 2 | 2 | 1〜2 時間 | 触ってみる / 効きそうか確認 |

minimal モードでも **permanent buffer は通常通り更新する**(後で full モードに移行した
ときに学びが引き継げる)。minimal で見えた手応えを根拠に full への投資判断ができる。

### test_cases_path の形式

```markdown
# Test Cases for <target skill name>

## case-01 [train]
### prompt
<対象スキルを subagent に渡し、このプロンプトを実行させる。
 タスクの具体的な入力 + 期待される結果の概要>

### rubric
- [critical] <最低ライン要件>
- <通常要件>
- <通常要件>

## case-02 [validation]
...
```

最低 **8 ケース推奨**(train 6 / validation 2)、できれば 12 ケース(train 8 / validation 4)。`n_minibatches=4` なら train を 4 分割するので、train ≥ 8 が望ましい。`[critical]` を最低 1 つ含めること。

---

## ワークフロー

### Step -1: pilot run + feedback function 設計(必須、最重要)

自動最適化の品質は採点ロジック(`feedback function μ_f`)の精度で決まる。pilot を飛ばして本ループに入ると、誤った勾配でスキルが歪んでいく。

実施手順:

1. test_cases から **5〜10 ケース** をピックアップ
2. 対象 SKILL を手動で適用(`/<対象skill名>` で実行 or Task tool で dispatch)
3. rubric で採点
4. **採点結果が人間の直感と一致するか確認**
   - 直感的に正解と思ったケースが × → 採点ロジックが間違い、rubric を修正
   - 直感的に明らかな失敗が ○ → 採点が甘い、`[critical]` を厳しくする
5. コーチへの初期メタプロンプトを下書き(`coach_meta_prompt.md`)
6. 1〜2 イテレーションだけ手動でループを回し、提案された編集案を人間が読んで妥当性確認
7. **コスト試算**: 1イテレーション当たりの LLM 呼び出し数 × 単価 × 想定イテレーション

成果物(対象 skill ディレクトリの隣に保存):

- `feedback_spec.md` — μ_f の仕様(入力、出力、評価基準、`[critical]` の判定根拠)
- `coach_meta_prompt.md` — コーチへの初期指示
- `pilot_report.md` — 数値結果 + 人間判断との一致度評価

### Step 0: 静的整合チェックと準備

#### 0a. description ↔ body 整合チェック

frontmatter `description` が謳う trigger / 用途と、body がカバーする範囲に乖離がないか確認。乖離があれば本ループに入る前に人間が修正(コーチが description に合わせて body を「再解釈」する false positive を防ぐ)。

#### 0b. 準備

1. `target_skill_path` を Read し、`current_skill` に保持
2. 対象ディレクトリ直下に **`.skill-opt/` を作成**(無ければ)し、その配下に以下を作成:
   - `.skill-opt/baseline.md` — 起点バックアップ
   - `.skill-opt/changelog.md` — 各イテレーションの差分と採否
   - `.skill-opt/rejection-buffer-run.yaml` — **per-run**(この実行のみ有効)
   - `.skill-opt/rejection-buffer-permanent.yaml` — **cross-session**(永続的失敗パターン)
3. 各実行先頭で **`.skill-opt/rejection-buffer-run.yaml` を空に初期化**。permanent は維持。
4. target ディレクトリ本体には `SKILL.md` 以外の visible ファイルを置かない。

### Step 1: 並列ミニバッチ rollout

train ケースを `n_minibatches` 個に均等分割し、各ミニバッチを **独立した選手 subagent** で並列 dispatch(単一メッセージ内で複数 Agent 呼び出し)。同 subagent は再利用しない。

回収する情報(完全トレース):

| 種類 | 取り方 |
|---|---|
| 最終成果物 | subagent レポート |
| tool_uses (execution trace) | Task tool 戻り値 `usage.tool_uses` |
| tool_outputs (evaluation trace) | subagent レポート内の「ツール戻り値要約」節 |
| reasoning(自己申告) | レポート「推論メモ」節 |
| 不明瞭点・裁量補完 | レポート末尾 |

選手 subagent のプロンプトは「選手 subagent 起動契約」節を参照。

### Step 2: ミニバッチ単位リフレクション(並列)

各ミニバッチごとに **新規のリフレクタ subagent** を独立 dispatch。コーチとは役割を分離する。

リフレクタ出力(全項目必須):

- 失敗パターン仮説(case ごとに「何が起きていそうか」を 1〜2 行)
- **rubric 判定文言マッピング**: 失敗 case ごとに、落ちた rubric 項目について `feedback_spec.md`
  の判定文言を **引用付き** で明示する。コーチが軸名から推測した的外れ修正を出さないための土台
- **失敗テーマ ID 割り振り**: 複数 case に共通する失敗を 1 つのテーマとしてラベリング
  (例: `theme-01: 該当箇所が file:line で示されていない`)。コーチがテーマ単位で
  関連する複数編集をバンドル提案できるようになる
- **構造欠陥ラベル**: 各テーマに `structure_defect_label: local | structural` を付与
  - `local`: 局所修正(`add` / `delete` / `replace`)で対処可能
  - `structural`: 構造再編成が必要。コーチは手を出さず、Step 9 で permanent buffer へ即時昇格
- 共通原因の抽出(ミニバッチ内で複数 case に共通する根本原因)
- 未充足な前提(スキルが暗黙に前提していて、選手が補完しきれていない情報)
- 削除候補(冗長 / 矛盾箇所)
- `tool_uses` 偏り観察(case 間で他比 3-5 倍以上なら構造的欠陥仮説)

精度のみで判断するとスキルの構造的欠陥が隠れる。**質的シグナルを常に主、メトリクスは補助**。

#### 出力フォーマット(YAML)

```yaml
minibatch_id: <ID>
case_diagnoses:
  - case_id: <ID>
    failure_hypothesis: <1〜2 行>
    rubric_mapping:
      - rubric_item_id: <r1, r2, ...>
        judgement_text_quote: |
          <feedback_spec の判定文言を引用>
        failure_mode: <なぜ落ちたか 1 行>
themes:
  - theme_id: theme-01
    description: <共通する失敗の言語化>
    affected_cases: [<case-id>, ...]
    structure_defect_label: local | structural
    structural_reason: <structural の場合のみ: なぜ局所修正では効かないか>
common_root_cause: <ミニバッチ内共通原因>
unmet_assumptions: <スキルが暗黙に前提していた情報>
delete_candidates: <冗長 / 矛盾箇所>
tool_uses_observation: <偏りがあれば構造的欠陥仮説 1 行>
```

### Step 3: 提案統合 → 学習率まで圧縮

**新規のコーチ subagent** に次を渡す:

- `current_skill` の全文
- 全ミニバッチのリフレクション(ミニバッチごとに区別)
- 全ミニバッチの **themes 一覧と structure_defect_label**
- `rejection-buffer-run.yaml`(この実行内の再提案禁止)
- `rejection-buffer-permanent.yaml`(恒久的失敗パターン、`coach_instruction` 絶対遵守)
- `coach_meta_prompt.md`(ドメイン依存前提 + 過去の学び)
- `feedback_spec.md`(rubric の判定文言。コーチが引用するため)
- `edit_budget` 上限

コーチの統合手順:

1. リフレクタが `structural` と診断した theme は **コーチが触らない**。
   `.skill-opt/.tmp-iter-<N>/structural_themes_for_promotion.yaml` に分けて記録
   (Step 9 で permanent buffer へ即時昇格)
2. `local` ラベルの theme についてのみ、ミニバッチごとに編集案を生成
3. 重複削除(同じ target/op はマージ)
4. 同一 theme に紐づく複数編集はバンドルとして扱う(EPT 風の「1 テーマ 1 イテレーション」)
5. 優先順位付け:複数ミニバッチで共有された theme への対処 > rationale 強さ > 単純さ
6. `edit_budget` 件まで truncate(超過分は破棄)
7. 提出前に「自己診断チェック」(コーチ起動契約節に列挙)を全件通す

#### 編集案フォーマット(全フィールド必須)

```yaml
- op: add | delete | replace
  target: <既存スキル内のアンカー行 or セクション見出し>
  before: |    # op=delete, replace のみ
    <置換/削除対象の元テキスト>
  after: |     # op=add, replace のみ
    <追加/置換後のテキスト>
  theme_id: <リフレクタが付与した theme-XX>
  rationale: <この編集が Step 2 のどの不明瞭点/失敗をどう解決するか 1〜2 行>
  hits_failure: [<case-id>, ...]
  supporting_minibatches: [<minibatch-id>, ...]

  # 以下は本スキル必須の自己診断フィールド:

  rubric_mapping:                       # 軸名から推測した的外れ修正を防ぐ
    - rubric_item_id: <r1, r2, ...>
      judgement_text_quote: |
        <feedback_spec.md から該当判定文言を引用(文字列一致必須)>
      contribution: <この編集が判定文言を満たすメカニズム 1 行>

  waves_pattern: 保守 | 上振れ | ゼロ振れ    # 修正の波及パターン自己診断
  # 保守: 1 修正で複数軸狙ったが 1 軸しか動かないと予想
  # 上振れ: 構造的情報(コマンド+設定+期待出力 等)が複数軸に効くと予想
  # ゼロ振れ: 軸名から推測しただけで判定文言に届かない可能性

  delete_considered: |                  # add の場合は必須
    <対応する delete を検討した結果、なぜ delete を選ばなかったか 1 行>
    <delete 案の場合は「N/A」>

  bloat_check: |                        # 肥大化シグナルチェック
    <「念のため」「必ず X」「もし〜なら」等の表現に該当しないことの自己確認 1 行>
```

**`rubric_mapping` が空、または引用が `feedback_spec.md` の判定文言と文字列一致しない編集案は
提出禁止**。その編集は「軸名から推測しただけの届かない修正」である可能性が高い。

#### op 別の rubric_mapping ルール

- `add` / `replace`: 「追加 / 置換テキストがどの判定文言を **満たすか**」を引用
- `delete`: 「削除対象が判定文言の充足を **妨げていた** 理由」を引用付きで明示
  (例: 削除対象が誤誘導するために、`r2` の判定文言「該当箇所が file:line で示されている」
  に反する出力を選手が出していた、等)

#### waves_pattern ごとの扱い

- `保守` / `上振れ`: 通常通り提出
- **`ゼロ振れ` の自己診断は提出禁止**。判定文言に届かないと自分で診断した編集案は
  そもそも提出する意味がない(rationale の正しさと validation スコアは別物)。
  「念のため出しておく」は肥大化バイアスの典型シグナル。

### Step 4: minibatch ゲート(改善検証)

サンプル効率の核心。各編集案ごとに:

1. `current_skill` をベースに編集を仮適用したパッチ案を作る
2. **同じミニバッチ群** で σ(編集前)と σ'(編集後)を測定
3. `σ' > σ` → Step 5 へ
4. `σ' ≤ σ` → 即却下、`rejection-buffer-run.yaml` 追記

validation を消費する前にここで篩い落とす。

### Step 5: held-out validation 評価

Step 4 通過案だけ:

1. validation ケース全てに対して新規の検証 subagent で試走
2. `val_score_after_<i>` を記録
3. 同じく `current_skill`(編集前)で validation 試走、`val_score_before` を記録(毎イテレーション取り直す)

### Step 6: 採否判定 と 適用

各通過案について:

- `delta_i = val_score_after_<i> - val_score_before`
- `delta_i >= acceptance_margin` なら採用候補
- 採用候補が複数なら `delta_i` 最大の 1 件のみ採用(1 イテレーション 1 編集)
- 採用 → `target_skill_path` に書き戻し、`changelog.md` に記録
- 不採用 → `rejection-buffer-run.yaml` に追記

#### rejection-buffer-run.yaml のフォーマット

```yaml
iter: <N>
rejected:
  - op: add | delete | replace
    target: <アンカー>
    rationale: <コーチの言い分>
    supporting_minibatches: [<id>, ...]
    rejected_at: minibatch | validation
    scores:
      sigma: <X>
      sigma_prime: <Y>
      val_before: <A>
      val_after: <B>
      delta: <Z>
    failure_mode_hypothesis: <なぜ効かなかったかの推測 1〜2行>
```

### Step 7: 終了判定

次のいずれかで停止:

- `max_iterations` 到達
- 連続 2 イテレーションで採用ゼロ(頭打ち)
- 連続 2 ロールバック(構造的に自動改善が機能していない → スキル設計を疑う)

停止時:

- baseline からの累積 train / validation スコア差分
- 採用 / 試行編集の総数
- ミニバッチ別の提案採用率
- 主要却下パターン(rejection-buffer-run から 3 つ)

### Step 8: 安全装置

- 各イテレーション開始時、`baseline.md` との validation スコア差をチェック。**baseline を下回ったら直前の編集をロールバック**
- `edit_budget` を超える編集案は `rationale` の強さで truncate
- `target_skill_path` の上書きは Step 6 の採用確定時のみ。仮適用は別ファイル

### Step 9: 棄却バッファの 2 層運用と昇格

実行終了時、`rejection-buffer-run.yaml` から `rejection-buffer-permanent.yaml` への昇格判定:

| 昇格条件 | 内容 |
|---|---|
| 頻度 | 過去の実行を通じて、類似編集案が 3 回以上却下された |
| 失敗モードの明確さ | `failure_mode_hypothesis` が一般化可能(名前付きパターン) |

昇格時、類似案を **クラスタリングして代表案 1 件** で記録(肥大化防止)。

なお「構造的に効かないテーマ」は per-run buffer を経由せず、リフレクタが
`structure_defect_label: structural` と判定した時点で次節の即時昇格ルートに乗る。

#### 構造欠陥テーマの即時昇格(per-run buffer を経由しない)

リフレクタが Step 2 で `structure_defect_label: structural` と診断したテーマは、
通常の昇格条件を満たさなくても **即時に permanent buffer へ昇格** する。

理由: structural と診断されたテーマは局所修正(`add` / `delete` / `replace`)では
対処不能。コーチが何度試しても採用されないため、最初から「コーチは触らない」と
固定するのが安全。

処理:
1. コーチが Step 3 で `structural_themes_for_promotion.yaml` に分けて記録した theme を読む
2. `rejection-buffer-permanent.yaml` の `structural_defects` セクションに追記
3. `coach_meta_prompt.md` の「過去の学び」セクションに **手動で転記する旨を Phase D で人間に促す**
   (1 ファイル固定方式なので自動マージはしない)
4. 次イテレーション以降、コーチはこのテーマを無視する

これは **手動チューニングに戻るシグナル**。`structural_defects` が増えてきたら、
本ループを継続する前に手動で構造再編成を行うのが筋。

#### rejection-buffer-permanent.yaml フォーマット

```yaml
promoted_patterns:
  - id: perm-001
    promoted_at: <date>
    pattern_name: "<例: 全件強制読み>"
    representative_edit: |
      op: add
      target: <ワークフロー冒頭>
      after: <恒久的に効かないと判明したパターン>
    evidence_count: 3
    structural_reason: |
      <なぜ構造的にこのスキルでは効かないかの説明>
    coach_instruction: |
      コーチへの恒久指示: <禁止事項 + 代替方針>

structural_defects:                      # Step 9 即時昇格分岐で追記される
  - id: sd-001
    promoted_at: <date>
    theme_id: <リフレクタが付与した theme-XX>
    pattern_name: <theme の short description>
    affected_cases: [<case-id>, ...]
    structural_reason: |
      <なぜ局所修正(add/delete/replace)では対処できないかの説明>
    recommended_action: |
      手動チューニングで <スキル名> の <該当セクション> を構造再編成する。
      局所修正での自動改善は本テーマには適用しない。
    coach_instruction: |
      コーチへの恒久指示: theme_id <theme-XX> と同じ症状を扱う編集案は
      提案禁止(手動チューニング待ち)。
```

`coach_instruction` フィールドはコーチが毎回読んで **絶対遵守** する恒久ルール。
`structural_defects` が 3 件以上溜まったら本ループを継続するより、まず手動で構造再編成
する方が ROI が高い。

---

## 選手 subagent 起動契約

```
あなたは <対象スキル名> を白紙で読む実行者です。

## 対象スキル
<current_skill の全文>

## タスク
<test case の prompt をそのまま貼る>

## 要件チェックリスト
<test case の rubric をそのまま貼る>
([critical] 全て○ なら成功(○)、1つでも× / 部分的 なら失敗(×))

## レポート構造
- 成果物: <生成物 or 実行結果サマリ>
- 要件達成: 各項目について ○ / × / 部分的(理由付き)
- ツール戻り値要約: 使ったツールごとに「呼び出し→戻り値の要点」を箇条書き
- 推論メモ: 解いている最中に考えたこと(主要分岐 3 つ程度)
- 不明瞭点: 対象スキルで詰まった箇所、解釈に迷った文言
- 裁量補完: 指示で決まっておらず自分の判断で埋めた箇所
- 再試行: 同じ判断をやり直した回数とその理由
```

## リフレクタ subagent 起動契約

```
あなたはミニバッチ <minibatch-id> の試走結果を診断するリフレクタです。
あなたは編集案を出しません。診断レポートだけを返します。

## 対象スキル
<current_skill の全文>

## このミニバッチの試走結果
<選手 subagent のレポート群、case-id ごとに整理>

## μ_f による採点
<feedback_text を含む採点結果>

## feedback_spec.md(rubric の判定文言、引用根拠)
<feedback_spec の全文>

## ヒント
- tool_uses の偏り(case 間で 3-5 倍以上の差)は構造的欠陥のサイン
- 不明瞭点と裁量補完の頻度を質的シグナルとして主に扱う
- 精度だけで判断しない

## 構造欠陥 vs 表層の判定基準
- `local`(表層): 1〜2 行の追記 / 削除 / 言い換えで判定文言を満たせる見込み
- `structural`(構造欠陥): 以下のいずれかに該当する場合
  - 既存セクション同士が矛盾していて、局所修正では両立できない
  - 対象スキルの目的(description)と body の役割が乖離している
  - 失敗が複数 case に渡って異質(共通テーマでまとめられない)
  - tool_uses が極端に偏り、references 横断探索が常態化している

## レポート出力(YAML 形式必須)

minibatch_id: <ID>

case_diagnoses:
  - case_id: <ID>
    failure_hypothesis: <1〜2 行>
    rubric_mapping:
      - rubric_item_id: <r1, r2, ...>
        judgement_text_quote: |
          <feedback_spec.md から該当判定文言を引用、文字列一致必須>
        failure_mode: <なぜ落ちたか 1 行>

themes:
  - theme_id: theme-01
    description: <共通する失敗の言語化、コーチが見出しとして使う>
    affected_cases: [<case-id>, ...]
    structure_defect_label: local | structural
    structural_reason: <structural の場合のみ: 上記基準のどれに該当するか>

common_root_cause: <ミニバッチ内共通原因>
unmet_assumptions: <スキルが暗黙に前提していた情報>
delete_candidates: <冗長 / 矛盾箇所>
tool_uses_observation: <偏りがあれば構造的欠陥仮説 1 行>
```

## コーチ subagent 起動契約

```
あなたはスキル文書のオプティマイザ(コーチ)です。タスクを実行しません。
複数ミニバッチのリフレクタ診断を統合し、編集案だけを生成します。

## 現在のスキル全文
<current_skill>

## feedback_spec.md(rubric の判定文言。引用根拠)
<feedback_spec の全文>

## メタプロンプト(あなたへの恒久指示 — ドメイン依存)
<coach_meta_prompt.md の全文>

## 各ミニバッチのリフレクション診断
### minibatch-01
<リフレクタの診断レポート YAML>
### minibatch-02
...

## per-run 棄却バッファ(この実行で再提案禁止)
<rejection-buffer-run.yaml の全文>

## permanent 棄却バッファ(恒久的に禁止)
<rejection-buffer-permanent.yaml の全文>
特に `coach_instruction` フィールドは絶対遵守。
`structural_defects` セクションの theme_id と一致する症状には絶対に手を出さない。

---

## 質的判断のシグナル基準(全イテレーションで適用)

### tool_uses 偏り
- case 間で他比 **3〜5 倍以上の偏り** = 構造的欠陥のサイン
- 該当ケースに「最小完成例 inline」または「いつ references を読むかの指針」を
  追加する編集案を優先

### 肥大化シグナル(編集案を出す前のチェック)
- 編集案に次の表現が含まれる場合は自己確認:
  - 「念のため」「必ず X を確認」「もし〜なら〜することもできます」
  - 重複した条件分岐 / オプション提示
- スキル全体が **500 行を超えている** → `add` を厳審査、`delete` を優先
- 1 セクションが **50 行を超えている** → 構造再編成サイン
  (局所修正では触らず、リフレクタが structural と診断すべき領域)

### 軸名 ≠ 判定文言の原則
- rubric の軸名(「精度」「明瞭性」等)から推測した修正は判定文言に届かないことが多い
- 各編集案には `rubric_mapping` を必須で添え、`feedback_spec.md` から判定文言を
  **文字列一致で引用** する
- 引用が判定文言と一致しない編集案は提出禁止

### 修正の波及パターン(事前見積もりの規律)
- **保守**: 1 修正で複数軸狙ったが 1 軸しか動かない見込み(通常通り提出)
- **上振れ**: 構造的情報(コマンド + 設定 + 期待出力 等)が複数軸に効く見込み
  (提出 OK だが、必ず「構造的情報の組合せ」になっていること)
- **ゼロ振れ**: 軸名から推測しただけで判定文言に届かない(一番ありがち)
  → **提出禁止**。自己診断でゼロ振れと判定したら棄却して次の候補へ
- 各編集案で `waves_pattern` を自己診断する(`保守` / `上振れ` / `ゼロ振れ` の 3 値)

### add と delete のバランス
- `add` 案には対応する `delete` を必ず検討し、`delete_considered` に
  「検討した結果なぜ delete を選ばなかったか」を記入
- リフレクタ診断の `delete_candidates` は最優先で拾う
- `delete` を出さずに `add` だけが続くイテレーションは肥大化バイアスのサイン

---

## 統合手順

1. リフレクタ診断の `themes` を読み、`structure_defect_label: structural` のものを
   別ファイル `structural_themes_for_promotion.yaml` に分離(Step 9 で permanent buffer 即時昇格)
2. `local` ラベルの theme についてのみ、各ミニバッチのリフレクションから編集案候補を洗い出す
3. 重複(同じ target / op)はマージ
4. 同一 `theme_id` に紐づく複数編集はバンドルとして扱う(EPT 風の柔軟さ)
5. 優先順位付け:
   - 複数ミニバッチで共有された theme > rationale の強さ > 提案の単純さ
6. <edit_budget> 件まで truncate

---

## 自己診断チェック(編集案を提出する前に全件通す)

各編集案について:
- [ ] `rubric_mapping` がある
- [ ] `rubric_mapping.judgement_text_quote` が `feedback_spec.md` の判定文言と文字列一致
- [ ] `waves_pattern` を自己診断した
- [ ] `waves_pattern: 上振れ` の場合、構造的情報の組合せになっている
- [ ] `add` 案には `delete_considered` を埋めた
- [ ] `bloat_check` で 1 行の自己確認がある
- [ ] permanent buffer の `coach_instruction` に違反していない
- [ ] permanent buffer の `structural_defects` の theme_id に該当しない
- [ ] per-run rejection buffer の類似案がない

ひとつでも × があれば、その編集案は **提出しない** か、**修正してから出す**。

---

## 制約(絶対遵守)
- 操作は `add` / `delete` / `replace` の 3 種類のみ
- 最大 <edit_budget> 件まで
- frontmatter (description, allowed-tools 等) の改変は禁止
- pilot 段階で確定した rubric 自体への言及・修正提案は禁止
- スキル全体の構造を書き換える大変更は禁止(構造変更が必要な theme は structural として除外済み)

## 出力
編集案を YAML リストで返す(上記「編集案フォーマット」全フィールド必須)
+ `structural_themes_for_promotion.yaml`(theme_id ごとに分離した structural テーマ一覧)
```

---

## 出力アーティファクト

実行後、対象スキルディレクトリ配下の `.skill-opt/` に runtime データを集約する(target ディレクトリ本体は `SKILL.md` のみを visible に保つ):

```
<target_skill ディレクトリ>/
├── SKILL.md                                  ← 進化済み(本体、visible)
└── .skill-opt/                               ← skill-evolve の runtime データ
    ├── baseline.md                           ← 起点バックアップ
    ├── changelog.md                          ← 全イテレーションの採用編集 + メトリクス推移
    ├── rejection-buffer-run.yaml             ← この実行の却下案(空なら作らない / 次実行で初期化)
    ├── rejection-buffer-permanent.yaml       ← 恒久的失敗パターン + structural_defects(削除厳禁)
    ├── feedback_spec.md                      ← Step -1 で確定した μ_f 仕様
    ├── coach_meta_prompt.md                  ← コーチへのメタ指示(1 ファイル固定、手動更新)
    ├── pilot_report.md                       ← Step -1 の人間判断との一致度
    └── .tmp-iter-<N>/                        ← per-iteration の作業ディレクトリ(自動 gc)
        └── structural_themes_for_promotion.yaml  ← コーチが分離した structural テーマ
```

**設計意図**:

- target ディレクトリ本体に SKILL.md 以外を散らさない(他のスキルと同じ "見た目" を保つ)
- `.skill-opt/` はドット先頭で `ls` のデフォルト表示から外れる(`.git/` と同じ慣例)
- 複数のスキルへ展開しても、それぞれの target が綺麗なまま

`rejection-buffer-permanent.yaml` と `coach_meta_prompt.md` は **長期資産**。削除や軽率なリセットは禁止。

`rejection-buffer-run.yaml` は per-run データなので、空のまま終わった実行では作らない(次実行が必要に応じて作成する)。

---

## Red flags(本ループ実行中の自戒)

主要 3 件のみ(詳細は README.md の Red flags 節を参照):

- **pilot を飛ばそう** → μ_f が壊れていると後段は全部無駄。飛ばし禁止
- **役割兼任で速くしよう** → 自己評価バイアスが入る。選手 / リフレクタ / コーチは必ず別 subagent
- **validation も train と同じケースで** → 過学習一直線。完全分離必須

---

## 関連

- 実行例テンプレート: `examples/pilot-template.md`
- コーチへのメタプロンプト雛形: `examples/coach_meta_prompt_template.md`
- 詳細な使い方とトラブルシューティング: USAGE.md
