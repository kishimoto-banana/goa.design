+++
title = "HTTP トランスポートマッピング"
weight = 2

[menu.main]
name = "HTTP トランスポートマッピング"
parent = "design"
+++

## ペイロードから HTTP リクエストへのマッピング

ペイロード型は、サービスメソッドへの引数として指定されたデータの形状を記述します。
HTTP トランスポート固有の DSL は、入力される HTTP リクエストからデータを構築する方法を定義します。

HTTP リクエストの状態は4つの異なる部分で構成されています：

* URL パスパラメーター（たとえばルート `/bottle/{id}` は`id` がパスパラメータであることを定義します）
* URL クエリ文字列パラメーター
* HTTP ヘッダ
* 最後に HTTP リクエストボディ

HTTP 式は、生成されたコードがリクエストをペイロード型にデコードする方法を指定します：

* `Param` 式は、パスもしくはクエリ文字列パラメータからロードされる値を定義します。
* `Header` 式は、HTTP ヘッダからロードされる値を定義します。
* `Body` 式は、リクエストボディからロードされる値を定義します。

次の2つのセクションでは、式について詳しく説明します。

生成されたコードは、ほとんどの場合に十分であるはずのデフォルトのデコーダー実装を提供することに注意してください。
しかしながら、（できればまれであってほしいですが）必要に応じてユーザー提供のデコーダーをプラグインすることもできます。

#### 非オブジェクト型を使用したペイロードのマッピング

ペイロード型が基本型（すなわち String, Integer, Float, Boolean, Byte 型のいずれか）、
配列型、マップ型であるとき、値は次の場所からロードされます：

* デザインで定義された最初の URL パスパラメータがあればそこから
* そうではなく、デザインで定義された最初のクエリ文字列パラメーターがあればそこから
* そうではなく、デザインで定義された最初のヘッダがあればそこから
* そうでなければ、ボディから

ただし次のような制限があります：

* パスパラメーターまたはヘッダの定義に使用できるのは、基本型か配列型のみです
* クエリ文字列パラメーターの定義に使用できるのは、基本型、配列型、およびマップ型のみです
* パスパラメーター、クエリ文字列パラメーター、またはヘッダの定義に使用される配列型およびマップ型は、
  それらの要素を基本型を使用して定義する必要があります

パスやヘッダにあらわれる配列は、カンマで区切った値で表現されます。

例：

* "識別子で GET" するシンプルな例。ここで識別子は整数。

```go
Method("show", func() {
    Payload(Int)
    HTTP(func() {
        GET("/{id}")
    })
})
```

| 生成されるメソッド | リクエストの例    | 対応する呼び出し     |
| ---------------- | --------------- | ------------------ |
| Show(int)        | GET /1          | Show(1)            |

* バルクの "識別子で DELETE" の例。ここで識別子は複数の文字列：

```go
Method("delete", func() {
    Payload(ArrayOf(String))
    HTTP(func() {
        DELETE("/{ids}")
    })
})
```

| 生成されるメソッド   | リクエストの例    | 対応する呼び出し             |
| ------------------ | --------------- | -------------------------- |
| Delete([]string)   | DELETE /a,b     | Delete([]string{"a", "b"}) |

> 前の例とこの例の両方で、パスパラメーターの名前が関係ないことに注意してください。

* クエリ文字列に配列が使われる例：

```go
Method("list", func() {
    Payload(ArrayOf(String))
    HTTP(func() {
        GET("")
        Param("filter")
    })
})
```

| 生成されるメソッド | リクエストの例            | 対応する呼び出し          |
| ---------------- | ----------------------- | ------------------------ |
| List([]string)   | GET /?filter=a&filter=b | List([]string{"a", "b"}) |

* ヘッダに浮動小数点が使われる例：

```go
Method("list", func() {
    Payload(Float32)
    HTTP(func() {
        GET("")
        Header("version")
    })
})
```

| 生成されるメソッド | リクエストの例        | 対応する呼び出し     |
| ---------------- | ------------------- | ------------------ |
| List(float32)    | GET / [version=1.0] | List(1.0)          |

* ボディにマップが使われる例：

```go
Method("create", func() {
    Payload(MapOf(String, Int))
    HTTP(func() {
        POST("")
    })
})
```

| 生成されるメソッド       | リクエストの例           | 対応する呼び出し                          |
| ---------------------- | ----------------------- | -------------------------------------- |
| Create(map[string]int) | POST / {"a": 1, "b": 2} | Create(map[string]int{"a": 1, "b": 2}) |

#### オブジェクト型を使用したペイロードのマッピング

HTTP 式は、ペイロードのオブジェクト型のアトリビュートが HTTP リクエストからどのようにロードされるかを記述します。
異なるアトリビュートは、リクエストの異なる部分からロードされるでしょう：
あるアトリビュートはリクエストパスからロードされ、あるものはクエリ文字列パラメーターからロードされ、その他はボディからロードされるなど。
同じ型の制限がパス、クエリ文字列、ヘッダのアトリビュートに適用されます
（パスとヘッダーを記述するアトリビュートは基本型もしくは基本型の配列でなければならず、
クエリ文字列パラメーターを記述するアトリビュートは基本型、基本型であるか基本型の配列かマップででなければなりません）。

`Body` 式を使うことで、リクエストボディを記述するペイロード型のアトリビュートを定義できます。
`Body` が省略された場合、ペイロード型を構成するすべてのアトリビュートで、パスパラメーター、クエリ文字列パラメーター、
あるいはヘッダーを定義するために使用されないすべてのアトリビュートが暗黙的にボディを記述します。

たとえば、ペイロードが与えられたとき：

```go
Method("create", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("name", String)
        Attribute("age", Int)
    })
})
```

次の HTTP 式により `id` アトリビュートはパスパラメータからロードされます、
さらに `name` と `age` はリクエストボディからロードされます：

```go
Method("create", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("name", String)
        Attribute("age", Int)
    })
    HTTP(func() {
        POST("/{id}")
    })
})
```

| 生成されるメソッド       | リクエストの例                    | 対応する呼び出し                                   |
| ---------------------- | ------------------------------- | ------------------------------------------------ |
| Create(*CreatePayload) | POST /1 {"name": "a", "age": 2} | Create(&CreatePayload{ID: 1, Name: "a", Age: 2}) |

`Body` を使用すると、配列やマップなどのオブジェクトではないリクエストボディを記述することができます。

次のようなペイロードを考えてみましょう：

```go
Method("rate", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("rates", MapOf(String, Float64))
    })
})
```

次の HTTP 式を使用すると、rates がボディからロードされます。

```go
Method("rate", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("rates", MapOf(String, Float64))
    })
    HTTP(func() {
        PUT("/{id}")
        Body("rates")
    })
})
```

| 生成されるメソッド   | リクエストの例                | 対応する呼び出し                                                           |
| ------------------ | --------------------------- | ------------------------------------------------------------------------ |
| Rate(*RatePayload) | PUT /1 {"a": 0.5, "b": 1.0} | Rate(&RatePayload{ID: 1, Rates: map[string]float64{"a": 0.5, "b": 1.0}}) |

`Body` を使用しなければ、リクエストボディの形状は、`rates` を単一のフィールドとしてもつオブジェクトになります。

#### HTTP の要素名からアトリビュート名へのマッピング

HTTP リクエスト要素 `Param`、`Header`、および `Body` を記述するために使用される式は、
要素の名前（クエリ文字列のキー、ヘッダー名、またはボディのフィールド名）と
対応するペイロードのアトリビュート名の間にマッピングを提供する場合があります。
このマッピングは、`"attribute name:element name"` の構文を用いて定義されます。
次に例を示します：

```go
Header("version:X-Api-Version")
```

上の例では、`version` アトリビュートの値は `X-Api-Version` HTTP ヘッダからロードされます。

`Body` 式は、ボディを構成するアトリビュートを明示的に列挙できる代替構文をサポートしています。
この構文により、次のように、入力データフィールド名とペイロードアトリビュート名の間のマッピングを指定できます。
次に例を示します：

```go
Method("create", func() {
    Payload(func() {
        Attribute("name", String)
        Attribute("age", Int)
    })
    HTTP(func() {
        POST("")
        Body(func() {
            Attribute("name:n")
            Attribute("age:a")
        })
    })
})
```

| 生成されるメソッド       | リクエストの例               | 対応する呼び出し                                   |
| ---------------------- | -------------------------- | ------------------------------------------------ |
| Create(*CreatePayload) | POST /1 {"n": "a", "a": 2} | Create(&CreatePayload{ID: 1, Name: "a", Age: 2}) |