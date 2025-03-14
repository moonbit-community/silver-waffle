# Create a LLM proxy

This part is divided as follows

1. Create a Server that says Hello World
2. Make the Server echos the body
3. Send request to get <https://www.example.com>
4. Send request to OpenRouter to access the LLM service

## Create a Server

In this section, we create a HTTP server using MoonBit with wasi-http 0.2.0,
that prints something in the response body.

### Task

- Clone <https://github.com/peter-jerry-ye/http-template.git>, or create a new repository and clone your own.
- Run the following script in the folder:
  ```bash
  moon update
  moon build --target wasm --debug
  wasm-tools component embed --encoding utf16 wit target/wasm/debug/build/http-server.wasm -o target/server.core.wasm
  wasm-tools component new target/server.core.wasm -o target/server.wasm
  # In the first terminal
  wasmtime serve target/server.wasm
  # In the second terminal
  curl localhost:8080
  ```

### Explanation

We've cloned a template for developing HTTP application in MoonBit. Then we ran
a few commands to create a componentized Wasm.

For the commands:

- `moon update`:\
  this command update the index of the package registry, <https://mooncakes.io>.
- `moon build --target wasm --debug`:\
  this command builds the project, generates the result in Wasm 1 backend (by
  default it would be Wasm GC, which is not supported yet by the Component Model
  proposal), in the debug mode (which you can remove when you are ready to
  publish).
- `wasm-tools component embed --encoding utf16 wit target/wasm/debug/build/http-server.wasm -o target/server.core.wasm`:\
  This embeds the interface information, under the folder `wit`, into the Wasm
  that we've just produced. We also specify that the encoding of the language is
  UTF16.
- `wasm-tools component new target/server.core.wasm -o target/server.wasm`:\
  This transforms the standard Wasm with the meta information into the so called
  componentized Wasm, i.e. the Wasm following the Component Model's standard.
- `wasmtime serve target/server.wasm`:\
  We use the wasmtime to serve this Wasm at port 8080. It will respond
  `Hello World!` in the body.

For the code:

The most important part is inside the `src/stub.mbt`, where we have a public
function called `handle`. This is the required function by the `wasi-http`
standard. It takes an incoming request, and an `ResponseOutParam`, on which we
should set the actual response.

In the function body, `@promise.spawn` spawns a coroutine defined with an async
function, which will call the function `top` and handle the error if anything
occurs. Then the event loop is initialized.

The `top` function defines the main logic. First, we create an outgoing response
with empty headers with the helper function of `@http.headers`. We then set the
status as 200, and creates a response body. The body returned is a `Result`
because the designer of the API wants to make sure that it is retrieved only
once. We simply `unwrap` it. We then set our response to the `response_out`.
Doing so will start the sending of the response immediately with the body
transmitted in the chunked manner. We then retrieve the output stream by
`outgoing_body.write().unwrap()`, and use `@io.println` to write a `String` into
it. We wait for its termination, and drop the output stream before finishing the
outgoing body.

All the imported packages can be checked in `src/moon.pkg.json`. All the
dependencies can be checked in `moon.mod.json`.

### Exercise

Try to print something other than `Hello World`.

```{seealso}
The documentation of the packages can be found in:

- wasi-imports: <https://mooncakes.io/docs/#/peter-jerry-ye/wasi-imports/>
- async (for `@promise` and `@loop`): <https://mooncakes.io/docs/#/peter-jerry-ye/async/>
- io (for `@channel`, `@io` and `@http`): <https://mooncakes.io/docs/#/peter-jerry-ye/io/>
```

```{note}
The template provides build scripts with `just` (<https://just.systems/>) in `justfile`, and http testing with `hurl` (<https://hurl.dev/>) in `test.hurl`. Both are excellent modern tools.
```

## Make the Server echos the body

In this section, we try to analyze the incoming request and make an eching
server.

### Task

Rewrite the content of the `top` in `src/stub.mbt` with:

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  let body = request.consume().unwrap()
  request.drop()
  let content = @http.text(body).0.await!!()
  let response = @types.OutgoingResponse::outgoing_response(@http.headers({}))
  response.set_status_code(200).unwrap()
  let outgoing_body = response.body().unwrap()
  response_out.set(Ok(response))
  let outgoing_stream = outgoing_body.write().unwrap()
  @io.println("\{content}", stream=outgoing_stream).await!!()
  outgoing_stream.drop()
  outgoing_body.finish(None).unwrap_or_error!()
}
```

Rerun all the build commands, and:

```bash
curl --data 'Hello World' localhost:8080
```

And we should get the content back.

### Explanation

We first use `request.consume().unwrap()` to get the `IncomingBody`, and we drop
the request since it's no longer needed. We then use the helper function
`@http.text` which will consume the whole body asynchronously and handle the
encoding conversion for us. The returned value will be a pair of `@promise.T`:
the text content, and the trailing headers.

### Exercise

Try to build a server that returns `Hello xxx` where `xxx` is retrieved from the
body.

## Send Request to <https://example.com>

In this section, we try to send a request and retrieve the content in the body.

### Task

Rewrite the content of the `top` in `src/stub.mbt` with:

```moonbit
async fn top(
  _request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  // Send request
  let request = @http.request!("example.com", path="/", scheme=Https)
  let response = @http.fetch(request).await!!()
  let body = response.consume().unwrap()
  let content = @http.text(body).0.await!!()
  response.drop()
  // Send response
  let response = @types.OutgoingResponse::outgoing_response(@http.headers({}))
  response.set_status_code(200).unwrap()
  let outgoing_body = response.body().unwrap()
  response_out.set(Ok(response))
  let outgoing_stream = outgoing_body.write().unwrap()
  @io.println("\{content}", stream=outgoing_stream).await!!()
  outgoing_stream.drop()
  outgoing_body.finish(None).unwrap_or_error!()
}
```

Rerun all the build commands, and:

```bash
curl localhost:8080
```

And we should get the content from the <https://example.com>.

### Explanation

Here we use `@http.request` to construct a request, sending to `example.com`
with the scheme of `Https`. The construction may throw error as we may pass in
invalid path and query or invalid scheme, etc. We then use `@http.fetch` to get
the response, and consume the body as what we've done for the incoming request.

### Exercise

Make a simple proxy server for a website that you chose.

## Send Request to OpenRouter

In this section, we try to send a request based on the API of OpenAI and handle
the response by interacting with JSON in MoonBit. The user message will be given
in the request body.

### Task

Rewrite the beginning of the `top` in `src/stub.mbt` into:

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  let token = Map::from_array(@environment.get_environment()).get(
    "OPENAI_API_KEY",
  )
  guard token is Some(token) else { fail!("OPENAI_API_KEY is not set") }
  // Get request
  let body = request.consume().unwrap()
  let content = @http.text(body).0.await!!()
  request.drop()
  // Prepare payload
  let payload : Json = {
    "model": "deepseek/deepseek-chat",
    "messages": [
      { "role": "system", "content": "You are a helpful assistant." },
      { "role": "user", "content": String(content) },
    ],
  }
  // Send request
  let request = @http.request!(
    "openrouter.ai",
    path="/api/v1/chat/completions",
    scheme=Https,
    method_=Post,
    headers=@http.headers({
      "Content-Type": [b"application/json".to_fixedarray()],
      "Authorization": [
        @encoding.encode(UTF8, "Bearer \{token}").to_fixedarray(),
      ],
    }),
  )
  let body = request.body().unwrap()
  let output_stream = body.write().unwrap()
  @io.println(payload.stringify(), stream=output_stream).await!!()
  output_stream.drop()
  body.finish(None).unwrap_or_error!()
  let response = @http.fetch(request).await!!()
  let body = response.consume().unwrap()
  let content = @http.json(body).0.await!!()
  response.drop()
  guard content
    is {
      "choices": [{ "message": { "content": String(content), .. }, .. }],
      ..
    } else {
    fail!("Unexpected response \{content}")
  }
  // Send response
  ...
}
```

Adjust `src/moon.pkg.json` to add import:

```json
{
  "link": {
    // ...
  },
  "import": [
    // ...
    "peter-jerry-ye/wasi-imports/interface/wasi/io/streams",
    "peter-jerry-ye/wasi-imports/interface/wasi/cli/environment",
    "moonbitlang/x/encoding"
  ]
}
```

And rewrite `wit/imports.wit` with

```wit
package moonbit-community:server;

world proxy {
  include wasi:http/proxy@0.2.0;
  // Get the OPENAI_API_KEY from environment
  import wasi:cli/environment@0.2.0;
}
```

Rebuild the project and run with:

```bash
OPENAI_API_KEY=your_key wasmtime serve -S cli=y --env OPENAI_API_KEY target/server.wasm
```

### Explanation

First, we try to get the `OPENAI_API_KEY` from the environment with
`@environment.get_environment()`. This requires using the `wasi:cli/environment`
interface, thus we need to adjust the declaration of our wit file, add
`-S cli=y` to allow the Wasm use this interface, add `--env OPENAI_API_KEY` to
allow the Wasm use this environment variable from the environment. You can also
set it directly with `--env OPENAI_API_KEY=your_key`.

Then, we get the content, prepare the payload following the API spec, and create
the request using `@http.request`. We specify that we want to `POST` the
request, and fill in the requested header, especially the `Authroization`. Then
we write it to the body of the request, which is retrieved by
`request.body().unwrap()`.

After sending the request, we use `@http.json` to get the response as `Json`,
and we use pattern matching to retrieve the answer. Since we are not using
streaming mode, it is easy and straight forward. We than print the result to the
response as what we've done before.

### Exercise

- Try to adjust the system prompt to give the AI some contexts and roles.
- Try to implement a streaming mode so that the user don't have to wait for the
  full content being ready. You may find `@io.read_line` and `@promise.spawn`
  helpful.

## Bonus: Publish on Spin

In this section, we publish the application that we've developped to Fermyon Cloud (<https://cloud.fermyon.com/>).

### Task

First, add the WIT dependency:

1. Install wit-deps (<https://github.com/bytecodealliance/wit-deps>). You can download the releases or run `cargo install wit-deps-cli` if you have the Rust toolchain.
2. Add dependency to `wit/deps.toml`:
    ```toml
    spin = "https://github.com/fermyon/spin/archive/refs/tags/v3.1.1.tar.gz"
    ```
3. Run `wit-dpes update`
4. Add dependency to `wit/imports.wit`:
    ```wit
    package moonbit-community:server;

    world proxy {
      include wasi:http/proxy@0.2.0;
      // Get the OPENAI_API_KEY from spin variables
      include fermyon:spin/platform@2.0.0;
    }
    ```

Then we add MoonBit dependency:

1. Add to `moon.mod.json`: `moon add peter-jerry-ye/spin-imports`
2. Add to `src/moon.pkg.json`:
    ```json
    {
      "link": {
        // ...
      },
      "import": [
        // ...
        "peter-jerry-ye/spin-imports/interface/fermyon/spin/variables"
      ]
    }
    ```

And we update the code in `top` in `src/stub.mbt` with:

```moonbit
async fn top(
  request : @types.IncomingRequest,
  response_out : @types.ResponseOutparam
) -> Unit! {
  let token = @variables.get("openai_api_key").unwrap_or_error!()
  // Get request
  ...
}
```

And finally, we define the Spin manifest in `spin.toml`:

```toml
spin_manifest_version = 2

[application]
name = "moonbit-server"
version = "1.0.0"
description = "A simple application."

[variables]
api_key = { required = true, secret = true }

# The application's sole trigger.
[[trigger.http]]
route = "/"
component = "server"

[component.server]
description = "A simple server."
# The Wasm module to run for the component
source = "target/server.wasm"
# Environment variable
allowed_outbound_hosts = ["https://openrouter.ai"]

# How to build the Wasm module from source
[component.server.build]
command = """
  moon build --target wasm
  wasm-tools component embed --encoding utf16 wit target/wasm/release/build/http-server.wasm -o target/server.core.wasm
  wasm-tools component new target/server.core.wasm -o target/server.wasm
"""

[component.server.variables]
openai_api_key="{{ api_key }}"
```

Build the project with: `WIT_REQUIRE_F32_F64=0 spin build`

Run the project with: `SPIN_VARIABLE_API_KEY="your_key" spin up`

After you have registered on the Fermyon Cloud (<https://cloud.fermyon.com/?signup>), run `spin login` to login. Then you can publish the project with: `spin deploy --variable api_key=your_key`.

### Explanation

We switch to use Spin's dynamic variable feature to get the secret key.

To do so, we need to add the Spin's interface in our WIT definition. We update the dependency and the definition. Then we use `peter-jerry-ye/spin-imports`, a module that contains the generated code for using the Spin APIs, to get the api key from the variable. Finally, we define a manifest to declare what application it is, what variable it contains, what trigger it has, what component it has and how to build it, etc., and the variable for it. And we publish it at last.

```{tip}
dotenvx (<https://dotenvx.com/>) is a handy tool for managing environment variables.
```