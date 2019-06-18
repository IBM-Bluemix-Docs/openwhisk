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


# 建立 Web 動作
{: #actions_web}

Web 動作是已註釋的 {{site.data.keyword.openwhisk}} 動作，可快速讓開發人員建置 Web 型應用程式。這些已註釋的動作容許開發人員對後端邏輯進行程式設計，而 Web 應用程式可匿名存取該後端邏輯，而不需要 {{site.data.keyword.openwhisk_short}} 鑑別金鑰。由動作開發人員決定實作其擁有的所需鑑別及授權（或 OAuth 流程）。
{: shortdesc}

Web 動作啟動會與已建立動作的使用者相關聯。此動作會將動作啟動的成本從呼叫者推遲到動作的擁有者身上。

請查看下列 JavaScript 動作 `hello.js`：
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

您可以搭配使用 CLI 的 `--web` 旗標與 `true` 或 `yes` 值，在名稱空間 `guest` 的套件 `demo` 中建立 _Web 動作_ **hello**：
```
ibmcloud fn package create demo
```
{: pre}

```
ibmcloud fn action create /guest/demo/hello hello.js --web true
```
{: pre}

搭配使用 `--web` 旗標與 `true` 或 `yes` 值，容許透過 REST 介面存取動作，而不需要使用認證。若要使用認證來配置 Web 動作，請參閱[保護 Web 動作](#actions_web_secure)小節。您可以使用結構如下的 URL 來呼叫 Web 動作：`https://{APIHOST}/api/v1/web/{namespace}/{packageName}/{actionName}.{EXT}`。

如果動作不在具名套件中，則套件名稱為 **default**。

範例為 `guest/demo/hello`。Web 動作 API 路徑可以與 `curl` 或 `wget` 搭配使用，而不需要 API 金鑰。它甚至可以直接輸入瀏覽器中。

請嘗試在 Web 瀏覽器中開啟 `https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane`。或者，嘗試使用 `curl` 來呼叫動作：
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane
```
{: pre}

在下列範例中，Web 動作會執行 HTTP 重新導向：
```javascript
function main() {
    return { 
    headers: { location: 'http://openwhisk.org' },
    statusCode: 302
  }
}
```
{: codeblock}

在下列範例中，Web 動作會設定單一 Cookie：
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

在下列範例中，Web 動作會設定多個 Cookie：
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

下列範例會傳回 `image/png`：
```javascript
function main() {
    let png = <base 64 encoded string>
    return { headers: { 'Content-Type': 'image/png' },
             statusCode: 200,
             body: png };
}
```
{: codeblock}

The following example returns `application/json`:
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

The default `Content-Type` for an HTTP response is `application/json`, and the body can be any allowed JSON value. The default `Content-Type` can be omitted from the headers.

It is important to be aware of the [回應大小限制](/docs/openwhisk?topic=cloud-functions-limits)（適用於動作），因為如果回應超出預先定義的系統限制，就會失敗。例如，大型物件不會透過 {{site.data.keyword.openwhisk_short}} 在行內傳送，而是改為延遲到物件儲存庫。

## 使用動作處理 HTTP 要求
{: #actions_web_http}

不是 Web 動作的 {{site.data.keyword.openwhisk_short}} 動作需要鑑別，而且必須回應 JSON 物件。

Web 動作可在未鑑別的情況下呼叫，而且可用來實作回應不同類型的 `headers`、`statusCode` 及 `body` 內容的 HTTP 處理程式。Web 動作必須傳回 JSON 物件。不過，如果 Web 動作的結果包括下列一個以上的項目作為最上層 JSON 內容，則控制器會以不同方式處理 Web 動作：

- `headers`：索引鍵為標頭名稱且值為字串、數字或布林值的 JSON 物件。若要傳送單一標頭的多個值，則標頭的值是多個值的 JSON 陣列。依預設，不會設定任何標頭。
- `statusCode`：有效的 HTTP 狀態碼。如果存在內文內容，則預設值為 `200 OK`。如果不存在任何內文內容，則預設值為 `204 No Content`。
- `body`：該字串可為純文字、JSON 物件或陣列或是 base64 編碼字串（適用於二進位資料）。如果內文為 `null`、空字串 `""` 或未定義，則會將其視為空白。預設值是空的內文。

控制器會將任何動作指定的標頭、狀態碼或內文傳遞給終止要求或回應的 HTTP 用戶端。如果未在動作結果的 `headers` 中宣告 `Content-Type` 標頭，則會將內文解譯為非字串值的 `application/json`，否則會解譯為 `text/html`。如果已定義 `Content-Type` 標頭，則控制器會判斷回應是二進位資料還是純文字，並視需要使用 base64 解碼器來解碼字串。如果內文未正確解碼，則會將錯誤傳回給用戶端。


## HTTP 環境定義
{: #actions_web_context}

呼叫時，所有 Web 動作都會接收 HTTP 要求詳細資料，作為動作輸入引數的參數。

請參閱下列 HTTP 參數：

- `__ow_method`（類型：字串）：要求的 HTTP 方法。
- `__ow_headers`（類型：將字串對映至字串）：要求標頭。
- `__ow_path`（類型：字串）：要求的不相符路徑（比對會在使用動作副檔名之後停止）。
- `__ow_user`（類型：字串）：識別 {{site.data.keyword.openwhisk_short}} 已鑑別身分的「名稱空間」
- `__ow_body`（類型：字串）：要求內文實體，當內容為二進位時，為 base64 編碼的字串，否則為一般字串
- `__ow_query`（類型：字串）：要求中的查詢參數，為未剖析的字串

要求無法置換任何具名的 `__ow_` 參數。這麼做會導致要求失敗，其狀態等於「400 不正確的要求」。

只有在 Web 動作[已註釋為需要鑑別](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions)，並容許 Web 動作實作自己的授權原則時，`__ow_user` 才會存在。只有在 Web 動作選擇要處理[「原始」HTTP 要求](#actions_web_raw_enable)時，才能使用 `__ow_query`。這是一個字串，其中包含從 URI 剖析的查詢參數（以 `&` 區隔）。在「原始」HTTP 要求中，或是當 HTTP 要求實體不是 JSON 物件或表單資料時，會出現 `__ow_body` 內容。否則，Web 動作會接收查詢和內文參數，作為動作引數中的第一個類別內容。內文參數的優先順序高於查詢參數，而查詢參數的優先順序高於動作和套件參數。

## HTTPS 端點支援
{: #actions_web_endpoint}

支援的 SSL 通訊協定：TLS 1.2、TLS 1.3（[初版 18](https://tools.ietf.org/html/draft-ietf-tls-tls13-18)）

不受支援的 SSL 通訊協定：SSLv2、SSLv3、TLS 1.0、TLS 1.1

## 額外特性
{: #actions_web_extra}

Web 動作提供的額外特性包括：

- `內容副檔名`：要求必須將其所需的內容類型指定為下列其中一項：`.json`、`.html`、`.http`、`.svg` 或 `.text`。指定類型的作法是將副檔名新增至 URI 中的動作名稱，因此動作 `/guest/demo/hello` 參照為 `/guest/demo/hello.http`（舉例來說），以接收傳回的 HTTP 回應。為了方便起見，如果未偵測到副檔名，則假設為 `.http` 副檔名。
- `投射結果中的欄位`：接在動作名稱後面的路徑是用來投射回應的一個以上層次。
`/guest/demo/hello.html/body`。此特性可讓傳回字典 `{body: "..." }` 的動作投射 `body` 內容，並且改為直接傳回其字串值。投射路徑遵循絕對路徑模型（如 XPath）。
- `作為輸入的查詢及內文參數`：動作會接收查詢參數，以及要求內文中的參數。參數的合併優先順序是：套件參數、動作參數、查詢參數及內文參數。這些參數每一個都會在重疊時覆寫任何先前的值。舉例來說，`/guest/demo/hello.http?name=Jane` 會將引數 `{name: "Jane"}` 傳遞給動作。
- `表單資料`：除了標準 `application/json` 之外，Web 動作還可以接收從資料 `application/x-www-form-urlencoded data` 編碼的 URL，作為輸入。
- `使用多個 HTTP 動詞來啟動`：Web 動作可以透過下列任何 HTTP 方法來呼叫：`GET`、`POST`、`PUT`、`PATCH` 和 `DELETE`，以及 `HEAD` 和 `OPTIONS`。
- `非 JSON 內文和原始 HTTP 實體處理`：Web 動作可以接受 JSON 物件以外的 HTTP 要求內文，並可選擇一律接收這類值作為不透明值（非二進位時為純文字，否則為 base64 編碼字串）。

下面範例簡短概述如何在 Web 動作中使用這些特性。請考量具有下列內文的動作 `/guest/demo/hello`：
```javascript
  function main(params) {
      return { response: params };
}
```

將此動作當作 Web 動作呼叫時，您可以從結果投射不同的路徑，以變更 Web 動作的回應。

例如，若要傳回整個物件，並查看動作所接收的引數：
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json
 ```
{: pre}

輸出範例：
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

若要使用查詢參數來執行，請參閱下列範例指令：
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane
 ```
{: pre}

輸出範例：
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

您也可以使用表單資料來執行：
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -d "name":"Jane"
 ```
{: pre}

輸出範例：
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

對 JSON 物件執行下列指令：
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: application/json' -d '{"name":"Jane"}'
```
{: pre}

輸出範例：
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

執行下列指令以投射名稱（以文字形式）：
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.text/response/name?name=Jane
```
{: pre}

輸出範例：
```
Jane
```
{: screen}

為了方便起見，查詢參數、表單資料和 JSON 物件內文實體全都被當作字典來處理，並且可以直接存取其值作為動作輸入內容。但是選擇以較直接的方式來處理 HTTP 要求實體的 Web 動作，或是當 Web 動作接收不是 JSON 物件的實體時，此行為就不是這樣了。

請參閱下列使用 "text" 內容類型的範例（如前所示）。
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: text/plain' -d "Jane"
```
{: pre}

輸出範例：
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

## 內容副檔名
{: #actions_web_ext}

通常需要有內容副檔名，才能呼叫 Web 動作。若沒有副檔名，則會假設 `.http` 為預設值。`.json` 及 `.http` 副檔名不需要投射路徑，而 `.html`、`.svg` 及 `.text` 副檔名則需要。為了方便起見，會假設預設路徑符合副檔名。若要呼叫 Web 動作並接收 `.html` 回應，動作必須回應包含稱為 `html` 的最上層內容的 JSON 物件（或者回應必須位於明確的路徑中）。換言之，`/guest/demo/hello.html` 相當於明確地投射 `html` 內容，如 `/guest/demo/hello.html/html`。動作的完整名稱必須包括其套件名稱，如果動作不在具名套件中，其為 `default`。

## 受保護的參數
{: #actions_web_protect}

動作參數也會受到保護並視為不可變。系統會自動完成參數，以啟用 Web 動作。
```
ibmcloud fn action create /guest/demo/hello hello.js --parameter name Jane --web true
```
{: pre}

這些變更的結果為 `name` 已連結至 `Jane`，而且無法因最終註釋而置換為查詢或內文參數。此設計可保護對意外或有意嘗試變更此值的查詢或內文參數所執行的動作。

## 保護 Web 動作
{: #actions_web_secure}

依預設，任何具有 Web 動作呼叫 URL 的人員都可以呼叫 Web 動作。使用 `require-whisk-auth` [Web 動作註釋](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions)來保護 Web 動作。`require-whisk-auth` 註釋設為 `true` 時，動作會根據動作擁有者的 Whisk 鑑別金鑰來鑑別呼叫要求的「基本授權」認證。設為數字或區分大小寫的字串時，動作的呼叫要求必須包括具有此相同值的 `X-Require-Whisk-Auth` 標頭。認證驗證失敗時，安全的 Web 動作會傳回`未獲授權`訊息。

或者，您也可以使用 `--web-secure` 旗標，自動設定 `require-whisk-auth` 註釋。設為 `true` 時，會產生亂數作為 `require-whisk-auth` 註釋值。設為 `false` 時，會移除 `require-whisk-auth` 註釋。設為任何其他值時，會使用該值作為 `require-whisk-auth` 註釋值。

使用 **--web-secure** 的範例：
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true --web-secure my-secret
```
{: pre}

使用 **require-whisk-auth** 的範例：
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true -a require-whisk-auth my-secret
```
{: pre}

使用 **X-Require-Whisk-Auth** 的範例：
```bash
curl https://${APIHOST}/api/v1/web/guest/demo/hello.json?name=Jane -X GET -H "X-Require-Whisk-Auth: my-secret"
```
{: pre}

請務必注意，Web 動作的擁有者擁有所有啟動記錄，且會引發在系統中執行動作的成本，不論呼叫動作的方式為何。

## 停用 Web 動作
{: #actions_web_disable}

若要停用透過 Web API (`https://openwhisk.bluemix.net/api/v1/web/`) 呼叫 Web 動作，請將 `false` 或 `no` 值傳遞給 `--web` 旗標，以使用 CLI 來更新動作。
```
ibmcloud fn action update /guest/demo/hello hello.js --web false
```
{: pre}

## 原始 HTTP 處理
{: #actions_web_raw}

Web 動作可以選擇直接解譯及處理送入的 HTTP 內文，而不需將 JSON 物件提升為可用於動作輸入的第一個類別內容（例如，`args.name` 與剖析 `args.__ow_query`）。透過 `raw-http` [註釋](/docs/openwhisk?topic=cloud-functions-annotations)即可完成此處理程序。使用稍早顯示的相同範例，但現在是作為「原始」HTTP Web 動作，它會接收 `name` 同時作為查詢參數以及 HTTP 要求內文中的 JSON 值：
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane -X POST -H "Content-Type: application/json" -d '{"name":"Jane"}'
```
{: pre}

輸出範例：
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

OpenWhisk 使用 [Akka Http](https://doc.akka.io/docs/akka-http/current/?language=scala) 架構來[判斷](https://doc.akka.io/api/akka-http/10.0.4/akka/http/scaladsl/model/MediaTypes$.html)哪些內容類型是二進位，哪些是純文字。

### 啟用原始 HTTP 處理
{: #actions_web_raw_enable}

若要啟用原始 HTTP Web 動作，可以搭配使用 `--web` 旗標與 `raw` 值。
```
ibmcloud fn action create /guest/demo/hello hello.js --web raw
```
{: pre}

### 停用原始 HTTP 處理
{: #actions_web_raw_disable}

將 `false` 或 `no` 值傳遞給 `--web` 旗標，即可停用原始 HTTP。
```
ibmcloud fn update create /guest/demo/hello hello.js --web false
```
{: pre}

### 從 Base64 解碼二進位內文內容
{: #actions_web_decode}

處理原始 HTTP 內容時，若要求 `Content-Type` 是二進位，則會以 Base64 編碼 `__ow_body` 內容。
下列函數示範如何解碼 Node、Python 及 Swift 中的內文內容。只需要將方法儲存至檔案、建立利用所儲存構件的原始 HTTP Web 動作，然後呼叫 Web 動作即可。

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

例如，將 Node 函數儲存為 `decode.js`，然後執行下列指令：
```
ibmcloud fn action create decode decode.js --web raw
```
{: pre}

輸出範例：
```
ok: created action decode
```
{: screen}

```
curl -k -H "content-type: application" -X POST -d "Decoded body" https:// us-south.functions.cloud.ibm.com/api/v1/web/guest/default/decodeNode.json
```
{: pre}

輸出範例：
```
{
  "body": "Decoded body"
}
```
{: screen}

## Options 要求
{: #actions_web_options}

依預設，對 Web 動作所提出的 OPTIONS 要求會導致在回應標頭自動新增 CORS 標頭。這些標頭允許所有原點，以及 options、get、delete、post、put、head 和 patch 等 HTTP 動詞。

請參閱下列標頭：
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: OPTIONS, GET, DELETE, POST, PUT, HEAD, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
```

或者，也可以由 Web 動作手動處理 OPTIONS 要求。若要啟用此選項，請在 Web 動作中新增 `web-custom-options` 註釋及值 `true`。啟用此特性時，不會自動將 CORS 標頭新增至要求回應。而是要由開發人員負責以程式設計方式附加他們想要的標頭。

請參閱下列範例，以建立 OPTIONS 要求的自訂回應：
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

將函數儲存到 `custom-options.js`，然後執行下列指令：
```
ibmcloud fn action create custom-option custom-options.js --web true -a web-custom-options true
```
{: pre}

```
$ curl https://${APIHOST}/api/v1/web/guest/default/custom-options.http -kvX OPTIONS
```
{: pre}

輸出範例：
```
< HTTP/1.1 200 OK
< Server: nginx/1.11.13
< Content-Length: 0
< Connection: keep-alive
< Access-Control-Allow-Methods: OPTIONS, GET
< Access-Control-Allow-Origin: example.com
```
{: screen}

## 錯誤處理
{: #actions_web_errors}

{{site.data.keyword.openwhisk_short}} 動作在兩種不同的可能失敗模式下失敗。第一個稱為_應用程式錯誤_，類似於捕捉的異常狀況：動作會傳回包含最上層 `error` 內容的 JSON 物件。第二個是動作災難性地失敗且未產生回應時所發生的_開發人員錯誤_（這類似於未捕捉到的異常狀況）。對於 Web 動作，控制器會如下處理應用程式錯誤：

- 忽略所有指定的路徑投射，控制器會改為投射 `error` 內容。
- 控制器會將依動作副檔名所隱含的內容處理套用至 `error` 內容的值。

開發人員必須知道 Web 動作的使用方式，以及如何產生適當的錯誤回應。例如，與 `.http` 副檔名搭配使用的 Web 動作傳回 HTTP 回應，類似於 `{error: { statusCode: 400 }`。無法這麼做時，會導致副檔名中所隱含的 `Content-Type` 與錯誤回應中的動作 `Content-Type` 不符。必須對具有序列的 Web 動作進行特殊考量，才能讓構成序列的元件可以在必要時產生足夠的錯誤。

## 範例：透過輸入產生 QR 碼影像
{: #actions_web_qr}

以下 Java Web Action 範例採用 `text` 作為輸入並產生 QR 碼影像。

1. 在 `java_example/src/main/java/qr` 目錄中，建立 `Generate.java` 檔案。

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

3. 從 `build.gradle` 檔案所在的 `java_example` 目錄中，執行下列指令來建置「Web 動作 JAR」。

  ```bash
  gradle jar
  ```
  {: pre}

4. 使用 JAR `build/libs/java_example-1.0.jar` 來部署 Web 動作。

  ```bash
  ibmcloud fn action update QRGenerate build/libs/java_example-1.0.jar --main qr.Generate -m 128 --web true
  ```
  {: pre}

5. 擷取 Web 動作端點的公用 URL，並將它指派給環境變數。

  ```bash
  ibmcloud fn action get QRGenerate --url
  URL=$(ibmcloud fn action get QRGenerate --url | tail -1)
  ```
  {: pre}

6. 您可以開啟 Web 瀏覽器，方法是使用此 `URL`，並附加具有編碼為 QR 影像之訊息的 `text` 查詢參數。您也可以使用 `curl` 這類 HTTP 用戶端來下載 QR 影像。

  ```bash
  curl -o QRImage.png $URL\?text=https://cloud.ibm.com
  ```
  {: pre}
