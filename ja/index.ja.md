# goa の改善と改良
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0097a7" -->

\#goastudy

2017/06/16 @tchssk

---

## 自己紹介 😎
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0097a7" -->

@tchssk (Taichi Sasaki)

<img src="../team.jpg" style="width:6em; border:none">

<span style="color:#b2ebf2">2016 年 4 月から</span> goa のコントリビュータ

<span style="color:#b2ebf2">2017 年 2 月から</span> goa 開発チームのメンバ

---

## アジェンダ 📝
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0097a7" -->

今日は私の goa に関する活動についてお話しします。

- v1 への機能追加とバグ修正
- ドキュメント改善
- v2 の開発

---

## v1 への<br>機能追加 ✨<br>と<br>バグ修正 🐛
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0097a7" -->

---

### OptionalPayload DSL の追加
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0288d1" -->

[https://github.com/goadesign/goa/pull/507](https://github.com/goadesign/goa/pull/507)

(v1.0.0 から)

---

#### ペイロードとは何か
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

> アクションペイロードは HTTP リクエストボディのデータ構造を表す。

goa では以下のように `Payload` DSL を使うことができる。

```go
Payload(func() {
	Member("id")
	Required("id")
})
```

---

#### 問題点
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

```go
Action("update", func() {
	Routing(
		PUT("/:accountID"),
	)
	Params(func() {
		Param("accountID", Integer, "Account ID")
	})
	Payload(func() {
		Member("name")
		Required("name")
	})
})
```

上記のアクションにペイロードなしのリクエストを送った。

```sh
curl localhost:8080/1
```

ペイロードがなかったら `400 Bad Request` が返却されるべきに見えるが、そうならない。

---

#### goa の作者 @raphael が言った。
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

<img src="../raphael.jpg" style="width:6em; border:none; border-radius:50%">

> 必須フィールド付きのペイロードは必須ペイロードと同じではありません。

---

#### 必須ペイロードと任意ペイロード
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

| 必須ペイロード | 任意ペイロード |
| --- | --- |
| クライアントはペイロード付きのリクエストを送らなければならない。 | クライアントはペイロードなしのリクエストを送ることができる。 |
| ペイロードがなかったらサーバは `400 Bad Request` を返却する。 | ペイロードがなくてもサーバは `400 Bad Request` を返さない。 ペイロードがあればフィールドをバリデートする。 |

---

<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

<img src="../raphael.jpg" style="width:6em; border:none; border-radius:50%">

> 新しい DSL が必要です。

---

#### 実装
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

1. `ActionDefinition` 構造体に `PayloadOptional` フィールドを追加する。

    ```go
    PayloadOptional bool
    ```

2. `PayloadOptional` が `true` のときペイロードの nil チェックを生成するようにする。

3. `OptionalPayload` DSL を追加する。それが `PayloadOptional` を `true` にする。

    ```go
    func Payload(p interface{}, dsls ...func()) {
    	payload(false, p, dsls...)
    }

    func OptionalPayload(p interface{}, dsls ...func()) {
    	payload(true, p, dsls...)
    }
    ```

---

#### 使い方
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

- 必須ペイロードを定義するために `Payload` DSL を使うことができる。

    ```go
    Payload(func() {
        Member("id")
        Required("id")
    })
    ```

- また任意ペイロードを定義するために `OptionalPayload` DSL を使うことができる。

    ```go
    OptionalPayload(func() {
        Member("id")
        Required("id")
    })
    ```

---

<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

<img src="../raphael.jpg" style="width:6em; border:none; border-radius:50%">

この言葉と共にプルリクエストがマージされた。

> すばらしい、ありがとう！

彼はとても褒め上手。

---

<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

このように、プルリクエストの度に議論をしている。  
とても楽しいです 😆

---

### Swagger extensions のサポート
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0288d1" -->

[https://github.com/goadesign/goa/pull/779](https://github.com/goadesign/goa/pull/779)

(v1.1.0 から)

---

#### Swagger とは何か
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

> API のための世界で最もポピュラーなフレームワーク。

OpenAPI 仕様としても知られている。  
goa は Swagger のためのビルトインジェネレータを備えている。

---

#### Swagger extensions とは何か
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

> Swagger 定義の幾つかの場所に追加できるカスタムプロバティ

```json
{
   "swagger": "2.0",
   "info": {
      "title": "The virtual wine cellar",
      "description": "A basic example of goa",
      "x-api": {"foo": "bar"}
    }
  }
}
```

---

#### 実装
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

1. `Metadata` DSL を拡張し `swagger:extension` をサポート。

    ```go
    Metadata("swagger:extension:x-api", `{"foo":"bar"}`)
    ```

2. Swagger のための構造体に `Extensions` というフィールドを追加。

    ```go
    Extensions map[string]interface{} `json:"-"`
    ```

3. それぞれの構造体の `json.Marshaler` をオーバーライド。

    ```go
    type _Info Info

    func (i Info) MarshalJSON() ([]byte, error) {
        // marshalJSON merges i.Extensions to i.
        // _Info is used to avoid recursive call of json.Marshal().
        return marshalJSON(_Info(i), i.Extensions)
    }
    ```

---

#### ユースケース
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

いくつかのクラウドコンピューティングサービスで使うことができる。

- [Amazon API Gateway (Amazon Web Services)](https://aws.amazon.com/api-gateway/)
- [Cloud Endpoints (Google Cloud Platform](https://cloud.google.com/endpoints/)

goa 公式ブログでは Swagger extensions を使って Cloud Endpoints に認証を追加する方法を紹介している。

- [From Design To Production](https://goa.design/blog/002-endpoints/)   

---

### テストヘルパーで引数と同じ名前の変数使用を避ける
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0288d1" -->

[https://github.com/goadesign/goa/pull/1084](https://github.com/goadesign/goa/pull/1084)

(v1.2.0 から)

---

#### テストヘルパー
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

テストヘルパーはコントローラの各ケースのテストをする関数群。

```go
func ShowAccountOK(
    t goatest.TInterface,
    ctx context.Context,
    service *goa.Service,
    ctrl app.AccountController,
    accountID int
) (http.ResponseWriter, *app.Account) {
    // ...
}
```

上記の関数は以下のケースのテストに使える。

```go
func (b *AccountController) Show(c *app.ShowAccountContext) error {
	account, ok := b.db.GetAccount(c.AccountID)
	if !ok {
		return c.NotFound()
	}
	return c.OK(ToAccountMedia(&account))  // This case.
}
```

---

#### 問題点
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->


- テストヘルパーはいくつかのローカル変数を使う。
- テストヘルパーはコントローラのテストに必要な引数を持つ。
- それらの名前が重複する可能性があった。

---

#### 実装
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

1. テスト対象の名前を予約。

    - URL パスパラメータ
    - URL クエリパラメータ
    - HTTP ヘッダ
    - HTTP リクエストペイロード
    - HTTP レスポンスメディアタイプ

2. 予約された名前をエスケープするテンプレート関数を追加。

3. テンプレート実行時にテストヘルパー内のローカル変数をエスケープする。

---

### 今後リリースされる改善
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0288d1" -->

- CollectionOf() への引数の追加

    [https://github.com/goadesign/goa/pull/1232](https://github.com/goadesign/goa/pull/1232)

- Format DSL で RFC1123 をサポート

    [https://github.com/goadesign/goa/pull/1247](https://github.com/goadesign/goa/pull/1247)

(v1.3.0 から?)

---

## ドキュメント改善 📄
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0097a7" -->

---

### 公式ウェブサイトの日本語翻訳 🇯🇵
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0288d1" -->

(2016 年秋から 2017 年春)

---

#### 日本語翻訳
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

https://goa.design/ja/ で見れます。

---

#### 翻訳
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

https://github.com/goadesign-jp で作業。

- @ikawaha
- @tchssk

<img src="../jp.png" style="width:6em; border:none; background-color:#fff; padding:1em">

(私を含む) 日本人開発者に有用だと思います。

---

## v2 の開発 🔨
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0097a7" -->

---

### v2 は開発中 🚧
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#0288d1" -->

---

#### v2 の概要
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

- v2 は "design first" アプローチを維持。
- 複数トランスポートをサポート。
    - REST (v1 でサポートされている形式)
    - gRPC
- 新しい DSL 。
    - v1 と v2 の DSL に互換性はない。
- 複数トランスポートの開発が難しい。
- もう少し時間がかかる。

---

#### v1 のこれから
<!-- .slide: data-transition="fade" -->
<!-- .slide: data-background-color="#455a64" -->

- v1 は当分の間メンテナンスされる。
- まだ改善の余地がある。
- 以下の手段でフィードバックを送ることができます。
    - GitHub で Issue を開く。
    - Twitter の \#goadesign ハッシュタグ。
    - Gophers Slack の \#goa チャンネル。
    - Gophers Slack の \#goa-jp チャンネル。
