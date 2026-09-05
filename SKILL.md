---
name: coderabbit-review-setup
description: CodeRabbitを任意のリポジトリへ導入・調整するときに、利用できるレビュー機能と.coderabbit.yamlの設定を踏まえ、そのリポジトリに合うレビュー方針を考えて最小限の設定を作成または更新する。
---

# CodeRabbit Review Setup

_作成・公式ドキュメント確認: 2026-09-05_

## 目的

CodeRabbitを、単なるLint botではなく、そのリポジトリに合ったPRレビュアーとして設定する。

固定テンプレートを無条件に適用しない。CodeRabbitに何ができるかを踏まえ、対象リポジトリで何を重視し、いつレビューし、どこまでmergeを止めるかを考え、必要な設定だけをrootの `.coderabbit.yaml` に書く。

## 基本モデル

| 要素 | 役割 |
|---|---|
| `.coderabbit.yaml` | レビュー言語、厳しさ、対象範囲、評価基準、起動方法を定義する |
| `@coderabbitai full review` | 現在の設定でPR全体を最初からレビューする |
| `@coderabbitai review` | 現在の設定で前回以降の変更だけをレビューする |
| CI | build、test、type checkなどを実際に実行する |
| GitHub Ruleset / Branch protection | reviewやcheckを本当のmerge条件として強制する |

`full review` は別のレビュー・プリセットではない。通常の `review` と同じ設定を使い、レビュー範囲だけをPR全体にする。

## CodeRabbitができるレビュー

CodeRabbitのレビュー指摘は主に次の6分類。

| 分類 | 見られること |
|---|---|
| Security & Privacy | 認証、認可、権限昇格、入力検証、secret、個人情報、データ露出 |
| Stability & Availability | crash、未処理例外、resource leak、race、deadlock、障害復旧 |
| Data Integrity & Integration | transaction、migration、schema、外部API、queue、filesystem、データ破壊 |
| Functional Correctness | ロジック、境界値、edge case、regression、要件との不一致 |
| Performance & Scalability | N+1、無駄なI/O、高コスト処理、memory/CPU消費、拡張時の問題 |
| Maintainability & Code Quality | 重複、責務混在、coupling、複雑性、可読性、構造、best practice |

保守性や品質も見られる。ただし、「別の書き方の方が好み」ではなく、将来の不具合、仕様不一致、変更コスト、テスト困難性などの具体的な影響がある問題を優先する。

CodeRabbitは各指摘を `Critical`、`Major`、`Minor`、`Trivial`、`Info` に分類する。重要な問題を優先し、Trivialな指摘でレビューを埋めない設定を目指す。

## レビューに使える文脈と補助機能

CodeRabbitは、変更行だけでなく、利用可能な範囲で周辺コード、依存関係、コードグラフ、Git履歴、PR本文、linked issue、CI failure logなども利用できる。

また、次のような機能がある。

- high-level summaryとchanged-files walkthrough
- sequence diagramとreview effort estimate
- linked issue assessment、related issues / PRs
- CI/CD failure分析
- 50以上のlinter・security/static analysis tools
- one-click suggestion / Autofix
- unit test、docstring、simplificationなどのFinishing Touches
- slop detection
- built-in / custom Pre-Merge Checks

これらを混同しない。

- 通常review: 問題を発見して説明する
- summary / walkthrough: 変更理解を助ける
- Pre-Merge Checks: 明示的にpass/failを判定する
- Finishing Touches: 修正や生成を行う
- GitHub Ruleset: mergeを強制制御する

## YAMLで設定できる主なこと

### 言語と口調

```yaml
language: ja-JP
tone_instructions: "簡潔で具体的に説明し、影響と修正方向を示す。主観的な好みだけでは指摘しない。"
```

例:

- 日本語: `ja-JP`
- 英語: `en-US`
- 中国語簡体字: `zh-CN`
- 中国語繁体字: `zh-TW`
- ロシア語: `ru-RU`
- イタリア語: `it-IT`

### レビューの厳しさ

```yaml
reviews:
  profile: chill
```

- `quiet`: 最重要の指摘中心
- `chill`: バランス型。多くのrepoで無難
- `assertive`: より広く積極的に指摘するが、細かく感じる可能性がある

何でも `assertive` にするより、必要なreview guidanceを的確に書く方がよい場合が多い。

### レビューの起動方法

必要なときだけ手動:

```yaml
reviews:
  auto_review:
    enabled: false
```

Labelでopt-in:

```yaml
reviews:
  auto_review:
    enabled: false
    drafts: false
    labels:
      - "review-ready"
```

自動レビュー:

```yaml
reviews:
  auto_review:
    enabled: true
    drafts: false
    auto_incremental_review: true
```

auto reviewは、description keyword、PR title、label、base branch、author、draft、auto-pauseでも制御できる。

### レビュー対象

```yaml
reviews:
  path_filters:
    - "!dist/**"
    - "!build/**"
    - "!vendor/**"
    - "!generated/**"
```

生成物やvendorなど、レビュー価値の低いものだけを除外する。migration、CI、infrastructure、設定、security-sensitive codeまで雑に除外しない。

### レビュー基準

`path_instructions` は、リポジトリ全体または特定pathに追加のreview guidanceを与える。

```yaml
reviews:
  path_instructions:
    - path: "**"
      instructions: |
        正しさ、セキュリティ、回帰、信頼性、データ整合性、
        互換性、性能、テスト品質、保守性、アーキテクチャを確認する。
        実行経路または将来コストを具体的に説明できる問題を優先する。
        formattingや主観的な書き換えだけをmerge blockerにしない。
```

特定領域だけ重点を変えることもできる。

```yaml
reviews:
  path_instructions:
    - path: "src/auth/**"
      instructions: |
        Security boundaryとして扱い、認証、認可、権限昇格、
        session/token、入力検証、secret露出を重点確認する。

    - path: "migrations/**"
      instructions: |
        既存データとの互換性、data loss、transaction、
        rollback、再実行、partial failureを重点確認する。
```

path名は例。対象repoの構造と現実的なリスクに合わせる。

### 既存の開発ガイドライン

CodeRabbitは次のようなファイルを自動検出し、review criteriaとして利用できる。

- `AGENTS.md`
- `AGENT.md`
- `CLAUDE.md`
- `GEMINI.md`
- `.cursorrules`
- GitHub Copilot instructions
- Cursor / Windsurf / Clineのrules

既存の規約を `.coderabbit.yaml` に丸ごと複製しない。YAMLではCodeRabbit固有の挙動とreview-specificな重点を設定する。

### Linter・解析ツール

CodeRabbitは50以上の外部ツールを統合する。ほとんどは既定で有効で、対象ファイルがある場合だけ自動選択されるため、通常は大量に列挙しない。

設定する主な理由は、ノイズの多いtoolを無効化するか、既存configの非標準pathを指定すること。

```yaml
reviews:
  tools:
    ruff:
      enabled: true
      config_file: "pyproject.toml"
```

tool keyと利用条件は現在の公式referenceで確認する。

## Request ChangesとPre-Merge Checks

### Request Changes Workflow

```yaml
reviews:
  request_changes_workflow: true
```

actionableなinline findingがあると、CodeRabbitがRequest Changesを出せる。通常は、最新commitのreviewが完了し、必要なthreadが解決し、failing Pre-Merge CheckがなくなるとApproveできる。

これはCodeRabbitのreview decisionを制御する設定。GitHub上で本当にmergeを止めるにはRulesetまたはbranch protectionも必要。

### Pre-Merge Checks

built-in checks:

- docstring coverage
- PR title
- PR description
- linked issue assessment

```yaml
reviews:
  pre_merge_checks:
    title:
      mode: warning
      requirements: "変更内容が分かる簡潔なタイトルにする。"
    description:
      mode: warning
    issue_assessment:
      mode: warning
```

custom checkでは自然言語でdeterministicなpass/fail条件を定義できる。

```yaml
reviews:
  pre_merge_checks:
    custom_checks:
      - name: "Maintainability regression"
        mode: warning
        instructions: |
          明確な保守性の後退がある場合だけfailする。
          例: 分岐しやすいbusiness logicの重複、
          architecture boundaryの明白な違反、
          将来変更を危険にするcoupling。
          主観的なstyleだけではfailしない。
```

mode:

- `off`: 無効
- `warning`: 警告するがblockしない
- `error`: Request Changes Workflowと組み合わせ、blockingに使える

新しいcheckは、まず `warning` で精度を確認してから `error` を検討する。planによって利用可否と上限が異なる。

## 主なコマンド

| コマンド | 役割 |
|---|---|
| `@coderabbitai full review` | PR全体を最初からreview |
| `@coderabbitai review` | 前回以降の変更だけreview |
| `@coderabbitai pause` / `resume` | 自動reviewを一時停止 / 再開 |
| `@coderabbitai ignore` | PR descriptionに置き、そのPRの自動reviewを無効化 |
| `@coderabbitai approve` | Request Changes Workflowでthread解決と承認を試行 |
| `@coderabbitai resolve` | CodeRabbitのthreadをresolve |
| `@coderabbitai configuration` | 現在適用中の設定を確認 |
| `@coderabbitai emit path instructions` | 蓄積されたguidanceからpath instructions更新PRを作成 |
| `@coderabbitai run pre-merge checks` | Pre-Merge Checksを再実行 |
| `@coderabbitai rate limit` | 残りreview allowanceを確認 |
| `@coderabbitai help` | command一覧を表示 |

Autofix、unit tests、docstrings、simplify、fix-ciなどは、review presetではなく修正・生成機能として扱う。

## リポジトリに合わせた重点の例

| Repoの性質 | 重点候補 |
|---|---|
| Web / API | auth/authz、validation、API compatibility、data exposure、idempotency |
| CLI / daemon / server | restart、signal、timeout、retry、permission、partial write |
| library / SDK | public API、semantic compatibility、error contract、dependency増加 |
| database-heavy | migration、transaction、locking、data loss、既存データ互換性 |
| frontend | state consistency、async race、loading/error state、accessibility |
| distributed system | duplicate delivery、ordering、retry、timeout、observability |
| infrastructure / IaC | privilege、public exposure、secret、destructive change、rollback |
| firmware / hardware | bounds、timing、power loss、unsafe device state、resource constraints |
| AI / ML / data | data leakage、reproducibility、evaluation leakage、resource use |

これは固定チェックリストではない。対象repoで現実に起こり得る壊れ方から、必要な重点だけを選ぶ。

## 代表的な設定パターン

### 手動レビューを基本にする

```yaml
# yaml-language-server: $schema=https://coderabbit.ai/integrations/schema.v2.json

language: ja-JP
tone_instructions: "簡潔で具体的に説明し、影響と修正方向を示す。主観的な好みだけでは指摘しない。"

reviews:
  profile: chill
  request_changes_workflow: false

  auto_review:
    enabled: false

  path_instructions:
    - path: "**"
      instructions: |
        バグ、セキュリティ、回帰、信頼性、データ整合性、
        互換性、テスト不足、具体的な保守性低下を優先する。
        formattingや主観的な好みだけの指摘は避ける。
```

完成時に `@coderabbitai full review`、修正後に `@coderabbitai review` を使う。

### CodeRabbitを承認役にする

```yaml
# yaml-language-server: $schema=https://coderabbit.ai/integrations/schema.v2.json

language: ja-JP

reviews:
  profile: chill
  request_changes_workflow: true

  auto_review:
    enabled: true
    drafts: false
    auto_incremental_review: true

  path_instructions:
    - path: "**"
      instructions: |
        正しさ、セキュリティ、回帰、信頼性、データ整合性、
        互換性、性能、テスト品質、保守性を確認する。
        concreteなfailure modeまたは意味のある将来コストがある問題を優先する。
        formattingのみの指摘をmerge blockerにしない。
```

mandatory merge gateにする場合は、別途GitHub Ruleset / branch protectionを設定する。

## 重要な注意

### `fail_commit_status`は通常finding用ではない

`reviews.fail_commit_status` は、CodeRabbitのreview処理自体がerrorになった場合に外向きstatusをfailさせる設定。

レビュー指摘が存在するだけでstatusをfailureにする単純なスイッチではない。

### 最新commitをreviewさせる

必須reviewでは、最新commitがCodeRabbitによってreview済みであることを確認する。自動incremental review、最後の手動 `@coderabbitai review`、CodeRabbitのreview progress、GitHub required checksを組み合わせる。

### `.coderabbit.yaml` があるrepoだけで使う

YAMLの存在自体はCodeRabbit Appのアクセス条件ではない。組織全体をopt-in方式にするなら、Organization側ではautomatic reviewを無効にし、使うrepoだけYAMLでmanual、label opt-in、automaticのいずれかを選ぶ。

### Public / OSS repo

2026-09-05時点の公式案内では、open-source public repositoriesには無償のOSS accessがある。ただしreview rateはprojectのcommunityとpopularityで変わる。

public repositoryが10 stars未満の場合、PR reviewは手動起動が必要。

```text
@coderabbitai full review
```

または:

```text
@coderabbitai review
```

plan、rate limit、feature availabilityは変わり得るため、導入時点の公式documentationを確認する。

## このスキルを使うとき

対象repoについて次を判断する。

- 助言役、承認役、mandatory merge gateのどれにするか
- manual、label opt-in、automaticのどれにするか
- review言語、tone、profile
- このrepoで現実に起こり得るfailure
- 重点reviewが必要なpath
- formatter、linter、CIへ任せる範囲
- 通常findingとPre-Merge Checkの使い分け
- review noiseと誤検知
- 現在のplanで利用可能か

その判断から、必要な項目だけを持つ最小の `.coderabbit.yaml` を作成または更新する。全schemaを出力したり、利用可能な機能を全部有効化したりしない。

成果物は原則として次の2つ。

1. repository rootの `.coderabbit.yaml`
2. 採用したreview mode、重点、blocking方針の短い説明

GitHub Rulesetが必要な場合は、YAMLとは別設定であることを明記する。

## 公式ドキュメント

- Configuration Reference  
  https://docs.coderabbit.ai/reference/configuration
- YAML Configuration  
  https://docs.coderabbit.ai/getting-started/yaml-configuration
- Pull Request Review Overview  
  https://docs.coderabbit.ai/guides/code-review-overview
- Review Commands  
  https://docs.coderabbit.ai/reference/review-commands
- Automatic Review Controls  
  https://docs.coderabbit.ai/configuration/auto-review
- Path-based Review Instructions  
  https://docs.coderabbit.ai/configuration/path-instructions
- Request Changes Workflow  
  https://docs.coderabbit.ai/pr-reviews/request-changes-workflow
- Pre-Merge Checks  
  https://docs.coderabbit.ai/pr-reviews/pre-merge-checks
- Third-party Tools  
  https://docs.coderabbit.ai/tools
- Code Guidelines  
  https://docs.coderabbit.ai/knowledge-base/code-guidelines
- Plans and Pricing  
  https://docs.coderabbit.ai/management/plans
