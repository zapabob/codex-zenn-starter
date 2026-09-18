# 『AIなしで手書きできますか？』認知的外骨格 Zenn記事作成 実装ログ

- 日時: 2026-09-19
- 実装AI: Antigravity
- 対象ファイル:
  - `articles/cognitive-exoskeleton-ai-handwriting.md` [新規作成]

---

## 1. 概要 (Overview)
就労支援面談での「AIを使うとしても、手でコードを書けることは必要」という問いかけに対し、20ファイル・約800行のSOP開発統制系、AGENTS.md、MCP（Sequential Thinking/Context7）、Skills、Tests/CIを組み合わせた「認知的外骨格」としてのエンジニアリング実践論を論証したZenn向け新規記事を作成した。

## 2. 背景と要求事項 (Background / Requirements)
- **背景**:
  - 発達障害（ASD/ADHD）によるワーキングメモリや多重タスク保持の認知的障壁を、AIエージェントと統制系ツールチェーンで外部化し、設計・安全性・意味論・検証証拠に集中する実践体制を構築している。
  - 「AIを外した手打ち試験（無支援状態への耐性）」と「管理された運用下で道具を使いこなして価値を生み出す能力（実務遂行力）」の測定設計のズレを提起する。
- **要求事項**:
  - ZennのMarkdown規格（Frontmatter、Slug規則、絵文字、トピック等）に完全準拠。
  - ルールに準拠し、初稿状態では `published: false` とする。
  - 読者目線（Reader-facing tone）で書かれ、不必要な個人攻撃を排し、現代の開発実務と測定設計の一般論として成立させる。
  - 制御構造のレイヤー表やフロー図（Mermaid）を組み込み、技術論・統制系論としての可読性と説得力を最大化する。

## 3. 前提と設計判断 (Assumptions / Decisions)
1. **スラグ名選定**:
   - `cognitive-exoskeleton-ai-handwriting.md` を選定。テーマ（認知的外骨格 × AI手書き批判）が明確で、Zennのスラグ命名規則（半角英数ハイフン）を満たす。
2. **構造の可視化**:
   - 崩れていた表形式をGitHub Flavored Markdown形式の3列テーブル（レイヤー / 工学上の役割 / 認知上の役割）に整形。
   - 人間主導の制御・証拠生成ループをMermaid図（`flowchart TD`）として構造化。
3. **論理の強度向上**:
   - 「AIを使えば誰でもエンジニア」論との明確な峻別（ハルシネーション、API捏造、テスト弱体化、勝手な変更に対する防御線）。
   - 「無支援耐性の測定」と「実務遂行能力の測定」の対比（計器飛行パイロットの比喩、眼鏡・車椅子の機能的アナロジー）。

## 4. 変更ファイル (Changed Files)
- `articles/cognitive-exoskeleton-ai-handwriting.md` (新規追加)
- `_docs/2026-09-19_認知的外骨格Zenn記事作成_Antigravity.md` (新規追加・本ログ)

## 5. 実行したコマンド (Commands Run)
- `pnpm exec zenn list:articles`: 記事が正常に一覧パースされるか検証。

## 6. 検証結果 (Verification Results)
- `zenn list:articles` 出力にて、新規記事がエラーなく正常に認識されたことを確認：
  ```
  cognitive-exoskeleton-ai-handwriting	『AIなしで手書きできますか？』は何を測っているのか――AGENTS.md・MCP・Skillsでつくる認知的外骨格
  ```
- Frontmatter構文、Markdown構文、テーブル構造、Mermaidブロックが正常であることを確認。

## 7. 残余リスク (Residual Risks)
- なし（ユーザー承認のもと `published: true` に設定し、GitHubへプッシュ公開）。

## 8. 推奨される次のアクション (Recommended Next Actions)
1. GitHub Actions / Zenn側での自動デプロイ完了を確認する。
2. 公開後の記事URL（zenn.dev/zapabob/articles/cognitive-exoskeleton-ai-handwriting）での表示を確認する。
