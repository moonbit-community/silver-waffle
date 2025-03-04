# AI Agent with MoonBit

The aim of this workshop is to develop an AI Agent that will generate a code review for each pull request on GitHub by leveraging the AI.

This workshop contains the following parts:

- Preparation
- Create a LLM proxy
- Create a GitHub App
- Create a Code Review Bot utilizing the previous knowledge

## Preparation

### Task

Install the required toolchains:

- MoonBit: <https://www.moonbitlang.com/download>
- Wasm-tools: <https://github.com/bytecodealliance/wasm-tools?tab=readme-ov-file#installation>
- Wasmtime: <https://docs.wasmtime.dev/cli-install.html>
- Spin: <https://developer.fermyon.com/spin/v3/install>
- smee: <https://smee.io/> (rely on npm: <https://docs.npmjs.com/downloading-and-installing-node-js-and-npm>)

### Explanation

Here we install the necessary toolchains.

- MoonBit: this is the programming language we are using, as it generates high-quality Wasm, and has a lower learning curve.
- Wasm-tools: this provides manipulations on Wasm, including transforming between text format and binary format, converting a core Wasm to a componentized Wasm following the Component Model, etc.
- Wasmtime: the runtime that supports a lot of post-MVP proposals. It also supports the Component Model, allowing people to develop Wasm as command line tools or http servers as long as it follows the standard interface.
- Spin: the runtime based on the Wasmtime. It provides some extra interfaces, such as the key-value storage. Fermyon, the company behind the Spin, also host a cloud for serving the Wasm.
- smee: a Webhook payload delivery service that ease the development of webhook. For example, it can proxy the payloads to locally running application, and it can replay the previous payloads.

## Create a LLM proxy

This part is divided as follows

1. Create a Server that says Hello World
2. Make the Server echos the body
3. Send request to get <https://www.example.com>
4. Send request to OpenRouter to access the LLM service

### Create a Server

#### Task

- Clone <https://github.com/peter-jerry-ye/http-template.git>
- Run the following script in the folder:
  ```bash
  moon build --target wasm --debug
  wasm-tools component embed --encoding utf16 wit target/wasm/debug/build/http-server.wasm -o target/server.core.wasm
  wasm-tools component new target/server.core.wasm -o target/server.wasm
  # In the first terminal
  wasmtime serve target/server.wasm
  # In the second terminal
  curl localhost:8080
  ```

#### Explanation

We've cloned a template for developing HTTP application in MoonBit. Then we ran a few commands to create a componentized Wasm.

For the commands:

- `moon build --target wasm --debug`:  
  this command builds the project, generates the result in Wasm 1 backend (by default it would be Wasm GC, which is not supported yet by the Component Model proposal), in the debug mode (which you can remove when you are ready to publish).
- `wasm-tools component embed --encoding utf16 wit target/wasm/debug/build/http-server.wasm -o target/server.core.wasm`:  
  This embeds the interface information, under the folder `wit`, into the Wasm that we've just produced. We also specify that the encoding of the language is UTF16.
- `wasm-tools component new target/server.core.wasm -o target/server.wasm`:  
  This transforms the standard Wasm with the meta information into the so called componentized Wasm, i.e. the Wasm following the Component Model's standard.
- `wasmtime serve target/server.wasm`:  
  We use the wasmtime to serve this Wasm at port 8080. It will respond `Hello World!` in the body.

For the code:

The keys are inside the `src/stub.mt`, where we have a public function called `handle`. This is the required function by the standard. It takes an incoming request, and an `ResponseOutParam`, on which we should set the actual response.

In the function body, `@promise.spawn` spawns a coroutine defined with an async function, which will call the function `top` and handle the error if anything occurs. Then the event loop is initialized. We can ignore this part.

The `top` function defines the main logic. First, we create an outgoing response with empty headers with the helper function of `@http.headers`. We then set the status as 200, and creates a response body. The body returned is a `Result` because the designer of the API wants to make sure that it is retrieved only once. Since we are only retrieving once, we simply unwrap it. We then set our response to the `response_out`. Doing so will start the sending of the response immediately in the chunked manner. We then retrieve the output stream by `outgoing_body.write().unwrap()`, and use `@io.println` to write a `String` into it. We wait for its termination, and drop the output stream before finishing the outgoing body.

All the imported packages can be checked in `src/moon.pkg.json`. All the dependencies can be checked in `moon.mod.json`.

### Make the Server echos the body

## Create a GitHub App

This part is divided as follows:

- Register on GitHub and create an App
- Create a server that handles the webhook
- Generate an access token based on the request

## Create a Code Review Bot

This part is divided as follows:

- Get the pull request's information
- Get the pull request's diff
- Get the previous list of comments
- Get the code comment from LLM
- Get the code comment from LLM