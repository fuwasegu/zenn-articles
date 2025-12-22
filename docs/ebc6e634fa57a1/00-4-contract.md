# Contract とは

## 一言で

Contract とは，モジュール間の「公開 API」です．

他のモジュールが参照していいのは Contract に定義されたものだけ．
逆に言えば，Contract に定義されていないもの（Domain 層の Entity 実装，Infrastructure 層の Repository 実装など）は内部実装として隠蔽されます．

---

## Laravel Illuminate に学ぶ

実は Laravel 自身がこの構造を採用しています．

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

Laravel を使うアプリケーション側は `Contracts` に定義された Interface に依存します．
実装が変わっても，Interface が変わらなければアプリケーションに影響しません．

本記事で紹介するモジュール構造も，この考え方を踏襲しています．

---

## Contract に配置するもの

| 種類 | 例 | 用途 |
|-----|-----|------|
| **ID（ValueObject）** | `MemberId`, `ProjectId` | 型安全な識別子 |
| **Entity Interface** | `MemberInterface` | Entity の読み取り専用契約 |
| **Service Interface** | `MemberQueryService` | 他モジュールに公開するサービスの契約 |
| **Exception** | `MemberNotFoundException` | ドメイン例外 |
| **Enum** | `ProjectType` | 共有される列挙型 |

---

## ディレクトリ構造

```
modules/Contract/
├── Kernel/                  # SharedKernel（全モジュール共通）
│   ├── Id/
│   │   └── HasId.php        # ID 用 trait
│   ├── Exception/
│   │   ├── DomainException.php
│   │   └── TechnicalException.php
│   └── Pagination/
│       └── Paginator.php
│
├── Member/                  # Member モジュールの契約
│   ├── Identities/
│   │   └── MemberId.php
│   ├── Entities/
│   │   └── MemberInterface.php
│   ├── Services/
│   │   └── MemberQueryService.php
│   └── Exceptions/
│       └── MemberNotFoundException.php
│
└── Project/                 # Project モジュールの契約
    ├── Identities/
    │   └── ProjectId.php
    ├── Entities/
    │   └── ProjectInterface.php
    └── Enums/
        └── ProjectType.php
```

---

## なぜ Contract が必要か

「なんちゃって」では，`app/Models/Member.php` は誰でも参照できます．
UseCase も同様に，どこからでも呼べてしまいます．

小規模なら問題ありませんが，チームが大きくなると：

- 「この Model のこのメソッド，勝手に変えていい？」
- 「この UseCase，他のチームも使ってる？」
- → **聞かないと分からない**

**「何が公開 API で，何が内部実装か」が不明確**なので，変更の影響範囲が読めません．

Contract はこの問題を解決します．
**Contract に置いたものだけが公開 API であり，他モジュールが参照していいもの．**
それ以外は内部実装として隠蔽されます．

---

## 記事での書き方（案）

```markdown
## Contract とは

Contract とは，モジュール間の「公開 API」です．

実は Laravel 自身がこの構造を採用しています．`Illuminate\Contracts` には Interface が，`Illuminate\Cache` などには実装が置かれています．

本記事で紹介するモジュール構造も，この考え方を踏襲しています．

### Contract に配置するもの

| 種類 | 例 | 用途 |
|-----|-----|------|
| ID（ValueObject） | `MemberId`, `ProjectId` | 型安全な識別子 |
| Entity Interface | `MemberInterface` | Entity の読み取り専用契約 |
| QueryService Interface | `MemberQueryService` | 読み取り専用サービスの契約 |
| Exception | `MemberNotFoundException` | ドメイン例外 |
| Enum | `ProjectType` | 共有される列挙型 |

### なぜ Contract が必要か

「なんちゃって」では，`app/Models/Member.php` は誰でも参照できます．
小規模なら問題ありませんが，チームが大きくなると「何が公開 API で，何が内部実装か」が不明確になります．

Contract を導入すると：

- **Contract に置いたものだけが公開 API**
- それ以外は内部実装として隠蔽
- 変更の影響範囲が明確になる
```

