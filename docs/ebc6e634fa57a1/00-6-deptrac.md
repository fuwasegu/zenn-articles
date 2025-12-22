# deptrac を使った静的解析

## deptrac とは

deptrac は、PHP プロジェクトの依存関係を静的解析するツールです。

https://github.com/qossmic/deptrac

「このレイヤー/モジュールは、このレイヤー/モジュールにのみ依存してよい」というルールを YAML で定義し、違反があれば検出してくれます。

---

## 静的解析できると何が嬉しいか

### 1. レビュー負荷の軽減

AI（GitHub Copilot, Cursor, Claude など）が大量のコードを生成する時代になりました。
生成されたコードが正しい依存関係を守っているか、目視で確認するのは大変です。

deptrac を CI に組み込めば、**依存違反は自動で検出**されます。
レビューで確認すべき項目が減り、ビジネスロジックに集中できます。

### 2. ルールの明文化

「このモジュールはあのモジュールを参照しちゃダメ」というルールを YAML に書くことで、チーム全員が同じ認識を持てます。
口頭やドキュメントで伝えるより確実です。

### 3. リファクタリングの安心感

モジュール構造を変更するとき、「どこかで依存が壊れてないか」を deptrac が検証してくれます。

---

## 設定ファイル構成

```
deptrac/
├── module.yaml         # モジュール間の依存ルール
└── contract.yaml       # Contract 内部の依存ルール
```

モジュール間の依存と、Contract 内部の依存は関心事が異なるため、設定ファイルを分けておくと管理しやすくなります。

---

## 設定例 1: module.yaml（モジュール間）

```yaml
# deptrac/module.yaml
deptrac:
  paths:
    - ./modules

  layers:
    - name: Contract
      collectors:
        - type: directory
          value: modules/Contract/.*
    - name: Member
      collectors:
        - type: directory
          value: modules/Member/.*
    - name: Project
      collectors:
        - type: directory
          value: modules/Project/.*
    - name: Assignment
      collectors:
        - type: directory
          value: modules/Assignment/.*
    - name: MemberSearch
      collectors:
        - type: directory
          value: modules/MemberSearch/.*

  ruleset:
    # Contract は自身のみに依存
    Contract:
      - Contract

    # 基本モジュールは Contract + 自身に依存
    Member:
      - Contract
      - Member
    Project:
      - Contract
      - Project
    Assignment:
      - Contract
      - Assignment

    # 複合モジュールも Contract + 自身に依存
    MemberSearch:
      - Contract
      - MemberSearch
```

### ポイント

- **自分自身も入れる**: `Member: [Contract, Member]` のように、自身への依存も明示
- **他モジュールの内部実装は参照しない**: Member が Project の内部実装を直接触るのは NG

---

## 設定例 2: contract.yaml（Contract 内部）

```yaml
# deptrac/contract.yaml
deptrac:
  paths:
    - ./modules/Contract

  layers:
    - name: Kernel
      collectors:
        - type: directory
          value: modules/Contract/Kernel/.*
    - name: Member
      collectors:
        - type: directory
          value: modules/Contract/Member/.*
    - name: Project
      collectors:
        - type: directory
          value: modules/Contract/Project/.*
    - name: Assignment
      collectors:
        - type: directory
          value: modules/Contract/Assignment/.*

  ruleset:
    # Kernel は自身のみ（他に依存しない）
    Kernel:
      - Kernel

    # Member は Kernel + 自身に依存
    Member:
      - Kernel
      - Member

    # Project は MemberId（managerId など）を参照するので Member に依存
    Project:
      - Kernel
      - Project
      - Member

    # Assignment は Member と Project の ID を参照するので依存を許可
    Assignment:
      - Kernel
      - Assignment
      - Member
      - Project
```

### ポイント

- **Kernel は他に依存しない**: 最も安定したレイヤー（SharedKernel）
- **各 Contract は Kernel に依存**: 共通型（HasId, DomainException など）を使う
- **関連エンティティへの依存は明示**: Project が `MemberId`（managerId など）を持つなら Member への依存を許可

---

## 実行方法

```bash
# インストール
composer require --dev qossmic/deptrac

# 実行
./vendor/bin/deptrac analyze deptrac.module.yaml
```

違反があればエラーが出力されます。CI に組み込んで、PR ごとにチェックするのがおすすめです。

---

## 記事での書き方（案）

```markdown
## deptrac を使った静的解析

### deptrac とは

deptrac は、PHP プロジェクトの依存関係を静的解析するツールです。

https://github.com/qossmic/deptrac

「このモジュールは、このモジュールにのみ依存してよい」というルールを YAML で定義し、違反があれば検出してくれます。

### なぜ静的解析が必要か

AI が大量のコードを生成する時代になりました。
生成されたコードが正しい依存関係を守っているか、目視で確認するのは大変です。

deptrac を CI に組み込めば、**依存違反は自動で検出**されます。
レビューで確認すべき項目が減り、ビジネスロジックに集中できます。

### 設定例

（yaml）

### ポイント

- **Contract は他に依存しない**: 最も安定したレイヤー
- **各モジュールは Contract のみに依存**: 他モジュールの内部実装を直接参照しない
```

