---
title: Creating NPX compatible cli tools with Bun 
published: true
---

# Creating NPX compatible cli tools with Bun

> This blog post is an exercise in exploration. Read to the end for why I wouldn't necessarily suggest doing this.

Users want to run scripts using tools already available in their project, usually this means installing a package and running its executable via `npx`, `npm run`, `pnpm dlx`, `bunx` or similar.

Script authors (like myself) want to create scripts using the best tools at their disposal. These past few years I've been using [Bun](https://bun.sh/) for creating most of my scripts because as an all-in-one toolchain+sdk it makes it ridiculously easy to build complex scripts with minimal time spent on project configuration, and many of its APIs like `file`, `glob` and `spawn` are ideal for building the kinds of tools I tend to work on.

I also like to author scripts that are fast and lightweight, which Bun's focus on performance and builtins helps make happen. I can often author complex scripts with zero or close-to-zero dependencies.

One challenge I've encountered has been how to create tools that work for users that don't use Bun or perhaps have an older version.

While our script can use a shebang (`#!/usr/bin/env bun`) to request running our script with `bun`, this will only work if the user has already adopted bun into their tooling stack, and has a recent enough version of bun on their machine to support the APIs we use.

Eventually, bun intends to enable building a project with `node` as a target in a way which includes any bun-specific APIs in the output. Whenever this lands it will immediately be incredibly useful, as most commonly the people trying to run scripts authored with bun APIs without bun are running node.

...but what if it didn't matter at all what tools and versions were locally available to the dev?

Below I'll outline a series of concepts that together allow us to do this.

## Compiling our Script

Often we don't think to bundle the node scripts we write. Or if we do its to support things like typescript, or esm/cjs interop. With bun, typescript and esm/cjs interop "just work". Thankfully, these ideas are now entering node as well so this particular advantage bun holds will fade away.

Sometimes, especially with larger node apps, we bundle just to reduce start-up time.
This is something bun makes easy, and it will even make optimizations like minification and inlining happen for you while it does.

For small CLI scripts, especially those meant for remote use by tools like `npx`, I think bundling is ideal as it lets us reduce the boot cost of our tool and potentially ship an even smaller package.

But bun optionally takes bundling two huge steps further:

- [`--bytecode`](https://bun.sh/docs/bundler/executables#bytecode-compilation) improves startup time of a bundle by pre-parsing the JS and storing that in the JS instead.
- [`--compile`](https://bun.sh/docs/bundler/executables) produces a single-file executable binary containing all necessary code and the bun runtime.

By default, bun compiles for the machine you are on, but we can adjust the target to compile for [any architecture that bun supports](https://bun.sh/docs/bundler/executables#supported-targets).

Assuming the entrypoint for our script is `./src/index.ts`, then we can build a simple script to generate compiled versions for each supported platform.

```ts
const KnownArchitectures = ['arm64', 'x64'];
const KnownOperatingSystems = ['linux', 'windows', 'darwin'];
const InvalidCombinations = ['windows-arm64'];

outer: for (const arch of KnownArchitectures) {
  for (const platform of KnownOperatingSystems) {
    if (InvalidCombinations.includes(`${platform}-${arch}`)) continue;

    // compile the given architecture
    const target = `bun-${platform}-${arch}-modern`;
    const args = [
      'bun',
      'build',
      './src/index.ts',
      `--outfile=dist/${target}-npx-bun-demo`,
      '--compile',
      '--minify',
      '--bytecode',
      '--env=disable',
      '--sourcemap=none',
      `--target=${target}`,
    ];

    const proc = Bun.spawn(args, {
      env: process.env,
      cwd: process.cwd(),
      stderr: 'inherit',
      stdout: 'inherit',
      stdin: 'inherit',
    });
    const result = await proc.exited;
    if (result !== 0) {
      break outer;
    }
  }
}
```

## setting up our package

### 1. move all dependencies to devDependencies

Since the code we need from our dependencies is all bundled into the executable, we don't want any tools to install them when trying to run our script.

So anything listed in `dependencies` should be moved to `devDependencies` in the `package.json`.

If you were using bun to build your API, this feature simplifies your deployment immensely! Zero-install / extremely lightweight image deploys!

### 2. Ensure we compile whenever we package up our project

In our `package.json` file, we setup the `prepack` script to ensure we build the compiled versions.

```json
  "scripts": {
    "build": "bun run build.ts",
    "prepack": "bun run build"
  },
```

### 3. setup a `bin` script for the package

Since we want our script to be usable from tools like `npx`, we need to add a `bin` entry to our `package.json` file.

NPM doesn't provide a utility for switching which asset to use for a bin script based on the consumer's architecture (though you could use node-gyp to achieve this). Below, I'll detail how to setup a script that juggles this for you, we'll name that file `arch-switch.mjs`.

```json
  "bin": {
    "npx-bun-demo": "./arch-switch.mjs"
  },
```

### 4. setup what files to publish with our package

Our script above outputs the compiled versions into `dist`, and requires the bin script `arch-switch.mjs`, nothing else is needed to use the tool, so these will be the only entries in our `package.json` file.

```json
  "files": [
    "dist",
    "arch-switch.mjs"
  ],
```

## Creating the bin script

Our `arch-switch` script will select which pre-compiled binary to execute to perform the `npx-bun-demo` command by detecting the platform and architecture of the machine in use.

For this, we want to make sure to use only features that are available to both node and bun, which generally means "use only node-available APIs" as bun largely has implemented support for anything node offers (being able to use any npm package and node built-in with bun is another of its superpowers).

```ts
import { spawn } from 'child_process';
import { arch as osArch, platform as osPlatform } from 'os';

const KnownArchitectures = ['arm64', 'x64'];
const KnownOperatingSystems = ['linux', 'windows', 'darwin'];
const InvalidCombinations = ['windows-arm64'];

const arch = osArch();
const _platform = osPlatform();
const platform = _platform === 'win32' ? 'windows' : _platform;

function ExitError(str) {
  console.log(`\n\n`);
  console.log(
    '\x1b[1m\x1b[4m\x1b[31m\x1b[40m',
    `\t\t${str}\t\t`
  );
  console.log(`\n\n`);
  process.exit(1);
}

if (!KnownArchitectures.includes(arch)) {
  ExitError(`Unsupported architecture '${arch}'`);
}

if (!KnownOperatingSystems.includes(platform)) {
  ExitError(`Unsupported platform '${platform}'`);
}

const executableName = `${platform}-${arch}`;
if (InvalidCombinations.includes(executableName)) {
  ExitError(`Unsupported architecture '${arch}' for current platform '${platform}'`);
}

spawn(
  `./dist/bun-${executableName}-modern-npx-bun-demo${platform === 'windows' ? '.exe' : ''}`,
  process.argv.slice(2),
  {
    cwd: process.cwd(),
    env: process.env,
    stdio: 'inherit',
  }
);
```

### Conditionally Running Bun via shebang

Our bin script above is missing just one piece: the shebang.

Usually, a bin script will specify `#!/usr/bin/env node` at the top of the file to ensure tools execute it with the node runtime.

We can alternatively specify `#!/usr/bin/env bun` instead to make use of the much faster start-up time and runtime offered by `bun`, but remember, we aren't sure our user has bun (or node for that matter) and we want to work either way.

Enter a conditional shebang, a bit of sorcery I leave to google (or lets be honest probably ChatGPT) to explain:

```sh
#!/bin/sh -
':'; /*-
test1=$(bun --version 2>&1) && exec bun "$0" "$@"
test2=$(node --version 2>&1) && exec node "$0" "$@"
exec printf '%s\n' "$test1" "$test2" 1>&2
*/
```

We can add a console log to the top of our file after this to help see in practice that this works:

```ts
#!/bin/sh -
':'; /*-
test1=$(bun --version 2>&1) && exec bun "$0" "$@"
test2=$(node --version 2>&1) && exec node "$0" "$@"
exec printf '%s\n' "$test1" "$test2" 1>&2
*/

console.log(
  `\n\nRunning '${['npx-bun-demo (arch-switch) |>', ...process.argv.slice(2)].join(' ')}' with: ${process.isBun ? 'bun' : 'node'}`
);
```

The general gist is that this uses various comment syntaxes and strings in a manner that a shell-script interprets it as a command and the JS runtime ignores it. If we find a bun command in our current path, we use it, else we try to use node.

And there you have it!

The source-code for this demo is [available on github](https://github.com/runspired/npx-bun-demo).

The example app details can be tried via `npx npx-bun-demo@1.0.0` or a similar tool of your choosing.

### What I don't like

The downside to the approach detailed here is the package size that results from producing the matrix of compiled binaries.

If all we cared about was a single platform and architecture, our package would be `56.8 MB`. This is viewable via `npm view npx-bun-demo@1.0.1`

This is larger than I'd like, but considering that is the runtime, dependencies and source-code I find it acceptable (most small scripts are much larger than this once dependency sizes are taken into account, before even considering the size of node itself).

With the full matrix of current targets, our package is `422.4 MB`. This is viewable via `npm view npx-bun-demo@1.0.0` Oof 🙈

There are times where this is probably acceptable (at least, in the short-haul until bun gives us the ability to consume its APIs outside of a bun environment and/or npm gives us the ability to pre-build separate tarballs for distinct architectures). 

If you want, you could risk dangerously relying on the postinstall hook (seriously folks, run your package installation with scripts disabled please) to detect and download the desired executable. The easy path for that would be to check the compiled versions into github and have the switch script fetch the file if not already present.

I really don't recommend this except *maybe* for scripts that are only ever used via a mechanism like `npx`.

This approach is demoable via `npx npx-bun-demo@1.0.7` for everything but windows (since github won't allow files > 100MB).
