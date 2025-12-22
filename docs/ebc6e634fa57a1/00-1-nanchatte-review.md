# 「なんちゃってクリーンアーキテクチャ」の振り返り

@mpyw さんの記事の要点を軽くおさらい。

---

## 元記事の主張（3行で）

1. **DDD や "真の" クリーンアーキテクチャは、大抵の現場ではオーバースペック**
2. **`app/UseCases` ディレクトリだけ切って、ドメインごとに単一責務なクラスを置くと使いやすい**
3. **ActiveRecord 指向のフレームワークで Repository パターンを無理に導入すると死ぬ**

---

## ディレクトリ構成（Post の Store を例に）

```
app/
├── Http/
│   ├── Controllers/
│   │   └── PostController.php         # store() メソッド
│   ├── Requests/
│   │   └── Post/
│   │       └── StoreRequest.php        # バリデーション
│   └── Resources/
│       └── PostResource.php            # レスポンス整形
├── Models/
│   ├── User.php
│   ├── Community.php
│   └── Post.php
└── UseCases/
    └── Post/
        ├── StoreAction.php             # ユースケース本体
        └── Exceptions/
            └── PostLimitExceededException.php
```

- UseCase を切り出すことで、Controller の肥大化を防ぐ
- 単一責務なクラス（1 UseCase = 1 クラス）
- Eloquent Model の機能は UseCase 内で普通に使う

---

## コード例（Post の Store）

### Controller

```php
final class PostController extends Controller
{
    public function store(StoreRequest $request, Community $community): PostResource
    {
        $post = (new StoreAction(
            community: $community,
            author: $request->user(),
        ))->run(
            title: $request->validated('title'),
            body: $request->validated('body'),
        );

        return new PostResource($post);
    }
}
```

### UseCase（StoreAction）

```php
final class StoreAction
{
    public function __construct(
        private Community $community,
        private User $author,
    ) {}

    public function run(string $title, string $body): Post
    {
        // ドメインルールのチェック
        if ($this->community->posts()->where('user_id', $this->author->id)->count() >= 10) {
            throw new PostLimitExceededException();
        }

        // Eloquent のリレーション機能をそのまま使う
        return $this->community->posts()->create([
            'user_id' => $this->author->id,
            'title' => $title,
            'body' => $body,
        ]);
    }
}
```

### Request

```php
final class StoreRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->communities()->where('id', $this->community->id)->exists();
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:100'],
            'body' => ['required', 'string', 'max:10000'],
        ];
    }
}
```

---

## 「なんちゃって」のいいところ

- **学習コストが低い**: Laravel の知識があればすぐ始められる
- **ファイル数が少ない**: Repository, Entity, VO などを作らない
- **Eloquent の恩恵を最大限受けられる**: リレーション、スコープ、アクセサなど
- **現実的な落としどころ**: 理想と現実のバランス

---

## 「なんちゃって」の限界（ここからステップアップする話）

元記事の Q&A より：

> **Q7. テーブル正規化されまくってて，超複雑かつ多機能なプロジェクトなんだけど？この案件でこの考え方使っても本当に大丈夫？**
>
> **無理です。潔く Eloquent Model を今すぐ完全に捨てて，本気で DDD やる構えを見せなさい。いくらなんでも適材適所ってものがあるよ！**

つまり、ある規模・複雑さを超えると「なんちゃって」では対応しきれない。

### 具体的に限界が来るケース

- モジュール間の依存関係が複雑化してきた
- 「この UseCase、どのドメインに属するんだっけ？」問題
- テストが書きづらくなってきた（Eloquent に依存しすぎ）
- 複数チームで開発するようになった（境界が曖昧だと衝突する）

---

## 記事での使い方（案）

「はじめに」の後に軽く挟む形：

```markdown
## 「なんちゃってクリーンアーキテクチャ」とは

詳細は元記事を読んでいただくとして、要点だけおさらいします。

- `app/UseCases` に単一責務なクラスを置く
- Repository パターンは使わず、Eloquent をそのまま活用
- Laravel の良さを殺さない、現実的な落としどころ

この設計は多くのプロジェクトで十分に機能します。
しかし、プロジェクトが大きくなり、複雑さが増してくると限界が見えてきます。

元記事でも言及されている通り：

> 無理です。潔く Eloquent Model を今すぐ完全に捨てて，本気で DDD やる構えを見せなさい。

本記事では、その「次のステップ」として、モジュラーモノリス × 4層アーキテクチャを紹介します。
```

