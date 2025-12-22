# 複合モジュールと Snapshot

## なぜ Entity の new は他モジュールでできないのか

### 問題

Entity は Domain 層に配置され、そのモジュール内でのみ生成される。

```php
// modules/Member/Domain/Member.php
namespace Modules\Member\Domain;

final class Member
{
    private function __construct(
        public readonly MemberId $id,
        public readonly string $name,
        public readonly Email $email,
    ) {}

    public static function create(...): self
    {
        // バリデーション、ドメインイベント発行など
        return new self(...);
    }
}
```

他モジュール（例: WorkloadOverview）が Member Entity を直接参照すると：

1. **依存方向の違反**: `WorkloadOverview → Member` という依存が生まれる
2. **境界の崩壊**: Member モジュールの内部実装に依存してしまう
3. **deptrac 違反**: モジュール間の直接参照は禁止

### 解決策: Snapshot

Entity の「読み取り専用ビュー」として Snapshot を Contract 層に公開する。

---

## Snapshot とは

- **読み取り専用**の DTO（Data Transfer Object）
- Contract 層に配置 → 他モジュールから参照可能
- Entity の状態を「スナップショット」として切り出したもの

```php
// modules/Contract/Member/Snapshots/MemberSnapshot.php
namespace Modules\Contract\Member\Snapshots;

use Modules\Contract\Member\Identities\MemberId;

final readonly class MemberSnapshot
{
    public function __construct(
        public MemberId $id,
        public string $name,
        public string $email,
    ) {}
}
```

---

## Entity vs Snapshot vs View

| 概念 | 配置 | 用途 | 他モジュールから |
|-----|------|------|----------------|
| **Entity** | Domain 層 | ビジネスロジックを持つ | ❌ 参照不可 |
| **Snapshot** | Contract 層 | 読み取り専用の公開データ | ✅ 参照可能 |
| **View** | Application 層 | UseCase の出力 DTO | ❌ 参照不可（モジュール内部） |

### Snapshot と View の違い

- **Snapshot**: 他モジュールへ公開するための DTO（Contract 層）
- **View**: 同一モジュール内の UseCase 出力（Application 層）

```php
// View: UseCase の出力（モジュール内部）
// modules/Member/Application/MemberView.php
final readonly class MemberView
{
    public function __construct(
        public string $id,      // string として返す（Presentation 層向け）
        public string $name,
        public string $email,
    ) {}

    public static function from(Member $member): self
    {
        return new self(
            id: $member->id->unwrap(),
            name: $member->name,
            email: $member->email->value(),
        );
    }
}
```

---

## QueryService と Snapshot

他モジュールからは QueryService 経由で Snapshot を取得：

```php
// modules/Contract/Member/Services/MemberQueryService.php
namespace Modules\Contract\Member\Services;

interface MemberQueryService
{
    public function findById(MemberId $id): MemberSnapshot;
    
    /** @return iterable<MemberSnapshot> */
    public function findByIds(array $memberIds): iterable;
}
```

実装は Infrastructure 層に：

```php
// modules/Member/Infrastructure/MemberQueryService.php
namespace Modules\Member\Infrastructure;

final readonly class MemberQueryService implements MemberQueryServiceContract
{
    public function findById(MemberId $id): MemberSnapshot
    {
        $model = MemberModel::find($id->unwrap());
        
        return new MemberSnapshot(
            id: MemberId::create($model->id),
            name: $model->name,
            email: $model->email,
        );
    }
}
```

---

## 複合モジュール（読み取り専用・横断）

複数のドメインを横断して読み取る場合、専用の複合モジュールを作成。

### 特徴

- **Domain 層を持たない**（ビジネスルールは各基本モジュールに）
- **Contract に公開しない**（他モジュールから参照されない）
- 他モジュールの `Snapshot` を組み合わせて返す

### ディレクトリ構造

```
modules/WorkloadOverview/        # 稼働率一覧を返す
├── ModuleServiceProvider.php
├── Presentation/
│   └── WorkloadOverviewController.php
├── Application/
│   ├── GetWorkloadOverviewUseCase.php
│   ├── GetWorkloadOverviewInteractor.php
│   └── WorkloadOverviewView.php
└── Infrastructure/
    └── WorkloadOverviewQueryService.php
```

### 実装例

```php
// modules/WorkloadOverview/Application/GetWorkloadOverviewInteractor.php
final readonly class GetWorkloadOverviewInteractor implements GetWorkloadOverviewUseCase
{
    public function __construct(
        private MemberQueryService $memberQuery,      // Contract 経由
        private ProjectQueryService $projectQuery,    // Contract 経由
        private AssignmentQueryService $assignmentQuery,
    ) {}

    public function handle(GetWorkloadOverviewInput $input): WorkloadOverviewView
    {
        $members = $this->memberQuery->findByIds($input->memberIds);
        $assignments = $this->assignmentQuery->findByMemberIds($input->memberIds);
        
        // Snapshot を組み合わせて View を構築
        return WorkloadOverviewView::from($members, $assignments);
    }
}
```

---

## ポイントまとめ

- **Entity の new は他モジュールでできない** → Snapshot で読み取り専用ビューを公開
- **Snapshot は Contract 層** → 他モジュールから参照可能
- **View は Application 層** → モジュール内部の UseCase 出力
- **複合モジュール**: Domain 層なし、他モジュールの Snapshot を組み合わせる
- **QueryService**: Snapshot を返す読み取り専用サービス

