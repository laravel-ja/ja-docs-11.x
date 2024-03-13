# Eloquent:ファクトリ

- [イントロダクション](#introduction)
- [モデルファクトリの定義](#defining-model-factories)
    - [ファクトリの生成](#generating-factories)
    - [ファクトリの状態](#factory-states)
    - [ファクトリのコールバック](#factory-callbacks)
- [ファクトリを使用するモデル生成](#creating-models-using-factories)
    - [モデルのインスタンス化](#instantiating-models)
    - [モデルの永続化](#persisting-models)
    - [連続データ](#sequences)
- [リレーションのファクトリ](#factory-relationships)
    - [Has Manyリレーション](#has-many-relationships)
    - [Belongs Toリレーション](#belongs-to-relationships)
    - [Many To Manyリレーション](#many-to-many-relationships)
    - [ポリモーフィックリレーション](#polymorphic-relationships)
    - [ファクトリ内でのリレーション定義](#defining-relationships-within-factories)
    - [リレーションでの既存モデルの再利用](#recycling-an-existing-model-for-relationships)

<a name="introduction"></a>
## イントロダクション

アプリケーションのテストやデータベースの初期値生成時に、データベースへレコードを挿入する必要がある場合があるでしょう。Laravelでは、各カラムの値を手作業で指定する代わりに、[Eloquentモデル](/docs/{{version}}/eloquent)それぞれに対して、モデルファクトリを使用し、デフォルト属性セットを定義できます。

ファクトリの作成方法の例を確認するには、アプリケーションの`database/factories/UserFactory.php`ファイルを見てください。このファクトリはすべての新しいLaravelアプリケーションに含まれており、以下のファクトリ定義が含まれています。

    namespace Database\Factories;

    use Illuminate\Database\Eloquent\Factories\Factory;
    use Illuminate\Support\Facades\Hash;
    use Illuminate\Support\Str;

    /**
     * @extends \Illuminate\Database\Eloquent\Factories\Factory<\App\Models\User>
     */
    class UserFactory extends Factory
    {
        /**
         * The current password being used by the factory.
         */
        protected static ?string $password;

        /**
         * モデルのデフォルト状態の定義
         *
         * @return array<string, mixed>
         */
        public function definition(): array
        {
            return [
                'name' => fake()->name(),
                'email' => fake()->unique()->safeEmail(),
                'email_verified_at' => now(),
                'password' => static::$password ??= Hash::make('password'),
                'remember_token' => Str::random(10),
            ];
        }

        /**
         * Indicate that the model's email address should be unverified.
         */
        public function unverified(): static
        {
            return $this->state(fn (array $attributes) => [
                'email_verified_at' => null,
            ]);
        }
    }

ご覧のとおり、一番基本的な形式では、ファクトリはLaravelの基本ファクトリクラスを拡張し、`definition`メソッドを定義するクラスです。`definition`メソッドは、ファクトリを使用してモデルを作成するときに適用する必要がある属性値のデフォルトセットを返します。

`fake`ヘルパを使うと、ファクトリで[Faker](https://github.com/FakerPHP/Faker) PHPライブラリにアクセスでき、テストやシードのためにさまざまな種類のランダムデータを生成でき、便利です。

> [!NOTE]
> You can change your application's Faker locale by updating the `faker_locale` option in your `config/app.php` configuration file.

<a name="defining-model-factories"></a>
## モデルファクトリの定義

<a name="generating-factories"></a>
### ファクトリの生成

ファクトリを作成するには、`make:factory` [Artisanコマンド](/docs/{{version}}/artisan)を実行します。

```shell
php artisan make:factory PostFactory
```

新しいファクトリクラスは、`database/factories`ディレクトリに配置されます。

<a name="factory-and-model-discovery-conventions"></a>
#### モデルと対応するファクトリの規約

ファクトリを定義したら、モデルのファクトリインスタンスをインスタンス化するために、`Illuminate\Database\Eloquent\Factories\HasFactory`トレイトが、モデルへ提供しているstaticな`factory`メソッドが使用できます。

`HasFactory`トレイトの`factory`メソッドは規約に基づいて、その トレイトが割り当てられているモデルに適したファクトリを決定します。具体的には、`Database\Factories`名前空間の中でモデル名と一致するクラス名を持ち、サフィックスが`Factory`であるファクトリを探します。この規約を特定のアプリケーションやファクトリで適用しない場合は、モデルの`newFactory`メソッドを上書きし、モデルと対応するファクトリのインスタンスを直接返してください。

    use Illuminate\Database\Eloquent\Factories\Factory;
    use Database\Factories\Administration\FlightFactory;

    /**
     * モデルの新ファクトリ・インスタンスの生成
     */
    protected static function newFactory(): Factory
    {
        return FlightFactory::new();
    }

次に、対応するファクトリで、`model`プロパティを定義します。

    use App\Administration\Flight;
    use Illuminate\Database\Eloquent\Factories\Factory;

    class FlightFactory extends Factory
    {
        /**
         * このファクトリに対応するモデル名
         *
         * @var class-string<\Illuminate\Database\Eloquent\Model>
         */
        protected $model = Flight::class;
    }

<a name="factory-states"></a>
### ファクトリの状態

状態操作メソッドを使用すると、モデルファクトリへ任意の組み合わせで適用できる個別の変更を定義できます。たとえば、`Database\Factories\UserFactory`ファクトリに、デフォルトの属性値の１つを変更する`suspended`状態メソッドが含まれているとしましょう。

状態変換メソッドは通常、Laravelの基本ファクトリクラスが提供する`state`メソッドを呼び出します。`state`メソッドは、このファクトリ用に定義する素の属性の配列を受け取るクロージャを受け入れ、変更する属性の配列を返す必要があります。

    use Illuminate\Database\Eloquent\Factories\Factory;

    /**
     * ユーザーが一時停止されていることを示す
     */
    public function suspended(): Factory
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        });
    }

<a name="trashed-state"></a>
#### 「ゴミ箱入り」状態

Eloquentモデルが[ソフトデリート](/docs/{{version}}/eloquent#soft-deleting)可能であれば、組み込み済みの「ゴミ箱入り(`trashed`)」状態メソッドを呼び出し、作成したモデルが既に「ソフトデリート済み」と示せます。すべてのファクトリで自動的に利用可能で、`trashed`状態を手作業で定義する必要はありません。

    use App\Models\User;

    $user = User::factory()->trashed()->create();

<a name="factory-callbacks"></a>
### ファクトリのコールバック

ファクトリコールバックは、`afterMaking`メソッドと`afterCreating`メソッドを使用して登録し、モデルの作成または作成後に追加のタスクを実行できるようにします。ファクトリクラスで`configure`メソッドを定義して、これらのコールバックを登録する必要があります。ファクトリがインスタンス化されるときにLaravelが自動的にこのメソッドを呼び出します。

    namespace Database\Factories;

    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Factory;

    class UserFactory extends Factory
    {
        /**
         * モデルファクトリの設定
         */
        public function configure(): static
        {
            return $this->afterMaking(function (User $user) {
                // ...
            })->afterCreating(function (User $user) {
                // ...
            });
        }

        // ...
    }

また、`state`メソッド内にファクトリ・コールバックを登録し、特定の状態に特化した追加タスクを実行することもできます。

    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Factory;

    /**
     * ユーザーが一時停止されていることを示す
     */
    public function suspended(): Factory
    {
        return $this->state(function (array $attributes) {
            return [
                'account_status' => 'suspended',
            ];
        })->afterMaking(function (User $user) {
            // ...
        })->afterCreating(function (User $user) {
            // ...
        });
    }

<a name="creating-models-using-factories"></a>
## ファクトリを使用するモデル生成

<a name="instantiating-models"></a>
### モデルのインスタンス化

ファクトリを定義したら、そのモデルのファクトリインスタンスをインスタンス化するために、`Illuminate\Database\Eloquent\Factories\HasFactory`トレイトにより、モデルが提供する静的な`factory`メソッドを使用できます。モデル作成のいくつかの例を見てみましょう。まず、`make`メソッドを使用して、データベースへ永続化せずにモデルを作成します。

    use App\Models\User;

    $user = User::factory()->make();

`count`メソッドを使用して多くのモデルのコレクションを作成できます。

    $users = User::factory()->count(3)->make();

<a name="applying-states"></a>
#### 状態の適用

[状態](#factory-states)のいずれかをモデルに適用することもできます。モデルへ複数の状態変換を適用する場合は、状態変換メソッドを直接呼び出すだけです。

    $users = User::factory()->count(5)->suspended()->make();

<a name="overriding-attributes"></a>
#### 属性のオーバーライド

モデルのデフォルト値の一部をオーバーライドしたい場合は、値の配列を`make`メソッドに渡してください。指定された属性のみが置き換えられ、残りの属性はファクトリで指定したデフォルト値へ設定したままになります。

    $user = User::factory()->make([
        'name' => 'Abigail Otwell',
    ]);

もしくは、`state`メソッドをファクトリインスタンスで直接呼び出して、インライン状態変更を実行することもできます。

    $user = User::factory()->state([
        'name' => 'Abigail Otwell',
    ])->make();

> [!NOTE]
> [複数代入保護](/docs/{{version}}/eloquent#mass-assignment)は、ファクトリを使用してのモデル作成時、自動的に無効になります。

<a name="persisting-models"></a>
### モデルの永続化

`create`メソッドはモデルインスタンスをインスタンス化し、Eloquentの`save`メソッドを使用してデータベースへ永続化します。

    use App\Models\User;

    // App\Models\Userインスタンスを１つ生成
    $user = User::factory()->create();

    // App\Models\Userインスタンスを３つ生成
    $users = User::factory()->count(3)->create();

属性の配列を`create`メソッドに渡すことで、ファクトリのデフォルトのモデル属性をオーバーライドできます。

    $user = User::factory()->create([
        'name' => 'Abigail',
    ]);

<a name="sequences"></a>
### 連続データ

モデルを生成するごとに、特定のモデル属性の値を変更したい場合があります。これは、状態変換を連続データとして定義することで実現できます。たとえば、作成されたユーザーごとに、`admin`カラムの値を`Y`と`N`の間で交互に変更したいとしましょう。

    use App\Models\User;
    use Illuminate\Database\Eloquent\Factories\Sequence;

    $users = User::factory()
                    ->count(10)
                    ->state(new Sequence(
                        ['admin' => 'Y'],
                        ['admin' => 'N'],
                    ))
                    ->create();

この例では、`admin`値が`Y`のユーザーが５人作成され、`admin`値が`N`のユーザーが５人作成されます。

必要に応じて、シーケンス値としてクロージャを含めることができます。新しい値をそのシーケンスが必要とするたびにクロージャを呼び出します。

    use Illuminate\Database\Eloquent\Factories\Sequence;

    $users = User::factory()
                    ->count(10)
                    ->state(new Sequence(
                        fn (Sequence $sequence) => ['role' => UserRoles::all()->random()],
                    ))
                    ->create();

シーケンスクロージャ内では，クロージャへ注入されるシーケンスインスタンスの`$index`または`$count`プロパティにアクセスできます。`$index`プロパティには、これまでに行われたシーケンスの反復回数が格納され、`$count`プロパティには、シーケンスが起動された合計回数が格納されます。

    $users = User::factory()
                    ->count(10)
                    ->sequence(fn (Sequence $sequence) => ['name' => 'Name '.$sequence->index])
                    ->create();

使いやすいように、シーケンスは、`sequence`メソッドを使用して適用することもできます。このメソッドは、内部的に`state`メソッドを呼び出すだけです。`sequence`メソッドには、クロージャまたはシーケンスの属性を表す配列を指定します。

    $users = User::factory()
                    ->count(2)
                    ->sequence(
                        ['name' => 'First User'],
                        ['name' => 'Second User'],
                    )
                    ->create();

<a name="factory-relationships"></a>
## リレーションのファクトリ

<a name="has-many-relationships"></a>
### Has Manyリレーション

次に、Laravelの流暢（fluent）なファクトリメソッドを使用して、Eloquentモデルのリレーションを構築する方法を見ていきましょう。まず、アプリケーションに`App\Models\User`モデルと`App\Models\Post`モデルがあると想定します。また、`User`モデルが`Post`との`hasMany`リレーションを定義していると想定しましょう。 Laravelのファクトリが提供する`has`メソッドを使用して、３つの投稿を持つユーザーを作成できます。`has`メソッドはファクトリインスタンスを引数に取ります。

    use App\Models\Post;
    use App\Models\User;

    $user = User::factory()
                ->has(Post::factory()->count(3))
                ->create();

規約により、`Post`モデルを`has`メソッドに渡す場合、Laravelは`User`モデルにリレーションを定義する`posts`メソッドが存在していると想定します。必要に応じ、操作するリレーション名を明示的に指定できます。

    $user = User::factory()
                ->has(Post::factory()->count(3), 'posts')
                ->create();

もちろん、関連モデルで状態を操作することもできます。さらに、状態変更で親モデルへのアクセスが必要な場合は、クロージャベースの状態変換が渡せます。

    $user = User::factory()
                ->has(
                    Post::factory()
                            ->count(3)
                            ->state(function (array $attributes, User $user) {
                                return ['user_type' => $user->type];
                            })
                )
                ->create();

<a name="has-many-relationships-using-magic-methods"></a>
#### マジックメソッドの使用

使いやすいように、Laravelのマジックファクトリリレーションメソッドを使用してリレーションを構築できます。たとえば、以下の例では、規約を使用して、`User`モデルの`posts`リレーションメソッドを介して作成する必要がある関連モデルを決定します。

    $user = User::factory()
                ->hasPosts(3)
                ->create();

マジックメソッドを使用してファクトリリレーションを作成する場合、属性の配列を渡して、関連モデルをオーバーライドできます。

    $user = User::factory()
                ->hasPosts(3, [
                    'published' => false,
                ])
                ->create();

状態の変更で親モデルへのアクセスが必要な場合は、クロージャベースの状態変換を提供できます。

    $user = User::factory()
                ->hasPosts(3, function (array $attributes, User $user) {
                    return ['user_type' => $user->type];
                })
                ->create();

<a name="belongs-to-relationships"></a>
### Belongs Toリレーション

ファクトリを使用して"has many"リレーションを構築する方法を検討したので、逆の関係を調べてみましょう。`for`メソッドを使用して、ファクトリが作成したモデルの属する親モデルを定義できます。たとえば、１人のユーザーに属する３つの`App\Models\Post`モデルインスタンスを作成できます。

    use App\Models\Post;
    use App\Models\User;

    $posts = Post::factory()
                ->count(3)
                ->for(User::factory()->state([
                    'name' => 'Jessica Archer',
                ]))
                ->create();

作成するモデルに関連付ける必要のある親モデルインスタンスがすでにある場合は、モデルインスタンスを`for`メソッドに渡すことができます。

    $user = User::factory()->create();

    $posts = Post::factory()
                ->count(3)
                ->for($user)
                ->create();

<a name="belongs-to-relationships-using-magic-methods"></a>
#### マジックメソッドの使用

便利なように、Laravelのマジックファクトリリレーションシップメソッドを使用して、"belongs to"リレーションシップを定義できます。たとえば、以下の例では、３つの投稿が`Post`モデルの`user`リレーションに属する必要があることを規約を使用して決定しています。

    $posts = Post::factory()
                ->count(3)
                ->forUser([
                    'name' => 'Jessica Archer',
                ])
                ->create();

<a name="many-to-many-relationships"></a>
### Many To Manyリレーション

[has manyリレーション](#has-many-relationships)と同様に、"many to many"リレーションは、`has`メソッドを使用して作成できます。

    use App\Models\Role;
    use App\Models\User;

    $user = User::factory()
                ->has(Role::factory()->count(3))
                ->create();

<a name="pivot-table-attributes"></a>
#### ピボットテーブルの属性

モデルをリンクするピボット／中間テーブルへ設定する属性を定義する必要がある場合は、`hasAttached`メソッドを使用します。このメソッドは、ピボットテーブルの属性名と値の配列を２番目の引数に取ります。

    use App\Models\Role;
    use App\Models\User;

    $user = User::factory()
                ->hasAttached(
                    Role::factory()->count(3),
                    ['active' => true]
                )
                ->create();

状態変更で関連モデルへのアクセスが必要な場合は、クロージャベースの状態変換を指定できます。

    $user = User::factory()
                ->hasAttached(
                    Role::factory()
                        ->count(3)
                        ->state(function (array $attributes, User $user) {
                            return ['name' => $user->name.' Role'];
                        }),
                    ['active' => true]
                )
                ->create();

作成しているモデルへアタッチしたいモデルインスタンスがすでにある場合は、モデルインスタンスを`hasAttached`メソッドへ渡せます。この例では、同じ３つの役割が３人のユーザーすべてに関連付けられます。

    $roles = Role::factory()->count(3)->create();

    $user = User::factory()
                ->count(3)
                ->hasAttached($roles, ['active' => true])
                ->create();

<a name="many-to-many-relationships-using-magic-methods"></a>
#### マジックメソッドの使用

利便性のため、Laravelのマジックファクトリリレーションメソッドを使用して、多対多のリレーションを定義できます。たとえば、次の例では、関連するモデルを`User`モデルの`roles`リレーションメソッドを介して作成する必要があることを規約を使用して決定します。

    $user = User::factory()
                ->hasRoles(1, [
                    'name' => 'Editor'
                ])
                ->create();

<a name="polymorphic-relationships"></a>
### ポリモーフィックリレーション

[ポリモーフィックな関係](/docs/{{version}}/eloquent-relationships#polymorphic-relationships)もファクトリを使用して作成できます。ポリモーフィックな"morph many"リレーションは、通常の"has many"リレーションと同じ方法で作成します。たとえば、 `App\Models\Post`モデルが`App\Models\Comment`モデルと`morphMany`関係を持っている場合は以下のようになります。

    use App\Models\Post;

    $post = Post::factory()->hasComments(3)->create();

<a name="morph-to-relationships"></a>
#### Morph Toリレーション

マジックメソッドを使用して`morphTo`関係を作成することはできません。代わりに、`for`メソッドを直接使用し、関係の名前を明示的に指定する必要があります。たとえば、`Comment`モデルに`morphTo`関係を定義する `commentable`メソッドがあると想像してください。この状況で`for`メソッドを直接使用し、１つの投稿に属する３つのコメントを作成できます。

    $comments = Comment::factory()->count(3)->for(
        Post::factory(), 'commentable'
    )->create();

<a name="polymorphic-many-to-many-relationships"></a>
#### ポリモーフィック多対多リレーション

ポリモーフィック「多対多」（`morphToMany`／` morphedByMany`）リレーションは、ポリモーフィックではない「多対多」リレーションと同じように作成できます。

    use App\Models\Tag;
    use App\Models\Video;

    $videos = Video::factory()
                ->hasAttached(
                    Tag::factory()->count(3),
                    ['public' => true]
                )
                ->create();

もちろん、`has`マジックメソッドを使用して、ポリモーフィックな「多対多」リレーションを作成することもできます。

    $videos = Video::factory()
                ->hasTags(3, ['public' => true])
                ->create();

<a name="defining-relationships-within-factories"></a>
### ファクトリ内でのリレーション定義

モデルファクトリ内でリレーションを定義するには、リレーションの外部キーへ新しいファクトリインスタンスを割り当てます。これは通常、`belongsTo`や`morphTo`リレーションなどの「逆」関係で行います。たとえば、投稿を作成時に新しいユーザーを作成する場合は、次のようにします。

    use App\Models\User;

    /**
     * モデルのデフォルト状態の定義
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'title' => fake()->title(),
            'content' => fake()->paragraph(),
        ];
    }

リレーションのカラムがそれを定義するファクトリに依存している場合は、属性にクロージャを割り当てることができます。クロージャは、ファクトリの評価済み属性配列を受け取ります。

    /**
     * モデルのデフォルト状態の定義
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'user_type' => function (array $attributes) {
                return User::find($attributes['user_id'])->type;
            },
            'title' => fake()->title(),
            'content' => fake()->paragraph(),
        ];
    }

<a name="recycling-an-existing-model-for-relationships"></a>
### リレーションでの既存モデルの再利用

あるモデルが、他のモデルと共通のリレーションを持っている場合、ファクトリが生成するすべてのリレーションで、モデルの単一インスタンスが再利用されるように、`recycle`メソッドを使用してください。

例えば、`Airline`、`Flight`、`Ticket`モデルがあり、チケット(Ticket)は航空会社(Airline)とフライト(Flight)に所属し、フライトも航空会社に所属しているとします。チケットを作成する際には、チケットとフライトの両方に同じ航空会社を使いたいでしょうから、`recycle`メソッドへ航空会社のインスタンスを渡します。

    Ticket::factory()
        ->recycle(Airline::factory()->create())
        ->create();

共通のユーザーやチームに所属するモデルがある場合、`recycle`メソッドが特に便利だと感じるでしょう。

`recycle`メソッドは、既存のモデルのコレクションを受け取ることもできます。コレクションを`recycle`メソッドへ渡すと、ファクトリがそのタイプのモデルを必要とするときに、コレクションからランダムにモデルを選びます。

    Ticket::factory()
        ->recycle($airlines)
        ->create();
