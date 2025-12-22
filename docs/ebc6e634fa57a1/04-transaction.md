# トランザクションの抽象化

## 目的

Laravel の `DB::transaction()` を直接使うと、Domain 層や Application 層が Laravel に依存してしまう。
これを Contract 層のインターフェースで抽象化する。

---

## TransactionExecutor インターフェース

```php
// modules/Contract/Kernel/Database/TransactionExecutor.php
namespace Modules\Contract\Kernel\Database;

interface TransactionExecutor
{
    /**
     * @template T
     * @param callable(): T $callback
     * @return T
     */
    public function execute(callable $callback): mixed;
}
```

---

## Laravel 実装

```php
// app/Support/Database/LaravelTransactionExecutor.php
namespace App\Support\Database;

use Illuminate\Support\Facades\DB;
use Modules\Contract\Kernel\Database\TransactionExecutor;

final readonly class LaravelTransactionExecutor implements TransactionExecutor
{
    public function execute(callable $callback): mixed
    {
        return DB::transaction($callback);
    }
}
```

ServiceProvider で bind：

```php
// app/Providers/AppServiceProvider.php
$this->app->bind(TransactionExecutor::class, LaravelTransactionExecutor::class);
```

---

## UseCase での使用

```php
final readonly class StoreAssignmentInteractor implements StoreAssignmentUseCase
{
    public function __construct(
        private TransactionExecutor $transactions,
        private AssignmentRepository $repository,
    ) {}

    public function handle(StoreAssignmentInput $input): AssignmentView
    {
        return $this->transactions->execute(function () use ($input) {
            $assignment = Assignment::create(...);
            $this->repository->save($assignment);
            return AssignmentView::from($assignment);
        });
    }
}
```

---

## DatabaseExceptionHandler との併用

**必ず `exceptionHandler->perform()` が外側、`transactions->execute()` が内側**

```php
return $this->exceptionHandler->perform(function () use ($assignment): void {
    $this->transactions->execute(function () use ($assignment): void {
        $this->repository->save($assignment);
    });
});
```

理由：
- トランザクション内で PDOException が発生した場合、まずロールバックされる
- その後 exceptionHandler がキャッチして、モジュール固有の例外に変換する
- 逆にすると、例外変換後にロールバック処理が走らない可能性がある

