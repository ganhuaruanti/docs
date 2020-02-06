# HTTP Session

- [介紹](#introduction)
    - [設定](#configuration)
    - [驅動需求](#driver-prerequisites)
- [使用 Session](#using-the-session)
    - [取得資料](#retrieving-data)
    - [儲存資料](#storing-data)
    - [快閃資料](#flash-data)
    - [刪除資料](#deleting-data)
    - [重新產生 Session ID](#regenerating-the-session-id)
- [新增自訂的 Session 驅動](#adding-custom-session-drivers)
    - [實作驅動](#implementing-the-driver)
    - [註冊驅動](#registering-the-driver)

<a name="introduction"></a>
## 介紹

由於 HTTP 驅動的應用程式是無狀態的，因此 Session 提供一種跨越多個請求來儲存關於使用者資訊的方法。Laravel 內建使用直觀且一致的 API 來存取各種 Session 後端，並支援目前較熱門的後端驅動，像是 [Memcached](https://memcached.org) 、 [Redis](https://redis.io) 和資料庫。

<a name="configuration"></a>
### 設定

Session 設定檔被存放在 `config/session.php`。請確實的查看檔案中的可用選項。預設的 Laravel 被設定使用 `file` Session 驅動，這對於許多應用程式來說算是不錯。By default, Laravel is configured to use the `cookie` session driver, which will work well for many applications.

Session `driver` 設定選項用來定義每個請求所儲存的 Session 資料位置。Laravel 內建幾個很棒的驅動，且能馬上應用：

<div class="content-list" markdown="1">
- `file` - Session 被儲存到 `storage/framework/sessions` 中。
- `cookie` - Session 被儲存到安全且被加密的 Cookie 中。
- `database` - Session 被儲存到關聯資料庫中。
- `memcached` / `redis` - Session 被儲存到其中一個快速且基於快取的儲存系統中。
- `array` - Session 被儲存到一個 PHP 陣列中且不會被保留。
</div>

> {tip} 陣列式驅動被用於[測試](/docs/{{version}}/testing)的時候，並防止 Session 中的資料被永久儲存。

<a name="driver-prerequisites"></a>
### 驅動需求

#### 資料庫

在使用 `database` Session 驅動時，你會需要建立一個資料表來放置 Session 項目。以下是使用 `Schema` 建立資料表的範例：

    Schema::create('sessions', function ($table) {
        $table->string('id')->unique();
        $table->unsignedInteger('user_id')->nullable();
        $table->string('ip_address', 45)->nullable();
        $table->text('user_agent')->nullable();
        $table->text('payload');
        $table->integer('last_activity');
    });

你可以使用 Artisan 的 `session:table` 指令來產生這個遷移檔：

    php artisan session:table

    php artisan migrate

#### Redis

Before using Redis sessions with Laravel, you will need to either install the PhpRedis PHP extension via PECL or install the `predis/predis` package (~1.0) via Composer. For more information on configuring Redis, consult its [Laravel documentation page](/docs/{{version}}/redis#configuration).

> {tip} In the `session` configuration file, the `connection` option may be used to specify which Redis connection is used by the session.

<a name="using-the-session"></a>
## Using The Session

<a name="retrieving-data"></a>
### Retrieving Data

There are two primary ways of working with session data in Laravel: the global `session` helper and via a `Request` instance. First, let's look at accessing the session via a `Request` instance, which can be type-hinted on a controller method. Remember, controller method dependencies are automatically injected via the Laravel [service container](/docs/{{version}}/container):

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;

    class UserController extends Controller
    {
        /**
         * Show the profile for the given user.
         *
         * @param  Request  $request
         * @param  int  $id
         * @return Response
         */
        public function show(Request $request, $id)
        {
            $value = $request->session()->get('key');

            //
        }
    }

When you retrieve an item from the session, you may also pass a default value as the second argument to the `get` method. This default value will be returned if the specified key does not exist in the session. If you pass a `Closure` as the default value to the `get` method and the requested key does not exist, the `Closure` will be executed and its result returned:

    $value = $request->session()->get('key', 'default');

    $value = $request->session()->get('key', function () {
        return 'default';
    });

#### The Global Session Helper

You may also use the global `session` PHP function to retrieve and store data in the session. When the `session` helper is called with a single, string argument, it will return the value of that session key. When the helper is called with an array of key / value pairs, those values will be stored in the session:

    Route::get('home', function () {
        // Retrieve a piece of data from the session...
        $value = session('key');

        // Specifying a default value...
        $value = session('key', 'default');

        // Store a piece of data in the session...
        session(['key' => 'value']);
    });

> {tip} There is little practical difference between using the session via an HTTP request instance versus using the global `session` helper. Both methods are [testable](/docs/{{version}}/testing) via the `assertSessionHas` method which is available in all of your test cases.

#### Retrieving All Session Data

If you would like to retrieve all the data in the session, you may use the `all` method:

    $data = $request->session()->all();

#### Determining If An Item Exists In The Session

To determine if an item is present in the session, you may use the `has` method. The `has` method returns `true` if the item is present and is not `null`:

    if ($request->session()->has('users')) {
        //
    }

To determine if an item is present in the session, even if its value is `null`, you may use the `exists` method. The `exists` method returns `true` if the item is present:

    if ($request->session()->exists('users')) {
        //
    }

<a name="storing-data"></a>
### Storing Data

To store data in the session, you will typically use the `put` method or the `session` helper:

    // Via a request instance...
    $request->session()->put('key', 'value');

    // Via the global helper...
    session(['key' => 'value']);

#### Pushing To Array Session Values

The `push` method may be used to push a new value onto a session value that is an array. For example, if the `user.teams` key contains an array of team names, you may push a new value onto the array like so:

    $request->session()->push('user.teams', 'developers');

#### Retrieving & Deleting An Item

The `pull` method will retrieve and delete an item from the session in a single statement:

    $value = $request->session()->pull('key', 'default');

<a name="flash-data"></a>
### Flash Data

Sometimes you may wish to store items in the session only for the next request. You may do so using the `flash` method. Data stored in the session using this method will be available immediately and during the subsequent HTTP request. After the subsequent HTTP request, the flashed data will be deleted. Flash data is primarily useful for short-lived status messages:

    $request->session()->flash('status', 'Task was successful!');

If you need to keep your flash data around for several requests, you may use the `reflash` method, which will keep all of the flash data for an additional request. If you only need to keep specific flash data, you may use the `keep` method:

    $request->session()->reflash();

    $request->session()->keep(['username', 'email']);

<a name="deleting-data"></a>
### Deleting Data

The `forget` method will remove a piece of data from the session. If you would like to remove all data from the session, you may use the `flush` method:

    // Forget a single key...
    $request->session()->forget('key');

    // Forget multiple keys...
    $request->session()->forget(['key1', 'key2']);

    $request->session()->flush();

<a name="regenerating-the-session-id"></a>
### Regenerating The Session ID

Regenerating the session ID is often done in order to prevent malicious users from exploiting a [session fixation](https://en.wikipedia.org/wiki/Session_fixation) attack on your application.

Laravel automatically regenerates the session ID during authentication if you are using the built-in `LoginController`; however, if you need to manually regenerate the session ID, you may use the `regenerate` method.

    $request->session()->regenerate();

<a name="adding-custom-session-drivers"></a>
## Adding Custom Session Drivers

<a name="implementing-the-driver"></a>
#### Implementing The Driver

Your custom session driver should implement the `SessionHandlerInterface`. This interface contains just a few simple methods we need to implement. A stubbed MongoDB implementation looks something like this:

    <?php

    namespace App\Extensions;

    class MongoSessionHandler implements \SessionHandlerInterface
    {
        public function open($savePath, $sessionName) {}
        public function close() {}
        public function read($sessionId) {}
        public function write($sessionId, $data) {}
        public function destroy($sessionId) {}
        public function gc($lifetime) {}
    }

> {tip} Laravel does not ship with a directory to contain your extensions. You are free to place them anywhere you like. In this example, we have created an `Extensions` directory to house the `MongoSessionHandler`.

Since the purpose of these methods is not readily understandable, let's quickly cover what each of the methods do:

<div class="content-list" markdown="1">
- The `open` method would typically be used in file based session store systems. Since Laravel ships with a `file` session driver, you will almost never need to put anything in this method. You can leave it as an empty stub. It is a fact of poor interface design (which we'll discuss later) that PHP requires us to implement this method.
- The `close` method, like the `open` method, can also usually be disregarded. For most drivers, it is not needed.
- The `read` method should return the string version of the session data associated with the given `$sessionId`. There is no need to do any serialization or other encoding when retrieving or storing session data in your driver, as Laravel will perform the serialization for you.
- The `write` method should write the given `$data` string associated with the `$sessionId` to some persistent storage system, such as MongoDB, Dynamo, etc.  Again, you should not perform any serialization - Laravel will have already handled that for you.
- The `destroy` method should remove the data associated with the `$sessionId` from persistent storage.
- The `gc` method should destroy all session data that is older than the given `$lifetime`, which is a UNIX timestamp. For self-expiring systems like Memcached and Redis, this method may be left empty.
</div>

<a name="registering-the-driver"></a>
#### Registering The Driver

Once your driver has been implemented, you are ready to register it with the framework. To add additional drivers to Laravel's session backend, you may use the `extend` method on the `Session` [facade](/docs/{{version}}/facades). You should call the `extend` method from the `boot` method of a [service provider](/docs/{{version}}/providers). You may do this from the existing `AppServiceProvider` or create an entirely new provider:

    <?php

    namespace App\Providers;

    use App\Extensions\MongoSessionHandler;
    use Illuminate\Support\Facades\Session;
    use Illuminate\Support\ServiceProvider;

    class SessionServiceProvider extends ServiceProvider
    {
        /**
         * Register any application services.
         *
         * @return void
         */
        public function register()
        {
            //
        }

        /**
         * Bootstrap any application services.
         *
         * @return void
         */
        public function boot()
        {
            Session::extend('mongo', function ($app) {
                // Return implementation of SessionHandlerInterface...
                return new MongoSessionHandler;
            });
        }
    }

Once the session driver has been registered, you may use the `mongo` driver in your `config/session.php` configuration file.
