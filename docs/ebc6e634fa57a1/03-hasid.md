# 型安全な ID（HasId トレイト）

## 目的

ID を単なる `string` ではなく、型で区別することで「MemberId を渡すべき場所に ProjectId を渡す」ミスを静的解析で検出する。

---

## なぜ ID 系 VO は「同一値 = 同一インスタンス」であるべきか

DDD における ValueObject の同一性は「値が同じなら等しい」。
しかし PHP では、同じ値を持つオブジェクトでも `===` で比較すると `false` になる：

```php
$id1 = new MemberId('abc-123');
$id2 = new MemberId('abc-123');

$id1 == $id2;   // true（値の比較）
$id1 === $id2;  // false（インスタンスの比較）😱
```

これが問題になる場面：
- 配列のキー
- `in_array(..., strict: true)`
- `array_unique(..., SORT_REGULAR)` など

**Flyweight パターン**を適用すると、同じ値に対して常に同一インスタンスが返されるため、`===` で正しく比較できる：

```php
$id1 = MemberId::create('abc-123');
$id2 = MemberId::create('abc-123');

$id1 === $id2;  // true ✅
```

---

## mpyw/sharable-value-objects

Flyweight パターンを簡単に実現するパッケージ。

```bash
composer require mpyw/sharable-value-objects
```

- 内部で `WeakReference` プールを使用
- 参照がなくなったインスタンスは自動的に GC される
- メモリリークの心配なし

GitHub: https://github.com/mpyw/sharable-value-objects

---

## HasId トレイトの実装

```php
// modules/Contract/Kernel/Id/HasId.php
namespace Modules\Contract\Kernel\Id;

use Mpyw\SharableValueObjects\SharableString;

trait HasId
{
    use SharableString;

    /**
     * 同じ値には同一インスタンスを返す（WeakReference プール利用）
     */
    public static function create(string $id): static
    {
        return static::acquire($id);
    }

    public static function createOrNull(?string $id): ?static
    {
        return $id !== null ? static::create($id) : null;
    }

    public function unwrap(): string
    {
        return $this->getOriginalValue();
    }
}
```

---

## ID クラスの定義

各モジュールの Contract 層に、ドメイン固有の ID クラスを定義：

```php
// modules/Contract/Member/Identities/MemberId.php
namespace Modules\Contract\Member\Identities;

use Modules\Contract\Kernel\Id\HasId;

final class MemberId
{
    use HasId;
}
```

```php
// modules/Contract/Project/Identities/ProjectId.php
namespace Modules\Contract\Project\Identities;

use Modules\Contract\Kernel\Id\HasId;

final class ProjectId
{
    use HasId;
}
```

---

## 使用例

```php
// Application 層: string → VO 変換
$memberIds = Collection::make($input->memberIds)
    ->map(fn (string $v) => MemberId::create($v))
    ->all();

// Infrastructure 層: Eloquent Model → Entity 変換
return new Assignment(
    id: AssignmentId::create($model->id),
    memberId: MemberId::create($model->member_id),
    projectId: ProjectId::create($model->project_id),
);
```

---

## アンチパターン

```php
// ❌ NG: __toString() を実装する（暗黙変換の事故を防ぐ）
final class MemberId
{
    use HasId;
    
    public function __toString(): string
    {
        return $this->unwrap();  // SQL に直接埋め込まれる事故が起きる
    }
}

// ❌ NG: new で直接インスタンス化（Flyweight が効かない）
$id = new MemberId('abc-123');

// ✅ OK: create() 経由でのみ生成
$id = MemberId::create('abc-123');
```

---

## ポイントまとめ

- `create()` を唯一のエントリポイントに（コンストラクタは trait が private にする）
- `__toString()` は実装しない（暗黙変換の事故を防ぐ）
- Flyweight パターンで `===` 比較が可能
- `WeakReference` プールでメモリリークなし

