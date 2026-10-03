# My Study

仕事や勉強で気になった技術的な疑問を、AIとの壁打ちで学ぶためのリポジトリです。疑問、整理した回答、復習クイズを一緒に残します。

```text
YYYY-MM-DD/
  topic-slug/
    Question.md
    Answer.md
    Quiz.md
templates/
.agents/skills/study-session/SKILL.md
AGENTS.md
```

日付は日本時間の学習開始日です。同じテーマの続きは元の場所に追記します。

## 使い方

AIに疑問をそのまま伝えるか、`templates/Question.md`を日付・話題のディレクトリにコピーして書きます。背景や仮説がなくても、疑問だけで始められます。

- 開始：「Terraformの依存関係について疑問があります。今日の学習として記録して、一緒に考えて」
- 継続：「このQuestion.mdの続きを考えたい。今回はこの挙動を掘り下げて」
- 整理：「ここまでの内容をAnswer.mdに整理して、復習クイズも作って」
- 復習：「このテーマのクイズを1問ずつ出して。解答は回答するまで見せないで」

Question.mdには元の疑問を残し、Answer.mdには根拠と未解決事項をまとめます。Quiz.mdの解答は折りたたみで確認できます。会話の全文保存は必須にせず、後で理解を再現できる要点を残します。

リポジトリ固有のルールは[AGENTS.md](AGENTS.md)、学習の進行手順は[study-session](.agents/skills/study-session/SKILL.md)にあります。Skillが自動検出されない環境でも、AGENTS.mdから参照できます。
