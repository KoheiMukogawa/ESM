# PROGRESS — ESM 進捗ログ

各フェーズ/作業で「何をやったか・なぜそう決めたか・次は何か」を記録する。**新しい作業ほど上に追記**。日付は絶対表記（例: 2026-06-20）。プロジェクト全体の指針は `CLAUDE.md` 参照。

凡例：✅ 完了 / 🚧 進行中 / ⏭ 次の予定

---

## Phase 0 — Claude APIプロキシ疎通　✅ 完了（2026-06-20）

**目的**：Claude APIキーを安全に隠したまま、フロント（GitHub Pages）からClaudeを呼べる土台を作る。設計の核＝「秘密キーを静的フロントに置かず、サーバ側の薄いプロキシに隠す」。

**やったこと**
- Firebase CLI 導入、`firebase login`、Blazeプランへ変更、Anthropicクレジット $5 購入。
- Cloud Functions（2nd gen / Node20 / asia-northeast1）に薄いプロキシ `ai` を実装・デプロイ。
  - Firebase IDトークン検証 → `ALLOWED_UIDS`（本人＋弟）で認可 → Claude `messages.stream` → SSEで中継。
  - CORSは `koheimukogawa.github.io` ＋ localhost のみ。既定モデル `claude-opus-4-8`、max_tokens上限8000。
- `ANTHROPIC_API_KEY` を Secret Manager に格納（フロント／リポジトリには置かない）。
- `index.html` に全AI機能の共通入口 `callAI()` を追加。
- ブラウザ実機でフル疎通確認（ログイン → プロキシ → Claudeの応答がストリーミング表示）。
- Artifact Registry の自動クリーンアップポリシー設定（保管料対策、1日より古いイメージを削除）。
- 初期セキュリティ点検：Firestoreルールが `users/{userId}` を本人のみ read/write になっていることを確認（安全・テストモードではない）。
- Phase 0 一式を commit `1666233` で `main` にpush。

**詰まり＆対処（再発時の参考）**
- 初回デプロイで `iam.serviceaccounts.actAs` の403 → IAM伝播待ちの一過性。1〜2分後に再デプロイで成功。
- `secrets:set` を `!`（非対話シェル）で実行 → 失敗。**実ターミナル**で実行する。
- 最初に誤ってプレースホルダ `sk-ant-…`（三点リーダ入り）を登録 → `not a legal HTTP header value`。**フルの正規キーを1行で**登録し直して解決。
- gitのpush認証が未設定（過去はGitHub Web編集）→ classic PAT（repoスコープ）で `git push`。

**決定事項**
- ホスティングは当面 GitHub Pages のまま。非公開化は実害が薄く（リポジトリに秘密は無い）、むしろサイトが落ちるので見送り。やるなら Firebase Hosting 移行後。
- 許可ユーザーは当面2名のみ（課金保護）。

⏭ **次：Phase 1（③添削）** — 既存の草案編集UIに `callAI()` を薄く接続。素材庫・草案に溶け込ませ、"機能のための機能"にしない（設計原則）。

---

## ドキュメント整備　✅ 完了（2026-06-21）
- `CLAUDE.md`（プロジェクト指針）と本 `PROGRESS.md`（進捗ログ）を作成。今後のセッションはまず CLAUDE.md を読んで文脈を引き継ぐ。
