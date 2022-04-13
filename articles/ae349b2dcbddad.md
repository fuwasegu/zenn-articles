---
title: "Eloquent に寄せたなんちゃって Entity でレイヤードアーキテクチャを楽する"
emoji: "🥱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Laravel", "PHP"]
published: true
---
# はじめに
先日，**PHPerKaigi 2022** に登壇させていただきました．ご清聴いただいた皆さま，ありがとうございました！
https://fortee.jp/phperkaigi-2022/proposal/7d7503c6-b152-40c5-8d51-e24145c522ef

また，登壇に使用したスライドも公開してありますのでぜひご覧ください．
@[speakerdeck](fee8e92dafea4b5d80bcfc12a352781f)

# 登壇内容をふりかえる
さて，今回の発表では主に以下のことについてお話しました．
* Laravel Facade を使いすぎると破綻するのでできるだけ使用を控えよう
* プロダクトの規模が大きくなるにつれて MVC アーキテクチャではつらみが出てくるので「なんちゃってクリーンアーキテクチャ」を導入しよう
* Unit テストを書こう．そのためにアプリケーションのロジックからフレームワーク依存を剥がそう

この内，3つ目の「Unit テストを書こう」では，ビジネスロジックが集まる UseCase 層が Laravel （主に Eloquent）に依存してしまうと Unit テストが書けないので， **「Entity ＋ Repository」を導入することによって Eloquent への依存を無くす** ，という解決策を示しました．

:::message alert
登壇内容および本記事では，mpyw 氏の「なんちゃってクリーンアーキテクチャ」の利用を前提としています．
なんちゃってクリーンアーキテクチャの記事は[コチラ](https://zenn.dev/mpyw/articles/ce7d09eb6d8117)
:::

スライド中に示した Entity のコード例は以下の通りです．

```php
class User
{
    public function __construct(
        private int $id,
        private string $name,
    ) {
    }

    public function id(): int
    {
        return $this->id;
    }

    public function name(): string
    {
        return $this->name;
    }
}
```

コンストラクタで各属性を受け取り，ゲッターで値を返す，とてもシンプルなものです．

DB を扱う上で想定される入出力のありがちな処理は以下のようなものが考えられます．
* 新しくレコードを登録する
* 条件指定してレコードを取得する
* 一括でレコードの内容を書き換える（REST の PUT 的な処理）
* カラムを指定して内容を書き換える（REST の PATCH 的な処理）
* レコードを削除する

上記のような Entity を用意している場合，対応する Repository は次のように実装できます．
前提として，以下のような User モデルがあるとします

```php
class User extends Model
{
    public function toEntity(): UserEntity
    {
        return new UserEntity(
            id: $this->id,
            name: $this->name,
        );
    }
}
```

### 新しくレコードを登録する
```php
public function create(string $name): UserEntity
{
    return UserModel::create([
        'name' => $name,
    ])->toEntity();
}
```

### 条件指定してレコードを取得する
```php
public function retrieveById(int $id): ?UserEntity
{
    return UserModel::query() // find() などでも可能
        ->whereKey($id)
        ->first()
        ?->toEntity()
}
```

### 一括でレコードの内容を書き換える（REST の PUT 的な処理）
```php
public function update(int $id, string $name): UserEntity
{
    if (! $model = UserModel::find($id)) {
        throw new UserNotFoundException();
    }

    $model->update([
        'name' => $name,
        // 他の値も引数で受け取り列挙
    ]);

    return $model->toEntity();
}
```

### カラムを指定して内容を書き換える（REST の PATCH 的な処理）
```php
public function updateName(int $id, string $name): UserEntity
{
    if (! $model = UserModel::find($id)) {
        throw new UserNotFoundException();
    }

    $model->update([
        'name' => $name,
    ]);

    return $model->toEntity();
}
```

### レコードを削除する
```php
public function delete(int $id): void
{
    if (! $model = UserModel::find($id)) {
        throw new UserNotFoundException();
    }

    $model->delete();
}
```

# カラムが増えたときのことを想像してみよう
`updateHoge()` を沢山生やすかどうかは，絶対にそのカラムだけを更新してほしい・他のカラムには触れないで欲しいなどの強い意志がある時や，更新する可能性があるカラムが**確定で**限られている時などになるとは思いますが，シンプルな `update()` メソッドの需要はそれなりにあると思います．
カラムが増えた時，Repository層 を通すととても大変です．なにか一つ更新したい場合でも以下のように全ての属性を列挙しなければいけません．

```php
// twtter の user name を更新したいだけなのに．．．
$user = $this->userRepository
    ->update(
        id: $user->id(),
        name: $user->name(),
        nickName: $user->nickName(),
        birthday: $user->birthday(),
        gender: $user->gender(),
        twitterUserName: $newTwitterUserName,
        // ...
    );
```

PHP8.0 から導入された**名前付き引数**のおかげで値の渡しミスはかなり防げますが，それでも列挙するのがとても大変です．

:::message
オプション引数を使って `?string $name = null,` にすればいいのでは？と思われる方もいらっしゃるかもしれません．
もちろんこれも一つの解決策ではありますが，Entity は普通の VO などとは違い DB のテーブルと対応することが多いため，値としての NULL を設定したい場合と区別が付きづらくなるという欠点がありますので，あまりおすすめできません．
:::

あぁ，Eloquent と仲良くしていた頃が懐かしいなぁ．ActiveRecord だったら

```php
$user = User::find($id);
$user->update([
    'twitter_user_name' => $newTwitterUserName,
]);
```

とか

```php
$user = User::find($id);
$user->twitter_user_name = $newTwitterUserName;
$user->save();
```

とかで，簡単に更新できたのになぁ．．．．

ん？ *ActiveRecord*？

## そうか． Entity も ActiveRecord っぽくしてしまえばいいんだ．
Eloquent と完全に縁を切ったわけではありません．禁止したわけじゃないんです．
ここは都合よく ActiveRecord 味の蜜を吸わせてもらいましょう．
Entity に `save()` を生やせるよう改造しちゃえばいいんです．

# Eloquent に依存する 「なんちゃって Entity」
さて，いきなりですが実装例を示します．

```diff php
class User
{
    public function __construct(
-        private int $id,
-        private string $name,
+        private UserModel $model
    ) {
+        assert($model->exists);
    }

    public function id(): int
    {
-        return $this->id;
+        return $this->model->id;
    }

    public function name(): string
    {
-        return $this->name;
+        return $this->model->name;
    }
+
+    public function setName(string $name): static
+    {
+        $this->model->name = $name;
+
+        return $this;
+    }
+
+    public function save(): static
+    {
+        $this->model->save();
+
+        return $this;
+    }
}
```

差分を簡単にまとめるとこうです．
* コンストラクタで直接 Eloquent Model を受け取るようにしました
* 各パラメータに対してセッターを実装しました
* `save()` メソッドを実装しました

このようにリファクタリングすることで先の twitter_user_name の例は次のように書き換えることができます．

```diff php
- $user = $this->userRepository
-    ->update(
-        id: $user->id(),
-        name: $user->name(),
-        nickName: $user->nickName(),
-        birthday: $user->birthday(),
-        gender: $user->gender(),
-        twitterUserName: $newTwitterUserName,
-        // ...
-    );
+$user->setTwitterUserName($newTwitterUserName);
+$user->save();
```

とてもスッキリしましたね！
ただ，これだけではイマイチイメージがつかない方や「テストどうするの？」と思われる方もいらっしゃると思うので，実際にこの「なんちゃって Entity」を使った UseCase の実装例とそのテストを示します．

:::details Models/Employee.php
```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use App\Entities\Employee as EmployeeEntitiy;

/**
 * @property int $id ID
 * @property string $name 氏名
 * @property int $employee_number 社員番号
 * @property string $emergency_contact_number 緊急連絡先
 */
class Employee extends Model
{
    public function toEntity(): EmployeeEntity
    {
        return new EmployeeEntity($this);
    }
}
```
:::

:::details Entities/Employee.php
```php
<?php

declare(strict_types=1);

namespace App\Entities;

use App\Models\Employee as EmployeerModel;

class Employeer
{
    public function __construct(
        private EmployeeModel $model
    ) {
        assert($this->model->exists);
    }

    public function id(): int
    {
        return $this->model->id;
    }

    public function name(): string
    {
        return $this->model->name;
    }

    public function employeeNumber(): int
    {
        return $this->model->employee_number;
    }

    public function emergencyContactNumber(): string
    {
        return $this->model->emergency_contact_number;
    }

    public function setName(string $value): static
    {
        $this->model->name = $value;
        return $this;
    }

    public function setEmployeeNumber(string $value): static
    {
        $this->model->employee_number = $value;
        return $this;
    }

    public function setEmergencyContactNumber(string $value): static
    {
        $this->model->emergency_contact_number = $value;
        return $this;
    }

    public function save(): static
    {
        $this->model->save();
        return $this;
    }
}
```
:::

:::details Repositories/EmployeeRepository.php
```php
<?php

declare(strict_types=1);

namespace App\UseCases\Repositories;

use App\Entities\Employee as EmployeeEntitiy;
use App\Models\Employee as EmployeeModel;

class EmployeeRepository
{
    public function retrieveByEmplpyeeNumber(int $employeeNumber): ?EmployeeEntity
    {
        return EmployeeModel::query()
            ->where('employee_number', $employeeNumber)
            ->first()
            ?->toEntity();
    }
}
```
:::

```php
<?php

declare(strict_types=1);

namespace App\UseCases\Employee;

use App\Exceptions\Employee\EmployeeNotFoundException;
use App\Exceptions\Employee\OverwiteSameValueException;
use App\Models\Employee;
use App\Repositories\EmployeeRepository;

class UpdateEmergencyContactNumberAction
{
    public function __construct(
        private EmployeeRepository $repository,
    ) {
    }

    public function __invoke(int $employeeNumber, string $emergencyContactNumber): Employee
    {
        // 社員番号から社員を検索
        $employee = $this->repository
            ->retrieveByEmplpyeeNumber($employeeNumber);

        // 存在チェック
        if (!$employee) {
            throw new EmployeeNotFoundException('該当する社員が見つかりませでした。');
        }

        // 同じ番号での上書きを禁止する
        if ($employee->emergencyContactNumber() === $emergencyContactNumber) {
            throw new OverwiteSameValueException('変更前と同じ番号です。');
        }

        // 緊急連絡先の更新
        $employee->setEmergencyContactNumber($emergencyContactNumber);
        $employee-save();

        return $employee;
    }
}

```

```php
<?php

declare(strict_types=1);

namespace Tests\Unit\UseCases;

use App\Entities\Employee as EmployeeEntity;
use App\Exceptions\Employee\EmployeeNotFoundException;
use App\Repositories\EmployeeRepository;
use App\UseCases\Employee\UpdateEmergencyContactNumberAction;
use Mockery;
use Mockery\MockInterface;

class UpdateEmergencyContactNumberActionTest
{
    /**
     * @var EmployeeRepository&MockInterface&mixed
     */
    private EmployeeRepository $repository;

    private UpdateEmergencyContactNumberAction $action;

    public function setUp(): void
    {
        parent::setUp();

        $this->repository = Mockery::mock(EmployeeRepository::class);
        $this->action = new UpdateEmergencyContactNumberAction($this->repository);
    }

    /**
     * @test
     */
    public function 緊急連絡先の更新(): void
    {
        // リポジトリの振る舞い
        $this->repository
            ->shouldReceive('retrieveByEmplpyeeNumber')
            ->once()
            ->with('123')
            ->andReturn($employee = Mockery::mock(EmployeeEntity::class));

        // エンティティの振る舞い
        $employee
            ->shouldReceive('emergencyContactNumber')
            ->once()
            ->andReturn('080-1234-5678');
        $employee
            ->shouldReceive('setEmergencyContactNumber')
            ->once()
            ->with('080-9876-5432');
        $employee
            ->shouldReceive('save')
            ->once()

        // 実行
        $result = $this->action(
            employeeNumber: '123',
            emergencyContactNumber: '080-9876-5432',
        );

        // 戻り値の検証
        $this->assertInstanceOf(EmployeeEntity::class, $result);
    }

    /**
     * @test
     */
    public function 緊急連絡先が存在しない(): void
    {
        // リポジトリの振る舞い
        $this->repository
            ->shouldReceive('retrieveByEmplpyeeNumber')
            ->once()
            ->with('123')
            ->andReturnNull();

        // 例外が投げられることを期待
        $this->expectException(EmployeeNotFoundException::class);

        // 実行
        $this->action(
            employeeNumber: '123',
            emergencyContactNumber: '080-9876-5432',
        );
    }
}
```

このように，Entity 自身は Eloquent Model に依存していますが，UseCase 自身は引き続き Eloquent Model に依存しないため，モックを使って UseCase の Unit テストを書くことができました！

# まとめ
* Entity ＋ Repository を導入することで UseCase から Eloquent を剥がし Unit テストが書けるようになる（PHPerKaigi の登壇内容）
* Entity がコンストラクタで各属性を受け取る形だと，カラムが増えてきたときに更新機能の実装が大変になる
* Entity が Eloquent Model を受け取る形にすることで，ActiveRecord のように Entity に `save()` メソッドなどを生やせるようになるので便利
