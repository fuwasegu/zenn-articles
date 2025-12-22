# 記事アウトライン

## タイトル
5年間 Laravel を使って辿り着いた，「なんちゃってクリーンアーキテクチャ」からのステップアップ

## 参考記事
- https://zenn.dev/mpyw/articles/ce7d09eb6d8117 （「なんちゃって」の元ネタ）

## 題材
稼働管理システム API
- **Project**: プロジェクト
- **Member**: メンバー
- **Assignment**: アサイン（メンバーをプロジェクトに割り当て、稼働率を管理）

---

## 目次構成

```
# 5年間 Laravel を使って辿り着いた，「なんちゃってクリーンアーキテクチャ」からのステップアップ

## はじめに ✅
## なんちゃってクリーンアーキテクチャを振り返る ✅

# モジュール分割と Contract
## 想定アプリケーション ✅
## モジュールとは ✅
## Contract とは ← Illuminate の例
## 今回のモジュール分割 ← Module レベルのディレクトリ構造
## モジュール間の依存関係 ← Mermaid 図
## deptrac による静的検証

# 4層アーキテクチャ
## 依存方向
## 各層に置くクラス
## ディレクトリ構造（Module 内）

# 実装パターン集（ダイジェスト）
## 型安全な ID（HasId トレイト）
## トランザクションの抽象化
## 例外設計（DomainException / TechnicalException / ModuleException）
## 複合モジュール
## PHP 8.4 の Property Hooks を活用した Entity

# Q&A: よくある疑問
- Q. Eloquent のリレーション使っちゃダメなの？
- Q. Collection は Domain 層で使えないの？
- Q. ここまでやる必要ある？

# おわりに
- 「なんちゃって」は悪くない、でもその先がある
- 詳細は Book で
```

---

## メモファイル一覧

| ファイル | 内容 |
|---------|------|
| `01-module-structure.md` | モジュール分割、Contract 層、deptrac |
| `02-four-layers.md` | 4層アーキテクチャの詳細 |
| `03-hasid.md` | 型安全な ID（HasId トレイト） |
| `04-transaction.md` | トランザクションの抽象化 |
| `05-exception.md` | 例外設計（ModuleException 含む） |
| `06-snapshot.md` | 複合モジュールと Snapshot |
| `07-property-hooks.md` | PHP 8.4 Property Hooks |
| `08-qa.md` | Q&A |

