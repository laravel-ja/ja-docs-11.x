# Eloquent:コレクション

- [イントロダクション](#introduction)
- [利用可能なメソッド](#available-methods)
- [カスタムコレクション](#custom-collections)

<a name="introduction"></a>
## イントロダクション

`get`メソッドで取得した結果や、リレーションによりアクセスした結果など、結果として複数のモデルを返すEloquentメソッドはすべて、`Illuminate\Database\Eloquent\Collection`クラスのインスタンスを返します。EloquentコレクションオブジェクトはLaravelの[基本的なコレクション](/docs/{{version}}/collections)を拡張しているため、基になるEloquentモデルの配列をスムーズに操作できるように使用する、数十のメソッドを自然に継承します。これらの便利なメソッドをすべて学ぶために、Laravelコレクションのドキュメントは必ず確認してください！

すべてのコレクションはイテレーターとしても機能し、単純なPHP配列のようにループで使えます。

    use App\Models\User;

    $users = User::where('active', 1)->get();

    foreach ($users as $user) {
        echo $user->name;
    }

ただし、前述のようにコレクションは配列よりもはるかに強力であり、直感的なインターフェイスを使用してチェーンする可能性を持つさまざまなマップ/リデュース操作を用意しています。たとえば、非アクティブなモデルをすべて削除してから、残りのユーザーの名を収集する場面を考えましょう。

    $names = User::all()->reject(function (User $user) {
        return $user->active === false;
    })->map(function (User $user) {
        return $user->name;
    });

<a name="eloquent-collection-conversion"></a>
#### Eloquentコレクションの変換

ほとんどのEloquentコレクションメソッドはEloquentコレクションの新しいインスタンスを返しますが、`collapse`、`flatten`、`flip`、`keys`、`pluck`、`zip`メソッドは、[基本のコレクション](/docs/{{version}}/collections)インスタンスを返します。同様に、`map`操作がEloquentモデルを含まないコレクションを返す場合、それは基本コレクションインスタンスに変換されます。

<a name="available-methods"></a>
## 利用可能なメソッド

すべてのEloquentコレクションはベースの[Laravelコレクション](/docs/{{version}}/collections#available-methods)オブジェクトを拡張します。したがって、これらは基本コレクションクラスによって提供されるすべての強力なメソッドを継承します。

さらに、`Illuminate\Database\Eloquent\Collection`クラスは、モデルコレクションの管理を支援するメソッドのスーパーセットを提供します。ほとんどのメソッドは`Illuminate\Database\Eloquent\Collection`インスタンスを返します。ただし、`modelKeys`などの一部のメソッドは、`Illuminate\Support\Collection`インスタンスを返します。

<style>
    .collection-method-list > p {
        columns: 14.4em 1; -moz-columns: 14.4em 1; -webkit-columns: 14.4em 1;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .collection-method code {
        font-size: 14px;
    }

    .collection-method:not(.first-collection-method) {
        margin-top: 50px;
    }
</style>

<div class="collection-method-list" markdown="1">

[append](#method-append)
[contains](#method-contains)
[diff](#method-diff)
[except](#method-except)
[find](#method-find)
[findOrFail](#method-find-or-fail)
[fresh](#method-fresh)
[intersect](#method-intersect)
[load](#method-load)
[loadMissing](#method-loadMissing)
[modelKeys](#method-modelKeys)
[makeVisible](#method-makeVisible)
[makeHidden](#method-makeHidden)
[only](#method-only)
[setVisible](#method-setVisible)
[setHidden](#method-setHidden)
[toQuery](#method-toquery)
[unique](#method-unique)

</div>

<a name="method-append"></a>
#### `append($attributes)` {.collection-method .first-collection-method}

`append`メソッドは、コレクション内の全モデルへ属性を[追加](/docs/{{version}}/eloquent-serialization#appending-values-to-json)するように指示するために使用します。このメソッドには、属性の配列か、単一の属性を指定します。

    $users->append('team');

    $users->append(['team', 'is_admin']);

<a name="method-contains"></a>
#### `contains($key, $operator = null, $value = null)` {.collection-method}

`contains`メソッドを使い、指定モデルインスタンスがコレクションに含まれているかどうかを判定できます。このメソッドは、主キーまたはモデルインスタンスを引数に取ります。

    $users->contains(1);

    $users->contains(User::find(1));

<a name="method-diff"></a>
#### `diff($items)` {.collection-method}

`diff`メソッドは、指定コレクションに存在しないすべてのモデルを返します。

    use App\Models\User;

    $users = $users->diff(User::whereIn('id', [1, 2, 3])->get());

<a name="method-except"></a>
#### `except($keys)` {.collection-method}

`except`メソッドは、指定する主キーを持たないすべてのモデルを返します。

    $users = $users->except([1, 2, 3]);

<a name="method-find"></a>
#### `find($key)` {.collection-method}

`find`メソッドは、指定キーと一致する主キーを持つモデルを返します。`$key`がモデルインスタンスの場合、`find`は主キーに一致するモデルを返そうとします。`$key`がキーの配列である場合、`find`は指定配列の中の主キーを持つすべてのモデルを返します。

    $users = User::all();

    $user = $users->find(1);

<a name="method-find-or-fail"></a>
#### `findOrFail($key)` {.collection-method}

`findOrFail`メソッドは、指定キーと一致する主キーを持つモデルを返すか、もしくは一致するモデルがコレクション内に見つからない場合に`Illuminate\Database\Eloquent\ModelNotFoundException`例外を投げます。

    $users = User::all();

    $user = $users->findOrFail(1);

<a name="method-fresh"></a>
#### `fresh($with = [])` {.collection-method}

`fresh`メソッドは、データベースからコレクション内の各モデルの新しいインスタンスを取得します。さらに、指定したリレーションはすべてEagerロードされます。

    $users = $users->fresh();

    $users = $users->fresh('comments');

<a name="method-intersect"></a>
#### `intersect($items)` {.collection-method}

`intersect`メソッドは、指定コレクションにも存在するすべてのモデルを返します。

    use App\Models\User;

    $users = $users->intersect(User::whereIn('id', [1, 2, 3])->get());

<a name="method-load"></a>
#### `load($relations)` {.collection-method}

`load`メソッドは、コレクション内のすべてのモデルに対して指定するリレーションをEagerロードします。

    $users->load(['comments', 'posts']);

    $users->load('comments.author');

    $users->load(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);

<a name="method-loadMissing"></a>
#### `loadMissing($relations)` {.collection-method}

`loadMissing`メソッドは、関係がまだロードされていない場合、コレクション内のすべてのモデルに対して指定するリレーションをEagerロードします。

    $users->loadMissing(['comments', 'posts']);

    $users->loadMissing('comments.author');

    $users->loadMissing(['comments', 'posts' => fn ($query) => $query->where('active', 1)]);

<a name="method-modelKeys"></a>
#### `modelKeys()` {.collection-method}

`modelKeys`メソッドは、コレクション内のすべてのモデルの主キーを返します。

    $users->modelKeys();

    // [1, 2, 3, 4, 5]

<a name="method-makeVisible"></a>
#### `makeVisible($attributes)` {.collection-method}

`makeVisible`メソッドは、通常コレクション内の各モデルで"hidden"になっている[属性をvisibleにします](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)。

    $users = $users->makeVisible(['address', 'phone_number']);

<a name="method-makeHidden"></a>
#### `makeHidden($attributes)` {.collection-method}

`makeHidden`メソッドは、通常コレクション内の各モデルで"visible"になっている[属性をhiddenにします](/docs/{{version}}/eloquent-serialization#hiding-attributes-from-json)。

    $users = $users->makeHidden(['address', 'phone_number']);

<a name="method-only"></a>
#### `only($keys)` {.collection-method}

`only`メソッドは、指定主キーを持つすべてのモデルを返します。

    $users = $users->only([1, 2, 3]);

<a name="method-setVisible"></a>
#### `setVisible($attributes)` {.collection-method}

`setVisible`メソッドは、コレクション内の各モデルの全てのvisible属性を[一時的に上書き](/docs/{{version}}/eloquent-serialization#temporarily-modifying-attribute-visibility)します。

    $users = $users->setVisible(['id', 'name']);

<a name="method-setHidden"></a>
#### `setHidden($attributes)` {.collection-method}

`setHidden`メソッドは、コレクション内の各モデルの全てのhidden属性を[一時的に上書き](/docs/{{version}}/eloquent-serialization#temporarily-modifying-attribute-visibility)します。

    $users = $users->setHidden(['email', 'password', 'remember_token']);

<a name="method-toquery"></a>
#### `toQuery()` {.collection-method}

`toQuery`メソッドは、コレクションモデルの主キーに対する`whereIn`制約を含むEloquentクエリビルダインスタンスを返します。

    use App\Models\User;

    $users = User::where('status', 'VIP')->get();

    $users->toQuery()->update([
        'status' => 'Administrator',
    ]);

<a name="method-unique"></a>
#### `unique($key = null, $strict = false)` {.collection-method}

`unique`メソッドは、コレクション内のすべての一意のモデルを返します。コレクション内の同じ主キーを持つモデルは、すべて削除します。

    $users = $users->unique();

<a name="custom-collections"></a>
## カスタムコレクション

指定したモデルを操作する際に、カスタム`Collection`オブジェクトを使用したい場合は、モデルへ`CollectedBy`属性を追加してください。

    <?php

    namespace App\Models;

    use App\Support\UserCollection;
    use Illuminate\Database\Eloquent\Attributes\CollectedBy;
    use Illuminate\Database\Eloquent\Model;

    #[CollectedBy(UserCollection::class)]
    class User extends Model
    {
        // ...
    }

あるいは、モデルへ`newCollection`メソッドを定義することもできます。

    <?php

    namespace App\Models;

    use App\Support\UserCollection;
    use Illuminate\Database\Eloquent\Collection;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * 新しいEloquentCollectionインスタンスの作成
         *
         * @param  array<int, \Illuminate\Database\Eloquent\Model>  $models
         * @return \Illuminate\Database\Eloquent\Collection<int, \Illuminate\Database\Eloquent\Model>
         */
        public function newCollection(array $models = []): Collection
        {
            return new UserCollection($models);
        }
    }

 一度、`newCollection`メソッドを定義するか、モデルへ`CollectedBy`属性を追加すると、Eloquentは通常、`Illuminate\Database\Eloquent\Collection`インスタンスを返します。

アプリケーションのすべてのモデルでカスタムコレクションを使用したい場合は、アプリケーションのすべてのモデルで拡張する基底モデルクラスで、`newCollection`メソッドを定義する必要があります。
