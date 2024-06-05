# Kotlin製のWebフレームワークKtorでAPIサーバー作成

## Ktor（ケイター）とは

- Jetbrainsが開発したKotlin製のWebフレームワーク
- 認証、認可機能
- HTTPクライアント
- WebSocket
- 非同期通信
- ロギング
- テンプレート

## Ktorの導入

- 以下のいずれかの方法でプロジェクトを作成する
  - IntelliJ IDEAにKtorプラグインをインストール(Ultimate版のみ)
  - Ktor Project Generatorを使ってプロジェクトを作成
    - https://start.ktor.io/#/settings

## プロジェクト構造

- build.gradle.kts
  - Gradleのビルド設定ファイル
  - Ktorサーバーを起動するための `ktor-server-netty` の指定などが記述されている
- Application.kt
  - サーバーのエントリーポイント
  - `main` 関数でサーバーを起動する
  - 8080ポートでサーバーを起動する設定になっている

## ルーティングの追加

- 以下のように、`routing` ブロック内にルーティングを追加する
- `http://localhost:8080/` にアクセスすると、`Hello Ktor!` と表示される
- 一般的なルーティング例

```kotlin:src/Application.kt
fun Application.module(testing: Boolean = false) {
    routing {
        get("/") {
            call.respondText("Hello Ktor!")
        }
    }
}
```

- ルーティングを別の場所に切り出す例
- `greetingRoute()` でルーティングを定義し、routingブロック内で呼び出す

```kotlin
fun Routing.greetingRoute() {
    get("/") {
        call.respondText("Hello Ktor!")
    }
}
```

```kotlin
fun Application.module(testing: Boolean = false) {
    routing {
        greetingRoute()
    }
}
```

## リクエストパラメータの追加

### パスパラメータ

- パスパラメータを受け取る場合は、`{}` で囲んで名前を指定する
- `call.parameters["name"]` で値を取得できる
- `http://localhost:8080/hello/John` にアクセスすると、`Hello John!` と表示される

```kotlin:src/Application.kt
  get("/hello/{name}") {
      val name = call.parameters["name"]
      call.respondText("Hello $name!")
  }
```

### クエリストリング

- クエリストリングを受け取る場合は、`call.request.queryParameters` で取得できる
- `http://localhost:8080/hello?name=John` にアクセスすると、`Hello John!` と表示される

```kotlin:src/Application.kt
  get("/hello") {
      val name = call.request.queryParameters["name"]
      call.respondText("Hello $name!")
  }
```

### 型安全なパラメータ

- build.gradle.ktsに`implementation("io.ktor:ktor-locations:$ktor_version")` を追加
- Application.kt に `install(Locations)`  を追加
- データクラスに `@Location` アノテーションを付与、userRoute()の中でGetUserLocation型を受け取り、paramで値を取得する

```kotlin
@Location("/user/{id}")
data class GetUserLocation(val id: Long)
```

```kotlin
fun Routing.userRoute() {
    get<GetUserLocation> { param ->
        val id = param.id
        call.respondText("id=$id")
    }
}
```

- パスを共通化したい場合は、ネストしたクラスを作成する

```kotlin
@Location("/user")
class UserLocation {
  @Location("/{id}")
  data class GetLocation(val id: Long)

  @Location("/detail/{id}")
  data class GetDetailLocation(val id: Long)

  @Location("/authenticated")
  data class AuthenticatedLocation(val id: Long)
}
```

```kotlin
fun Routing.userRoute() {
    get<UserLocation.GetLocation> { param ->
        val id = param.id
        call.respondText("get id=$id")
    }

    get<UserLocation.GetDetailLocation> { param ->
        val id = param.id
        call.respondText("getDetail id=$id")
    }
}
```

## REST APIの実装

### Jacksonのフィーチャーを追加

- build.gradle.ktsに`implementation("io.ktor:ktor-jackson:$ktor_version")` を追加
- `install(ContentNegotiation)` でJacksonを有効化
  - `call.respond` でデータクラスを返すと、JSON形式で返却される
  - 以下のようにJackson処理のカスタマイズが可能

```kotlin
    install(ContentNegotiation) {
        jackson {
            // シリアライズ、デシリアライズ処理のカスタマイズを記述
        }
    }
```

### JSONでレスポンスを返却する

- リクエストは `BookLocation` で受け取り、レスポンスは `BookResponse` で返却する
- `call.respond(response)` でデータクラスを返却すると、JSON形式で返却される

```kotlin
fun Routing.bookRoute() {
    route("/book") {
        @Location("/detail/{bookId}")
        data class BookLocation(val bookId: Long)
        get<BookLocation> { request ->
            val response = BookResponse(request.bookId, "Kotlin入門", "Kotlin太郎")
            call.respond(response)
        }
    }
}

data class BookResponse(
  val id: Long,
  val title: String,
  val author: String
)
```

- `http://localhost:8080/book/detail/1` にアクセスすると、以下のJSONが返却される

```json
{
  "id": 1,
  "title": "Kotlin入門",
  "author": "Kotlin太郎"
}
```

### JSONでPOSTリクエストを送信する

- postメソッドでリクエストを受け取る
- `call.receive` でリクエストボディを受け取る

```kotlin
fun Routing.bookRoute() {
    route("/book") {
        post("/register") {
            val request = call.receive<RegisterRequest>()
            val response = RegisterResponse(request.id, request.title, request.author)
            call.respond(response)
        }
    }   
}

data class RegisterRequest(
  val id: Long,
  val title: String,
  val author: String
)

data class RegisterResponse(
  val id: Long,
  val title: String,
  val author: String
)
```

- `http://localhost:8080/book/register` にPOSTリクエストを送信すると、以下のJSONが返却される

```shell
curl -H 'Content-Type:application/json' -X POST -d '{"id": 200, "title": "Spring入門", "author": "ス プリング太郎"}' http://localhost:8080/book/register
{"id":200,"title":"Spring入門","author":"スプリング太郎"}
```

## 認証機構の実装

- 対応している認証
  - Basic認証
  - Form認証
  - Digest認証
  - JWT認証・JWK認証
  - LDAP認証
  - OAuth認証

### 認証のフィーチャーを追加

- build.gradle.ktsに`implementation("io.ktor:ktor-auth:$ktor_version")` を追加
- Application.kt に `install(Authentication)`  を追加
- 以下は、Basic認証で、ユーザー名が `user` 、パスワードが `password` の場合に認証を通過する例
- 成功した場合、`UserIdPrincipal(credentials.name)` で認証を通過したユーザー名を取得できる

```kotlin
fun Application.module(testing: Boolean = false) {
    install(Authentication) {
        basic {
            validate { credentials ->
                if (credentials.name == "user" && credentials.password == "password") {
                    UserIdPrincipal(credentials.name)
                } else {
                    null
                }
            }
        }
    }
}
```

### 認証の対象とするパスをルーティングに追加

- authenticateブロック内に認証の対象としたいパスを追加する
- 認証が通過した場合、`call.authentication.principal<UserIdPrincipal>()` で認証を通過したユーザー名を取得できる

```kotlin
routing {
    authenticate {
        get("/authenticated") {
            val user = call.authentication.principal<UserIdPrincipal>()
            call.respondText("authenticated id=${user?.name}")
        }
    }
}
```

### セッション情報の項目を変更

- `UserIdPrincipal` 以外の情報をセッション情報に追加する場合は、`Principal` インターフェースを実装したクラスを作成する

```kotlin
data class MyUserPrincipal(val id: Long, val name: String, val profile: String) : Principal
```



