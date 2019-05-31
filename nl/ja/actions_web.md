---

copyright:
  years: 2017, 2019
lastupdated: "2019-05-16"

keywords: web actions, serverless

subcollection: cloud-functions

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:pre: .pre}
{:table: .aria-labeledby="caption"}
{:codeblock: .codeblock}
{:tip: .tip}
{:note: .note}
{:important: .important}
{:deprecated: .deprecated}
{:download: .download}
{:gif: data-image-type='gif'}


# Web アクションの作成
{: #actions_web}

Web アクションは、開発者が Web ベースのアプリケーションを迅速に構築できるようにするためにアノテーションが付けられた {{site.data.keyword.openwhisk}} アクションです。 アノテーションが付けられたこれらのアクションにより、{{site.data.keyword.openwhisk_short}} 認証キーを必要とせずに Web アプリケーションが匿名でアクセスできるバックエンド・ロジックを、開発者がプログラムできます。 アクション開発者は自身の判断で、必要とする独自の認証および許可 (OAuth フロー) を実装します。
{: shortdesc}

Web アクションのアクティベーションは、アクションを作成したユーザーと関連付けられます。 このアクションでは、アクションのアクティベーションのコストが、呼び出し元からアクションの所有者に移ります。

以下に、JavaScript アクション `hello.js` の例を示します。
```javascript
function main({name}) {
  var msg = 'you did not tell me who you are.';
  if (name) {
    msg = `hello ${name}!`
  }
  return {body: `<html><body><h3>${msg}</h3></body></html>`}
}
```
{: codeblock}

以下のように、CLI の `--web` フラグを値 `true` または `yes` で指定し、_Web アクション_ **hello** を、名前空間 `guest` のパッケージ `demo` 内に作成できます。
```
ibmcloud fn package create demo
```
{: pre}

```
ibmcloud fn action create /guest/demo/hello hello.js --web true
```
{: pre}

値 `true` または `yes` の `--web` フラグを使用すると、資格情報を必要とせずに、REST インターフェース経由でアクションにアクセスできます。 資格情報を伴う Web アクションを構成するには、『[Web アクションの保護](#actions_web_secure)』セクションを参照してください。 Web アクションは、以下のように構造化された URL を使用して呼び出すことができます。
`https://{APIHOST}/api/v1/web/{namespace}/{packageName}/{actionName}.{EXT}`

アクションが名前付きパッケージ内にない場合、パッケージ名は **default** になります。

例えば、`guest/demo/hello` などです。 API キーなしで Web アクション API パスを `curl` または`wget` で使用できます。 ブラウザーに直接入力することも可能です。

`https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane` を、ご使用の Web ブラウザーで開いてみてください。 あるいは、次のように `curl` を使用してこのアクションを呼び出してみてください。
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane
```
{: pre}

次の例では、Web アクションが HTTP リダイレクトを実行します。
```javascript
function main() {
  return {
    headers: { location: 'http://openwhisk.org' },
    statusCode: 302
  }
}
```
{: codeblock}

次の例では、Web アクションが単一の Cookie を設定します。
```javascript
function main() {
  return {
    headers: {
      'Set-Cookie': 'UserID=Jane; Max-Age=3600; Version=',
      'Content-Type': 'text/html'
    },
    statusCode: 200,
    body: '<html><body><h3>hello</h3></body></html>' }
}
```
{: codeblock}

次の例では、Web アクションが複数の Cookie を設定します。
```javascript
function main() {
  return {
    headers: {
      'Set-Cookie': [
        'UserID=Jane; Max-Age=3600; Version=',
        'SessionID=asdfgh123456; Path = /'
      ],
      'Content-Type': 'text/html'
    },
    statusCode: 200,
    body: '<html><body><h3>hello</h3></body></html>' }
}
```
{: codeblock}

次の例は、`image/png` を返します。
```javascript
function main() {
    let png = <base 64 encoded string>
    return { headers: { 'Content-Type': 'image/png' },
             statusCode: 200,
             body: png };
}
```
{: codeblock}

次の例は、`application/json` を返します。
```javascript
function main(params) {
    return {
        statusCode: 200,
        headers: { 'Content-Type': 'application/json' },
        body: params
    };
}
```
{: codeblock}

HTTP 応答のデフォルトの `Content-Type` は `application/json` であり、本体は、許可される任意の JSON 値にすることができます。デフォルトの `Content-Type` はヘッダーから省略できます。

事前定義されたシステム限度を超える応答は失敗するため、アクションの[応答サイズ制限](/docs/openwhisk?topic=cloud-functions-limits)を知っておくことが重要です。例えば、ラージ・オブジェクトは {{site.data.keyword.openwhisk_short}} を通じてインラインで送信するのではなく、オブジェクト・ストアに置きます。

## アクションによる HTTP 要求の処理
{: #actions_web_http}

Web アクションでない {{site.data.keyword.openwhisk_short}} アクションは、認証を必要とし、JSON オブジェクトで応答することも必要です。

Web アクションは認証なしで起動でき、さまざまなタイプの `headers`、`statusCode`、および `body` コンテンツで応答する HTTP ハンドラーを実装するために使用できます。
Web アクションは JSON オブジェクトを返す必要があります。 ただし、コントローラーは、結果に以下の 1 つ以上が最上位 JSON プロパティーとして含まれる場合、Web アクションを別の方法で処理します。

- `headers`: キーがヘッダー名であり、値がストリング値、数値、またはブール値である JSON オブジェクト。 1 つのヘッダーで複数の値を送信する場合、ヘッダーの値は、複数の値の JSON 配列になります。 デフォルトで設定されるヘッダーはありません。
- `statusCode`: 有効な HTTP 状況コード。 本体コンテンツが存在する場合、デフォルトは `200 OK` です。 本体コンテンツが存在しない場合、デフォルトは `204 No Content` です。
- `body`: プレーン・テキスト、JSON オブジェクトや配列、または Base64 エンコードのストリング (バイナリー・データの場合) のいずれかであるストリング。 body が `null`、空ストリング `""`、または未定義の場合、本体は空であると見なされます。 デフォルトは空の本体です。

アクションで指定されたヘッダー、状況コード、または本体があれば、コントローラーはそれを、要求や応答を終了する HTTP クライアントに渡します。 アクション結果の `headers` で `Content-Type` ヘッダーが宣言されていない場合、本体は、非ストリング値であれば `application/json`、その他の場合は `text/html` として解釈されます。 `Content-Type` ヘッダーが定義されている場合、コントローラーは応答がバイナリー・データなのかプレーン・テキストなのかを判別し、必要に応じて base64 デコーダーを使用してストリングをデコードします。 本体が正しくデコードされていない場合は、クライアントにエラーが返されます。


## HTTP コンテキスト
{: #actions_web_context}

すべての Web アクションは、起動されると、アクション入力引数へのパラメーターとして、HTTP 要求の詳細を受け取ります。

以下の HTTP パラメーターを参照してください。

- `__ow_method` (タイプ: ストリング): 要求の HTTP メソッド。
- `__ow_headers` (タイプ: ストリング間マップ): 要求ヘッダー。
- `__ow_path` (タイプ: ストリング): マッチングされていない要求パス (マッチングは、アクションの拡張子が取り込まれると停止します)。
- `__ow_user` (タイプ: ストリング): {{site.data.keyword.openwhisk_short}} 認証済みサブジェクトを識別する名前空間
- `__ow_body` (タイプ: ストリング): コンテンツがバイナリーの場合は base64 エンコード・ストリング、それ以外の場合はプレーン・ストリングの要求本体エンティティー。
- `__ow_query` (タイプ: ストリング): 構文解析されていないストリングである、要求からの照会パラメーター。

要求は、指定された `__ow_` パラメーターのいずれもオーバーライドできません。 これを行うと、要求は失敗し、状況は 400 の Bad Request になります。

`__ow_user` は、Web アクションが[認証を必要とするものとしてアノテーションが付けられている](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions)場合にのみ存在し、これにより、Web アクションは独自の許可ポリシーを実装することができます。 `__ow_query` は、Web アクションが[「未加工」 HTTP 要求 ](#actions_web_raw_enable)を処理することを選択した場合にのみ使用可能です。 これは、URI からの解析される照会パラメーターを含むストリングです (`&` で分離)。 `__ow_body` プロパティーは、「未加工」HTTP 要求の中、または HTTP 要求エンティティーが JSON オブジェクトでも形式データでもない場合に存在します。 それ以外の場合、Web アクションは、アクション引数内の第 1 クラス・プロパティーとして、照会および本体のパラメーターを受け取ります。 本体パラメーターは照会パラメーターより優先され、照会パラメーターはアクション・パラメーターおよびパッケージ・パラメーターより優先されます。

## HTTPS エンドポイント・サポート
{: #actions_web_endpoint}

サポートされる SSL プロトコル: TLS 1.2、TLS 1.3 ([ドラフト・バージョン 18](https://tools.ietf.org/html/draft-ietf-tls-tls13-18))

サポートされない SSL プロトコル: SSLv2、SSLv3、TLS 1.0、TLS 1.1

## 追加の機能
{: #actions_web_extra}

Web アクションは、以下を含む追加機能を提供します。

- `コンテンツ拡張子`: 要求は、その要求にとって望ましいコンテンツ・タイプを、`.json`、`.html`、`.http`、`.svg`、または `.text` として指定する必要があります。 タイプは、URI 内のアクション名に拡張子を追加することで指定されます。これによって、例えば、アクション `/guest/demo/hello` は `/guest/demo/hello.http` として示され、HTTP 応答を受け取ります。 便宜上、拡張子が検出されない場合は `.http` 拡張子が想定されます。
- `結果からのフィールドの射影`: アクション名に続くパスを使用して、応答の 1 つ以上のレベルを射影します。
`/guest/demo/hello.html/body`。 この機能により、ディクショナリー `{body: "..." }` を返すアクションは `body` プロパティーを射影して、代わりにそのストリング値を直接戻すことができます。 射影パスは、絶対パス・モデル (例: XPath) に従います。
- `入力としての照会および本体のパラメーター`: アクションは、照会パラメーターと要求本体のパラメーターを受け取ります。 パラメーターをマージするときの優先順位は、パッケージ・パラメーター、アクション・パラメーター、照会パラメーター、および本体パラメーターです。 これらの各パラメーターは、オーバーラップが発生すると、前の値をオーバーライドできます。 例えば、`/guest/demo/hello.http?name=Jane` は、引数 `{name: "Jane"}` をアクションに渡します。
- `形式データ`: 標準 `application/json` に加えて、Web アクションは、入力としてデータ `application/x-www-form-urlencoded data` から、エンコードされた URL を受け取ることができます。
- `複数の HTTP 動詞を使用するアクティベーション`: Web アクションは、HTTP メソッド (`GET`、`POST`、`PUT`、`PATCH`、および `DELETE`、ならびに `HEAD` および `OPTIONS`) のどれを通しても起動できます。
- `非 JSON 本体および未加工 HTTP エンティティーの処理`: Web アクションは、JSON オブジェクト以外の HTTP 要求本体を受け入れることができ、不透明値のような値 (バイナリーではない場合はプレーン・テキスト、バイナリーの場合は base64 エンコード・ストリング) を常に受け取ることを選択できます。

以下の例では、Web アクションでこれらの機能を使用する方法を簡単に説明します。 以下の本体を持つアクション `/guest/demo/hello` を考えてみます。
```javascript
function main(params) {
    return { response: params };
}
```

このアクションが Web アクションとして呼び出されるときに、結果と異なるパスを射影することによって、Web アクションの応答を変更できます。

例えば、オブジェクト全体を返し、アクションが受け取る引数を確認するには、以下のようにします。
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json
```
{: pre}

出力例:
```
{
  "response": {
    "__ow_method": "get",
    "__ow_headers": {
      "accept": "*/*",
      "connection": "close",
      "host": "172.17.0.1",
      "user-agent": "curl/7.43.0"
    },
    "__ow_path": ""
  }
}
```
{: screen}

照会パラメーターで実行するには、以下のコマンド例を参照してください。
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane
 ```
{: pre}

出力例:
```
{
  "response": {
    "name": "Jane",
    "__ow_method": "get",
    "__ow_headers": {
      "accept": "*/*",
      "connection": "close",
      "host": "172.17.0.1",
      "user-agent": "curl/7.43.0"
    },
    "__ow_path": ""
  }
}
```
{: screen}

次のように形式データを使用して実行することもできます。
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -d "name":"Jane"
 ```
{: pre}

出力例:
```
{
  "response": {
    "name": "Jane",
    "__ow_method": "post",
    "__ow_headers": {
      "accept": "*/*",
      "connection": "close",
      "content-length": "10",
      "content-type": "application/x-www-form-urlencoded",
      "host": "172.17.0.1",
      "user-agent": "curl/7.43.0"
    },
    "__ow_path": ""
  }
}
```
{: screen}

JSON オブジェクトの場合、以下のコマンドを実行します。
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: application/json' -d '{"name":"Jane"}'
```
{: pre}

出力例:
```
{
  "response": {
    "name": "Jane",
    "__ow_method": "post",
    "__ow_headers": {
      "accept": "*/*",
      "connection": "close",
      "content-length": "15",
      "content-type": "application/json",
      "host": "172.17.0.1",
      "user-agent": "curl/7.43.0"
    },
    "__ow_path": ""
  }
}
```
{: screen}

名前を (テキストとして) 射影するには、以下のコマンドを実行します。
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.text/response/name?name=Jane
```
{: pre}

出力例:
```
Jane
```
{: screen}

便宜上、照会パラメーター、形式データ、および JSON オブジェクト本体エンティティーはすべてディクショナリーとして扱われ、それぞれの値は、アクション入力プロパティーとして直接アクセス可能です。 より直接的に HTTP 要求エンティティーを処理しようとする Web アクションの場合、あるいは Web アクションが JSON オブジェクトではないエンティティーを受け取る場合には、この動作は当てはまりません。

前に示したように、「text」の content-type を使用した以下の例を参照してください。
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: text/plain' -d "Jane"
```
{: pre}

出力例:
```
{
  "response": {
    "__ow_method": "post",
    "__ow_headers": {
      "accept": "*/*",
      "connection": "close",
      "content-length": "4",
      "content-type": "text/plain",
      "host": "172.17.0.1",
      "user-agent": "curl/7.43.0"
    },
    "__ow_path": "",
    "__ow_body": "Jane"
  }
}
```
{: screen}

## コンテンツ拡張子
{: #actions_web_ext}

Web アクションを呼び出すには通常、コンテンツ拡張子が必要です。 拡張子がない場合は、デフォルトとして `.http` が想定されます。 射影パスは、`.json` と `.http` の拡張子には必要ありませんが、`.html`、`.svg`、`.text` の拡張子には必要です。 便宜上、デフォルトのパスは、拡張子名と一致すると想定されます。 Web アクションを起動して、`.html` 応答を受け取るには、アクションは、`html` という名前の最上位プロパティーを含む JSON オブジェクトを用いて応答する必要があります (または、応答は明示的パスに存在する必要があります)。 言い換えると、`/guest/demo/hello.html` は、`/guest/demo/hello.html/html` のように、`html` プロパティーを明示的に射影することと等価です。 アクションの完全修飾名には、そのパッケージ名が含まれている必要があります。このパッケージ名は、アクションが名前付きパッケージ内にない場合は `default` になります。

## 保護されたパラメーター
{: #actions_web_protect}

アクション・パラメーターは保護され、変更不可能として処理されます。 Web アクションを有効にするために、パラメーターは自動的にファイナライズされます。
```
ibmcloud fn action create /guest/demo/hello hello.js --parameter name Jane --web true
```
{: pre}

これらの変更の結果、`name` が `Jane` にバインドされ、final アノテーションがあるため照会パラメーターまたは本体パラメーターでオーバーライドすることはできません。 この設計は、意図的または偶発的にこの値を変更しようとする照会パラメーターまたは本体パラメーターからアクションを保護します。

## Web アクションの保護
{: #actions_web_secure}

デフォルトでは、Web アクションの呼び出し URL を持っていれば誰でも Web アクションを呼び出すことができます。 Web アクションを保護するには、[Web アクションのアノテーション](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions) `require-whisk-auth` を使用します。`require-whisk-auth` アノテーションが `true` に設定されている場合、アクションは、呼び出し要求の基本許可資格情報をアクション所有者の whisk 認証キーに照らして認証します。 数字または大/小文字の区別があるストリングに設定されている場合、アクションの呼び出し要求には、それと同じ値が設定された `X-Require-Whisk-Auth` ヘッダーが含まれていなければなりません。 保護された Web アクションは、資格情報の検証が失敗した場合はメッセージ `Not Authorized` を返します。

代替方法として、`--web-secure` フラグを使用して `require-whisk-auth` アノテーションを自動的に設定することもできます。  `true` に設定すると、`require-whisk-auth` アノテーション値として乱数が生成されます。 `false` に設定すると、`require-whisk-auth` アノテーションは削除されます。  その他の値に設定すると、その値が `require-whisk-auth` アノテーション値として使用されます。

**--web-secure** の使用例:
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true --web-secure my-secret
```
{: pre}

**require-whisk-auth** の使用例:
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true -a require-whisk-auth my-secret
```
{: pre}

**X-Require-Whisk-Auth** の使用例:
```bash
curl https://${APIHOST}/api/v1/web/guest/demo/hello.json?name=Jane -X GET -H "X-Require-Whisk-Auth: my-secret"
```
{: pre}

Web アクションの所有者がすべてのアクティベーション・レコードを所有し、アクションがどのように呼び出されたのかに関係なくシステムでのアクションの実行のコストを負担することに留意することは重要です。

## Web アクションの無効化
{: #actions_web_disable}

Web API (`https://openwhisk.bluemix.net/api/v1/web/`) を介した Web アクションの起動を無効にするには、`--web` フラグに値 `false` または `no` を渡して、CLI で アクションを更新します。
```
ibmcloud fn action update /guest/demo/hello hello.js --web false
```
{: pre}

## 未加工 HTTP 処理
{: #actions_web_raw}

Web アクションは、アクション入力に使用できる第 1 クラス・プロパティーに JSON オブジェクトをプロモーションせずに、着信 HTTP 本体を直接解釈して処理することを選択できます (例: `args.name` 対 `args.__ow_query` の構文解析)。 この処理は、`raw-http` [アノテーション](/docs/openwhisk?topic=cloud-functions-annotations)を介して行います。 前述の同じ例を使用しますが、今度は「未加工」HTTP Web アクションとして使用し、以下のように、照会パラメーターと、HTTP 要求本体内の JSON 値の両方として `name` を受け取ります。
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane -X POST -H "Content-Type: application/json" -d '{"name":"Jane"}'
```
{: pre}

出力例:
```
{
  "response": {
    "__ow_method": "post",
    "__ow_query": "name=Jane",
    "__ow_body": "eyJuYW1lIjoiSmFuZSJ9",
    "__ow_headers": {
      "accept": "*/*",
      "connection": "close",
      "content-length": "15",
      "content-type": "application/json",
      "host": "172.17.0.1",
      "user-agent": "curl/7.43.0"
    },
    "__ow_path": ""
  }
}
```
{: screen}

OpenWhisk は、[Akka HTTP](https://doc.akka.io/docs/akka-http/current/?language=scala) フレームワークを使用して、どのコンテンツ・タイプがバイナリーであり、どれがプレーン・テキストであるかを[判別](https://doc.akka.io/api/akka-http/10.0.4/akka/http/scaladsl/model/MediaTypes$.html)します。

### 未加工 HTTP 処理の有効化
{: #actions_web_raw_enable}

未加工 HTTP Web アクションは、`--web` フラグで値 `raw` を使用して有効化します。
```
ibmcloud fn action create /guest/demo/hello hello.js --web raw
```
{: pre}

### 未加工 HTTP 処理の無効化
{: #actions_web_raw_disable}

未加工 HTTP を無効にするには、`--web` フラグに値 `false` または `no` を渡します。
```
ibmcloud fn update create /guest/demo/hello hello.js --web false
```
{: pre}

### バイナリーの本体コンテンツを Base64 からデコード
{: #actions_web_decode}

未加工 HTTP コンテンツが処理されるとき、要求の `Content-Type` がバイナリーの場合、`__ow_body` コンテンツは、Base64 でエンコードされます。 以下の関数は、Node、Python、および Swift で本体コンテンツをデコードする方法を示しています。 単純に、メソッドをファイルに保存し、保存された成果物を使用する未加工 HTTP Web アクションを作成し、その後、Web アクションを起動します。

#### Node
{: #actions_web_decode_js}

```javascript
function main(args) {
    decoded = new Buffer(args.__ow_body, 'base64').toString('utf-8')
    return {body: decoded}
}
```
{: codeblock}

#### Python
{: #actions_web_decode_python}

```python
def main(args):
    try:
        decoded = args['__ow_body'].decode('base64').strip()
        return {"body": decoded}
    except:
        return {"body": "Could not decode body from Base64."}
```
{: codeblock}

#### Swift
{: #actions_web_decode_swift}

```swift
extension String {
    func base64Decode() -> String? {
        guard let data = Data(base64Encoded: self) else {
            return nil
        }

        return String(data: data, encoding: .utf8)
    }
}

func main(args: [String:Any]) -> [String:Any] {
    if let body = args["__ow_body"] as? String {
        if let decoded = body.base64Decode() {
            return [ "body" : decoded ]
        }
    }

    return ["body": "Could not decode body from Base64."]
}
```
{: codeblock}

例として、この Node 関数を `decode.js` として保存し、以下のコマンドを実行します。
```
ibmcloud fn action create decode decode.js --web raw
```
{: pre}

出力例:
```
ok: created action decode
```
{: screen}

```
curl -k -H "content-type: application" -X POST -d "Decoded body" https:// us-south.functions.cloud.ibm.com/api/v1/web/guest/default/decodeNode.json
```
{: pre}

出力例:
```
{
  "body": "Decoded body"
}
```
{: screen}

## OPTIONS 要求
{: #actions_web_options}

デフォルトで、Web アクションに対して OPTIONS 要求を行うと、自動的に CORS ヘッダーが応答ヘッダーに追加されます。 これらのヘッダーにより、すべてのオリジン、および options、get、delete、post、put、head、および patch の各 HTTP 動詞が可能になります。

以下のヘッダーを参照してください。
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: OPTIONS, GET, DELETE, POST, PUT, HEAD, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
```

あるいは、OPTIONS 要求は、Web アクションによって手動で処理することもできます。 このオプションを有効にするには、`web-custom-options` アノテーションを `true` の値に設定して Web アクションに追加します。 このフィーチャーが有効になると、CORS ヘッダーは自動的に要求応答に追加されません。 代わりに、開発者の責任で、希望するヘッダーをプログラマチックに追加することになります。

OPTIONS 要求に対するカスタム応答を作成するには、次の例を参照してください。
```js
function main(params) {
  if (params.__ow_method == "options") {
    return {
      headers: {
        'Access-Control-Allow-Methods': 'OPTIONS, GET',
        'Access-Control-Allow-Origin': 'example.com'
      },
      statusCode: 200
    }
  }
}
```
{: codeblock}

この関数を `custom-options.js` に保存し、以下のコマンドを実行します。
```
ibmcloud fn action create custom-option custom-options.js --web true -a web-custom-options true
```
{: pre}

```
$ curl https://${APIHOST}/api/v1/web/guest/default/custom-options.http -kvX OPTIONS
```
{: pre}

出力例:
```
< HTTP/1.1 200 OK
< Server: nginx/1.11.13
< Content-Length: 0
< Connection: keep-alive
< Access-Control-Allow-Methods: OPTIONS, GET
< Access-Control-Allow-Origin: example.com
```
{: screen}

## エラー処理
{: #actions_web_errors}

{{site.data.keyword.openwhisk_short}} アクションは、2 つの異なる障害モードで失敗します。 1 つは_アプリケーション・エラー_ と呼ばれるものであり、キャッチされた例外に似ています。アクションが返す JSON オブジェクトには、最上位に `error` プロパティーが含まれます。 もう 1 つは _開発者エラー_ であり、これはアクションで壊滅的な障害が起こって、応答が生成されない場合に発生します (キャッチされていない例外に似ています)。 Web アクションの場合、コントローラーは次のようにアプリケーション・エラーを処理します。

- 指定されたパス射影はすべて無視され、コントローラーは代わりに `error` プロパティーを射影します。
- コントローラーはアクションの拡張子で暗黙に示されるコンテンツ処理を `error` プロパティーの値に適用します。

開発者は、Web アクションの使用方法を認識し、適切なエラー応答を生成する必要があります。 例えば、`.http` 拡張子と共に使用される Web アクションは、`{error: { statusCode: 400 }` のような HTTP 応答を返します。 適切にエラーが生成されないと、拡張子で暗黙に示される `Content-Type` と、エラー応答内のアクション `Content-Type` とが一致しません。 Web アクションがシーケンスになっている場合は、シーケンスを形成しているコンポーネントが必要に応じて適切なエラーを生成できるように、特別な考慮が必要です。

## 例: 入力からの QR コード・イメージの生成
{: #actions_web_qr}

入力として `text` を取り込んで QR コード・イメージを生成する Java Web アクションの例を以下に示します。

1. ディレクトリー `java_example/src/main/java/qr` にファイル `Generate.java` を作成します。

  ```java
  package qr;

  import java.io.*;
  import java.util.Base64;

  import com.google.gson.JsonObject;

  import com.google.zxing.*;
  import com.google.zxing.client.j2se.MatrixToImageWriter;
  import com.google.zxing.common.BitMatrix;

  public class Generate {
    public static JsonObject main(JsonObject args) throws Exception {
      String property = "text";
      String text = "Hello. Try with a 'text' value next time.";
      if (args.has(property)) {
        text = args.get(property).toString();
      }

      ByteArrayOutputStream baos = new ByteArrayOutputStream();
      OutputStream b64os = Base64.getEncoder().wrap(baos);

      BitMatrix matrix = new MultiFormatWriter().encode(text, BarcodeFormat.QR_CODE, 300, 300);
      MatrixToImageWriter.writeToStream(matrix, "png", b64os);
      b64os.close();

      String output = baos.toString("utf-8");

      JsonObject response = new JsonObject();
      JsonObject headers = new JsonObject();
      headers.addProperty("content-type", "image/png; charset=UTF-8");
      response.add("headers", headers);
      response.addProperty("body", output);
      return response;
    }
  }
  ```
  {: codeblock}

3. `java_example` (ファイル `build.gradle` が入っているディレクトリー) から以下のコマンドを実行して、Web アクション JAR をビルドします。

  ```bash
  gradle jar
  ```
  {: pre}

4. JAR `build/libs/java_example-1.0.jar` を使用して Web アクションをデプロイします。

  ```bash
  ibmcloud fn action update QRGenerate build/libs/java_example-1.0.jar --main qr.Generate -m 128 --web true
  ```
  {: pre}

5. Web アクション・エンドポイントのパブリック URL を取得して環境変数に割り当てます。

  ```bash
  ibmcloud fn action get QRGenerate --url
  URL=$(ibmcloud fn action get QRGenerate --url | tail -1)
  ```
  {: pre}

6. この `URL` を使用して Web ブラウザーを開くことができます。QR イメージにエンコードして組み込むメッセージを、その照会パラメーター `text` に追加します。 `curl` などの HTTP クライアントを使用して QR イメージをダウンロードすることもできます。

  ```bash
  curl -o QRImage.png $URL\?text=https://cloud.ibm.com
  ```
  {: pre}
