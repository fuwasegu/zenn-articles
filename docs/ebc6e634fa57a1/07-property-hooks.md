# PHP 8.4 の Property Hooks を活用した Entity 設計

## TL;DR

- Interface は「public に読める」を契約（`{ get; }`）
- Class は `readonly` で不変を保証
- 非対称可視性（`private(set)`）はクラス側でのみ使用

---

## Interface（契約）

```php
// modules/Contract/Assignment/Entities/AssignmentInterface.php
namespace Modules\Contract\Assignment\Entities;

use Modules\Contract\Assignment\Identities\AssignmentId;
use Modules\Contract\Member\Identities\MemberId;
use Modules\Contract\Project\Identities\ProjectId;

interface AssignmentInterface
{
    public AssignmentId $id { get; }
    public MemberId $memberId { get; }
    public ProjectId $projectId { get; }
    public Workload $workload { get; }
}
```

- `{ get; }` で「public から get できる」だけを約束
- set の契約は課さない（実装側が決める）
- **`private(set)` は Interface に持ち込まない**（Fatal Error になる）

---

## 実装（保証）

```php
// modules/Assignment/Domain/Assignment.php
namespace Modules\Assignment\Domain;

use Modules\Contract\Assignment\Entities\AssignmentInterface;

final class Assignment implements AssignmentInterface
{
    public function __construct(
        public readonly AssignmentId $id,
        public readonly MemberId $memberId,
        public readonly ProjectId $projectId,
        public readonly Workload $workload,
    ) {}

    public static function create(
        AssignmentId $id,
        MemberId $memberId,
        ProjectId $projectId,
        Workload $workload,
    ): self {
        // バリデーション、ドメインイベント発行など
        return new self($id, $memberId, $projectId, $workload);
    }
}
```

- `readonly` で不変を保証
- Mockery が `readonly class` をモックできないため、`readonly property` を推奨

---

## 非対称可視性（クラス側のみ）

内部で変更が必要な場合は非対称可視性を使用：

```php
final class Token implements TokenInterface
{
    public private(set) string $value { get; }

    public function __construct(string $seed)
    {
        $this->value = $seed;
    }

    public function rotate(string $next): void
    {
        $this->value = $next;  // クラス内部では書き込み可
    }
}
```

- `readonly` だと内部からも変更不可
- 外部からは読み取り専用、内部では変更可能

---

## Property Hooks での正規化

```php
final class Member implements MemberInterface
{
    public readonly MemberId $id;

    public string $name {
        get { return $this->name; }
        set(string $value) { $this->name = trim($value); }  // 正規化
    }

    public function __construct(MemberId $id, string $name)
    {
        $this->id = $id;
        $this->name = $name;  // set hook が呼ばれる
    }
}
```

---

## 設計原則

| 原則 | 説明 |
|-----|------|
| **契約は最小** | Interface は `{ get; }` のみ |
| **保証は実装強め** | `readonly` を積極利用 |
| **振る舞いは Hook** | 正規化・バリデーション・派生値 |
| **可視性はクラス調整** | 外部書き込み制限 |

---

## よくあるエラー

| エラー | 原因 |
|-------|------|
| `Fatal Error` | Interface に非対称可視性（`private(set)`）を書いた |
| `Error` | readonly プロパティに set Hook を書いた |
| 無限ループ | backed property で `$this->prop` を使わなかった |

---

## ゲッターメソッド不要のメリット

```php
// Before: ゲッターメソッドが必要
interface MemberInterface
{
    public function getId(): MemberId;
    public function getName(): string;
}

// After: Property Hooks でシンプルに
interface MemberInterface
{
    public MemberId $id { get; }
    public string $name { get; }
}
```

コード量が減り、IDE の補完も効きやすくなる。

