# WORKFLOW.md

## タイトル
AI成果物レビュー駆動ワークフロー

## バージョン
0.2.0

## ステータス
Draft

---

## 1. 概要

本ドキュメントは、設計・レビュー・質問回答・改訂・承認・実装までを段階的に進めるためのワークフローを定義する。  
目的は、AI駆動開発における「作ったが止まらない」「レビューしたが進めない」「不明点が会話の中に埋もれる」といった問題を防ぎ、成果物単位で状態を明確に管理しながら、機械的に次のアクションを判断できるようにすることである。

本バージョンでは、**最初に `WORKFLOW.md` だけを配置すれば、システムが `state/state.json` を自動生成して起動できる**ことを前提にしている。  
つまり、`WORKFLOW.md` は単なる説明書ではなく、**進行ルールと初期状態生成ルールを兼ねた起動定義ファイル**として扱う。

---

## 2. 目的

このワークフローの目的は以下の通りである。

- 人間の手作業によるコピペと進行管理を減らす
- 成果物ごとのレビューと承認を強制する
- 未解決事項が残っている状態で次工程へ進むことを防ぐ
- 設計文書間の整合性を維持したまま実装へ進める
- Claude Code、Codex CLI、別AI/API を役割分担して運用可能にする
- `WORKFLOW.md` 単体から初期状態を再現できるようにする

---

## 3. 基本原則

本ワークフローは、以下の原則に従って動作する。

1. **承認されるまで次工程へ進まない**  
   最新レビュー結果が `APPROVED` でない成果物は未完了とみなす。

2. **不明点は必ず質問として分離する**  
   レビュー本文に埋め込まず、`qa/*_questions.md` に切り出す。

3. **回答は必ずファイルとして保存する**  
   質問に対する判断は `qa/*_answers.md` として残し、改訂根拠を追跡可能にする。

4. **改訂後は必ず再レビューする**  
   一度レビュー済みであっても、内容が変わった場合は再度レビューを行う。

5. **状態は機械可読に管理する**  
   現在地、未解決事項、次アクションは `state/state.json` で管理する。

6. **AIは推測で仕様を確定しすぎない**  
   不足情報がある場合は質問として切り出し、勝手に埋めて進めない。

7. **`state/state.json` が存在しない場合は `WORKFLOW.md` から初期生成する**  
   起動時に `state/state.json` が無ければ、ブートストラップ処理を行う。

---

## 4. 想定する役割分担

### 4.1 Claude Code
Claude Code は主に作成・改訂・進行管理を担当する。

- 成果物の新規作成
- 回答反映による成果物改訂
- `state/state.json` を読んだ次アクション判定
- 次工程への進行制御
- 必要に応じた外部コマンド実行
- `state/state.json` の初期生成

### 4.2 Codex CLI
Codex CLI は主にレビューを担当する。

- 成果物レビュー
- 曖昧な記述、不足、矛盾の指摘
- レビュー結果の判定
- 必要に応じた質問一覧の抽出

### 4.3 回答担当AI
回答担当AIは、レビューで抽出された質問への回答を担当する。

- 方針の明確化
- 未確定事項の補完
- 仕様分岐の選択
- 改訂に必要な判断の文書化

---

## 5. 管理対象成果物

本ワークフローで管理する主な成果物は以下とする。

- 必須成果物: `SPEC.md`, `USECASE.md`, `SEQUENCE.md`, `CLASS.md`, `TEST.md`
- 任意成果物: `IMPLEMENTATION_PLAN.md` および案件固有で追加する設計文書
- 実装コード
- テストコード
- レビュー報告
- 質問ファイル
- 回答ファイル
- 実行ログ
- `state/state.json`

---

## 6. 成果物の標準進行順

設計から実装までの標準的な進行順は以下とする。

1. `SPEC.md`
2. `USECASE.md`
3. `SEQUENCE.md`
4. `CLASS.md`
5. `TEST.md`
6. 任意成果物（例: `IMPLEMENTATION_PLAN.md`, `API.md`）
7. 実装
8. 実装レビュー
9. テスト実行
10. 修正
11. 最終完了判定

この順序は標準形であり、案件に応じて追加成果物を挿入してもよい。  
ただし、依存関係を無視して後続成果物を先に進めてはならない。任意成果物は `WORKFLOW.md` の機械可読定義に列挙されたものだけを有効化する。

### 6.1 機械可読な成果物定義

`WORKFLOW.md` を起動定義ファイルとして扱うため、成果物一覧は `## Artifact Definitions` 見出し直下の YAML ブロックで宣言する。

````markdown
## Artifact Definitions
```yaml
artifacts:
  - name: SPEC.md
    required: true
  - name: USECASE.md
    required: true
  - name: SEQUENCE.md
    required: true
  - name: CLASS.md
    required: true
  - name: TEST.md
    required: true
  - name: IMPLEMENTATION_PLAN.md
    required: false
  - name: API.md
    required: false
```
````

ブートストラップ時は `## Artifact Definitions` セクションの定義を優先して `artifact_order` と `artifacts` を生成する。  
定義が存在しない場合は、後方互換として `SPEC.md` `USECASE.md` `SEQUENCE.md` `CLASS.md` `TEST.md` の5成果物を標準定義として用いる。  
任意成果物はこの定義に列挙したものだけを有効化し、列挙しない限り `IMPLEMENTATION_PLAN.md` も state には含めない。

---

## 7. 成果物単位の状態定義

各成果物は、少なくとも以下の状態を持つ。

- `not_started`  
  まだ開始していない状態。

- `drafted`  
  初版が作成された状態。

- `review_requested`  
  レビュー依頼済みで、レビュー結果待ちの状態。

- `reviewed`  
  レビュー報告が作成された状態。

- `questions_pending`  
  未回答質問が存在する状態。

- `answers_completed`  
  質問への回答が作成済みの状態。

- `revised`  
  回答を反映して成果物を改訂した状態。

- `approved`  
  最新レビュー結果が `APPROVED` の状態。

- `blocked`  
  人間判断待ち、依存不足、重大矛盾などにより進行不能な状態。

---

## 8. レビュー結果の定義

レビュー担当AIは、各レビュー報告の末尾に必ず以下のいずれかの結果を出力する。

- `APPROVED`
- `NEEDS_REVISION`
- `NEEDS_ANSWER`
- `BLOCKED`

### 8.1 各結果の意味

#### `APPROVED`
主要な問題が解消されており、次工程へ進行可能。

#### `NEEDS_REVISION`
改訂は必要だが、追加質問なしで修正可能。

#### `NEEDS_ANSWER`
仕様判断や未確定事項があり、回答ファイルの作成が必要。

#### `BLOCKED`
依存不足や重大矛盾があり、自動進行を停止すべき状態。

---

## 9. レビュー報告の必須形式

各レビュー報告の末尾には、以下のような機械可読ブロックを必須とする。

```text
## Review Result
Status: APPROVED
Blocking: no
Next Action: start_next_artifact
```

または

```text
## Review Result
Status: NEEDS_ANSWER
Blocking: yes
Next Action: answer_questions
```

この形式により、ワークフロー制御側は自由文全体を解釈せずとも、次アクションを安定して判定できる。

---

## 10. ディレクトリ構成の推奨例

```text
project/
  WORKFLOW.md
  SPEC.md
  USECASE.md
  SEQUENCE.md
  CLASS.md
  TEST.md
  IMPLEMENTATION_PLAN.md

  reviews/
    SPEC_review.md
    USECASE_review.md
    SEQUENCE_review.md
    CLASS_review.md
    TEST_review.md
    IMPLEMENTATION_PLAN_review.md

  qa/
    SPEC_questions.md
    SPEC_answers.md
    USECASE_questions.md
    USECASE_answers.md
    SEQUENCE_questions.md
    SEQUENCE_answers.md
    CLASS_questions.md
    CLASS_answers.md
    TEST_questions.md
    TEST_answers.md

  state/
    state.json

  logs/
    workflow.log

  prompts/
    create_artifact.md
    review_artifact.md
    answer_questions.md
    revise_artifact.md
```

上記は `IMPLEMENTATION_PLAN.md` を有効化した場合の推奨例である。  
ブートストラップ時は最低限 `state/` `reviews/` `qa/` `logs/` を自動生成する。  
`prompts/` は任意とし、固定テンプレート運用を採る場合のみ後から追加してよい。

---

## 11. 標準処理手順

### 11.1 ブートストラップ
起動時に `state/state.json` が存在しない場合、`WORKFLOW.md` の「初期状態生成ルール」に従って `state/` `reviews/` `qa/` `logs/` を自動生成し、`state/state.json` を生成する。`prompts/` は任意とし、必要時に遅延生成する。

### 11.2 生成
対象成果物が存在しない場合、作成担当AIが新規作成する。

### 11.3 レビュー
成果物生成後、レビュー担当AIがレビュー報告を作成する。

### 11.4 質問抽出
レビュー結果が `NEEDS_ANSWER` の場合、質問ファイルを生成する。

### 11.5 回答生成
質問ファイルが存在する場合、回答担当AIが回答ファイルを生成する。

### 11.6 改訂
回答ファイルを参照し、作成担当AIが成果物を改訂する。

### 11.7 再レビュー
改訂済み成果物を再度レビューする。

### 11.8 承認
最新レビュー結果が `APPROVED` なら、その成果物を承認済みとする。

### 11.9 次工程進行
現在成果物が承認済みの場合のみ、依存関係を満たす次の成果物へ進む。

---

## 12. 進行判定ルール

### 12.1 次工程に進める条件
以下をすべて満たす場合にのみ、次工程へ進める。

- 対象成果物が存在する
- 最新レビュー結果が `APPROVED`
- 未解決質問が存在しない
- `state/state.json` に blocking issue がない

### 12.2 差し戻し条件
以下のいずれかに該当する場合、現在成果物の改訂または回答生成に戻る。

- レビュー結果が `NEEDS_REVISION`
- レビュー結果が `NEEDS_ANSWER`
- 回答未反映の状態である
- 前工程との整合性エラーがある

### 12.3 停止条件
以下のいずれかに該当する場合、ワークフローを停止する。

- レビュー結果が `BLOCKED`
- 必須入力ファイルが不足している
- 外部コマンド実行に失敗し復旧不能
- 人間判断待ち事項がある
- セキュリティ上の判断が必要である

---

## 13. 自動進行ループ

ワークフロー制御ロジックは、概ね以下の順で処理を行う。

1. `WORKFLOW.md` の存在を確認する
2. `state/state.json` が無ければ、`state/` `reviews/` `qa/` `logs/` とあわせて初期生成する
3. `state/state.json` を読む
4. `current_artifact` を取得する
5. 対象成果物が未作成なら生成する
6. レビュー報告が未作成ならレビューする
7. 質問ファイルがあり、回答ファイルがなければ回答生成する
8. 回答ファイルがあり、未反映なら成果物を改訂する
9. 改訂済みなら再レビューする
10. 最新レビュー結果が `APPROVED` なら次成果物へ進む
11. 必須成果物と有効化された任意成果物が承認済みなら実装フェーズへ進む
12. 実装・テスト完了後、`final_status` を `done` に更新する

---

## 14. `state/state.json` 初期生成ルール

`state/state.json` は以下のルールで自動生成する。

### 14.1 生成タイミング
- 起動時に `state/state.json` が存在しない場合
- もしくは初期化要求が明示された場合

### 14.2 補助ディレクトリの初期生成
- `state/`
- `reviews/`
- `qa/`
- `logs/`

`prompts/` は任意とし、初回ブートストラップでは生成必須としない。

### 14.3 成果物定義の解決
- `WORKFLOW.md` 内に `## Artifact Definitions` セクションがある場合は、その定義を優先して使う
- 定義が無い場合は `SPEC.md` `USECASE.md` `SEQUENCE.md` `CLASS.md` `TEST.md` を標準の必須成果物として使う
- `IMPLEMENTATION_PLAN.md` や追加成果物は、機械可読定義に列挙された場合のみ有効化する

### 14.4 初期値
- `project_name`: ルートフォルダ名、または固定名
- `current_artifact`: `SPEC.md`
- `current_phase`: `spec`
- `final_status`: `in_progress`
- `blocking_issues`: 空配列
- `human_decisions_pending`: 空配列
- `next_action`: `generate_spec`
- `artifact_order`: 成果物定義セクション、または標準定義に従う

### 14.5 成果物ごとの初期化
有効化された各成果物は以下の初期状態で生成する。

- `required`: 必須成果物なら `true`、任意成果物なら `false`
- `status`: `not_started`
- `review_status`: `null`
- `review_file`: 規約パスを設定
- `questions_file`: 規約パスを設定
- `answers_file`: 規約パスを設定
- `revision_count`: `0`
- `approved`: `false`

ただし、`current_artifact` である `SPEC.md` は開始対象であるため、`next_action` は `generate_spec` とする。

### 14.6 規約パス
例:

- `reviews/SPEC_review.md`
- `qa/SPEC_questions.md`
- `qa/SPEC_answers.md`

---

## 15. `state/state.json` 自動生成テンプレート

以下は `WORKFLOW.md` に機械可読定義が存在しない場合に生成される `state/state.json` の標準テンプレートである。

```json
{
  "project_name": "sample-project",
  "current_artifact": "SPEC.md",
  "current_phase": "spec",
  "final_status": "in_progress",
  "artifact_order": [
    "SPEC.md",
    "USECASE.md",
    "SEQUENCE.md",
    "CLASS.md",
    "TEST.md"
  ],
  "artifacts": {
    "SPEC.md": {
      "required": true,
      "status": "not_started",
      "review_status": null,
      "review_file": "reviews/SPEC_review.md",
      "questions_file": "qa/SPEC_questions.md",
      "answers_file": "qa/SPEC_answers.md",
      "revision_count": 0,
      "approved": false
    },
    "USECASE.md": {
      "required": true,
      "status": "not_started",
      "review_status": null,
      "review_file": "reviews/USECASE_review.md",
      "questions_file": "qa/USECASE_questions.md",
      "answers_file": "qa/USECASE_answers.md",
      "revision_count": 0,
      "approved": false
    },
    "SEQUENCE.md": {
      "required": true,
      "status": "not_started",
      "review_status": null,
      "review_file": "reviews/SEQUENCE_review.md",
      "questions_file": "qa/SEQUENCE_questions.md",
      "answers_file": "qa/SEQUENCE_answers.md",
      "revision_count": 0,
      "approved": false
    },
    "CLASS.md": {
      "required": true,
      "status": "not_started",
      "review_status": null,
      "review_file": "reviews/CLASS_review.md",
      "questions_file": "qa/CLASS_questions.md",
      "answers_file": "qa/CLASS_answers.md",
      "revision_count": 0,
      "approved": false
    },
    "TEST.md": {
      "required": true,
      "status": "not_started",
      "review_status": null,
      "review_file": "reviews/TEST_review.md",
      "questions_file": "qa/TEST_questions.md",
      "answers_file": "qa/TEST_answers.md",
      "revision_count": 0,
      "approved": false
    }
  },
  "blocking_issues": [],
  "human_decisions_pending": [],
  "next_action": "generate_spec"
}
```

任意成果物を有効化する場合は、機械可読定義セクションに列挙した上で、同じ構造のエントリを `artifact_order` と `artifacts` に追加する。任意成果物の `required` は `false` とする。

---

## 16. 実装フェーズへの進行条件

以下をすべて満たした場合のみ、実装フェーズへ進行できる。

- `SPEC.md` が承認済み
- `USECASE.md` が承認済み
- `SEQUENCE.md` が承認済み
- `CLASS.md` が承認済み
- `TEST.md` が承認済み
- `artifact_order` に含まれる任意成果物がある場合、それらも承認済み

---

## 17. テストフェーズへの進行条件

以下をすべて満たした場合のみ、テスト実行へ進行できる。

- 実装コードが存在する
- テストコードが存在する
- 実装レビューが `APPROVED`
- 実装と設計文書の重大矛盾が解消済みである

---

## 18. ログ出力要件

すべての処理は `logs/workflow.log` に記録する。  
最低限記録する内容は以下とする。

- 開始時刻
- 対象成果物
- 実行ステップ
- 使用AIまたは実行コマンド
- 出力ファイル
- 判定結果
- エラー内容
- 次アクション

ログは後から「なぜ止まったか」「どの回答で方針が変わったか」を追跡するために用いる。

---

## 19. エラー時の取り扱い

### 19.1 一時的エラー
API 呼び出し失敗や CLI 実行失敗など、一時的なエラーは再試行対象とする。

### 19.2 論理エラー
レビューで重大矛盾が出た場合は、自動再試行せず `blocked` とし、人間または回答担当AIの判断を待つ。

### 19.3 ファイル欠落
前提ファイルが欠けている場合、当該成果物は進行不能とし、`blocking_issues` に記録する。

---

## 20. 最小構成で必要なファイル

本ワークフローを最低限動かすために必要なファイルは以下の1つである。

- `WORKFLOW.md`

本バージョンでは、`WORKFLOW.md` が存在すれば、そこから `state/` `reviews/` `qa/` `logs/` と `state/state.json` を初期生成できる。  
その後、ワークフローは成果物定義に従って最初の成果物を確定し、自動的に設計フェーズを開始する。

---

## 21. 完了条件

以下をすべて満たした場合に、ワークフローは完了とみなす。

- 必須成果物がすべて `approved`
- `artifact_order` に含まれる任意成果物がある場合、それらも `approved`
- 実装レビューが `APPROVED`
- テスト結果が成功
- `state/state.json.final_status == "done"`


## 22. Codex CLI 実行規約

### レビュー実行
codex exec "SPEC.md をレビューし、曖昧点・不足・矛盾を列挙し、最後に APPROVED / NEEDS_ANSWER / NEEDS_REVISION / BLOCKED を出力せよ"

### JSONL 実行
codex exec --json "USECASE.md をレビューせよ"

---

## 23. まとめ

このワークフローの本質は、AIを賢くすることではなく、**AIが迷わないように状態と停止条件を明示すること**にある。  
従来は `SPEC.md`、`WORKFLOW.md`、`state/state.json` の3点セットを最小構成としていたが、本バージョンでは `WORKFLOW.md` 自体に `state/state.json` の初期生成ルールと成果物定義ルールを埋め込むことで、起動時の必要ファイルをさらに削減した。

したがって、AI駆動開発を最小構成で始めたい場合、人間が最初に用意すべきファイルは次の1つになる。

- `WORKFLOW.md`

この `WORKFLOW.md` を起点として、`state/state.json` の初期化、成果物定義の解決、レビュー、質問回答、改訂、承認、実装までを段階的に機械化しやすくなる。
