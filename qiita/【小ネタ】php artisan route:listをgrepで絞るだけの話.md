## 投稿の経緯
先日、新人が`php artisan route:list`を実行して、延々とスクロールしていました。
「grep使えばいいよ」と伝えると、「これ神です…！」と感動していました。
それだけの話ですが、記事にしてみました。

## コマンドと実行結果
### 例：`api`を含むルートだけ表示

```bash
php artisan route:list | grep api
```
```
GET|HEAD  api/user .......... App\Http\Controllers\UserController@index
POST      api/login ......... App\Http\Controllers\AuthController@login
（他は表示されません）
```
> `api` が含まれる行だけが表示されます。

### 例：特定コントローラを含むルートだけ表示

```bash
php artisan route:list | grep UserController
```
```
GET|HEAD  users ............. App\Http\Controllers\UserController@index
POST      users ............. App\Http\Controllers\UserController@store
（他は表示されません）
```
> `UserController` が含まれる行だけが表示されます。
---

スクロールするより、パイプ（`|`）打った方が早いです。

新人の頃ってパイプ（`|`）使わないですよね…。
僕も初心者のころ使いどころがわからなかったな～と思い、記事にしてみました。
