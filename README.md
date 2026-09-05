# coderabbit-review-setup

CodeRabbit を任意のリポジトリへ導入・調整するための **Agent Skill** です。

固定の `.coderabbit.yaml` を配るのではなく、CodeRabbit で可能なレビュー機能を知識として持たせ、対象リポジトリに応じて **どのようなレビューをさせるべきか / 自動・手動のどちらがよいか / どこまで merge を止めるべきか** をエージェント自身に判断させることを目的にしています。

## 構成

README と Agent Skill 本体は分離しています。

```text
coderabbit-review-setup/
├── README.md
└── SKILLS/
    └── coderabbit-review-setup/
        └── SKILL.md
```

Agent Skills として利用する本体は [`SKILLS/coderabbit-review-setup/SKILL.md`](./SKILLS/coderabbit-review-setup/SKILL.md) です。

## 扱う内容

- CodeRabbit が確認できるレビュー領域
  - Security & Privacy
  - Stability & Availability
  - Data Integrity & Integration
  - Functional Correctness
  - Performance & Scalability
  - Maintainability & Code Quality
- `.coderabbit.yaml` の主要設定
  - レビュー言語・tone
  - `quiet / chill / assertive`
  - automatic / manual / label opt-in review
  - `path_filters`
  - `path_instructions`
  - Request Changes Workflow
  - Pre-Merge Checks
  - Slop Detection
  - linter / security / static-analysis tools
- `@coderabbitai full review` / `@coderabbitai review` などの主要コマンド
- GitHub Ruleset / Branch protection と CodeRabbit の役割分担
- Web/API、CLI、daemon、library、DB、frontend、distributed system、IaC、firmware など、リポジトリの性質に応じたレビュー観点

## 使い方

`SKILLS/coderabbit-review-setup` ディレクトリを、そのまま Agent Skills 対応エージェントの skills ディレクトリへ配置して利用してください。

例えばエージェントに次のように依頼します。

```text
このリポジトリにCodeRabbitを導入して。
レビュー方針もこのrepoに合うように考えて.coderabbit.yamlを作って。
```

エージェントはSkillを参照し、必要以上に設定を増やさず、そのリポジトリに合う `.coderabbit.yaml` を作成・更新する想定です。

## 考え方

| 要素 | 役割 |
|---|---|
| `.coderabbit.yaml` | CodeRabbit の恒常的なレビュー方針 |
| `@coderabbitai full review` | その方針で PR 全体をレビュー |
| `@coderabbitai review` | その方針で差分を再レビュー |
| CI | build / test / type check などを実際に実行 |
| GitHub Ruleset | review/check を本当の merge 条件として強制 |

CodeRabbit を単なる formatting bot にせず、**正しさ・セキュリティ・信頼性・データ整合性・互換性・性能・テスト品質・保守性**を中心にレビューさせることを重視しています。

## 公式ドキュメント

詳細と参照URLは [`SKILLS/coderabbit-review-setup/SKILL.md`](./SKILLS/coderabbit-review-setup/SKILL.md) にまとめています。

---

Skill 作成・公式ドキュメント確認: **2026-09-05**
