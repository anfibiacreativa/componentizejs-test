This is a test implementation of [bytecodealliance/componentize-js](https://github.com/bytecodealliance/componentize-js)

## Pre-requisites
- [Node.js](https://nodejs.org/en/) and [npm](https://www.npmjs.com/) are required to build and run this project.

- [Cargo](https://doc.rust-lang.org/cargo/getting-started/installation.html) must be installed, following the instructions in the link, for your specific platform.

_Please notice that I have implemented this demo with node.js 16.16.0 and npm 8.11.0_

## Creating the repo from scratch

### Create the project folder
```bash
mkdir componentize-demo && cd componentize-demo
```

Now we will initialize this folder as an npm package, running

```bash
npm init -y
```
(we pass the -y flag to skip the interactive questions).

We will then open the generated `package.json` file and add the following line:

```json
"type": "module"
```

This is required to enable ES modules in Node.js.

Now we will install the `componentize-js` package:

```bash
npm install --save-dev @bytecodealliance/componentize-js
```

and we're going to create a new file called `componentize.mjs` that will be used to generate our Wasm module. Copy and paste the following code to it.

```js
import { componentize } from '@bytecodealliance/componentize-js';
import { readFile, writeFile } from 'node:fs/promises';

const jsSource = await readFile('hello.js', 'utf8');
const witSource = await readFile('hello.wit', 'utf8');

const { component } = await componentize(jsSource, witSource);

await writeFile('hello.component.wasm', component);
```

### Create the WIT and JS files

Now we're going to create a WIT file called `hello.wit`. WIT stands for WebAssembly Interface Types and it's a file that describes the interface of a WebAssembly module. Copy and paste the following code to it.

```wasm
default world hello {
  export hello: func(name: string) -> string
}
```

We are also going to create a JavaScript file called `hello.js` that will be used to implement the WIT file. This file exports a single function. Copy and paste the function to the file.

```js
export function hello (name) {
  return `Hello ${name}`;
}
```

### Generate the Wasm module

Now we're ready to generate our Wasm module:
  
  ```bash
  node componentize.mjs
  ```

  You should see the generated binary file `hello.component.wasm` in your project folder.

### Create the Rust project

Now we're going to create a Rust project that will be used to load and run our Wasm module. 

For that, we're going to run the following command:

```bash
cargo init --bin
```

at the root of our project, just like we did with npm for Node.js.

This will create a `Cargo.toml` file and a `src` folder with a `main.rs` file inside.

Now we're going to add the following lines to the `Cargo.toml` file:

```toml
[package]
name = "wasmtime-test"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[workspace]

[dependencies]
anyhow = "1.0.65"
async-std = { version = "1.12.0", features = ["attributes"] }
cap-std = "1.0.12"
wasmtime = { git = "https://github.com/bytecodealliance/wasmtime", rev = "299131ae2d6655c49138bfab2c4469650763ef3b", features = ["component-model"] }
wasi-common =  { git = "https://github.com/bytecodealliance/preview2-prototyping", rev = "dd34a00d4386000cd00071cff18b9e3a12788075" }
wasi-cap-std-sync = { git = "https://github.com/bytecodealliance/preview2-prototyping", rev = "dd34a00d4386000cd00071cff18b9e3a12788075" }
wasmtime-wasi-sockets =  { git = "https://github.com/bytecodealliance/preview2-prototyping", rev = "dd34a00d4386000cd00071cff18b9e3a12788075" }
wasmtime-wasi-sockets-sync = { git = "https://github.com/bytecodealliance/preview2-prototyping", rev = "dd34a00d4386000cd00071cff18b9e3a12788075" }
```

This will add the dependencies we need to run our Wasm module.

We will also go to `src/main.rs` and add the following code to it:

```rust
use anyhow::Result;
use wasi_cap_std_sync::WasiCtxBuilder;
use wasi_common::{wasi, Table, WasiCtx, WasiView};
use wasmtime::{
    component::{Component, Linker},
    Config, Engine, Store, WasmBacktraceDetails,
};
use wasmtime_wasi_sockets::{WasiSocketsCtx, WasiSocketsView};
use wasmtime_wasi_sockets_sync::WasiSocketsCtxBuilder;

wasmtime::component::bindgen!({
    world: "hello",
    path: "hello.wit",
    async: true
});

#[async_std::main]
async fn main() -> Result<()> {
    let builder = WasiCtxBuilder::new().inherit_stdio();
    let mut table = Table::new();
    let wasi = builder.build(&mut table)?;

    let mut config = Config::new();
    config.cache_config_load_default().unwrap();
    config.wasm_backtrace_details(WasmBacktraceDetails::Enable);
    config.wasm_component_model(true);
    config.async_support(true);

    let engine = Engine::new(&config)?;
    let mut linker = Linker::new(&engine);

    let component = Component::from_file(&engine, "hello.component.wasm").unwrap();


    struct CommandCtx {
        table: Table,
        wasi: WasiCtx,
        sockets: WasiSocketsCtx,
    }
    impl WasiView for CommandCtx {
        fn table(&self) -> &Table {
            &self.table
        }
        fn table_mut(&mut self) -> &mut Table {
            &mut self.table
        }
        fn ctx(&self) -> &WasiCtx {
            &self.wasi
        }
        fn ctx_mut(&mut self) -> &mut WasiCtx {
            &mut self.wasi
        }
    }
    let sockets = WasiSocketsCtxBuilder::new()
        .inherit_network(cap_std::ambient_authority())
        .build();
    impl WasiSocketsView for CommandCtx {
        fn table(&self) -> &Table {
            &self.table
        }
        fn table_mut(&mut self) -> &mut Table {
            &mut self.table
        }
        fn ctx(&self) -> &WasiSocketsCtx {
            &self.sockets
        }
        fn ctx_mut(&mut self) -> &mut WasiSocketsCtx {
            &mut self.sockets
        }
    }

    wasi::command::add_to_linker(&mut linker)?;
    let mut store = Store::new(
        &engine,
        CommandCtx {
            table,
            wasi,
            sockets,
        },
    );

    let (instance, _instance) =
        Hello::instantiate_async(&mut store, &component, &linker).await?;

    let res = instance.call_hello(&mut store, "ComponentizeJS").await?;
    println!("{}", res);
    Ok(())
}
```

This code will load our Wasm module and run it.

### Building and running the Rust project

The first thing we need to do, is to build our Rust project. For that, we're going to run the following command:

```bash
cargo build --release
```
Now we're ready to run it.

```bash
cargo run ./target/release/componentizejs
```

After running this command, you should see the following output:

```bash
Hello ComponentizeJS
```

## Transpiling and running it in the Node.js

Now we're going to transpile our Wasm module to JavaScript and run it in Node.js. First of all, we need to install additional dependencies:

```bash
npm install --save @bytecodealliance/jco @bytecodealliance/preview2-shim
```

We will also update our `package.json` file to add the following lines:

```json
"scripts": {
    "transpile": "jco transpile hello.component.wasm -o hello --wasi-shim",
    ...
}
```
Note I did not install the jco CLI globally, but locally. What this line does is execute jsco to transpile the .wasm component using Typescript interfaces, and generate the moules in a new folder called `hello`. For that, it uses the flag `--wasi-shim` to resolve the imports.

Now we can go back to the terminal and run the following command:

```bash
npm run transpile
```

We should see the following output:
  
```bash
Transpiled JS Component Files:

 - hello/hello.component.core.wasm     6.44 MiB
 - hello/hello.component.core2.wasm    30.1 KiB
 - hello/hello.component.d.ts          0.58 KiB
 - hello/hello.component.js            44.2 KiB
 - hello/imports/environment.d.ts      0.09 KiB
 - hello/imports/exit.d.ts             0.16 KiB
 - hello/imports/filesystem.d.ts       3.85 KiB
 - hello/imports/monotonic-clock.d.ts  0.14 KiB
 - hello/imports/preopens.d.ts         0.47 KiB
 - hello/imports/random.d.ts            0.1 KiB
 - hello/imports/streams.d.ts          0.58 KiB
 - hello/imports/wall-clock.d.ts
```
Those have been saved to a new folder in your project called `hello`. 

Now you can use the REPL to run the following command:

```bash
node -e "import('./hello/hello.component.js').then(m => console.log(m.hello('ComponentizeJS')))"
```

You should see the following output:

```bash
Hello ComponentizeJS
```


