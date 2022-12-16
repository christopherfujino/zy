# 🐚 zy

A fork of google/zx with less dependencies.

```js
#!/usr/bin/env zy

await $`cat package.json | grep name`

let branch = await $`git branch --show-current`
await $`dep deploy --branch=${branch}`

await Promise.all([
  $`sleep 1; echo 1`,
  $`sleep 2; echo 2`,
  $`sleep 3; echo 3`,
])

let name = 'foo bar'
await $`mkdir /tmp/${name}`
```

Bash is great, but when it comes to writing more complex scripts,
many people prefer a more convenient programming language.
JavaScript is a perfect choice, but the Node.js standard library
requires additional hassle before using. The `zy` package provides
useful wrappers around `child_process`, escapes arguments and
gives sensible defaults.

## Install

```bash
git clone https://github.com/christopherfujino/zy
cd zy
npm install
npm link # this will create a symlink to your global node_modules
echo 'await $`uname -a` | zy'
```

**Requirement**: Node version >= 16.0.0

## Goods

[$](#command-) · [cd()](#cd) · [fetch()](#fetch) · [question()](#question) · [sleep()](#sleep) · [echo()](#echo) · [stdin()](#stdin) · [within()](#within) ·
[fs](#fs-package) · [os](#os-package) · [path](#path-package) · [glob](#globby-package) · [yaml](#yaml-package) · [minimist](#minimist-package) · [which](#which-package) ·
[__filename](#__filename--__dirname) · [__dirname](#__filename--__dirname) · [require()](#require)

## Documentation

Write your scripts in a file with an `.mjs` extension in order to
use `await` at the top level. If you prefer the `.js` extension,
wrap your scripts in something like `void async function () {...}()`.

Add the following shebang to the beginning of your `zy` scripts:
```bash
#!/usr/bin/env zy
```

Now you will be able to run your script like so:
```bash
chmod +x ./script.mjs
./script.mjs
```

Or via the `zy` executable:

```bash
zy ./script.mjs
```

All functions (`$`, `cd`, `fetch`, etc) are available straight away
without any imports.

Or import globals explicitly (for better autocomplete in VS Code).

```js
import 'zy/globals'
```

### ``$`command` ``

Executes a given command using the `spawn` func
and returns [`ProcessPromise`](#processpromise).

Everything passed through `${...}` will be automatically escaped and quoted.

```js
let name = 'foo & bar'
await $`mkdir ${name}`
```

**There is no need to add extra quotes.** Read more about it in
[quotes](docs/quotes.md).

You can pass an array of arguments if needed:

```js
let flags = [
  '--oneline',
  '--decorate',
  '--color',
]
await $`git log ${flags}`
```

If the executed program returns a non-zero exit code,
[`ProcessOutput`](#processoutput) will be thrown.

```js
try {
  await $`exit 1`
} catch (p) {
  console.log(`Exit code: ${p.exitCode}`)
  console.log(`Error: ${p.stderr}`)
}
```

### `ProcessPromise`

```ts
class ProcessPromise extends Promise<ProcessOutput> {
  stdin: Writable
  stdout: Readable
  stderr: Readable
  exitCode: Promise<number>
  pipe(dest): ProcessPromise
  kill(): Promise<void>
  nothrow(): this
  quiet(): this
}
```

Read more about the [ProcessPromise](docs/process-promise.md).

### `ProcessOutput`

```ts
class ProcessOutput {
  readonly stdout: string
  readonly stderr: string
  readonly signal: string
  readonly exitCode: number
  toString(): string // Combined stdout & stderr.
}
```

The output of the process is captured as-is. Usually, programs print a new line `\n` at the end.
If `ProcessOutput` is used as an argument to some other `$` process,
**zy** will use stdout and trim the new line.

```js
let date = await $`date`
await $`echo Current date is ${date}.`
```

## Functions

### `cd()`

Changes the current working directory.

```js
cd('/tmp')
await $`pwd` // => /tmp
```

### `fetch()`

A wrapper around the [node-fetch](https://www.npmjs.com/package/node-fetch) package.

```js
let resp = await fetch('https://medv.io')
```

### `question()`

A wrapper around the [readline](https://nodejs.org/api/readline.html) package.

```js
let bear = await question('What kind of bear is best? ')
```

### `sleep()`

A wrapper around the `setTimeout` function.

```js
await sleep(1000)
```

### `echo()`

A `console.log()` alternative which can take [ProcessOutput](#processoutput).

```js
let branch = await $`git branch --show-current`

echo`Current branch is ${branch}.`
// or
echo('Current branch is', branch)
```

### `stdin()`

Returns the stdin as a string.

```js
let content = JSON.parse(await stdin())
```

### `within()`

Creates a new async context.

```js
await $`pwd` // => /home/path

within(async () => {
  cd('/tmp')

  setTimeout(async () => {
    await $`pwd` // => /tmp
  }, 1000)
})

await $`pwd` // => /home/path
```

```js
let version = await within(async () => {
  $.prefix += 'export NVM_DIR=$HOME/.nvm; source $NVM_DIR/nvm.sh; '
  await $`nvm use 16`
  return $`node -v`
})
```

## Packages

The following packages are available without importing inside scripts.

### `fs` package

The [fs-extra](https://www.npmjs.com/package/fs-extra) package.

```js
let {version} = await fs.readJson('./package.json')
```

### `os` package

The [os](https://nodejs.org/api/os.html) package.

```js
await $`cd ${os.homedir()} && mkdir example`
```

### `path` package

The [path](https://nodejs.org/api/path.html) package.

```js
await $`mkdir ${path.join(basedir, 'output')}`
```

### `globby` package

The [globby](https://github.com/sindresorhus/globby) package.

```js
let packages = await glob(['package.json', 'packages/*/package.json'])
```

### `yaml` package

The [yaml](https://www.npmjs.com/package/yaml) package.

```js
console.log(YAML.parse('foo: bar').foo)
```

### `minimist` package

The [minimist](https://www.npmjs.com/package/minimist) package available
as global const `argv`.

```js
if( argv.someFlag ){ echo('yes') }
```

### `which` package

The [which](https://github.com/npm/node-which) package.

```js
let node = await which('node')
```

## Configuration

### `$.shell`

Specifies what shell is used. Default is `which bash`.

```js
$.shell = '/usr/bin/bash'
```

Or use a CLI argument: `--shell=/bin/bash`

### `$.spawn`

Specifies a `spawn` api. Defaults to `require('child_process').spawn`.

### `$.prefix`

Specifies the command that will be prefixed to all commands run.

Default is `set -euo pipefail;`.

Or use a CLI argument: `--prefix='set -e;'`

### `$.quote`

Specifies a function for escaping special characters during
command substitution.

### `$.verbose`

Specifies verbosity. Default is `true`.

In verbose mode, `zy` prints all executed commands alongside with their
outputs.

Or use the CLI argument `--quiet` to set `$.verbose = false`.

### `$.env`

Specifies an environment variables map.

Defaults to `process.env`.

### `$.cwd`

Specifies a current working directory of all processes created with the `$`.

The [cd()](#cd) func changes only `process.cwd()` and if no `$.cwd` specified,
all `$` processes use `process.cwd()` by default (same as `spawn` behavior).

### `$.log`

Specifies a [logging function](src/core.ts).

```ts
import { LogEntry, log } from 'zy/core'

$.log = (entry: LogEntry) => {
  switch (entry.kind) {
    case 'cmd':
      // for example, apply custom data masker for cmd printing
      process.stderr.write(masker(entry.cmd))
      break
    default:
      log(entry)
  }
}
```

## Polyfills

### `__filename` & `__dirname`

In [ESM](https://nodejs.org/api/esm.html) modules, Node.js does not provide
`__filename` and `__dirname` globals. As such globals are really handy in scripts,
`zy` provides these for use in `.mjs` files (when using the `zy` executable).

### `require()`

In [ESM](https://nodejs.org/api/modules.html#modules_module_createrequire_filename)
modules, the `require()` function is not defined.
The `zy` provides `require()` function, so it can be used with imports in `.mjs`
files (when using `zy` executable).

```js
let {version} = require('./package.json')
```

## Experimental

The zy provides a few experimental functions. Please leave feedback about
those features in [the discussion](https://github.com/google/zy/discussions/299).
To enable new features via CLI pass `--experimental` flag.

### `retry()`

Retries a callback for a few times. Will return after the first
successful attempt, or will throw after specifies attempts count.

```js
import { retry, expBackoff } from 'zy/experimental'

let p = await retry(10, () => $`curl https://medv.io`)

// With a specified delay between attempts.
let p = await retry(20, '1s', () => $`curl https://medv.io`)

// With an exponential backoff.
let p = await retry(30, expBackoff(), () => $`curl https://medv.io`)
```

### `spinner()`

Starts a simple CLI spinner.

```js
import { spinner } from 'zy/experimental'

await spinner(() => $`long-running command`)

// With a message.
await spinner('working...', () => $`sleep 99`)
```

## FAQ

### Passing env variables

```js
process.env.FOO = 'bar'
await $`echo $FOO`
```

### Passing array of values

When passing an array of values as an argument to `$`, items of the array will be escaped
individually and concatenated via space.

Example:
```js
let files = [...]
await $`tar cz ${files}`
```

### Importing into other scripts

It is possible to make use of `$` and other functions via explicit imports:

```js
#!/usr/bin/env node
import {$} from 'zy'
await $`date`
```

### Scripts without extensions

If script does not have a file extension (like `.git/hooks/pre-commit`), zy
assumes that it is an [ESM](https://nodejs.org/api/modules.html#modules_module_createrequire_filename)
module.

### Markdown scripts

The `zy` can execute [scripts written as markdown](docs/markdown.md):

```bash
zy docs/markdown.md
```

### TypeScript scripts

```ts
import {$} from 'zy'
// Or
import 'zy/globals'

void async function () {
  await $`ls -la`
}()
```

Set [`"type": "module"`](https://nodejs.org/api/packages.html#packages_type)
in **package.json** and [`"module": "ESNext"`](https://www.typescriptlang.org/tsconfig/#module)
in **tsconfig.json**.

### Executing remote scripts

If the argument to the `zy` executable starts with `https://`, the file will be
downloaded and executed.

```bash
zy https://medv.io/game-of-life.js
```

### Executing scripts from stdin

The `zy` supports executing scripts from stdin.

```js
zy <<'EOF'
await $`pwd`
EOF
```

### Executing scripts via --eval

Evaluate the following argument as a script.

```bash
cat package.json | zy --eval 'let v = JSON.parse(await stdin()).version; echo(v)'
```

### Installing dependencies via --install

```js
// script.mjs:
import sh from 'tinysh'
sh.say('Hello, world!')
```

Add `--install` flag to the `zy` command to install missing dependencies 
automatically.

```bash
zy --install script.mjs
```

You can also specify needed version by adding comment with `@` after 
the import.

```js
import sh from 'tinysh' // @^1
```

### Attaching a profile

By default `child_process` does not include aliases and bash functions.
But you are still able to do it by hand. Just attach necessary directives 
to the `$.prefix`.

```js
$.prefix += 'export NVM_DIR=$HOME/.nvm; source $NVM_DIR/nvm.sh; '
await $`nvm -v`
```

### Using GitHub Actions

The default GitHub Action runner comes with `npx` installed.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Build
      env:
        FORCE_COLOR: 3
      run: |
        npx zy <<'EOF'
        await $`...`
        EOF
```

## License

[Apache-2.0](LICENSE)

Disclaimer: _This is not an officially supported Google product._
