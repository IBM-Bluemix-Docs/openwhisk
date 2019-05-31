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


# Creación de acciones web
{: #actions_web}

Las acciones web son acciones de {{site.data.keyword.openwhisk}} que se anotan para permitir rápidamente los desarrolladores creen aplicaciones basadas en la web. Estas acciones anotadas permiten a los desarrolladores programar la lógica subyacente a la que la aplicación web puede acceder de forma anónima, sin necesidad de disponer de una clave de autenticación de {{site.data.keyword.openwhisk_short}}. Depende del desarrollador de la acción si desea implementar su propio sistema de autenticación y autorización o utilizar un flujo de OAuth.
{: shortdesc}

Las activaciones de acciones web están asociadas al usuario que ha creado la acción. Esta acción transfiere el coste de la activación de una acción del que efectúa la llamada al propietario de la acción.

Consulte la siguiente acción JavaScript `hello.js`:
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

Puede crear una _acción web_ **hello** en el paquete `demo` para el espacio de nombres `guest` utilizando el distintivo `--web` de la CLI con el valor `true` o `yes`:
```
ibmcloud fn package create demo
```
{: pre}

```
ibmcloud fn action create /guest/demo/hello hello.js --web true
```
{: pre}

Si especifica el distintivo `--web` con el valor `true` o `yes`, se puede acceder a una acción mediante una interfaz REST sin necesidad de credenciales. Para configurar una acción web con credenciales, consulte la sección sobre [Protección de acciones web](#actions_web_secure). Una acción web se puede invocar utilizando un URL estructurado del siguiente modo:
`https://{APIHOST}/api/v1/web/{namespace}/{packageName}/{actionName}.{EXT}`.

El nombre de paquete es **default** si la acción no está en un paquete con nombre.

Un ejemplo es `guest/demo/hello`. La vía de acceso a la API de la acción web se puede utilizar con `curl` o `wget` sin una clave de API. Incluso se puede escribir directamente en el navegador.

Intente abrir `https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane` en su navegador web. O intente invocar la acción utilizando `curl`:
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane
```
{: pre}

En el siguiente ejemplo, una acción web realiza una redirección HTTP:
```javascript
function main() {
  return {
    headers: { location: 'http://openwhisk.org' },
    statusCode: 302
  }
}
```
{: codeblock}

En el ejemplo siguiente, una acción web establece una cookie única:
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

En el siguiente ejemplo, una acción web establece varias cookies:
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

El ejemplo siguiente devuelve `image/png`:
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

It is important to be aware of the [límite de tamaño de respuesta](/docs/openwhisk?topic=cloud-functions-limits) de las acciones, ya que las respuestas que superen los límites predefinidos del sistema fallan. Los objetos grandes no envían en línea a través de {{site.data.keyword.openwhisk_short}}, sino que se defieren a un almacén de objetos, por ejemplo.

## Manejo de solicitudes HTTP con acciones
{: #actions_web_http}

Una acción de {{site.data.keyword.openwhisk_short}} que no sea una acción web necesita autenticación y debe responder con un objeto JSON.

Las acciones web se pueden invocar sin autenticación y se pueden utilizar para implementar manejadores HTTP que respondan con contenido de `headers`, `statusCode` y `body` de distintos tipos.
Las acciones web deben devolver un objeto JSON. No obstante, el controlador trata una acción web de forma distinta si su resultado incluye una o más de las siguientes como propiedades JSON de nivel superior:

- `headers`: un objeto JSON en el que las claves son nombres de cabecera y los valores son valores de serie, numéricos o booleanos. Para enviar varios valores para una sola cabecera, el valor de la cabecera es una matriz JSON de varios valores. No hay ninguna cabecera establecida de forma predeterminada.
- `statusCode`: un código de estado HTTP válido. Si el contenido del cuerpo está presente, el valor predeterminado es
`200 OK`. Si no hay contenido de cuerpo presente, el valor predeterminado es `204 No Content`.
- `body`: una serie que puede ser texto sin formato, una matriz u objeto JSON, o una serie codificada en base64 para datos binarios. El cuerpo se considera vacío si es `null`, la serie vacía `""` o es indefinido. El valor predeterminado es un cuerpo vacío.

El controlador pasa las cabeceras especificadas por acciones, el código de estado o el cuerpo al cliente HTTP que finaliza la solicitud o respuesta. Si no se declara la cabecera `Content-Type` en la sección `headers` del resultado de la acción, el cuerpo se interpreta como `application/json` para valores que no sean de tipo serie y `text/html` en cualquier otro caso. Si se define la cabecera `Content-Type`, el controlador determina si la respuesta son datos binarios o texto sin formato y decodifica la serie utilizando el decodificador base64 según sea necesario. Si el cuerpo no se decodifica correctamente, se devuelve un error al cliente.


## Contexto HTTP
{: #actions_web_context}

Todas las acciones web, cuando se invocan, reciben detalles de solicitud HTTP como parámetros del argumento de entrada de la acción.

Consulte los parámetros HTTP siguientes:

- `__ow_method` (tipo: serie): El método HTTP de la solicitud.
- `__ow_headers` (tipo: correlación entre serie y serie): Cabeceras de la solicitud.
- `__ow_path` (tipo: serie): Vía de acceso de la solicitud que no coincide (la comparación finaliza al consumirse la extensión de la acción).
- `__ow_user` (tipo: serie): Espacio de nombres que identifica el asunto autenticado de {{site.data.keyword.openwhisk_short}}
- `__ow_body` (tipo: serie): Entidad del cuerpo de la solicitud, como serie codificada en base64 cuando el contenido es binario o serie de texto sin formato en caso contrario
- `__ow_query` (tipo: serie): Parámetros de la consulta procedentes de la solicitud como serie sin analizar

Una solicitud no puede sustituir ninguno de los parámetros `__ow_` mencionados. Si lo hiciera, la solicitud fallaría con el estado 400 Solicitud errónea.

El valor `__ow_user` solo aparece cuando la acción web tiene una [anotación que indica que requiere autenticación](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions) y permite que una acción web implemente su propia política de autorización. `__ow_query` solo está disponible cuando una acción web elige manejar la [solicitud HTTP "sin procesar"](#actions_web_raw_enable). Es una serie que contiene los parámetros de la consulta analizados del URI (separados por `&`). La propiedad `__ow_body` aparece en solicitudes HTTP "sin procesar" o cuando la entidad de la solicitud HTTP no es un objeto JSON ni datos de formulario. De lo contrario, las acciones web reciben parámetros de consulta y de cuerpo como propiedades de primera clase en los argumentos de la acción. Los parámetros de cuerpo prevalecen sobre los parámetros de consulta, que a su vez prevalecen sobre los parámetros de acción y de paquete.

## Soporte para puntos finales HTTPS
{: #actions_web_endpoint}

Protocolos SSL admitidos: TLS 1.2, TLS 1.3 ([versión de borrador 18](https://tools.ietf.org/html/draft-ietf-tls-tls13-18))

Protocolos SSL sin soporte: SSLv2, SSLv3, TLS 1.0, TLS 1.1

## Características adicionales
{: #actions_web_extra}

Las acciones web proporcionan características adicionales que incluyen:

- `Extensiones de contenido`: La solicitud debe especificar el tipo de contenido deseado como `.json`, `.html`, `.http`, `.svg` o `.text`. El tipo se especifica añadiendo una extensión al nombre de la acción en el URI de modo que se haga referencia a la acción `/guest/demo/hello` como `/guest/demo/hello.http`, por ejemplo para recibir una respuesta HTTP. Para su comodidad, supondremos que se utiliza la extensión `.http` cuando no se detecta ninguna extensión.
- `Proyección de campos desde el resultado`: La vía de acceso que sigue al nombre de la acción se utiliza para proyectar uno o varios niveles de respuesta.
`/guest/demo/hello.html/body`. Esta característica permite que una acción que devuelve un diccionario `{body: "..." }` proyecte la propiedad `body` y devuelva directamente el valor de su serie. La vía de acceso proyectada sigue un modelo de vía de acceso absoluta (como en XPath).
- `Parámetros de consulta y de cuerpo como entrada`: La acción recibe parámetros de consulta y parámetros en el cuerpo de la solicitud. El orden de prioridad para fusionar parámetros es el siguiente: parámetros de paquete, parámetros de acción, parámetros de consulta y parámetros de cuerpo. Cada uno de estos parámetros puede sustituir cualquier valor anterior en caso de solapamiento. Por ejemplo, `/guest/demo/hello.http?name=Jane` puede pasar el argumento `{name: "Jane"}` a la acción.
- `Datos de formulario`: Además de `application/json` estándar, las acciones web pueden recibir como entrada un URL codificado procedente de los datos `application/x-www-form-urlencoded data`.
- `Activación utilizando varios verbos HTTP`: Se puede invocar una acción web mediante uno de estos métodos HTTP: `GET`, `POST`, `PUT`, `PATCH` y `DELETE`, así como `HEAD` y `OPTIONS`.
- `Gestión de entidades que no son de cuerpo JSON ni HTTP sin procesar (RAW HTTP)`: Una acción web puede aceptar un cuerpo de solicitud HTTP que no sea un objeto JSON y puede optar por recibir siempre estos valores como valores opacos (texto sin formato cuando no están en binario o serie codificada en base64 en caso contrario).

El ejemplo siguiente muestra brevemente cómo puede utilizar estas características en una acción web. Supongamos que tenemos una acción `/guest/demo/hello` con el siguiente cuerpo:
```javascript
function main(params) {
    return { response: params };
}
```

Cuando se invoca esta acción una acción web, puede alterar la respuesta de la acción web protegiendo diferentes vías de acceso desde el resultado.

Por ejemplo, para devolver el objeto completo y ver los argumentos que recibe la acción:
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json
```
{: pre}

Salida de ejemplo:
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

Para ejecutar con un parámetro de consulta, consulte el mandato de ejemplo siguiente:
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane
 ```
{: pre}

Salida de ejemplo:
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

También puede ejecutar con datos de formulario:
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -d "name":"Jane"
 ```
{: pre}

Salida de ejemplo:
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

Ejecute el mandato siguiente para un objeto JSON:
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: application/json' -d '{"name":"Jane"}'
```
{: pre}

Salida de ejemplo:
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

Ejecute el siguiente mandato para proyectar el nombre (como texto):
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.text/response/name?name=Jane
```
{: pre}

Salida de ejemplo:
```
Jane
```
{: screen}

Para su comodidad, los parámetros de consulta, datos de formulario y entidades de cuerpo de objeto JSON se tratan como diccionarios ya que se puede acceder directamente a sus valores como propiedades de entrada de la acción. Este comportamiento no se aplica a las acciones web que optan por gestionar las entidades de solicitud HTTP de forma más directa ni cuando la acción web recibe una entidad que no es un objeto JSON.

Consulte el ejemplo siguiente, que utiliza un tipo de contenido "text", como hemos visto anteriormente.
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: text/plain' -d "Jane"
```
{: pre}

Salida de ejemplo:
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

## Extensiones de contenido
{: #actions_web_ext}

Generalmente se necesita una extensión de contenido cuando se invoca una acción web. Si no se especifica, se adopta `.http` como valor predeterminado. Las extensiones `.json` y `.http` no necesitan una vía de acceso de proyección; en cambio, las extensiones `.html`, `.svg` y `.text`, sí. Para su comodidad, se presupone que la vía de acceso predeterminada coincide con el nombre de la extensión. Para invocar una acción web y recibir una respuesta `.html`, la acción debe responder con un objeto JSON que conste de una propiedad de nivel superior denominada `html` (o la respuesta debe estar en la vía de acceso explícita). Es decir, `/guest/demo/hello.html` equivale a proyectar la propiedad `html` de forma explícita, como en el caso de `/guest/demo/hello.html/html`. El nombre completo de la acción debe incluir el nombre de su paquete, que es `default` si la acción no está en un paquete con nombre.

## Parámetros protegidos
{: #actions_web_protect}

Los parámetros de la acción están protegidos y se tratan como inalterables. Los parámetros finalizan automáticamente para habilitar las acciones web.
```
ibmcloud fn action create /guest/demo/hello hello.js --parameter name Jane --web true
```
{: pre}

El resultado de estos cambios es que `name` se vincula a `Jane` y no lo pueden modificar ni los parámetros de la consulta ni los del cuerpo debido a la anotación final. Este diseño protege la acción frente a parámetros de consulta o de cuerpo que intenten modificar este valor, ya sea intencionadamente o por accidente.

## Protección de acciones web
{: #actions_web_secure}

De forma predeterminada, cualquier persona que tenga el URL de invocación de la acción web puede invocar una acción web. Utilice la [anotación de acción web](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions) `require-whisk-auth` para proteger la acción web. Cuando la anotación `require-whisk-auth` está establecida en `true`, la acción autenticará las credenciales de autorización básicas de la solicitud de invocación comparándola con la clave de autenticación whisk del propietario de la acción. Cuando se establece en un número o una serie de caracteres sensible a mayúsculas y minúsculas, la solicitud de invocación de la acción debe incluir una cabecera `X-Require-Whisk-Auth` que tenga el mismo valor. Las acciones web seguras devolverán el mensaje `No autorizado` cuando la validación de credenciales falle.

Como alternativa, puede utilizar el distintivo `--web-secure` para establecer automáticamente la anotación `require-whisk-auth`.  Si tiene el valor `true`, se genera un número aleatorio como valor de la anotación `require-whisk-auth`. Si tiene el valor `false`, la anotación `require-whisk-auth` se elimina.  Si tiene cualquier otro valor, se utiliza dicho valor como el valor de la anotación `require-whisk-auth`.

Ejemplo utilizando **--web-secure**:
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true --web-secure my-secret
```
{: pre}

Ejemplo utilizando **require-whisk-auth**:
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true -a require-whisk-auth my-secret
```
{: pre}

Ejemplo utilizando **X-Require-Whisk-Auth**:
```bash
curl https://${APIHOST}/api/v1/web/guest/demo/hello.json?name=Jane -X GET -H "X-Require-Whisk-Auth: my-secret"
```
{: pre}

Es importante tener en cuenta que el propietario de la acción web es el propietario de todos los registros de activación y es el responsable del coste de ejecutarlas en el sistema, independientemente de cómo se haya invocado la acción.

## Inhabilitación de acciones web
{: #actions_web_disable}

Para inhabilitar la invocación de una acción web mediante la API web (`https://openwhisk.bluemix.net/api/v1/web/`), pase el valor `false` o `no` al distintivo `--web` para actualizar una acción con la CLI.
```
ibmcloud fn action update /guest/demo/hello hello.js --web false
```
{: pre}

## Manejo de HTTP sin procesar
{: #actions_web_raw}

Una acción web puede optar por interpretar y procesar un cuerpo HTTP directamente, sin la promoción de un objeto JSON a las propiedades de primera clase disponibles para la entrada de la acción (es decir, `args.name` frente a analizar `args.__ow_query`). Este proceso se hace mediante `raw-http` [annotation](/docs/openwhisk?topic=cloud-functions-annotations). Utilizando el ejemplo anterior, pero ahora como una acción web HTTP "sin procesar" que recibe `name` como parámetro de consulta y como valor JSON en el cuerpo de la solicitud HTTP:
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane -X POST -H "Content-Type: application/json" -d '{"name":"Jane"}'
```
{: pre}

Salida de ejemplo:
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

OpenWhisk utiliza la infraestructura [Akka Http](https://doc.akka.io/docs/akka-http/current/?language=scala) para [determinar](https://doc.akka.io/api/akka-http/10.0.4/akka/http/scaladsl/model/MediaTypes$.html) qué tipos de contenido son binarios y cuáles son texto sin formato.

### Habilitación del manejo de HTTP sin procesar
{: #actions_web_raw_enable}

Las acciones web HTTP sin procesar se habilitan asignando al distintivo `--web` el valor `raw`.
```
ibmcloud fn action create /guest/demo/hello hello.js --web raw
```
{: pre}

### Inhabilitación del manejo de HTTP sin procesar
{: #actions_web_raw_disable}

La inhabilitación de HTTP sin procesar se consigue pasando el valor `false` o `no` al distintivo `--web`.
```
ibmcloud fn update create /guest/demo/hello hello.js --web false
```
{: pre}

### Decodificación de contenido de cuerpo binario de Base64
{: #actions_web_decode}

Al procesar contenido HTTP sin procesar, el contenido `__ow_body` se codifica en Base64 si la solicitud `Content-Type` es binaria. Las siguientes funciones muestran cómo decodificar el contenido del cuerpo en Node, Python y Swift. Simplemente guarde el método en un archivo, cree una nueva acción web HTTP sin procesar que utilice el artefacto guardado y, a continuación, invoque la acción web.

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

Como ejemplo, guarde la función Node como `decode.js` y ejecute los mandatos siguientes:
```
ibmcloud fn action create decode decode.js --web raw
```
{: pre}

Salida de ejemplo:
```
ok: created action decode
```
{: screen}

```
curl -k -H "content-type: application" -X POST -d "Decoded body" https:// us-south.functions.cloud.ibm.com/api/v1/web/guest/default/decodeNode.json
```
{: pre}

Salida de ejemplo:
```
{
  "body": "Decoded body"
}
```
{: screen}

## Solicitudes de Options
{: #actions_web_options}

De forma predeterminada, una solicitud OPTIONS realizada a una acción web da como resultado que las cabeceras de CORS se añadan automáticamente a las cabeceras de respuesta. Estas cabeceras permiten todos los orígenes y los verbos HTTP options, get, delete, post, put, head y patch.

Consulte las siguientes cabeceras:
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: OPTIONS, GET, DELETE, POST, PUT, HEAD, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
```

Como alternativa, se pueden manejar solicitudes OPTIONS manualmente mediante una acción web. Para habilitar esta opción, añada una anotación de `web-custom-options` con un valor de `true` a una acción web. Cuando esta característica está habilitada, las cabeceras CORS no se añaden automáticamente a la respuesta de la solicitud. En su lugar, es responsabilidad del desarrollador añadir sus cabeceras deseadas mediante programación.

Consulte el siguiente ejemplo para crear respuestas personalizadas en solicitudes OPTIONS:
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

Guarde la función en `custom-options.js` y ejecute los mandatos siguientes:
```
ibmcloud fn action create custom-option custom-options.js --web true -a web-custom-options true
```
{: pre}

```
$ curl https://${APIHOST}/api/v1/web/guest/default/custom-options.http -kvX OPTIONS
```
{: pre}

Salida de ejemplo:
```
< HTTP/1.1 200 OK
< Server: nginx/1.11.13
< Content-Length: 0
< Connection: keep-alive
< Access-Control-Allow-Methods: OPTIONS, GET
< Access-Control-Allow-Origin: example.com
```
{: screen}

## Manejo de errores
{: #actions_web_errors}

Cuando una acción de {{site.data.keyword.openwhisk_short}} falla, hay dos modalidades de anomalía posibles. La primera se conoce como _error de aplicación_ y es similar a una excepción interceptada: la acción devuelve un objeto JSON que contiene una propiedad `error` de nivel superior. La segunda es un _error de desarrollador_, que se produce cuando la acción falla de forma catastrófica y no genera ninguna respuesta (similar a una excepción no capturada). En el caso de acciones web, el controlador trata los errores de aplicación del siguiente modo:

- Cualquier proyección de vía de acceso especificada se pasa por alto y el controlador proyecta en su lugar la propiedad `error`.
- El controlador aplica el manejo de contenido implícito según la extensión de la acción al valor de la propiedad `error`.

Los desarrolladores deben saber cómo se pueden utilizar las acciones web y deben generar respuestas adecuadas a los errores. Por ejemplo, una acción web que se utilice con la extensión `.http` devuelve una respuesta HTTP, como por ejemplo `{error: { statusCode: 400 }`. Si no es así, se produce una discrepancia entre el `Content-Type` implícito a partir de la extensión y el `Content-Type` de la acción en la respuesta al error. Hay que tener especial cuidado con las acciones web que son secuencias, de forma que los componentes que forman una secuencia puedan generar errores adecuados cuando es necesario.

## Ejemplo: Generación de una imagen de código QR a partir de la entrada
{: #actions_web_qr}

A continuación se muestra un ejemplo de una acción web Java que acepta `text` como entrada y genera una imagen de código QR.

1. Cree un archivo `Generate.java` en el directorio `java_example/src/main/java/qr`.

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

3. Compile el JAR de la acción web ejecutando el mandato siguiente desde el directorio `java_example` donde se encuentra el archivo `build.gradle`.

  ```bash
  gradle jar
  ```
  {: pre}

4. Despliegue la acción web utilizando el JAR `build/libs/java_example-1.0.jar`.

  ```bash
  ibmcloud fn action update QRGenerate build/libs/java_example-1.0.jar --main qr.Generate -m 128 --web true
  ```
  {: pre}

5. Recupere el URL público del punto final de acción web y asígnelo a una variable de entorno.

  ```bash
  ibmcloud fn action get QRGenerate --url
  URL=$(ibmcloud fn action get QRGenerate --url | tail -1)
  ```
  {: pre}

6. Puede abrir un navegador web utilizando este `URL` y añadir el parámetro de consulta
`text` con el mensaje a codificar en la imagen QR. También puede utilizar un cliente HTTP como
`curl` para descargar una imagen QR.

  ```bash
  curl -o QRImage.png $URL\?text=https://cloud.ibm.com
  ```
  {: pre}
