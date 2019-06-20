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


# Création d'actions Web
{: #actions_web}

Les actions Web sont des actions {{site.data.keyword.openwhisk}} qui sont annotées pour permettre aux développeurs de générer rapidement des applications Web. Ces actions annotées permettent aux développeurs de programmer une logique de back end à laquelle votre application Web peut accéder de manière anonyme, sans avoir à fournir une clé d'authentification {{site.data.keyword.openwhisk_short}}. Il revient au développeur d'action d'implémenter son propre mécanisme d'authentification et d'autorisation, ou un flux de protocole d'autorisation OAuth.
{: shortdesc}

Des activations d'action Web sont associées à l'utilisateur qui a créé l'action. Cette action reporte le coût d'une action de l'appelant au propriétaire de l'action.

Examinez l'action JavaScript `hello.js` suivante :
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

Vous pouvez créer une  _action Web_ **hello** dans le package `demo` pour l'espace de nom `guest` en utilisant l'indicateur `--web` de l'interface de ligne de commande avec la valeur `true` ou `yes` :
```
ibmcloud fn package create demo
```
{: pre}

```
ibmcloud fn action create /guest/demo/hello hello.js --web true
```
{: pre}

L'utilisation de l'indicateur `--web` avec la valeur `true` ou `yes` rend une action accessible au moyen d'une interface REST sans avoir à spécifier des données d'identification. Pour configurer une action Web avec des données d'identification, voir la section [Sécurisation des actions Web](#actions_web_secure). Une action Web peut être appelée en utilisant une URL structurée comme suit :
`https://{APIHOST}/api/v1/web/{namespace}/{packageName}/{actionName}.{EXT}`.

Le nom du package est **default** si l'action n'est pas un package nommé.

Par exemple, `guest/demo/hello`. Le chemin de l'API de l'action Web peut être utilisé avec `curl` ou `wget` sans clé d'API. Il peut même être entré directement dans votre navigateur.

Tentez d'ouvrir `https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane` dans votre navigateur Web. Ou essayez d'appeler l'action en utilisant `curl` :
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello?name=Jane
```
{: pre}

Dans l'exemple suivant, une action Web effectue une redirection HTTP :
```javascript
function main() {
  return {
    headers: { location: 'http://openwhisk.org' },
    statusCode: 302
  }
}
```
{: codeblock}

Dans l'exemple suivant, une action Web définit un seul cookie :
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

Dans l'exemple suivant, une action Web définit plusieurs cookies :
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

L'exemple suivant renvoie un élément `image/png` :
```javascript
function main() {
    let png = <base 64 encoded string>
    return { headers: { 'Content-Type': 'image/png' },
             statusCode: 200,
             body: png };
}
```
{: codeblock}

L'exemple suivant renvoie `application/json` :
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

La valeur par défaut de `Content-Type` pour une réponse HTTP est `application/json`, et le corps peut être n'importe quelle valeur JSON autorisée. La valeur par défaut de `Content-Type` peut être omise dans les en-têtes.

Il est important de tenir compte de la [taille limite de la réponse](/docs/openwhisk?topic=cloud-functions-limits) pour les actions car une réponse dépassant les limites du système prédéfinies est vouée à l'échec. Les objets volumineux ne sont pas envoyés en ligne via {{site.data.keyword.openwhisk_short}}, mais différés vers un conteneur d'objets, par exemple.

## Traitement des demandes HTTP à l'aide d'actions
{: #actions_web_http}

Une action {{site.data.keyword.openwhisk_short}} qui n'est pas une action Web requiert une authentification et doit répondre avec un objet JSON.

Les actions Web peuvent être appelées sans authentification et peuvent être utilisées pour implémenter des gestionnaires HTTP qui répondent avec un contenu `headers`, `statusCode` et `body` de différents types.
Les actions Web doivent renvoyer un objet JSON. Toutefois, le contrôleur traite une action Web différemment si
son résultat comprend une ou plusieurs des propriétés JSON de niveau supérieur suivantes :

- `headers` : objet JSON dans lequel les clés sont des noms d'en-tête et les valeurs sont des valeurs de chaîne, numériques ou booléennes. Pour que plusieurs valeurs puissent être envoyées pour un seul en-tête, la valeur de l'en-tête est un tableau de ces valeurs. Aucun en-tête n'est défini par défaut.
- `statusCode` : code d'état HTTP valide. Si le corps comporte un contenu, la valeur par défaut est `200 OK`. Si aucun contenu n'est présent pour le corps, la valeur par défaut est `204 No Content`.
- `body` : chaîne qui est un texte en clair, un tableau ou un objet JSON, ou une chaîne codée en base64 pour les données binaires. Le corps est considéré comme vide s'il comporte la valeur `null`, la chaîne vide `""`, ou s'il n'est pas défini. La valeur par défaut est un corps vide.

Le contrôleur transmet les en-têtes spécifiés par une action, le code d'état ou le corps au client HTTP qui met fin à la demande ou à la réponse. Si l'en-tête `Content-Type` n'est pas déclaré dans le contenu `headers` du résultat de l'action, le corps est interprété en tant que `application/json` pour les valeurs qui ne sont pas des chaînes, et en tant que `text/html` pour les autres. Lorsque l'en-tête `Content-Type` est défini, le contrôleur détermine si la réponse contient des données binaires ou un texte en clair et décode la chaîne à l'aide                  *d'un décodeur de base64 si besoin. Si le corps n'est pas décodé correctement, une erreur est renvoyée au client.


## Contexte HTTP
{: #actions_web_context}

Lorsqu'elles sont appelées, toutes les actions Web reçoivent des informations de demande HTTP en tant que paramètres pour l'argument d'entrée d'action.

Examinez les paramètres HTTP suivants :

- `__ow_method` (type : chaîne) : méthode HTTP de la demande.
- `__ow_headers` (type : mappe chaîne à chaîne) : en-têtes de demande.
- `__ow_path` (type : chaîne) : chemin sans correspondance de la demande (la correspondance cesse une fois l'extension d'action consommée).
- `__ow_user` (type : chaîne) : espace nom qui identifie l'objet authentifié par {{site.data.keyword.openwhisk_short}}.
- `__ow_body` (type : chaîne) : entité de corps de demande sous la forme d'une chaîne codée en base64 lorsque le contenu est binaire, sinon, sous la forme d'une chaîne en clair.
- `__ow_query` (type : chaîne) : paramètres de requête de la demande en tant que chaîne non analysée.

Une demande ne peut remplacer aucun des paramètres `__ow_` nommés. Si cela se produit, la demande échoue avec l'état 400 Bad Request.

Le paramètre `__ow_user` n'est présent que lorsque l'action Web est [annotée pour indiquer que l'authentification est requise](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions) et permet à une action Web d'implémenter sa propre règle d'autorisation. Le paramètre `__ow_query` n'est disponible que lorsqu'une action Web choisit de traiter la [demande HTTP "raw"](#actions_web_raw_enable). Il s'agit d'une chaîne contenant les paramètres de requête qui sont analysés à partir de l'URI (séparés par `&`). La propriété `__ow_body` est présente dans les demandes HTTP brutes ou lorsque l'entité de demande HTTP n'est pas un objet JSON ni des données de formulaire. Sinon, les actions Web reçoivent les paramètres de requête et de corps comme propriétés de première classe dans les arguments d'action. Les paramètres de corps sont prioritaires sur les paramètres de requête, lesquels sont prioritaires sur les paramètres d'action et de package.

## Prise en charge de noeud final HTTPS
{: #actions_web_endpoint}

Protocoles SSL pris en charge : TLS 1.2, TLS 1.3 ([version brouillon 18](https://tools.ietf.org/html/draft-ietf-tls-tls13-18))

Protocoles SSL non pris en charge : SSLv2, SSLv3, TLS 1.0, TLS 1.1

## Fonctions supplémentaires
{: #actions_web_extra}

Les actions Web offrent des fonctions supplémentaires, parmi lesquelles :

- `Extensions de contenu` : la demande doit préciser le type de contenu souhaité, par exemple, `.json`, `.html`, `.http`, `.svg` ou `.text`. Le type est spécifié en ajoutant une extension au nom de l'action dans l'URI, ainsi, une action `/guest/demo/hello` est référencée sous la forme `/guest/demo/hello.http`, par exemple, pour recevoir une réponse HTTP en retour. Par souci de commodité, l'extension `.http` est utilisée par défaut lorsqu'aucune extension n'est détectée.
- `Projection de zones à partir du résultat` : le chemin qui suit le nom de l'action est utilisé pour projeter un ou plusieurs niveaux de la réponse.
`/guest/demo/hello.html/body`. Cette fonction permet à une action qui renvoie un dictionnaire `{body: "..." }` de projeter la propriété `body` et de renvoyer directement sa valeur de chaîne à la place. Le chemin projeté suit un modèle de chemin d'accès absolu (comme dans XPath).
- `Paramètres de requête et de corps en tant qu'entrées` : l'action reçoit des paramètres de requête, mais également des paramètres dans le corps de la demande. L'ordre de priorité pour la fusion des paramètres est le suivant : paramètres de package, paramètres d'action, paramètres de requête et paramètres de corps. Chacun de ces paramètres peut remplacer les valeurs précédentes en cas de chevauchement. Par exemple, `/guest/demo/hello.http?name=Jane` peut transmettre l'argument `{name: "Jane"}` à l'action.
- `Données de formulaire` : outre l'élément `application/json` standard, les actions Web peuvent recevoir une URL codée à partir des données `application/x-www-form-urlencoded data` comme entrée.
- `Activation utilisant plusieurs verbes HTTP` : une action Web peut être appelée via l'une des méthodes HTTP suivantes : `GET`, `POST`, `PUT`, `PATCH` et `DELETE`, ainsi que `HEAD` et `OPTIONS`.
- `Traitement de corps autre que JSON et d'entité HTTP brute` : une action Web peut accepter un corps de demande HTTP autre qu'un objet JSON et peut choisir de toujours recevoir de telles valeurs comme valeurs opaques (texte en clair avec de données non binaires, sinon, chaîne codée en base64).

L'exemple ci-dessous montre de façon sommaire comment utiliser ces fonctions dans une action Web. Imaginons une action `/guest/demo/hello` avec le corps suivant :
```javascript
function main(params) {
    return { response: params };
}
```

Lorsque cette action est appelée en tant qu'action Web, vous pouvez modifier la réponse de l'action Web en projetant différents chemins à partir du résultat.

Par exemple, pour renvoyer la totalité de l'objet et voir les arguments que l'action reçoit :
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json
```
{: pre}

Exemple de sortie :
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

Pour effectuer une exécution avec un paramètre de requête, examinez l'exemple de commande suivant :
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane
 ```
{: pre}

Exemple de sortie :
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

Vous pouvez également effectuer une exécution avec des données de formulaire :
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -d "name":"Jane"
 ```
{: pre}

Exemple de sortie :
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

Exécutez la commande suivante pour un objet JSON :
```
 curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: application/json' -d '{"name":"Jane"}'
```
{: pre}

Exemple de sortie :
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

Exécutez la commande suivante pour projeter le nom (en tant que texte) :
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.text/response/name?name=Jane
```
{: pre}

Exemple de sortie :
```
Jane
```
{: screen}

Par souci de commodité, les paramètres de requête, les données de formulaire et les entités de corps d'objet JSON sont tous traités en tant que dictionnaires, car leurs valeurs sont directement accessibles en tant que propriétés d'entrée d'action. Ce n'est pas le cas des actions Web qui choisissent de traiter les entités de demande HTTP plus directement ou lorsque l'action Web reçoit une entité qui n'est pas un objet JSON.

Examinez l'exemple suivant qui utilise un type de contenu "text", comme illustré précédemment :
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json -H 'Content-Type: text/plain' -d "Jane"
```
{: pre}

Exemple de sortie :
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

## Extensions de contenu
{: #actions_web_ext}

Une extension de contenu est généralement nécessaire pour appeler une action Web. En l'absence d'extension, `.http` est utilisé par défaut. Les extensions `.json` et `.http` ne requièrent pas de chemin de projection, contrairement aux extensions `.html`, `.svg` et `.text`. Par souci de commodité, on considère que le chemin par défaut correspond au nom d'extension. Pour appeler une action Web et recevoir une réponse `.html`, l'action doit répondre avec un objet JSON qui contient une propriété de niveau supérieur nommée `html` (ou bien la réponse doit figurer dans le chemin explicite). En d'autres termes, `/guest/demo/hello.html` revient à projeter la propriété `html` de manière explicite, comme dans `/guest/demo/hello.html/html`. Le nom qualifié complet de l'action doit inclure son nom de package, qui est `default` si l'action ne figure pas dans un package nommé.

## Paramètres protégés
{: #actions_web_protect}

Les paramètres d'action sont protégés et traités comme non modifiables. Les paramètres sont finalisés automatiquement pour activer des actions Web.
```
ibmcloud fn action create /guest/demo/hello hello.js --parameter name Jane --web true
```
{: pre}

Ces modifications ont pour conséquence de lier `name` à `Jane`, ce qui ne peut pas être remplacé par des paramètres de requête ou de corps en raison de l'annotation finale. Cette conception permet de sécuriser l'action sur les paramètres de requête ou de corps qui tentent de modifier cette valeur de manière intentionnelle ou fortuite.

## Sécurisation des actions Web
{: #actions_web_secure}

Par défaut, une action Web peut être appelée par toute personne disposant de l'URL d'appel de l'action Web. Utilisez l'[annotation d'action Web](/docs/openwhisk?topic=cloud-functions-annotations#annotations-specific-to-web-actions) `require-whisk-auth` pour sécuriser l'action Web. Lorsque l'annotation `require-whisk-auth` a la valeur `true`, l'action authentifie les données d'identification pour l'autorisation de base de la demande d'appel par rapport à la clé d'authentification Whisk du propriétaire de l'action. Lorsqu'elle est définie avec un nombre ou une chaîne sensible à la casse, la demande d'appel de l'action doit inclure un en-tête `X-Require-Whisk-Auth` ayant la même valeur. Les actions Web sécurisées renverront le message `Not Authorized` en cas d'échec de validation des données d'identification.

Sinon, utilisez l'indicateur `--web-secure` pour définir automatiquement l'annotation `require-whisk-auth`.  Si la valeur définie est `true`, un nombre aléatoire est généré comme valeur pour l'annotation `require-whisk-auth`. Si la valeur définie est `false`, l'annotation `require-whisk-auth` est retirée.  Avec une autre valeur définie, c'est cette valeur qui est utilisée comme valeur pour l'annotation `require-whisk-auth`.

Exemple d'utilisation de **--web-secure** :
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true --web-secure my-secret
```
{: pre}

Exemple d'utilisation de **require-whisk-auth** :
```bash
ibmcloud fn action update /guest/demo/hello hello.js --web true -a require-whisk-auth my-secret
```
{: pre}

Exemple d'utilisation de **X-Require-Whisk-Auth** :
```bash
curl https://${APIHOST}/api/v1/web/guest/demo/hello.json?name=Jane -X GET -H "X-Require-Whisk-Auth: my-secret"
```
{: pre}

Il est important de noter que le propriétaire de l'action Web détient tous les enregistrements des activations et supporte les coûts liés à l'exécution de l'action dans le système quel que soit le mode d'appel de l'action employé.

## Désactivation d'actions Web
{: #actions_web_disable}

Pour désactiver l'appel d'une action Web via l'API Web (`https://openwhisk.bluemix.net/api/v1/web/`), attribuez la valeur `false` ou `no` à l'indicateur `--web` pour mettre à jour une action à l'aide de l'interface de ligne de commande.
```
ibmcloud fn action update /guest/demo/hello hello.js --web false
```
{: pre}

## Traitement de demande HTTP brute
{: #actions_web_raw}

Une action Web peut choisir d'interpréter et de traiter directement un corps HTTP entrant, sans la promotion d'un objet JSON vers les propriétés de première classe disponibles pour l'entrée d'action (par exemple, `args.name` par rapport à l'analyse `args.__ow_query`). Ce processus s'effectue via une [annotation](/docs/openwhisk?topic=cloud-functions-annotations) `raw-http`. Utilisation de l'exemple précédent, mais cette fois-ci avec une action Web HTTP brute qui reçoit `name` comme paramètre de requête et comme valeur JSON dans le corps de demande HTTP :
```
curl https://us-south.functions.cloud.ibm.com/api/v1/web/guest/demo/hello.json?name=Jane -X POST -H "Content-Type: application/json" -d '{"name":"Jane"}'
```
{: pre}

Exemple de sortie :
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

OpenWhisk utilise l'infrastructure [Akka Http](https://doc.akka.io/docs/akka-http/current/?language=scala) pour [déterminer](https://doc.akka.io/api/akka-http/10.0.4/akka/http/scaladsl/model/MediaTypes$.html) les types de contenu binaires et les types de contenu de texte en clair.

### Activation du traitement de demande HTTP brute
{: #actions_web_raw_enable}

Les actions Web HTTP brutes sont activées via l'indicateur `--web` en utilisant la valeur `raw`.
```
ibmcloud fn action create /guest/demo/hello hello.js --web raw
```
{: pre}

### Désactivation du traitement de demande HTTP brute
{: #actions_web_raw_disable}

La désactivation des actions HTTP brutes est réalisée en attribuant la valeur `false` ou `no` à l'indicateur `--web`.
```
ibmcloud fn update create /guest/demo/hello hello.js --web false
```
{: pre}

### Décodage de contenu de code binaire depuis Base64
{: #actions_web_decode}

Lorsqu'un contenu HTTP brut est traité, le contenu `__ow_body` est codé en Base64 lorsque la partie `Content-Type` de la demande est binaire. Les fonctions ci-dessous montrent comment décoder le contenu du corps dans Node, Python et Swift. Il suffit d'enregistrer une méthode dans un fichier, de créer une action Web HTTP brute qui utilise l'artefact sauvegardé et d'appeler l'action Web.

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

Par exemple, enregistrez la fonction Node en tant que `decode.js` et exécutez les commandes suivantes :
```
ibmcloud fn action create decode decode.js --web raw
```
{: pre}

Exemple de sortie :
```
ok: created action decode
```
{: screen}

```
curl -k -H "content-type: application" -X POST -d "Decoded body" https:// us-south.functions.cloud.ibm.com/api/v1/web/guest/default/decodeNode.json
```
{: pre}

Exemple de sortie :
```
{
  "body": "Decoded body"
}
```
{: screen}

## Demandes OPTIONS
{: #actions_web_options}

Par défaut, une demande OPTIONS adressée à une action Web entraîne l'ajout automatique d'en-têtes CORS dans les en-têtes de réponse. Ces en-têtes admettent toutes les origines et les options, ainsi que les termes HTTP get, delete, post, put, head et patch.

Examinez les en-têtes suivants :
```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: OPTIONS, GET, DELETE, POST, PUT, HEAD, PATCH
Access-Control-Allow-Headers: Authorization, Content-Type
```

Les demandes OPTIONS peuvent également être traitées manuellement par une action Web. Pour activer cette option, ajoutez une annotation
`web-custom-options` avec la valeur `true` à une action Web. Lorsque cette fonction est activée, les en-têtes CORS ne sont pas ajoutés automatiquement dans la réponse à la demande. Il incombe à la place au développeur d'ajouter les en-têtes voulus à l'aide d'un programme.

Examinez l'exemple ci-dessous qui illustre la création de réponses personnalisées à des demandes OPTIONS :
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

Enregistrez la fonction dans le fichier `custom-options.js` et exécutez les commandes suivantes :
```
ibmcloud fn action create custom-option custom-options.js --web true -a web-custom-options true
```
{: pre}

```
$ curl https://${APIHOST}/api/v1/web/guest/default/custom-options.http -kvX OPTIONS
```
{: pre}

Exemple de sortie :
```
< HTTP/1.1 200 OK
< Server: nginx/1.11.13
< Content-Length: 0
< Connection: keep-alive
< Access-Control-Allow-Methods: OPTIONS, GET
< Access-Control-Allow-Origin: example.com
```
{: screen}

## Traitement des erreurs
{: #actions_web_errors}

Une action {{site.data.keyword.openwhisk_short}} échoue dans deux modes d'échec possibles. Le premier correspondant à une erreur d'application (_application error_) est semblable à une exception interceptée : l'action renvoie un objet JSON contenant une propriété `error` de niveau supérieur. Le second correspondant à une erreur de développeur (_developer error_) est activé lorsque l'action échoue de manière désastreuse sans générer de réponse (semblable à une exception non interceptée). Pour les actions Web, le contrôleur traite les erreurs d'application comme suit :

- Toute projection de chemin spécifiée est ignorée et le contrôleur projette la propriété `error` à la place.
- Le contrôleur applique le traitement de contenu induit par l'extension d'action à la valeur de la propriété `error`.

Les développeurs doivent savoir comment les actions Web peuvent être utilisées et ils doivent générer des réponses d'erreur en conséquence. Par exemple, une action Web qui est utilisée avec l'extension `.http` renvoie une réponse HTTP de ce type : `{error: { statusCode: 400 }`. Si tel n'est pas le cas, une non-concordance est générée entre le type de contenu (`Content-Type`) induit par l'extension et le type de contenu (`Content-Type`) de l'action dans la réponse d'erreur. Une attention particulière doit être accordée aux actions Web qui sont des séquences, de sorte que les composants qui constituent une séquence puissent générer les erreurs appropriées, le cas échéant.

## Exemple : Génération d'une image de code QR à partir d'une entrée
{: #actions_web_qr}

Voici un exemple d'action Web Java qui utilise `text` comme entrée et génère une image de code QR.

1. Créez un fichier `Generate.java` dans le répertoire `java_example/src/main/java/qr`.

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

3. Créez le fichier JAR de l'action Web en exécutant la commande suivante à partir du répertoire `java_example`, où se trouve le fichier `build.gradle`.

  ```bash
  gradle jar
  ```
  {: pre}

4. Déployez l'action Web à l'aide du fichier JAR `build/libs/java_example-1.0.jar`.

  ```bash
  ibmcloud fn action update QRGenerate build/libs/java_example-1.0.jar --main qr.Generate -m 128 --web true
  ```
  {: pre}

5. Extrayez l'URL publique du noeud final de l'action Web et affectez-la à une variable d'environnement.

  ```bash
  ibmcloud fn action get QRGenerate --url
  URL=$(ibmcloud fn action get QRGenerate --url | tail -1)
  ```
  {: pre}

6. Vous pouvez ouvrir un navigateur Web à l'aide de cette `URL` et lui ajouter le paramètre de requête `text` avec le message à coder dans l'image QR. Vous pouvez également utiliser un client HTTP comme `curl` pour télécharger une image QR.

  ```bash
  curl -o QRImage.png $URL\?text=https://cloud.ibm.com
  ```
  {: pre}
