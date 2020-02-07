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

在使用 Laravel 搭配 Redis Session 之前，你會需要透過 Composer 來安裝`predis/predis` 套件（~1.0）。For more information on configuring Redis, consult its [Laravel documentation page](/docs/{{version}}/redis#configuration).

> {tip} 在 `session` 設定檔中，`connection` 選項可被用於指定 Session 要使用哪一個 Redis 連線。

<a name="using-the-session"></a>
## 使用 Session

<a name="retrieving-data"></a>
### 取得資料

Laravel 有兩種主要的方式來使用 Session 資料：全域的 `session` 輔助函式和透過 `Request` 實例。首先，讓我們觀察透過 `Request` 實例來存取該 Session，這個實例能直接注入在控制器的方法上。但請你記得，控制器方法依賴是透過 Laravel [服務容器](/docs/{{version}}/container)來自動注入的：

    <?php

    namespace App\Http\Controllers;

    use App\Http\Controllers\Controller;
    use Illuminate\Http\Request;

    class UserController extends Controller
    {
        /**
         * 顯示特定使用者的個人資料。
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

當你從 Session 中取得一個值得時候，你也可以傳入一個預設的值作為 `get` 方法的第二個參數。如果 Session 中沒有指定的鍵，就會回傳這個預設值。如果你傳入一個`閉包`作為 `get` 方法的預設值，結果請求的值並未存在時，就會執行`閉包`與回傳它的結果：

    $value = $request->session()->get('key', 'default');

    $value = $request->session()->get('key', function () {
        return 'default';
    });

#### 全域 Session 輔助函式

你也可以使用全域的 `session` PHP 函式來取得，並儲存 Session 中的資料。如果只使用一個字串參數來呼叫 `session` 輔助函式的時候，它會回傳 Session 鍵的值。當使用一組陣列的鍵與值來呼叫輔助函式時，這些值會被存到 Session 中：

    Route::get('home', function () {
        // 從 Session 中取得一段資料...
        $value = session('key');

        // 指定一個預設值...
        $value = session('key', 'default');

        // 在 Session 中儲存一段資料...
        session(['key' => 'value']);
    });

> {tip} 不論是透過 HTTP 請求實例，還是使用全域的 `session` 輔助函式，這兩者之間並無實際上的差異。能透過 `assertSessionHas` 方法在所有測試案例中[測試](/docs/{{version}}/testing)這兩種方法。

#### 取得所有 Session 資料

如果你想要在 Session 中取得所有資料，你可以使用 `all` 方法：

    $data = $request->session()->all();

#### 確定項目是否存在於 Session

要確定一個值是否存在於 Session，你可以使用 `has` 方法。如果該值存在，`has` 方法就會回傳 `true`，如果沒有則回傳 `null`：

    if ($request->session()->has('users')) {
        //
    }

要確認一個值是否存在於 Session，縱使這個值是 `null`，你可以使用 `exists` 方法。`exists` 方法會在該值存在時回傳 `true`：

    if ($request->session()->exists('users')) {
        //
    }

<a name="storing-data"></a>
### 儲存資料

要在 Session 儲存資料，你通常會使用 `put` 方法或 `session` 輔助函式：

    // 透過一個請求實例...
    $request->session()->put('key', 'value');

    // 透過全域輔助函式...
    session(['key' => 'value']);

#### 將 Session 值推入陣列中

`push` 方法可被用於將一個新的值推入一組放置 Session 值的陣列。例如，如果 `user.teams` 鍵中有一組團隊名稱的陣列，你可以將一個新值推入陣列中，就像：

    $request->session()->push('user.teams', 'developers');

#### 取得與刪除一個項目

`pull` 方法只用一行句子就能從 Session 中取得和刪除一個項目：

    $value = $request->session()->pull('key', 'default');

<a name="flash-data"></a>
### 快閃資料

有時你可能希望只有在下一個請求中才將項目儲存到 Session 裡。你可以使用 `flash` 方法來做到。使用這個方法來將資料儲存到 Session，資料只會保留到下一個 HTTP 請求之前，然後就會清除這些資料。快閃資料主要用於短期的狀態訊息：

    $request->session()->flash('status', 'Task was successful!');

如果你需要保留快閃資料給更多的請求，你可以使用 `reflash` 方法，這方法會將所有快閃資料保留到額外的請求。如果你只需要指定快閃資料，你可以使用 `keep` 方法：

    $request->session()->reflash();

    $request->session()->keep(['username', 'email']);

<a name="deleting-data"></a>
### 刪除資料

`forget` 方法會從 Session 中移除資料片段。如果你想要從 Session 中移除所有資料，你可以使用 `flush` 方法：

    // 移除單一鍵值資料
    $request->session()->forget('key');

    // 移除多個鍵值資料
    $request->session()->forget(['key1', 'key2']);

    $request->session()->flush();

<a name="regenerating-the-session-id"></a>
### 重新產生 Session ID

Regenerating the session ID is often done in order to prevent malicious users from exploiting a [session fixation](https://en.wikipedia.org/wiki/Session_fixation) attack on your application.

Laravel automatically regenerates the session ID during authentication if you are using the built-in `LoginController`; however, if you need to manually regenerate the session ID, you may use the `regenerate` method.

    $request->session()->regenerate();

<a name="adding-custom-session-drivers"></a>
## 新增自訂的 Session 驅動

<a name="implementing-the-driver"></a>
#### 實作驅動

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
