# モジュール分割と Contract 層

## なぜ app/ の中だけでは破綻するのか

「なんちゃって」では `app/UseCases` に UseCase を、`app/Models` に Eloquent Model を置く。
小〜中規模では十分に機能するが、プロジェクトが成長すると以下の問題が顕在化する。

---

### 問題 1: UseCase の境界が曖昧

```
app/UseCases/
├── Member/
│   ├── StoreAction.php
│   └── UpdateAction.php
├── Project/
│   ├── StoreAction.php
│   ├── UpdateAction.php
│   └── AssignMemberAction.php  # これは Project? Assignment?
└── Report/
    └── WorkloadReportAction.php  # Member と Project 両方使う
```

- `AssignMemberAction` は Project の機能？それとも Assignment という独立したドメイン？
- `WorkloadReportAction` は複数のドメインを横断している。どこに置く？
- **結果**: ディレクトリ構成がチームメンバーによってブレる。「正解」がない。

---

### 問題 2: 複数チーム開発時の衝突

Member チームと Project チームが同時に開発しているとき：

- 両チームが `app/Models/Member.php` を触る
- 両チームが `app/UseCases/` 配下にファイルを追加する
- **Git のコンフリクトが頻発**

「なんちゃって」はファイル数が少ないのがメリットだが、
複数チームで触ると**そのメリットがデメリットに反転**する。

---

### まとめ: 「境界」がないことが問題

Eloquent Model 自体は、クエリビルダの入口として使うだけなら大きな問題にはならない。

本質的な問題は**「ドメイン境界がコード上で表現されていない」**こと。

- どこからどこまでが Member の責務か？
- Project チームは Member の何を使っていいのか？
- 変更したら誰に影響する？

これらが**暗黙知**になっている状態が、大規模開発では破綻する原因になる。

---

## modules/ ディレクトリ構造

```
modules/
├── Contract/                    # モジュール間の公開API
│   ├── Kernel/                  # 共通型（ID, Exception, Pagination）
│   ├── Member/                  # Memberモジュールの契約
│   ├── Project/                 # Projectモジュールの契約
│   └── Assignment/              # Assignmentモジュールの契約
│
├── Member/                      # 基本モジュール
│   ├── Presentation/
│   ├── Application/
│   ├── Domain/
│   └── Infrastructure/
│
├── Project/                     # 基本モジュール
│   └── ...
│
├── Assignment/                  # 基本モジュール
│   └── ...
│
└── WorkloadOverview/            # 複合モジュール（読み取り専用）
    ├── Presentation/
    ├── Application/
    └── Infrastructure/
```

---

## Contract 層 ─ モジュール間の「公開API」e

### Laravel Illuminate の構造に学ぶ

実は Laravel 自身が、この「Contract と実装を分離する」構造を採用している。

```
vendor/laravel/framework/src/Illuminate/
├── Contracts/           # Interface（公開 API）
│   ├── Cache/
│   │   └── Repository.php
│   ├── Queue/
│   │   └── Queue.php
│   └── ...
├── Cache/               # 実装
│   └── Repository.php
├── Queue/               # 実装
│   └── Queue.php
└── ...
```

- `Illuminate\Contracts\Cache\Repository` → Interface（契約）
- `Illuminate\Cache\Repository` → 実装

Laravel を使うアプリケーション側は、`Contracts` に定義された Interface に依存する。
実装が変わっても、Interface が変わらなければアプリケーションに影響しない。

本記事で紹介するモジュール構造も、この考え方を踏襲している。

---

### なぜ Contract が必要か

「なんちゃって」では、`app/Models/Member.php` は誰でも参照できる。
UseCase も同様に、どこからでも呼べてしまう。

これ自体は小規模なら問題ない。しかし、チームが大きくなると：

- 「この Model のこのメソッド、勝手に変えていい？」
- 「この UseCase、他のチームも使ってる？」
- → **聞かないと分からない**

**「何が公開 API で、何が内部実装か」が不明確**なので、変更の影響範囲が読めない。

Contract 層は、この問題を解決する。
**Contract に置いたものだけが「公開 API」であり、他モジュールが参照していいもの。**
それ以外（Domain 層の Entity、Infrastructure 層の実装など）は内部実装として隠蔽される。

---

### Contract に配置するもの

Contract には以下を配置：

| 種類 | 例 | 用途 |
|-----|-----|------|
| Interface | `MemberQueryService` | 他モジュールが参照できる振る舞い |
| ValueObject | `MemberId`, `ProjectId` | 型安全な識別子 |
| Enum | `AssignmentStatus` | 共有される列挙型 |
| Exception | `MemberNotFoundException` | ドメイン例外 |
| Snapshot | `MemberSnapshot` | 読み取り専用のデータ構造 |

```php
// modules/Contract/Member/Identities/MemberId.php
final class MemberId
{
    use HasId;
}

// modules/Contract/Member/Services/MemberQueryService.php
interface MemberQueryService
{
    public function findById(MemberId $id): MemberSnapshot;
}
```

---

## deptrac で依存関係を静的検証

[deptrac](https://github.com/qossmic/deptrac) を使って、モジュール間の依存違反を CI で検出。

```yaml
# deptrac-modules.yaml
deptrac:
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

  ruleset:
    Member:
      - Contract
    Project:
      - Contract
    # Member → Project は NG（Contract 経由でのみ通信）
```

```bash
vendor/bin/deptrac analyse --config-file=deptrac-modules.yaml
```

