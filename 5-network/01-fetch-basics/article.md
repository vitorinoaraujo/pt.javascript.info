
# Fetch

JavaScript pode enviar pedidos de rede para o servidor e carregar nova informação sempre que for necessária.

Por exemplo, podemos usar um pedido de rede para:

- Submeter um pedido,
- Carregar informação sobre user,
- Receber updates mais recentes do servidor,
- ...etc.

...E tudo isto sem fazer _reload_ da página!

There's an umbrella term "AJAX" (abbreviated <b>A</b>synchronous <b>J</b>avaScript <b>A</b>nd <b>X</b>ML) for network requests from JavaScript. We don't have to use XML though: the term comes from old times, that's why that word is there. You may have heard that term already.

Existem vários métodos de fazer um pedido de rede e obter informação de um servidor.

O método `fetch()` é moderno e versátil, portanto será esse pelo qual iremos começar. Não é suportado por browsers mais antigos (podem ser usados polyfills), mas tem suporte muito bom nos mais modernos.

A sintaxe básica é a seguinte:

```js
let promise = fetch(url, [options])
```

- **`url`** -- o URL a aceder.
- **`options`** -- parâmetros opcionais: método, cabeçalhos etc.

Sem `options`, trata-se de um simples pedido GET, fazendo o download do conteúdo do `url`.

O browser inicia o pedido de imediato e retorna uma `promise` de que o código invocado deve ser usado para obter o resultado.

Obter uma resposta é normalmente um processo que se divide em dois passos.

**Primeiro, a `promise`, retornada por `fetch`, responde com um objecto do tipo [Response](https://fetch.spec.whatwg.org/#response-class) class assim que o servidor responde com os headers.**

Neste instante podemos validar o status HTTP, para verificar se teve sucesso ou não, validar os headers, mas ainda não temos o body.

A `promise`rejeita se o `fetch`não conseguir obter o pedido HTTP, ex. problemas de rede, ou a não existência do site. Códigos de status HTTP tais como 404 ou 500 não causam erro.

Podemos ver os Status HTTP nas propriedades da resposta:

- **`status`** -- código de status HTTP, ex. 200.
- **`ok`** -- booleano, `true` se o código de status HTTP for 200-299.

Por exemplo:

```js
let response = await fetch(url);

if (response.ok) { // if HTTP-status is 200-299
  // get the response body (the method explained below)
  let json = await response.json();
} else {
  alert("HTTP-Error: " + response.status);
}
```
**Segundo, para obter o corpo da resposta, temos de usar uma chamada de método adicional**

`Response` devolve vários métodos do tipo `promise`que permitem aceder ao corpo da resposta em vários formatos:

- **`response.text()`** -- lê a resposta e devolve como texto,
- **`response.json()`** -- lê a resposta e devolve como JSON,
- **`response.formData()`** -- lê a resposta e devolve como objecto `FormData` (explicado no [próximo capítulo](info:formdata)),
- **`response.blob()`** -- lê a resposta e devolve como [Blob](info:blob) (tipo de dados binário),
- **`response.arrayBuffer()`** -- lê a resposta e devolve como [ArrayBuffer](info:arraybuffer-binary-arrays) (representação de baixo nível de tipos de dado binário),
- adicionalmente, `response.body` é um objecto do tipo [ReadableStream](https://streams.spec.whatwg.org/#rs-class) , permite que o cporto da resposta seja lido `chunk-by-chunk`, veremos um exemplo mais à frente.

Por exemplo, vamos obter um objecto JSON com os commits mais recentes do GitHub:

```js run async
let url = 'https://api.github.com/repos/javascript-tutorial/en.javascript.info/commits';
let response = await fetch(url);

*!*
let commits = await response.json(); // read response body and parse as JSON
*/!*

alert(commits[0].author.login);
```

Ou, o mesmo mas sem o uso de `await`, usando o sintaxe das `promises`:

```js run
fetch('https://api.github.com/repos/javascript-tutorial/en.javascript.info/commits')
  .then(response => response.json())
  .then(commits => alert(commits[0].author.login));
```

Para obter o texto de resposta, `await response.text()` em vez de `.json()`:

```js run async
let response = await fetch('https://api.github.com/repos/javascript-tutorial/en.javascript.info/commits');

let text = await response.text(); // read response body as text

alert(text.slice(0, 80) + '...');
```

Para demonstrat o uso da leitura em formato binário, vamos obter e mostrar uma imagem de ["fetch" specification](https://fetch.spec.whatwg.org) (ver capítulo [Blob](info:blob) para detalhes acerca  `Blob`):

```js async run
let response = await fetch('/article/fetch/logo-fetch.svg');

*!*
let blob = await response.blob(); // download as Blob object
*/!*

// create <img> for it
let img = document.createElement('img');
img.style = 'position:fixed;top:10px;left:10px;width:100px';
document.body.append(img);

// show it
img.src = URL.createObjectURL(blob);

setTimeout(() => { // hide after three seconds
  img.remove();
  URL.revokeObjectURL(img.src);
}, 3000);
```

````warn
We can choose only one body-reading method.

If we've already got the response with `response.text()`, then `response.json()` won't work, as the body content has already been processed.

```js
let text = await response.text(); // response body consumed
let parsed = await response.json(); // fails (already consumed)
````

## Response headers

The response headers are available in a Map-like headers object in `response.headers`.

It's not exactly a Map, but it has similar methods to get individual headers by name or iterate over them:

```js run async
let response = await fetch('https://api.github.com/repos/javascript-tutorial/en.javascript.info/commits');

// get one header
alert(response.headers.get('Content-Type')); // application/json; charset=utf-8

// iterate over all headers
for (let [key, value] of response.headers) {
  alert(`${key} = ${value}`);
}
```

## Headers dos pedidos

To set a request header in `fetch`, we can use the `headers` option. It has an object with outgoing headers, like this:

```js
let response = fetch(protectedUrl, {
  headers: {
    Authentication: 'secret'
  }
});
```

...mas existe uma lista de [headers HTTP proibidos](https://fetch.spec.whatwg.org/#forbidden-header-name) que não podemos utilizar:

- `Accept-Charset`, `Accept-Encoding`
- `Access-Control-Request-Headers`
- `Access-Control-Request-Method`
- `Connection`
- `Content-Length`
- `Cookie`, `Cookie2`
- `Date`
- `DNT`
- `Expect`
- `Host`
- `Keep-Alive`
- `Origin`
- `Referer`
- `TE`
- `Trailer`
- `Transfer-Encoding`
- `Upgrade`
- `Via`
- `Proxy-*`
- `Sec-*`

Estes métodos asseguram um adequado e seguro HTTP, por isso são controlados exlusivamente pelo browser.

## Pedidos POST

Para fazer um pedido `POST`, ou um pedido com um outro método, temos de usar as opções `fetch`:

- **`method`** -- HTTP-method, e.g. `POST`,
- **`body`** -- the request body, one of:
  - a string (e.g. JSON-encoded),
  - `FormData` object, to submit the data as `form/multipart`,
  - `Blob`/`BufferSource` to send binary data,
  - [URLSearchParams](info:url), to submit the data in `x-www-form-urlencoded` encoding, rarely used.

O formato JSON é usado na maioria das vezes.

Por exemplo, este código submete o objecto `user` como JSON:

```js run async
let user = {
  name: 'John',
  surname: 'Smith'
};

*!*
let response = await fetch('/article/fetch/post/user', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json;charset=utf-8'
  },
  body: JSON.stringify(user)
});
*/!*

let result = await response.json();
alert(result.message);
```

Please note, if the request `body` is a string, then `Content-Type` header is set to `text/plain;charset=UTF-8` by default.

But, as we're going to send JSON, we use `headers` option to send `application/json` instead, the correct `Content-Type` for JSON-encoded data.

## Envio de uma imagem

Também podemos submeter dados binários com `fetch` usando objectos do tipo `Blob` ou `BufferSource`.

Neste exemplo, existe um `<canvas>` onde podemos desenhar movendo o ponteiro do rato. Um click no botão de "submit" envia a imagem para o servidor:

```html run autorun height="90"
<body style="margin:0">
  <canvas id="canvasElem" width="100" height="80" style="border:1px solid"></canvas>

  <input type="button" value="Submit" onclick="submit()">

  <script>
    canvasElem.onmousemove = function(e) {
      let ctx = canvasElem.getContext('2d');
      ctx.lineTo(e.clientX, e.clientY);
      ctx.stroke();
    };

    async function submit() {
      let blob = await new Promise(resolve => canvasElem.toBlob(resolve, 'image/png'));
      let response = await fetch('/article/fetch/post/image', {
        method: 'POST',
        body: blob
      });

      // the server responds with confirmation and the image size
      let result = await response.json();
      alert(result.message);
    }

  </script>
</body>
```

Please note, here we don't set `Content-Type` header manually, because a `Blob` object has a built-in type (here `image/png`, as generated by `toBlob`). For `Blob` objects that type becomes the value of `Content-Type`.

A função `submit()` pode ser reescrita sem `async/await` do seguinte modo:

```js
function submit() {
  canvasElem.toBlob(function(blob) {        
    fetch('/article/fetch/post/image', {
      method: 'POST',
      body: blob
    })
      .then(response => response.json())
      .then(result => alert(JSON.stringify(result, null, 2)))
  }, 'image/png');
}
```

## Summário

Um pedido típico de fetch consiste em duas chamadas `await`:

```js
let response = await fetch(url, options); // resolves with response headers
let result = await response.json(); // read body as json
```

Ou, sem `await`:

```js
fetch(url, options)
  .then(response => response.json())
  .then(result => /* process result */)
```

Response properties:
- `response.status` -- código HTTP da resposta,
- `response.ok` -- `true` se o status é 200-299.
- `response.headers` -- objecto do tipo Map que contém os HTTP headers.

Methods to get response body:
- **`response.text()`** -- retorna a resposta como texto,
- **`response.json()`** -- parsa a resposta como um objecto JSON,
- **`response.formData()`** -- return the response as `FormData` object (form/multipart encoding, see the next chapter),
- **`response.blob()`** -- return the response as [Blob](info:blob) (binary data with type),
- **`response.arrayBuffer()`** -- return the response as [ArrayBuffer](info:arraybuffer-binary-arrays) (low-level binary data),

Opções fetch disponíveis:
- `method` -- método HTTP,
- `headers` -- um objecto com os request headers (nem todos os headers são permitidos),
- `body` -- a data a ser enviada (o body do pedido) como `string`, `FormData`, `BufferSource`, `Blob` ou objecto `UrlSearchParams`.

Nos próximos capitulos iremos ver mais opções e casos de uso para `fetch`.
