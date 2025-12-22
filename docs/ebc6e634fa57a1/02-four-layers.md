# 4層アーキテクチャ

## なぜ4層か

### なんちゃってとの対比

| | なんちゃって | 4層アーキテクチャ |
|---|-------------|------------------|
| 層 | Controller + UseCase + Model | Presentation + Application + Domain + Infrastructure |
| Eloquent 依存 | UseCase が直接 Eloquent を使う | Infrastructure に閉じ込める |
| Repository | なし | Domain に Interface、Infrastructure に実装 |
| DTO | なし（Request/Response を直接使う） | Input/Output で型安全に |
| テスタビリティ | Eloquent に依存するので DB が必要 | Domain は純粋 PHP、モックしやすい |

### なぜ厳格にするのか

なんちゃってでは Eloquent への依存を許容していた。
これはシンプルさとのトレードオフとして正しい選択だった。

しかし、規模が大きくなると：

- **UseCase が肥大化**: バリデーション、認可、永続化、レスポンス整形が混在
- **テストが書きづらい**: Eloquent に依存しているので DB が必要
- **変更の影響範囲が読めない**: Eloquent のリレーションを使っているので、どこまで影響するか分からない

4層に分けることで、**責務が明確になり、テストしやすく、変更の影響範囲が限定**される。

### ファイル数は増えるけど…

確かに、4層に分けるとファイル数は増える。
1つの CRUD 機能に対して、Controller, Request, Assembler, UseCase, Interactor, Input, Output, Entity, Repository(IF), Repository(実装), Model, Factory, Policy...

でも、**AI がいる今なら問題ない**。

- ファイル生成は AI に任せられる
- 責務が明確だから AI も正しいファイルに正しいコードを書ける
- むしろ責務が曖昧な方が AI が変な場所にコードを突っ込んでくる

:::message
5年前ならオーバースペックだった。でも今は違う。
AI にコードを書かせる時代だからこそ、厳格な責務分割が活きてくる。
:::

---

## クリーンアーキテクチャとの対応

クリーンアーキテクチャでは、同心円状にレイヤーを分けて依存方向を内側に向ける。
本記事では、実用性を重視して以下の4層に整理している。

| 本記事の層 | クリーンアーキテクチャでの対応 |
|-----------|---------------------------|
| Presentation | Interface Adapters (Controllers, Presenters) |
| Application | Use Cases |
| Domain | Entities |
| Infrastructure | Frameworks & Drivers |

---

## 依存方向

```
Presentation → Application → Domain ← Infrastructure
```

- **Presentation** は Application のみに依存
- **Application** は Domain のみに依存
- **Infrastructure** は Domain の Interface を実装（依存性逆転）
- **Domain** は何にも依存しない（純粋な PHP）

### 依存性逆転（DIP）

通常、上位モジュール（Application）が下位モジュール（Infrastructure）に依存しがち。

```php
// ❌ Application が Infrastructure に依存
class StoreProjectInteractor
{
    public function __construct(
        private ProjectEloquentRepository $repository, // 具象に依存
    ) {}
}
```

依存性逆転では、Domain に Interface を置き、Infrastructure が実装する。

```php
// ✅ Domain に Interface
// modules/Project/Domain/ProjectRepository.php
interface ProjectRepository
{
    public function store(Project $project): void;
}

// ✅ Infrastructure が実装
// modules/Project/Infrastructure/ProjectRepository.php
class ProjectRepository implements ProjectRepositoryInterface
{
    public function store(Project $project): void
    {
        // Eloquent で永続化
    }
}

// ✅ Application は Interface に依存
class StoreProjectInteractor
{
    public function __construct(
        private ProjectRepository $repository, // Interface に依存
    ) {}
}
```

---

## 各層に置くクラス

| 層 | 配置するクラス | 責務 |
|----|--------------|------|
| **Presentation** | Controller, FormRequest, Assembler | HTTPリクエストの受付・レスポンス整形 |
| **Application** | UseCase(IF), Interactor, Input, Output | ユースケースの実行・調整 |
| **Domain** | Entity, VO, Repository(IF), Policy, Factory | ビジネスルール |
| **Infrastructure** | Repository実装, EloquentModel, ExceptionHandler | 技術的詳細 |

---

## 各層の詳細

### Presentation 層

HTTP リクエストを受け取り、Application 層に渡す入口。

| クラス | 責務 |
|-------|------|
| **Controller** | ルーティングの終端。UseCase を呼び出し、結果を返す |
| **FormRequest** | バリデーション。Laravel の機能をそのまま使う |
| **Assembler** | FormRequest → Input DTO への変換 |

```php
// Controller
class ProjectController
{
    public function store(
        StoreProjectRequest $request,
        ProjectInputAssembler $assembler,
        StoreProjectUseCase $useCase,
    ): array {
        $input = $assembler->toStoreInput($request->validated());
        $output = $useCase($input);
        return $output->toArray();
    }
}

// FormRequest
class StoreProjectRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => ['required', 'string', 'max:255'],
            'client_id' => ['nullable', 'string', 'uuid'],
            'manager_id' => ['required', 'string', 'uuid'],
            'period' => ['required', 'array'],
            'period.start' => ['required', 'date'],
            'period.end' => ['required', 'date', 'after:period.start'],
            'type' => ['required', 'string', Rule::enum(ProjectType::class)],
        ];
    }
}

// Assembler
// ※ MemberId, DateRange, ProjectType は Contract に定義されているので
//   Presentation 層でも参照できる（Domain にあったら依存違反になる）
class ProjectInputAssembler
{
    public function toStoreInput(array $validated): StoreProjectInput
    {
        return new StoreProjectInput(
            name: $validated['name'],
            clientId: isset($validated['client_id'])
                ? ClientId::from($validated['client_id'])
                : null,
            managerId: MemberId::from($validated['manager_id']),
            period: DateRange::from(
                $validated['period']['start'],
                $validated['period']['end'],
            ),
            type: ProjectType::from($validated['type']),
        );
    }
}
```

### Application 層

ユースケースの実行・調整を担当。

| クラス | 責務 |
|-------|------|
| **UseCase (Interface)** | ユースケースの契約（テスト時に差し替え可能） |
| **Interactor** | UseCase の実装。Domain を組み合わせてユースケースを実現 |
| **Input** | 入力 DTO。Presentation からの入力を型安全に受け取る |
| **Output / View** | 出力 DTO。Presentation に返すデータ |

#### 補足説明

**UseCase と Interactor を分ける理由**

UseCase を Interface として定義し、Interactor で実装している。
テスト時に UseCase をモックに差し替えられるため、Controller のテストが楽になる。

**Interactor は「調整役」**

Interactor 自身はビジネスロジックを持たない。
Domain 層のクラス（Entity, Policy, Factory, Repository）を組み合わせてユースケースを実現する。

```
Interactor の役割:
1. Policy で認可チェック
2. Factory で Entity 生成
3. Repository で永続化
4. Output DTO に変換して返す
```

**トランザクション管理**

基本は集約単位、つまり Repository 内でトランザクションを貼る。

ただし、複数 Module をまたいだ更新系ユースケースが出てきた場合は、抽象化した `TransactionExecutor` を使って Interactor 側でトランザクションを貼ることを認める。

Laravel はネストしたトランザクションの管理がちゃんとしている（`savepoint` を使った擬似ネスト）ので、Repository 内で貼ったトランザクションと Interactor で貼ったトランザクションが競合しない。

```php
// 複数 Module をまたぐ場合の例（Assignment Module → Project Module）
// Requirement のスロットにメンバーを充てて Assignment に変換する
class SwapRequirementToAssignmentInteractor implements SwapRequirementToAssignmentUseCase
{
    public function __construct(
        private TransactionExecutor $transaction,
        private AssignmentFactory $factory,
        private AssignmentRepository $repository,
        private RequirementQueryService $requirementQuery,   // Project Module（参照）
        private RequirementCommandService $requirementCommand, // Project Module（更新）
    ) {}

    public function __invoke(SwapRequirementToAssignmentInput $input): void
    {
        // スロットに対応する Requirement を取得
        $requirements = $this->requirementQuery->findBySlotIds($input->slotIds);

        $this->transaction->execute(function () use ($requirements, $input) {
            // Requirement → Assignment に変換（member_id を充てる）
            // Factory が IdGenerator を使って UUID を生成
            $assignments = $requirements->map(
                fn ($req) => $this->factory->create(
                    year: $req->year,
                    month: $req->month,
                    projectId: $req->projectId,
                    memberId: $input->memberId,
                    workload: $req->workload,
                ),
            );

            // Assignment Module: Bulk Insert
            $this->repository->bulkStore($assignments);

            // Project Module: 該当スロットの Requirement を削除
            $this->requirementCommand->deleteBySlotIds($input->slotIds);
        });
    }
}
```

#### コード例

```php
// UseCase Interface
interface StoreProjectUseCase
{
    public function __invoke(StoreProjectInput $input): ProjectView;
}

// Interactor（実装）
class StoreProjectInteractor implements StoreProjectUseCase
{
    public function __construct(
        private ProjectFactory $factory,
        private ProjectRepository $repository,
        private StoreProjectPolicy $policy,
    ) {}

    public function __invoke(StoreProjectInput $input): ProjectView
    {
        $this->policy->authorize($input);
        $project = $this->factory->create($input);
        $this->repository->store($project);
        return ProjectView::fromEntity($project);
    }
}
```

### Domain 層

ビジネスルールを表現。フレームワークに依存しない純粋な PHP。

| クラス | 責務 |
|-------|------|
| **Entity** | ドメインオブジェクト。ビジネスルールを持つ |
| **Repository (Interface)** | 永続化の契約。実装は Infrastructure |
| **Policy** | 認可ルール。ビジネスルールとしての権限判定 |
| **Factory** | Entity の生成ロジックをカプセル化 |
| **IdGenerator (Interface)** | ID 生成の契約。実装は Infrastructure |

#### UUID を使うべき理由

1. **DB INSERT 前に ID が確定する**: Factory で Entity を生成した時点で ID を持てる。Repository に渡す前に ID が確定しているので、関連オブジェクトの生成や戻り値の構築が楽。
2. **Auto Increment だと困るケース**: INSERT してからでないと ID がわからない。トランザクション内で関連レコードを作る際に困る。
3. **分散システムでの衝突回避**: マイクロサービス化を視野に入れる場合、UUID なら衝突しない。

#### Factory の create と reconstruct

Factory には 2 つの役割がある：
- `create`: 新規生成。IdGenerator で UUID を発行
- `reconstruct`: DB からの復元。既存の ID を受け取る

```php
// IdGenerator Interface（Domain）
interface IdGenerator
{
    public function generate(): string;
}

// Factory
// - create: VO を受け取る（Assembler で型付けされた Input から）
// - reconstruct: primitive を受け取る（DB からの復元）
class ProjectFactory
{
    public function __construct(
        private IdGenerator $idGenerator,
    ) {}

    public function create(
        string $name,
        ?ClientId $clientId,
        MemberId $managerId,
        LocalDate $since,
        LocalDate $until,
        ProjectType $type,
    ): Project {
        return new Project(
            id: ProjectId::create($this->idGenerator->generate()),
            name: $name,
            clientId: $clientId,
            managerId: $managerId,
            period: new DateRange($since, $until),
            type: $type,
        );
    }

    public function reconstruct(
        string $id,
        string $name,
        ?string $clientId,
        string $managerId,
        string $since,
        string $until,
        string $type,
    ): Project {
        return new Project(
            id: ProjectId::create($id),
            name: $name,
            clientId: ClientId::createOrNull($clientId),
            managerId: MemberId::create($managerId),
            period: new DateRange(
                LocalDate::parse($since),
                LocalDate::parse($until),
            ),
            type: ProjectType::from($type),
        );
    }
}
```

#### IdGenerator の実装（Infrastructure）

```php
// Ramsey UUID を使った実装
class RamseyIdGenerator implements IdGenerator
{
    public function generate(): string
    {
        return Uuid::uuid7()->toString();
    }
}

// テスト用の固定値実装
class FixedIdGenerator implements IdGenerator
{
    public function __construct(
        private string $value = 'test-uuid',
    ) {}

    public function generate(): string
    {
        return $this->value;
    }
}
```

#### Entity の例

```php
class Project
{
    public function __construct(
        public readonly ProjectId $id,
        public readonly string $name,
        public readonly ?ClientId $clientId,
        public readonly MemberId $managerId,
        public readonly DateRange $period,
        public readonly ProjectType $type,
    ) {}
}
```

### Infrastructure 層

技術的詳細を担当。フレームワーク依存はここに閉じ込める。

| クラス | 責務 |
|-------|------|
| **Repository（実装）** | Domain の Repository Interface を実装 |
| **Eloquent Model** | DB テーブルとのマッピング。クエリビルダとして使う |
| **ExceptionHandler** | DB 例外を Domain 例外に変換 |
| **QueryService（実装）** | Contract の QueryService Interface を実装 |

```php
// Repository 実装
class ProjectRepository implements ProjectRepositoryInterface
{
    public function __construct(
        private ProjectModel $model,
        private ProjectDatabaseExceptionHandler $exceptionHandler,
    ) {}

    public function store(Project $project): void
    {
        $this->exceptionHandler->handleOnStore(
            fn () => $this->model->newQuery()->create([
                'id' => $project->id->value,
                'manager_id' => $project->managerId->value,
                'name' => $project->name,
                'type' => $project->type->value,
                // ...
            ]),
        );
    }
}
```

---

## ディレクトリ例: Project モジュール

```
modules/Project/
├── ModuleServiceProvider.php
├── Presentation/
│   ├── ProjectController.php
│   ├── StoreProjectRequest.php
│   └── ProjectInputAssembler.php
├── Application/
│   ├── StoreProjectUseCase.php       # Interface
│   ├── StoreProjectInteractor.php    # 実装
│   ├── StoreProjectInput.php
│   └── ProjectView.php
├── Domain/
│   ├── Project.php                   # Entity
│   ├── ProjectRepository.php         # Interface
│   ├── ProjectFactory.php
│   └── Policy/
│       └── StoreProjectPolicy.php
└── Infrastructure/
    ├── ProjectRepository.php         # 実装
    ├── ProjectDatabaseExceptionHandler.php
    └── Model/
        └── ProjectModel.php          # Eloquent
```

---

## deptrac で4層の依存も検証

モジュール間の依存だけでなく、4層の依存方向も deptrac で検証できる。

```yaml
# deptrac/domain.yaml
deptrac:
  paths:
    - ./modules

  layers:
    - name: Presentation
      collectors:
        - type: directory
          value: modules/.*/Presentation/.*
    - name: Application
      collectors:
        - type: directory
          value: modules/.*/Application/.*
    - name: Domain
      collectors:
        - type: directory
          value: modules/.*/Domain/.*
    - name: Infrastructure
      collectors:
        - type: directory
          value: modules/.*/Infrastructure/.*

  ruleset:
    Presentation:
      - Presentation
      - Application
    Application:
      - Application
      - Domain
    Domain:
      - Domain
    Infrastructure:
      - Infrastructure
      - Application
      - Domain
```

---

## 「UseCase だけ」から「4層」へ

「UseCase を作っただけ」の状態と4層アーキテクチャの違いは、**責務の明確な分離**。

```php
// ❌ UseCase に全部詰め込んだ例
class StoreProjectUseCase
{
    public function handle(Request $request): JsonResponse
    {
        // バリデーション、認可、永続化、レスポンス整形が全部ここに...
    }
}

// ✅ 4層で分離した例
// Presentation: リクエスト受付、バリデーション、レスポンス整形
// Application: ユースケース調整（認可 → 生成 → 永続化）
// Domain: ビジネスルール（Entity, Policy, Factory）
// Infrastructure: 永続化の実装
```

---

## 記事での書き方（案）

```markdown
# 4層アーキテクチャ

各モジュール内は、4つの層で構成されます。

## 依存方向

（図）

Presentation → Application → Domain ← Infrastructure

ポイントは Infrastructure が Domain に依存している（依存性逆転）こと。

## 各層の責務

（表）

## ディレクトリ構造

（例）

## deptrac で依存方向を検証

（yaml）
```

