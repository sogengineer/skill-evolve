# skill-evolve pilot 起動キット(汎用テンプレート)

`skill-evolve` を任意の対象 SKILL.md に対して走らせるための **pilot(Step -1)起動キット** のテンプレート。

このドキュメントは pilot 段階のテンプレートであり、本ループに入る前の最重要工程である「採点ロジック(`μ_f`)を人間直感と突合する」作業を効率化するために用意されている。

## 目的

対象 SKILL の改善ループを安全に始められる状態にする。具体的には:

1. **`μ_f`(採点関数)を確定**して `feedback_spec.md` に固定する
2. **test_cases の最小セット**を作って、後段の本ループが回せる粒度にする
3. **コーチへの初期メタプロンプト**を `coach_meta_prompt.md` として用意する

これらが揃って初めて、Step 0 以降の本ループに進める。

---

## 前提条件

- `skill-evolve` SKILL を熟読済み
- 対象スキルを通常実行し、入出力の形を理解している
- 構造的整合性(description ↔ body)は事前に手で揃えている

---

## 1. `μ_f`(feedback function)の設計テンプレート

対象スキルの出力に応じて以下を埋める:

```yaml
# feedback_spec.md (Step -1 で確定)
input:
  - <対象スキルへの入力(ファイル一覧、プロンプト、コンテキスト等)>
  - <期待される結果(ground truth)>

expected_output_shape:
  - <出力の構造仕様(テーブル、JSON、自由記述 etc.)>
  - <必須フィールド一覧>

rubric:
  - id: r1
    text: "[critical] <最低ラインを 1 文で記述>"
    weight: 1.0
  - id: r2
    text: "[critical] <存在意義そのものに関わる項目>"
    weight: 1.0
  - id: r3
    text: "[critical] <下流処理が parse 可能な形式要件>"
    weight: 1.0
  - id: r4
    text: "<通常要件 / 偽陽性ゼロ など>"
    weight: 0.8
  - id: r5
    text: "<通常要件 / 判定基準の一貫性 など>"
    weight: 0.8
  - id: r6
    text: "<補助要件 / 漏れ防止チェック>"
    weight: 0.6
  - id: r7
    text: "<補助要件 / レポート粒度>"
    weight: 0.6
  - id: r8
    text: "<付加価値 / 分類・優先度付け>"
    weight: 0.4

scoring:
  precision: (Σ achieved_weight) / (Σ all_weight)
  pass_critical: 全 [critical] が ○ なら ○、1つでも × / 部分的 なら ×
```

`[critical]` の判定根拠を明確にする(例):

- **r1**(形式要件)— 下流処理の parse 可能性を担保
- **r2**(存在意義)— このスキルが解こうとしている問題そのもの
- **r3**(actionable)— 結果が次のアクションに繋がるか

これら 3 つ(あるいは別の本質的要件)が揃わない出力は実用上失敗扱いにする。

---

## 2. test_cases 最小セット(starter pack)

実際の fixtures は別途、対象ドメインの本物 or 合成データで構築する必要がある。以下は **構造のテンプレート**。

> [!note] fixtures の置き場所
> 推奨: `examples/<target-skill-name>-pilot/fixtures/case-XX/` 配下に入力データを配置。
> 1 ケース 1 ディレクトリで完結する小さな compound にする。

### case-01 [train] — <代表的な成功パターン>

**prompt** (選手 subagent に渡す):

```
<対象スキルを起動するプロンプト。
 input fixtures のパスや前提を明示する>
```

**rubric** (`[critical]` を必ず 1 つ含む):

```
- [critical] <最低ラインの判定基準>
- <通常要件>
- <通常要件>
```

**ground truth** (採点で参照):

```
<期待される出力の要点。完全一致である必要はない>
```

### case-02 [train] — <代表的な失敗パターン>

(同様の構造で、対象スキルが従来取りこぼしていた / 弱かったケースを記述)

### case-03 [train] — <境界ケース>

(rubric が分かれる微妙なケース。Step -1 で人間直感との一致度を測る上で重要)

### case-04 [validation] — <hold-out>

(train とは別の入力で、汎化性能を測るためのケース)

---

## 3. コーチへの初期メタプロンプト(coach_meta_prompt.md)

```markdown
# Coach Meta Prompt

あなたは <対象スキル> のオプティマイザです。以下の方針を恒久遵守してください。

## ドメイン前提
- <対象ドメインの特性 例: 静的なテキスト生成 / コードレビュー / 要約 / 検索 など>
- <ユーザーが期待する出力スタイル>

## 編集方針
- 局所修正のみ。スキル全体の構造を書き換える大変更は禁止
- 既存のチェックリストや見出し構造は維持する
- 新規セクションを追加するときは「いつ参照するか」のトリガー条件を明示

## 過去の学び(必要に応じて追記される)
- (Step 9 で `coach_instruction` が permanent buffer から昇格してきたものをここに追記)

## 禁止事項
- frontmatter (description, allowed-tools 等) の書き換え
- pilot 段階で確定した rubric 自体への言及・修正提案
```

---

## 4. pilot 実行手順

1. 上記 1 - 3 を埋める
2. 5〜10 ケースを手動で対象スキルに通す
3. rubric で採点し、人間直感と突合する
   - 不一致が出たら **rubric を修正**(対象スキルではなく rubric が悪い)
4. 1 - 2 イテレーション、コーチを手動で起動して編集案を出させる
5. 編集案が「効きそう」と人間が判断できるか確認
6. コスト試算(LLM 呼び出し × 単価 × 想定イテレーション)
7. 結果を `pilot_report.md` にまとめる

pilot で μ_f の信頼性が確保できなければ、本ループに入らず rubric 設計に戻ること。

---

## 5. pilot_report.md のテンプレート

```markdown
# Pilot Report — <target skill name>

## 実施日
YYYY-MM-DD

## サンプル数
- 試走ケース: N 件
- うち [critical] 全達成: M 件

## 採点結果と人間直感の一致度
| case | 採点 | 人間判断 | 一致 |
|---|---|---|---|
| case-01 | ○ | ○ | ✅ |
| case-02 | × | × | ✅ |
| case-03 | ○ | × | ❌ (r4 を厳しくする) |
...

一致率: X / N

## 採点ロジック修正履歴
- <rubric の修正点>

## コーチ手動起動の所感
- <提案された編集案の妥当性>
- <Red flags に該当する提案が出たか>

## コスト試算
- 1 イテレーション LLM 呼び出し: ~K 回
- 想定総コスト: $Y (max_iterations=3, n_minibatches=4 想定)

## 本ループ Go / No-Go 判定
- [ ] μ_f が人間直感と一致する
- [ ] コーチが妥当な編集案を出す
- [ ] コスト範囲内
→ 全て ✅ なら本ループへ。1 つでも ❌ なら設計に戻る。
```
