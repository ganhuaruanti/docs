# Laravel Socialite

- [簡介](#introduction)
- [升級 Socialite](#upgrading-socialite)
- [安裝](#installation)
- [設置](#configuration)
- [路由](#routing)
- [可選的參數](#optional-parameters)
- [存取範圍](#access-scopes)
- [無狀態認證](#stateless-authentication)
- [取得用戶細節](#retrieving-user-details)

<a name="introduction"></a>
## 簡介

除了傳統的使用表單進行認證，Laravel 還提供了 [Laravel Socialite](https://github.com/laravel/socialite)，可以簡單方便的搭配 OAuth 提供者進行認證。

Socialite 目前支援 Facebook，Twitter，LinkedIn，Google，GitHub，GitLab 和 Bitbucket 的認證。

> {tip} Adapters for other platforms are listed at the community driven [Socialite Providers](https://socialiteproviders.netlify.com/) website.

<a name="upgrading-socialite"></a>
## 升級 Socialite

要升級新版的 Socialite 之前，一定要詳細閱讀[升級指南](https://github.com/laravel/socialite/blob/master/UPGRADE.md)。

<a name="installation"></a>
## 安裝

要開始使用 Socialite，我們可以用 composer 在專案相依元件內增加此套件：

    composer require laravel/socialite

<a name="configuration"></a>
## 設置

使用 Socialite 之前，你還要加上所使用 OAuth 服務的相關認證。這些認證應該要放在 `config/services.php` 設置檔案內，並且根據專案的需求，用 `facebook`，`twitter`，`linkedin`，`google`，`github`，`gitlab` 或 `bitbucket` 這些 key 進行區分。例如：

    'github' => [
        'client_id' => env('GITHUB_CLIENT_ID'),
        'client_secret' => env('GITHUB_CLIENT_SECRET'),
        'redirect' => 'http://your-callback-url',
    ],

> {tip} If the `redirect` option contains a relative path, it will automatically be resolved to a fully qualified URL.

<a name="routing"></a>
## 路由

Next, you are ready to authenticate users! You will need two routes: one for redirecting the user to the OAuth provider, and another for receiving the callback from the provider after authentication. We will access Socialite using the `Socialite` facade:

    <?php

    namespace App\Http\Controllers\Auth;

    use Socialite;

    class LoginController extends Controller
    {
        /**
         * Redirect the user to the GitHub authentication page.
         *
         * @return \Illuminate\Http\Response
         */
        public function redirectToProvider()
        {
            return Socialite::driver('github')->redirect();
        }

        /**
         * Obtain the user information from GitHub.
         *
         * @return \Illuminate\Http\Response
         */
        public function handleProviderCallback()
        {
            $user = Socialite::driver('github')->user();

            // $user->token;
        }
    }

The `redirect` method takes care of sending the user to the OAuth provider, while the `user` method will read the incoming request and retrieve the user's information from the provider.

You will need to define routes to your controller methods:

    Route::get('login/github', 'Auth\LoginController@redirectToProvider');
    Route::get('login/github/callback', 'Auth\LoginController@handleProviderCallback');

<a name="optional-parameters"></a>
## 可選的參數

A number of OAuth providers support optional parameters in the redirect request. To include any optional parameters in the request, call the `with` method with an associative array:

    return Socialite::driver('google')
        ->with(['hd' => 'example.com'])
        ->redirect();

> {note} When using the `with` method, be careful not to pass any reserved keywords such as `state` or `response_type`.

<a name="access-scopes"></a>
## 存取範圍

Before redirecting the user, you may also add additional "scopes" on the request using the `scopes` method. This method will merge all existing scopes with the ones you supply:

    return Socialite::driver('github')
        ->scopes(['read:user', 'public_repo'])
        ->redirect();

You can overwrite all existing scopes using the `setScopes` method:

    return Socialite::driver('github')
        ->setScopes(['read:user', 'public_repo'])
        ->redirect();

<a name="stateless-authentication"></a>
## 無狀態認證

要取消狀態認證，你可以用 `stateless` 函式。這個函式在 API 需要加入社群網站認證時很有用：

    return Socialite::driver('google')->stateless()->user();

<a name="retrieving-user-details"></a>
## 取得用戶細節

一旦你有了用戶物件，你可以取得更多用戶的細節：

    $user = Socialite::driver('github')->user();

    // OAuth Two Providers
    $token = $user->token;
    $refreshToken = $user->refreshToken; // not always provided
    $expiresIn = $user->expiresIn;

    // OAuth One Providers
    $token = $user->token;
    $tokenSecret = $user->tokenSecret;

    // All Providers
    $user->getId();
    $user->getNickname();
    $user->getName();
    $user->getEmail();
    $user->getAvatar();

#### 從 Token 取得用戶細節 (OAuth2)

如果你有某個用戶的合法 token，你可以透過 `userFromToken` 取得用戶物件：

    $user = Socialite::driver('github')->userFromToken($token);
    
#### 從 Token 和 Secret 取得用戶細節(OAuth1)

如果你有某個用戶的合法 token 和 secret，你可以透過 `userFromTokenAndSecret` 取得用戶物件：

    $user = Socialite::driver('twitter')->userFromTokenAndSecret($token, $secret);
