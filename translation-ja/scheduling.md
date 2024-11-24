# タスクスケジュール

- [イントロダクション](#introduction)
- [スケジュール定義](#defining-schedules)
    - [Artisanコマンドのスケジュール](#scheduling-artisan-commands)
    - [キュー投入するジョブのスケジュール](#scheduling-queued-jobs)
    - [シェルコマンドのスケジュール](#scheduling-shell-commands)
    - [繰り返しのスケジュールオプション](#schedule-frequency-options)
    - [タイムゾーン](#timezones)
    - [タスク多重起動の停止](#preventing-task-overlaps)
    - [単一サーバ上でのタスク実行](#running-tasks-on-one-server)
    - [バックグランドタスク](#background-tasks)
    - [メンテナンスモード](#maintenance-mode)
    - [スケジュールグループ](#schedule-groups)
- [スケジュールの実行](#running-the-scheduler)
    - [秒単位のタスクスケジュール](#sub-minute-scheduled-tasks)
    - [スケジュールをローカルで実行](#running-the-scheduler-locally)
- [タスク出力](#task-output)
- [タスクフック](#task-hooks)
- [イベント](#events)

<a name="introduction"></a>
## イントロダクション

以前は、サーバでスケジュールする必要のあるタスクごとにcron設定エントリを作成する必要がありました。しかしながら、タスクスケジュールがソース管理されないため、これはすぐに苦痛になる可能性があります。既存のcronエントリを表示したり、エントリを追加したりするには、サーバへSSHで接続する必要がありました。

Laravelのコマンドスケジューラは、サーバ上のタスクのスケジュールを管理する新しいアプローチを提供します。このスケジューラを使用すると、Laravelアプリケーション内でコマンドスケジュールをスムーズかつ表現的に定義できます。スケジューラを使用する場合、サーバ上に必要なcronエントリーは１つだけです。タスクスケジュールは通常、アプリケーションの`routes/console.php`ファイルで定義します。

<a name="defining-schedules"></a>
## スケジュール定義

アプリケーションの`routes/console.php`ファイルですべてのスケジュールタスクを定義できます。始めに、例を見てみましょう。この例では、毎日午前0時に呼び出すクロージャをスケジュールしています。クロージャの中で、テーブルをクリアするためにデータベースクエリを実行しています。

    <?php

    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Facades\Schedule;

    Schedule::call(function () {
        DB::table('recent_users')->delete();
    })->daily();

クロージャを使用したスケジュールに加えて、[呼び出し可能なオブジェクト](https://secure.php.net/manual/en/language.oop5.magic.php#object.invoke)をスケジュールすることもできます。呼び出し可能なオブジェクトは、`__invoke`メソッドを含む単純なPHPクラスです。

    Schedule::call(new DeleteRecentUsers)->daily();

`routes/console.php`ファイルをコマンド定義のためだけに使用したい場合は、アプリケーションの`bootstrap/app.php`ファイルで`withSchedule`メソッドを使用して、スケジュールするタスクを定義できます。このメソッドは、スケジューラのインスタンスを受け取るクロージャを引数に取ります。

    use Illuminate\Console\Scheduling\Schedule;

    ->withSchedule(function (Schedule $schedule) {
        $schedule->call(new DeleteRecentUsers)->daily();
    })

スケジュールしたタスクの概要と、次に実行がスケジュールされている時間を表示したい場合は、`schedule:list` Artisanコマンドを使用します。

```bash
php artisan schedule:list
```

<a name="scheduling-artisan-commands"></a>
### Artisanコマンドのスケジュール

クロージャのスケジュールに加えて、[Artisanコマンド](/docs/{{version}}/artisan)およびシステムコマンドをスケジュールすることもできます。たとえば、`command`メソッドを使用して、コマンドの名前またはクラスのいずれかを使用してArtisanコマンドをスケジュールできます。

コマンドのクラス名を使用してArtisanコマンドをスケジュールする場合、コマンドが呼び出されたときにコマンドに提供する必要がある追加のコマンドライン引数の配列を渡せます。

    use App\Console\Commands\SendEmailsCommand;
    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send Taylor --force')->daily();

    Schedule::command(SendEmailsCommand::class, ['Taylor', '--force'])->daily();

<a name="scheduling-artisan-closure-commands"></a>
#### Artisanクロージャコマンドのスケジュール

クロージャで定義したArtisanコマンドをスケジューリングしたい場合は、コマンド定義の後にスケジューリング関連のメソッドをチェーンしてください。

    Artisan::command('delete:recent-users', function () {
        DB::table('recent_users')->delete();
    })->purpose('Delete recent users')->daily();

クロージャコマンドへ引数を渡す必要がある場合は、`schedule`メソッドへ渡してください。

    Artisan::command('emails:send {user} {--force}', function ($user) {
        // ...
    })->purpose('Send emails to the specified user')->schedule(['Taylor', '--force'])->daily();

<a name="scheduling-queued-jobs"></a>
### キュー投入するジョブのスケジュール

[キュー投入するジョブ](/docs/{{version}}/queues)をスケジュールするには、`job`メソッドを使います。このメソッドを使うと、ジョブをキューに入れるためのクロージャを自前で作成する`call`メソッドを使わずとも、ジョブをスケジュール実行できます。

    use App\Jobs\Heartbeat;
    use Illuminate\Support\Facades\Schedule;

    Schedule::job(new Heartbeat)->everyFiveMinutes();

オプションの２番目と３番目の引数を`job`メソッドに指定して、ジョブのキューに入れるために使用するキュー名とキュー接続を指定できます。

    use App\Jobs\Heartbeat;
    use Illuminate\Support\Facades\Schedule;

    // "sqs"接続の"heartbeats"キューにジョブをディスパッチ
    Schedule::job(new Heartbeat, 'heartbeats', 'sqs')->everyFiveMinutes();

<a name="scheduling-shell-commands"></a>
### シェルコマンドのスケジュール

オペレーティングシステムでコマンドを実行するためには`exec`メソッドを使います。

    use Illuminate\Support\Facades\Schedule;

    Schedule::exec('node /home/forge/script.js')->daily();

<a name="schedule-frequency-options"></a>
### 繰り返しのスケジュールオプション

指定間隔で実行するようにタスクを設定する方法の例をいくつか見てきました。しかし、タスクに割り当てることができるタスクスケジュールの間隔は他にもたくさんあります。

<div class="overflow-auto">

| メソッド                           | 説明                                   |
| ---------------------------------- | -------------------------------------- |
| `->cron('* * * * *');`             | カスタムcronスケジュールでタスクを実行 |
| `->everySecond();`                 | 毎秒タスク実行                         |
| `->everyTwoSeconds();`             | ２秒毎にタスク実行                     |
| `->everyFiveSeconds();`            | ５秒毎にタスク実行                     |
| `->everyTenSeconds();`             | １０秒ごとにタスク実行                 |
| `->everyFifteenSeconds();`         | １５秒毎にタスク実行                   |
| `->everyTwentySeconds();`          | ２０秒ごとにタスク実行                 |
| `->everyThirtySeconds();`          | ３０秒ごとにタスク実行                 |
| `->everyMinute();`                 | 毎分タスク実行                         |
| `->everyTwoMinutes();`             | ２分毎にタスク実行                     |
| `->everyThreeMinutes();`           | ３分毎にタスク実行                     |
| `->everyFourMinutes();`            | ４分毎にタスク実行                     |
| `->everyFiveMinutes();`            | ５分毎にタスク実行                     |
| `->everyTenMinutes();`             | １０分毎にタスク実行                   |
| `->everyFifteenMinutes();`         | １５分毎にタスク実行                   |
| `->everyThirtyMinutes();`          | ３０分毎にタスク実行                   |
| `->hourly();`                      | 毎時タスク実行                         |
| `->hourlyAt(17);`                  | １時間ごと、毎時１７分にタスク実行     |
| `->everyOddHour($minutes = 0);`    | 奇数時間ごとにタスク実行               |
| `->everyTwoHours($minutes = 0);`   | ２時間毎にタスク実行                   |
| `->everyThreeHours($minutes = 0);` | ３時間毎にタスク実行                   |
| `->everyFourHours($minutes = 0);`  | ４時間毎にタスク実行                   |
| `->everySixHours($minutes = 0);`   | ６時間毎にタスク実行                   |
| `->daily();`                       | 毎日深夜１２時に実行                   |
| `->dailyAt('13:00');`              | 毎日13:00に実行                        |
| `->twiceDaily(1, 13);`             | 毎日1:00と13:00に実行                  |
| `->twiceDailyAt(1, 13, 15);`       | 毎日1:15と13:15に実行                  |
| `->weekly();`                      | 毎週日曜日の00:00にタスク実行          |
| `->weeklyOn(1, '8:00');`           | 毎週月曜日の8:00に実行                 |
| `->monthly();`                     | 毎月１日の00:00にタスク実行            |
| `->monthlyOn(4, '15:00');`         | 毎月4日の15:00に実行                   |
| `->twiceMonthly(1, 16, '13:00');`  | 毎月１日と１６日の13:00にタスク実行    |
| `->lastDayOfMonth('15:00');`       | 毎月最終日の15:00に実行                |
| `->quarterly();`                   | 四半期の初日の00:00にタスク実行        |
| `->quarterlyOn(4, '14:00');`       | 四半期の４日の14:00に実行              |
| `->yearly();`                      | 毎年１月１日の00:00にタスク実行        |
| `->yearlyOn(6, 1, '17:00');`       | 毎年６月１日の17:00にタスク実行        |
| `->timezone('America/New_York');`  | タスクのタイムゾーンを設定             |

</div>

これらの方法を追加の制約と組み合わせてると、特定の曜日にのみ実行する、さらに細かく調整したスケジュールを作成できます。たとえば、毎週月曜日に実行するようにコマンドをスケジュールできます。

    use Illuminate\Support\Facades\Schedule;

    // 週に１回、月曜の13:00に実行
    Schedule::call(function () {
        // ...
    })->weekly()->mondays()->at('13:00');

    // ウィークデーの８時から１７時まで１時間ごとに実行
    Schedule::command('foo')
              ->weekdays()
              ->hourly()
              ->timezone('America/Chicago')
              ->between('8:00', '17:00');

追加のスケジュール制約のリストを以下にリストします。

<div class="overflow-auto">

| メソッド                                 | 説明                                         |
| ---------------------------------------- | -------------------------------------------- |
| `->weekdays();`                          | ウィークデーのみに限定                       |
| `->weekends();`                          | ウィークエンドのみに限定                     |
| `->sundays();`                           | 日曜だけに限定                               |
| `->mondays();`                           | 月曜だけに限定                               |
| `->tuesdays();`                          | 火曜だけに限定                               |
| `->wednesdays();`                        | 水曜だけに限定                               |
| `->thursdays();`                         | 木曜だけに限定                               |
| `->fridays();`                           | 金曜だけに限定                               |
| `->saturdays();`                         | 土曜だけに限定                               |
| `->days(array\|mixed);`                  | 特定の日付だけに限定                         |
| `->between($startTime, $endTime);`       | 開始と終了時間間にタスク実行を制限           |
| `->unlessBetween($startTime, $endTime);` | 開始と終了時間間にタスクを実行しないよう制限 |
| `->when(Closure);`                       | クロージャの戻り値が`true`の時のみに限定     |
| `->environments($env);`                  | 指定の環境でのみタスク実行を限定             |

</div>

<a name="day-constraints"></a>
#### 曜日の限定

`days`メソッドはタスクを週の指定した曜日に実行するように制限するために使用します。たとえば、日曜日と水曜日に毎時コマンドを実行するようにスケジュールするには次のように指定します。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')
                    ->hourly()
                    ->days([0, 3]);

または、タスクを実行する日を定義するときに、`Illuminate\Console\Scheduling\Schedule`クラスで使用可能な定数を使用することもできます。

    use Illuminate\Support\Facades;
    use Illuminate\Console\Scheduling\Schedule;

    Facades\Schedule::command('emails:send')
                    ->hourly()
                    ->days([Schedule::SUNDAY, Schedule::WEDNESDAY]);

<a name="between-time-constraints"></a>
#### 時間制限

`between`メソッドは一日の時間に基づき、実行時間を制限するために使用します。

    Schedule::command('emails:send')
                        ->hourly()
                        ->between('7:00', '22:00');

同じように、`unlessBetween`メソッドは、その時間にタスクの実行を除外するために使用します。

    Schedule::command('emails:send')
                        ->hourly()
                        ->unlessBetween('23:00', '4:00');

<a name="truth-test-constraints"></a>
#### 論理テスト制約

`when`メソッドを使用して、特定の論理テストの結果に基づいてタスクの実行を制限できます。言い換えると、指定するクロージャが`true`を返す場合、他の制約条件がタスクの実行を妨げない限り、タスクは実行されます。

    Schedule::command('emails:send')->daily()->when(function () {
        return true;
    });

`skip`メソッドは`when`をひっくり返したものです。`skip`メソッドへ渡したクロージャが`true`を返した時、スケジュールタスクは実行されません。

    Schedule::command('emails:send')->daily()->skip(function () {
        return true;
    });

`when`メソッドをいくつかチェーンした場合は、全部の`when`条件が`true`を返すときのみスケジュールされたコマンドが実行されます。

<a name="environment-constraints"></a>
#### 環境制約

`environments`メソッドは、指定する環境でのみタスクを実行するために使用できます（`APP_ENV`[環境変数](/docs/{{version}}/configuration#environment-configuration)で定義されます。）

    Schedule::command('emails:send')
                ->daily()
                ->environments(['staging', 'production']);

<a name="timezones"></a>
### タイムゾーン

`timezone`メソッドを使い、タスクのスケジュールをどこのタイムゾーンとみなすか指定できます。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('report:generate')
             ->timezone('America/New_York')
             ->at('2:00')

すべてのスケジュールタスクに同じタイムゾーンを繰り返し割り当てている場合は、アプリケーションの`app`設定ファイルで`schedule_timezone`オプションを定義すれば、すべてのスケジュールに割り当てるタイムゾーンを指定できます。

    'timezone' => env('APP_TIMEZONE', 'UTC'),

    'schedule_timezone' => 'America/Chicago',

> [!WARNING]
> タイムゾーンの中には夏時間を取り入れているものがあることを忘れないでください。夏時間の切り替えにより、スケジュールしたタスクが２回実行されたり、まったくされないことがあります。そのため、可能であればタイムゾーンによるスケジュールは使用しないことを推奨します。

<a name="preventing-task-overlaps"></a>
### タスク多重起動の防止

デフォルトでは以前の同じジョブが起動中であっても、スケジュールされたジョブは実行されます。これを防ぐには、`withoutOverlapping`メソッドを使用してください。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')->withoutOverlapping();

この例の場合、`emails:send` [Artisanコマンド](/docs/{{version}}/artisan)は実行中でない限り毎分実行されます。`withoutOverlapping`メソッドは指定したタスクの実行時間の変動が非常に大きく、予想がつかない場合にとくに便利です。

必要であれば、「重起動の防止(without overlapping)」ロックを期限切れにするまでに、何分間経過させるかを指定できます。時間切れまでデフォルトは、２４時間です。

    Schedule::command('emails:send')->withoutOverlapping(10);

`withoutOverlapping`メソッドは、動作の裏でアプリケーションの[キャッシュ](/docs/{{version}}/cache)を利用してロックを取得します。必要であれば、`schedule:clear-cache` Artisanコマンドを使用して、これらのキャッシュ・ロックを解除できます。これは通常、予期しないサーバの問題でタスクがスタックした場合のみ必要です。

<a name="running-tasks-on-one-server"></a>
### 単一サーバ上でのタスク実行

> [!WARNING]
> この機能を利用するには、アプリケーションのデフォルトのキャッシュドライバとして`database`、`memcached`、`dynamodb`、`redis`キャッシュドライバを使用している必要があります。さらに、すべてのサーバが同じ中央キャッシュサーバと通信している必要があります。

アプリケーションのスケジューラを複数のサーバで実行する場合は、スケジュールしたジョブを単一のサーバでのみ実行するように制限できます。たとえば、毎週金曜日の夜に新しいレポートを生成するスケジュールされたタスクがあるとします。タスクスケジューラが３つのワーカーサーバで実行されている場合、スケジュールされたタスクは３つのサーバすべてで実行され、レポートを３回生成してしまいます。これは良くありません！

タスクをサーバひとつだけで実行するように指示するには、スケジュールタスクを定義するときに`onOneServer`メソッドを使用します。このタスクを最初に取得したサーバが、同じタスクを同じCronサイクルで他のサーバで実行しないように、ジョブにアトミックなロックを確保します。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('report:generate')
                    ->fridays()
                    ->at('17:00')
                    ->onOneServer();

<a name="naming-unique-jobs"></a>
#### サーバジョブに一意名を付ける

Laravelに対して単一サーバ上でジョブの各順列を実行するように指示しながら、同じジョブを異なるパラメータでディスパッチするようにスケジュールする必要がある場合があります。これを実現するには、`name`メソッドを使用して各スケジュール定義に一意の名前を割り当てます。

```php
Schedule::job(new CheckUptime('https://laravel.com'))
            ->name('check_uptime:laravel.com')
            ->everyFiveMinutes()
            ->onOneServer();

Schedule::job(new CheckUptime('https://vapor.laravel.com'))
            ->name('check_uptime:vapor.laravel.com')
            ->everyFiveMinutes()
            ->onOneServer();
```

同様に、１サーバで実行することを意図している場合、スケジュールするクロージャへ名前を割り当てる必要があります。

```php
Schedule::call(fn () => User::resetApiRequestCount())
    ->name('reset-api-request-count')
    ->daily()
    ->onOneServer();
```

<a name="background-tasks"></a>
### バックグランドタスク

デフォルトでは、同時にスケジュールされた複数のタスクは、`schedule`メソッドで定義された順序に基づいて順番に実行されます。長時間実行されるタスクがある場合、これにより、後続のタスクが予想よりもはるかに遅く開始される可能性があります。タスクをすべて同時に実行できるようにバックグラウンドで実行する場合は、`runInBackground`メソッドを使用できます。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('analytics:report')
             ->daily()
             ->runInBackground();

> [!WARNING]
> `runInBackground`メソッドは`command`か`exec`メソッドにより、タスクをスケジュールするときにのみ使用してください。

<a name="maintenance-mode"></a>
### メンテナンスモード

アプリケーションが[メンテナンスモード](/docs/{{version}}/configuration#maintenance-mode)の場合、アプリケーションのスケジュールされたタスクは実行されません。これは、タスクがそのサーバで実行している未完了のメンテナンスに干渉することを望まないためです。ただし、メンテナンスモードでもタスクを強制的に実行したい場合は、タスクを定義するときに`evenInMaintenanceMode`メソッドを呼び出すことができます。

    Schedule::command('emails:send')->evenInMaintenanceMode();

<a name="schedule-groups"></a>
### スケジュールグループ

似た設定を持つ複数のスケジュールタスクを定義する場合、Laravelのタスクグループ化機能を使用すると、各タスクで同じ設定を繰り返さずに済みます。タスクのグループ化により、コードがシンプルになり、関連するタスク間の一貫性が確保されます。

スケジュール済みタスクのグループを作成するには、必要なタスク設定メソッドを起動し、続いて `group` メソッドを起動します。`group`メソッドには、指定した設定を共有するタスクを定義するクロージャを渡します。

```php
use Illuminate\Support\Facades\Schedule;

Schedule::daily()
    ->onOneServer()
    ->timezone('America/New_York')
    ->group(function () {
        Schedule::command('emails:send --force');
        Schedule::command('emails:prune');
    });
```

<a name="running-the-scheduler"></a>
## スケジューラの実行

スケジュールするタスクを定義する方法を学習したので、サーバで実際にタスクを実行する方法について説明しましょう。`schedule:run` Artisanコマンドは、スケジュールしたすべてのタスクを評価し、サーバの現在の時刻に基づいてタスクを実行する必要があるかどうかを判断します。

したがって、Laravelのスケジューラを使用する場合、サーバに１分ごとに`schedule:run`コマンドを実行する単一のcron設定エントリを追加するだけで済みます。サーバにcronエントリを追加する方法がわからない場合は、[Laravel Forge](https://forge.laravel.com)などのcronエントリを管理できるサービスの使用を検討してください。

```shell
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

<a name="sub-minute-scheduled-tasks"></a>
### 秒単位のタスクスケジュール

ほとんどのオペレーティングシステムでは、cronジョブの実行は最短で１分に１回に制限されています。しかし、Laravelのスケジューラーを使えば、１秒に１回など、より細かい間隔でタスクを実行できます。

    use Illuminate\Support\Facades\Schedule;

    Schedule::call(function () {
        DB::table('recent_users')->delete();
    })->everySecond();

アプリケーション内で秒単位実行タスクが定義されている場合、`schedule:run`コマンドは実行後即時終了せず、現在の分が終了するまで実行し続けます。これにより、その分内で実行する必要がある、全ての秒単位実行タスクを起動します。

秒単位実行タスクの実行に予想以上の時間がかかると、後の秒単位実行タスクの実行が遅れる可能性があるため、すべての秒単位実行タスクは、キュー投入ジョブまたはバックグラウンドコマンドをディスパッチし、確実にタスクを処理できるようにすることを推奨します。

    use App\Jobs\DeleteRecentUsers;

    Schedule::job(new DeleteRecentUsers)->everyTenSeconds();

    Schedule::command('users:delete')->everyTenSeconds()->runInBackground();

<a name="interrupting-sub-minute-tasks"></a>
#### 秒単位のタスクの中断

`schedule:run`コマンドは秒単位実行タスクが定義されている場合、起動した分の間ずっと実行を持続するため、アプリケーションをデプロイするときにコマンドを中断する必要があるかもしれません。そうしないと、すでに実行されている`schedule:run`コマンドのインスタンスは現在の分が終了するまで、アプリケーションの以前にデプロイされたコードを使い続けることになります。

実行中の`schedule:run`コマンドを中断するには、アプリケーションのデプロイスクリプトへ、`schedule:interrupt`コマンドを追加します。このコマンドはアプリケーションのデプロイが終わった後に実行する必要があります：

```shell
php artisan schedule:interrupt
```

<a name="running-the-scheduler-locally"></a>
### スケジュールをローカルで実行

通常、ローカル開発マシンにスケジューラのcronエントリを追加することはありません。代わりに、`schedule:work` Artisanコマンドを使用できます。このコマンドはフォアグラウンドで実行し、コマンドを終了するまで１分ごとにスケジューラーを呼び出します。

```shell
php artisan schedule:work
```

<a name="task-output"></a>
## タスク出力

Laravelスケジューラはスケジュールしたタスクが生成する出力を取り扱う便利なメソッドをたくさん用意しています。最初に`sendOutputTo`メソッドを使い、後ほど内容を調べられるようにファイルへ出力してみましょう。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')
             ->daily()
             ->sendOutputTo($filePath);

出力を指定したファイルに追加したい場合は、`appendOutputTo`メソッドを使います。

    Schedule::command('emails:send')
             ->daily()
             ->appendOutputTo($filePath);

`emailOutputTo`メソッドを使用して、選択した電子メールアドレスへ出力を電子メールで送信できます。タスクの出力をメールで送信する前に、Laravelの[メールサービス](/docs/{{version}}/mail)を設定する必要があります。

    Schedule::command('report:generate')
             ->daily()
             ->sendOutputTo($filePath)
             ->emailOutputTo('taylor@example.com');

スケジュールしたArtisanまたはシステムコマンドが、ゼロ以外の終了コードで終了した場合にのみ出力を電子メールで送信する場合は、`emailOutputOnFailure`メソッドを使用します。

    Schedule::command('report:generate')
             ->daily()
             ->emailOutputOnFailure('taylor@example.com');

> [!WARNING]
> `emailOutputTo`、 `emailOutputOnFailure`、`sendOutputTo`、`appendOutputTo`メソッドは、`command`と`exec`メソッドに対してのみ指定できます。

<a name="task-hooks"></a>
## タスクフック

`before`および`after`メソッドを使用して、スケジュール済みのタスクを実行する前後に実行するコードを指定できます。

    use Illuminate\Support\Facades\Schedule;

    Schedule::command('emails:send')
             ->daily()
             ->before(function () {
                 // タスクが実行されようとしている
             })
             ->after(function () {
                 // タスクが実行された
             });

`onSuccess`メソッドと`onFailure`メソッドを使用すると、スケジュールされたタスクが成功または失敗した場合に実行されるコードを指定できます。失敗は、スケジュールされたArtisanまたはシステムコマンドがゼロ以外の終了コードで終了したことを示します。

    Schedule::command('emails:send')
             ->daily()
             ->onSuccess(function () {
                 // タスク成功時…
             })
             ->onFailure(function () {
                 // タスク失敗時…
             });

コマンドから出力を利用できる場合は、フックのクロージャの定義で`$output`引数として`Illuminate\Support\Stringable`インスタンスを型指定することで、`after`、`onSuccess`、または`onFailure`フックでアクセスできます。

    use Illuminate\Support\Stringable;

    Schedule::command('emails:send')
             ->daily()
             ->onSuccess(function (Stringable $output) {
                 // タスク成功時…
             })
             ->onFailure(function (Stringable $output) {
                 // タスク失敗時…
             });

<a name="pinging-urls"></a>
#### URLへのPing

`pingBefore`メソッドと`thenPing`メソッドを使用すると、スケジューラーはタスクの実行前または実行後に、指定するURLに自動的にpingを実行できます。このメソッドは、[Envoyer](https://envoyer.io)などの外部サービスに、スケジュールされたタスクが実行を開始または終了したことを通知するのに役立ちます。

    Schedule::command('emails:send')
             ->daily()
             ->pingBefore($url)
             ->thenPing($url);

`pingBeforeIf`および`thenPingIf`メソッドは、特定の条件が`true`である場合にのみ、特定のURLにpingを実行するために使用します。

    Schedule::command('emails:send')
             ->daily()
             ->pingBeforeIf($condition, $url)
             ->thenPingIf($condition, $url);

`pingOnSuccess`メソッドと`pingOnFailure`メソッドは、タスクが成功または失敗した場合にのみ、特定のURLにpingを実行するために使用します。失敗は、スケジュールされたArtisanまたはシステムコマンドがゼロ以外の終了コードで終了したことを示します。

    Schedule::command('emails:send')
             ->daily()
             ->pingOnSuccess($successUrl)
             ->pingOnFailure($failureUrl);

<a name="events"></a>
## イベント

Laravelは、スケジュールの処理中に様々な[イベント](/docs/{{version}}/events)をディスパッチします。以下のイベントに対して、リスナを[定義](/docs/{{version}}/events)できます。

<div class="overflow-auto">

| イベント名 |
| --- |
| `Illuminate\Console\Events\ScheduledTaskStarting` |
| `Illuminate\Console\Events\ScheduledTaskFinished` |
| `Illuminate\Console\Events\ScheduledBackgroundTaskFinished` |
| `Illuminate\Console\Events\ScheduledTaskSkipped` |
| `Illuminate\Console\Events\ScheduledTaskFailed` |

</div>
