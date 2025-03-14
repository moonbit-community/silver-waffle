# Preparation

## Task

Install the required toolchains:

- MoonBit: <https://www.moonbitlang.com/download>
- Wasm-tools:
  <https://github.com/bytecodealliance/wasm-tools?tab=readme-ov-file#installation>
- Wasmtime: <https://docs.wasmtime.dev/cli-install.html>
- Spin: <https://developer.fermyon.com/spin/v3/install>
- smee: <https://smee.io/> (rely on npm:
  <https://docs.npmjs.com/downloading-and-installing-node-js-and-npm>)

## Explanation

Here we install the necessary toolchains.

- MoonBit: this is the programming language we are using, as it generates
  high-quality Wasm, and has a lower learning curve.
- Wasm-tools: this provides manipulations on Wasm, including transforming
  between text format and binary format, converting a core Wasm to a
  componentized Wasm following the Component Model, etc.
- Wasmtime: the runtime that supports a lot of post-MVP proposals. It also
  supports the Component Model, allowing people to develop Wasm as command line
  tools or http servers as long as it follows the standard interface.
- Spin: the runtime based on the Wasmtime. It provides some extra interfaces,
  such as the key-value storage. Fermyon, the company behind the Spin, also host
  a cloud for serving the Wasm.
- smee: a Webhook payload delivery service that ease the development of webhook.
  For example, it can proxy the payloads to locally running application, and it
  can replay the previous payloads.