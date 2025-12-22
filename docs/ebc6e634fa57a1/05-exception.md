# 例外設計

## 2つのカテゴリ

| カテゴリ | 意味 | HTTPステータス | 例 |
|---------|------|---------------|-----|
| **DomainException** | ビジネスルール違反 | 400系 | `AssignmentNotFoundException` |
| **TechnicalException** | 技術的障害 | 500系 | `ConnectionException` |

---

## 基底クラス（Contract/Kernel）

```php
// modules/Contract/Kernel/Exception/DomainException.php
namespace Modules\Contract\Kernel\Exception;

abstract class DomainException extends \Exception {}
```

```php
// modules/Contract/Kernel/Exception/TechnicalException.php
namespace Modules\Contract\Kernel\Exception;

abstract class TechnicalException extends \RuntimeException {}
```

```php
// modules/Contract/Kernel/Exception/NotFoundException.php
namespace Modules\Contract\Kernel\Exception;

abstract class NotFoundException extends DomainException {}
```

---

## ModuleException マーカーインターフェース

### なぜ必要か

1. **例外の出所を明確にする**: どのモジュールから発生した例外か判別可能
2. **ハンドリングの粒度**: モジュール単位で例外を catch できる
3. **境界の明示**: 他モジュールの例外を直接 catch することを抑止

### 実装

```php
// modules/Contract/Kernel/Exception/ModuleException.php
namespace Modules\Contract\Kernel\Exception;

/**
 * 各モジュールの例外が実装するマーカーインターフェース
 */
interface ModuleException {}
```

各モジュールにマーカーインターフェースを定義：

```php
// modules/Contract/Assignment/Exceptions/AssignmentModuleException.php
namespace Modules\Contract\Assignment\Exceptions;

use Modules\Contract\Kernel\Exception\ModuleException;

interface AssignmentModuleException extends ModuleException {}
```

モジュール固有の例外がマーカーを実装：

```php
// modules/Contract/Assignment/Exceptions/AssignmentNotFoundException.php
namespace Modules\Contract\Assignment\Exceptions;

use Modules\Contract\Kernel\Exception\NotFoundException;

final class AssignmentNotFoundException extends NotFoundException 
    implements AssignmentModuleException
{
    public function __construct()
    {
        parent::__construct('Assignment not found.');
    }
}
```

### 例外の階層構造

```
Throwable
├── Exception
│   └── DomainException (抽象)
│       └── NotFoundException (抽象)
│           └── AssignmentNotFoundException
│               ├─ extends NotFoundException
│               └─ implements AssignmentModuleException
└── RuntimeException
    └── TechnicalException (抽象)
        └── ConnectionException (抽象)
            └── AssignmentConnectionException
                ├─ extends ConnectionException
                └─ implements AssignmentModuleException
```

---

## HTTP レスポンスへの変換

`bootstrap/app.php` で統一的に変換：

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->render(function (NotFoundException $e) {
        return response()->json(['message' => $e->getMessage()], 404);
    });
    
    $exceptions->render(function (DomainException $e) {
        return response()->json(['message' => $e->getMessage()], 400);
    });
    
    $exceptions->render(function (TechnicalException $e) {
        // ログには詳細を残すが、ユーザーには汎用メッセージ
        Log::error($e->getMessage(), ['exception' => $e]);
        return response()->json(['message' => 'Internal server error.'], 500);
    });
})
```

---

## DatabaseExceptionHandler

Repository で発生する PDOException をモジュール固有の例外に変換：

```php
// modules/Assignment/Infrastructure/AssignmentDatabaseExceptionHandler.php
namespace Modules\Assignment\Infrastructure;

use Modules\Contract\Assignment\Exceptions\AssignmentConnectionException;

final readonly class AssignmentDatabaseExceptionHandler
{
    public function __construct(
        private AssignmentConnectionException $connectionException,
    ) {}

    /**
     * @template T
     * @param callable(): T $callback
     * @return T
     */
    public function perform(callable $callback): mixed
    {
        try {
            return $callback();
        } catch (\PDOException $e) {
            throw $this->connectionException;
        }
    }
}
```

---

## ポイントまとめ

- **DomainException**: ビジネスルール違反 → 400系
- **TechnicalException**: 技術的障害 → 500系
- **ModuleException**: マーカーIF で例外の出所を明示
- **DatabaseExceptionHandler**: PDOException をモジュール固有例外に変換
- **bootstrap/app.php**: 例外クラスごとに HTTP レスポンスを統一変換

