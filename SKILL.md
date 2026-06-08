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

人間規律ベースの先行手法と組み合わせる場合、新規 / 大幅改訂は手動チューニングで骨格を固め、本スキルで継続改善するのが推奨フロー。

---

### いつ使うか

- 頻繁に使われる対象スキルを堅牢化したいとき
- 自動改善ループの収束が悪い、または偽陽性 / 偽陰性が増えてきたとき
- 新規スキルを追加した直後、現実の入力で精度を確認したいとき
- モデル切替(例: sonnet → opus 等)で挙動が変わった際の再チューニング

**使わない**場面:

- 1 回限りのアドホックなプロンプト改善(評価コストが割に合わない)
- rubric が立てられないタスク(採点不能のため自動最適化は機能しない)
- orchestrator スキル(単体 SKILL 最適化とは別問題。別途設計が必要)

### 手動チューニングとの使い分け

| 観点 | 手動チューニング | skill-evolve |
|---|---|---|
| ループの主導者 | 人間(あるいは親 LLM) | コーチ subagent |
| 編集生成 | 人間が考える | コーチが `add` / `delete` / `replace` の3操作で提案 |
| 試走の構造 | シナリオ並列 | 並列ミニバッチ + ミニバッチ単位リフレクション + 提案統合 |
| 検証 | 同シナリオでの再評価 + 収束時のみ hold-out | 毎イテレーション held-out validation |
| 失敗の長期記憶 | なし | per-run + permanent の2層 |
| pilot の必須化 | 推奨 | **必須**(Step -1) |

推奨フロー:

```
新規スキル作成
  → 手動チューニング   (人間判断で骨格を固める、構造欠陥を取り除く)
  → skill-evolve       (継続改善、permanent buffer を育てる)
  → 頭打ち時は手動チューニングに戻る(構造を見直す段階)
```

### 入力契約

呼び出し時、ユーザー(または上位 skill)は以下を指定:

```yaml
target_skill_path:   <改善対象 SKILL.md の絶対パス>
test_cases_path:     <テストケース md の絶対パス>
feedback_spec_path:  <Step -1 で確定した μ_f 仕様書のパス>
max_iterations: 3
edit_budget: 4            # 学習率。SkillOpt アブレーションでの最適値
acceptance_margin: 5      # validation 改善がこのポイント以上で採用
n_minibatches: 4          # 並列ミニバッチ数
```

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
5. コーチへの初期メタプロンプトを下書き(`coach_meta_prompt_v0.md`)
6. 1〜2 イテレーションだけ手動でループを回し、提案された編集案を人間が読んで妥当性確認
7. **コスト試算**: 1イテレーション当たりの LLM 呼び出し数 × 単価 × 想定イテレーション

成果物(対象 skill ディレクトリの隣に保存):

- `feedback_spec.md` — μ_f の仕様(入力、出力、評価基準、`[critical]` の判定根拠)
- `coach_meta_prompt_v0.md` — コーチへの初期指示
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

リフレクタ出力:

- 失敗パターン仮説(case ごとに「何が起きていそうか」)
- 共通原因の抽出(ミニバッチ内で複数 case に共通する根本原因)
- 未充足な前提(スキルが暗黙に前提していて、選手が補完しきれていない情報)
- 削除候補(冗長 / 矛盾箇所)
- `tool_uses` 偏り観察(他比 3-5 倍以上なら構造的欠陥仮説)

精度のみで判断するとスキルの構造的欠陥が隠れる。**質的シグナルを常に主、メトリクスは補助**。

### Step 3: 提案統合 → 学習率まで圧縮

**新規のコーチ subagent** に次を渡す:

- `current_skill` の全文
- 全ミニバッチのリフレクション(ミニバッチごとに区別)
- `rejection-buffer-run.yaml`(この実行内の再提案禁止)
- `rejection-buffer-permanent.yaml`(恒久的失敗パターン、`coach_instruction` 絶対遵守)
- `coach_meta_prompt_v<latest>.md`
- `edit_budget` 上限

コーチの統合手順:

1. ミニバッチごとの提案を生成
2. 重複削除(同じ target/op はマージ)
3. 優先順位付け:複数ミニバッチで共有された問題への対処 > rationale 強さ > 単純さ
4. `edit_budget` 件まで truncate(超過分は破棄)

#### 編集案フォーマット

```yaml
- op: add | delete | replace
  target: <既存スキル内のアンカー行 or セクション見出し>
  before: |    # op=delete, replace のみ
    <置換/削除対象の元テキスト>
  after: |     # op=add, replace のみ
    <追加/置換後のテキスト>
  rationale: <この編集が Step 2 のどの不明瞭点/失敗をどう解決するか 1〜2 行>
  hits_failure: [<case-id>, ...]
  supporting_minibatches: [<minibatch-id>, ...]
```

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
| 構造性 | リフレクタが「構造的に効かない」と明示診断 |
| 失敗モードの明確さ | `failure_mode_hypothesis` が一般化可能(名前付きパターン) |

昇格時、類似案を **クラスタリングして代表案 1 件** で記録(肥大化防止)。

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
```

`coach_instruction` フィールドはコーチが毎回読んで **絶対遵守** する恒久ルール。

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

## ヒント
- tool_uses の偏り(case 間で 3-5 倍以上の差)は構造的欠陥のサイン
- 不明瞭点と裁量補完の頻度を質的シグナルとして主に扱う
- 精度だけで判断しない

## レポート構造
- 失敗パターン仮説: case-id ごとに「何が起きていそうか」を 1〜2 行
- 共通原因: 当該ミニバッチ内で複数 case に共通する根本原因(あれば)
- 未充足な前提: スキルが暗黙に前提していて、選手が補完しきれていない情報
- 削除候補: スキル内の冗長 / 矛盾箇所
- tool_uses 観察: 偏りがあれば構造的欠陥仮説を 1 行
```

## コーチ subagent 起動契約

```
あなたはスキル文書のオプティマイザ(コーチ)です。タスクを実行しません。
複数ミニバッチのリフレクタ診断を統合し、編集案だけを生成します。

## 現在のスキル全文
<current_skill>

## メタプロンプト(あなたへの恒久指示)
<coach_meta_prompt_v_latest.md の全文>

## 各ミニバッチのリフレクション
### minibatch-01
<リフレクタの診断レポート>
### minibatch-02
...

## per-run 棄却バッファ(同じ提案禁止)
<rejection-buffer-run.yaml の全文>

## permanent 棄却バッファ(恒久的に禁止)
<rejection-buffer-permanent.yaml の全文>
特に `coach_instruction` フィールドは絶対遵守。

## 制約
- 操作は add / delete / replace の 3 種類のみ
- 最大 <edit_budget> 件まで(優先順位付けで切り詰める)
- 各案に rationale(失敗のどれをどう解決するか)を必須で添える
- 各案に supporting_minibatches(どのミニバッチが同じ問題を指摘したか)を添える
- 過去却下リスト、permanent パターンに該当するものは提案禁止
- スキル全体の構造を書き換える大変更は禁止(局所修正のみ)

## 統合手順
1. 各ミニバッチのリフレクションから提案候補を洗い出す
2. 重複(同じ target / op)はマージ
3. 優先順位付け: 複数ミニバッチで共有された問題 > rationale の強さ > 提案の単純さ
4. <edit_budget> 件まで truncate

## 出力
編集案を YAML リストで返す(上記「編集案フォーマット」に従う)
```

---

## 出力アーティファクト

実行後、対象スキルディレクトリ配下の `.skill-opt/` に runtime データを集約する(target ディレクトリ本体は `SKILL.md` のみを visible に保つ):

```
<target_skill ディレクトリ>/
├── SKILL.md                              ← 進化済み(本体、visible)
└── .skill-opt/                           ← skill-evolve の runtime データ
    ├── baseline.md                       ← 起点バックアップ
    ├── changelog.md                      ← 全イテレーションの採用編集 + メトリクス推移
    ├── rejection-buffer-run.yaml         ← この実行の却下案(空なら作らない / 次実行で初期化)
    ├── rejection-buffer-permanent.yaml   ← 恒久的失敗パターン(削除厳禁)
    ├── feedback_spec.md                  ← Step -1 で確定した μ_f 仕様
    ├── coach_meta_prompt_v<N>.md         ← コーチへのメタ指示
    └── pilot_report.md                   ← Step -1 の人間判断との一致度
```

**設計意図**:

- target ディレクトリ本体に SKILL.md 以外を散らさない(他のスキルと同じ "見た目" を保つ)
- `.skill-opt/` はドット先頭で `ls` のデフォルト表示から外れる(`.git/` と同じ慣例)
- 複数のスキルへ展開しても、それぞれの target が綺麗なまま

`rejection-buffer-permanent.yaml` と `coach_meta_prompt_v*.md` は **長期資産**。削除や軽率なリセットは禁止。

`rejection-buffer-run.yaml` は per-run データなので、空のまま終わった実行では作らない(次実行が必要に応じて作成する)。

---

## Red flags(合理化に注意)

| 出てくる合理化 | 実態 |
|---|---|
| 「Step -1 pilot は時間がかかるので飛ばそう」 | μ_f が壊れていると後段は全部無駄。**飛ばし禁止**。 |
| 「選手と同じ subagent にリフレクタもコーチもやらせれば速い」 | 試走結果を見た当事者は客観評価できない。役割は 3 つに必ず分ける。 |
| 「ミニバッチ分割せず train 全件を 1 回で渡せばよくない?」 | 多視点集約が本手法の核心。1 ミニバッチ視点に依存すると診断が偏る。 |
| 「validation も train と同じケースで良くない?」 | 過学習一直線。validation は train から完全分離。 |
| 「コーチがいい案を出したから minibatch ゲート飛ばして validation」 | minibatch ゲートはサンプル効率の心臓。飛ばすと validation コストが線形に増える。 |
| 「permanent buffer はパターン化が手間なので per-run だけで十分」 | セッション跨ぎの長期失敗が再提案される。Step 9 の昇格運用は必須。 |
| 「edit_budget を 1 イテで 7 件にしたい」 | 編集ごとの寄与が混ざって測れなくなる。デフォルト 4 を超えるなら根拠を明示。 |
| 「精度が baseline 下回ったが rationale は正しい」 | rationale の正しさと validation スコアは別物。ロールバック発動。 |
| 「orchestrator スキルにも本スキルを当てたい」 | orchestrator は単体 SKILL 最適化とは別問題。別途設計が必要。 |

---

## 長期資産化のヒント

- **対象スキル選定**: rubric が立てやすいタスク(明確な成功/失敗判定が可能)から始める
- **避けるべき対象**: orchestrator スキル、主観性が高くて rubric 設計が難しいスキル
- **長期資産化**: 各対象スキルの `rejection-buffer-permanent.yaml` が貯まると、ドメイン固有の「効かない提案パターン辞典」が形成される

## 将来拡張 (v2 計画、本 SKILL の範囲外)

| 拡張 | 概要 | 必要になる場面 |
|---|---|---|
| Phase 9 メタ最適化 | エポック単位でコーチのメタプロンプト自体を更新 | 同じスキルを何ヶ月も継続改善するとき |
| multi-task Pareto frontier | 候補プールを Pareto frontier で管理 | 局所最適に頻繁にハマる場面 |
| System Aware Merge | 2 候補の良いとこ取りで新候補を作る genetic crossover | 系統樹分岐後の統合 |
| multi-objective Pareto(デプロイ選択) | 精度 / 簡潔さ / 安定性 / tool_economy の 4 軸 Pareto | 用途別に最適候補を選ぶ |

これらは別 SKILL(例: `skill-evolve-pareto`)として分離する想定。本 SKILL で `rejection-buffer-permanent.yaml` と `coach_meta_prompt_v*.md` を蓄積しておくと、v2 移行時の入力として活用できる。

---

## 関連

- 元論文: SkillOpt https://arxiv.org/abs/2605.23904 / GEPA(ICLR 2026)https://arxiv.org/abs/2507.19457
- 実行例テンプレート: `examples/pilot-template.md`(本ディレクトリ内)
