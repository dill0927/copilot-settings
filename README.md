# copilot-settings

GitHub Copilot を使いこなすための個人設定・スキル・カスタムエージェントをまとめたリポジトリ。

参考: [awesome-copilot](https://awesome-copilot.github.com/)

## Copilot カスタマイズ手段の全体像

| カテゴリ | ファイル形式 | 役割 | 本リポジトリ |
|---|---|---|---|
| **Repo Instructions** | `copilot-instructions.md` | リポジトリ全体に常時適用（GitHub.com にも反映） | `.github/` |
| **Prompts** | `.prompt.md` | チャットで `/コマンド` として呼び出す再利用プロンプト | `.github/prompts/` |
| **Instructions** | `.instructions.md` | ファイルパターンに自動適用されるコーディング規約 | `.github/instructions/` |
| **Agents** | `.agent.md` | 専用ツール・ペルソナを持つカスタムチャットモード | `.github/agents/` |
| **Skills** | `SKILL.md`（フォルダ） | 関連リソースをまとめた自己完結型スキルパッケージ | `.github/skills/` |
| **MCP** | `mcp.json` | 外部ツール・API を Copilot に接続する | `.vscode/` |
| **Hooks** | ワークフロー定義 | エージェントイベントをトリガーとした自動化 | 今後追加予定 |
| **Workflows** | GitHub Actions | AI を組み込んだ CI/CD ワークフロー | 今後追加予定 |

## ディレクトリ構成

```
copilot-settings/
├── .github/                       # VS Code が自動検出するパス（settings.json 不要）
│   ├── copilot-instructions.md    # リポジトリ全体への共通指示（GitHub.com にも反映）
│   ├── prompts/                   # .prompt.md — スラッシュコマンド用プロンプト
│   │   ├── code-review.prompt.md
│   │   └── write-tests.prompt.md
│   ├── instructions/              # .instructions.md — ファイルパターンに自動適用
│   │   ├── general.instructions.md
│   │   └── security.instructions.md
│   └── agents/                    # .agent.md — カスタムエージェント
│       └── planning.agent.md
├── .vscode/
│   └── mcp.json                   # MCP サーバー設定
│   └── skills/                    # SKILL.md フォルダ — 自己完結型スキル
│       └── code-review/
│           └── SKILL.md
```

## 使い方

### .github/prompts/ — スラッシュコマンド
`.github/prompts/` に置いた `.prompt.md` は VS Code が自動検出する。
Copilot Chat で `/code-review` `/write-tests` のように呼び出せる。

### .github/instructions/ — 自動適用される指示
`.instructions.md` の frontmatter で `applyTo` を指定すると、
そのパターンに一致するファイルを編集する際に Copilot が自動で参照する。

```markdown
---
applyTo: '**/*.ts'
---
# TypeScript コーディング規約
...
```

### .github/agents/ — カスタムエージェント
`.github/agents/` に置いた `.agent.md` は VS Code が自動検出する。

### .github/skills/ — スキルパッケージ
`SKILL.md` を含むフォルダ単位で管理。
Copilot CLI や awesome-copilot 形式のスキルとして利用できる。

### .github/copilot-instructions.md — リポジトリ共通指示
リポジトリのルートに置くと、`applyTo` によるフィルタリングなしに**常時**参照される。
GitHub.com 上の Copilot にも反映されるため、チーム共通のルールを書くのに適している。

```markdown
# プロジェクト共通指示
- 日本語で回答する
- シンプルで読みやすいコードを優先する
```

### .vscode/mcp.json — MCP サーバー設定
MCP（Model Context Protocol）サーバーを登録することで、ファイルシステムや外部 API などの
カスタムツールを Copilot に接続できる。

```json
{
  "servers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${env:GITHUB_TOKEN}" }
    }
  }
}
```
