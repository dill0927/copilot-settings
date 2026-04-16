# copilot-settings

GitHub Copilot を使いこなすための個人設定・スキル・カスタムエージェントをまとめたリポジトリ。

## ディレクトリ構成

```
copilot-settings/
├── prompts/        # 再利用可能なプロンプト集 (.prompt.md)
├── agents/         # カスタムエージェント定義 (.chatParticipant.json など)
└── instructions/   # コーディング規約・指示ファイル (.instructions.md)
```

## 使い方

### prompts/
VS Code の `chat.promptFilesLocations` に指定することで、Copilot Chat で `/` スラッシュコマンドとして使えるプロンプトファイルを格納します。

### agents/
カスタムエージェント（Chat Participants）の定義ファイルを格納します。

### instructions/
`.github/copilot-instructions.md` や `.instructions.md` など、Copilot に常時参照させる指示ファイルを格納します。
