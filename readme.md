# TypeScript ESM URL Search Query

Right now, TypeScript supports the ESM syntax:

```ts
import thing from './mod.js';
```

It doesn't, however support URL ESM:

```js
import thing from 'https://thing.com/mod.js';
```

Not even for `file:` URLs:

```js
import thing from 'file:///C:/thing/mod.js';
```

The only supported syntax is the one using file paths as per the above.

Node and browsers support a bit more, specifically, the URL search query part can be included, and that is true
even for the file paths:

```ts
import thingTest from './mod.js?test';
```

This will correctly resolve to `mod.js`. Why would this be useful? Well, you'll get the search query part of
the URL in your `import.meta` in `mod.js` so you can tailor your static exports to the caller based on the
search query part of the URL.

Is this a good practice? It is for my personal side project which I approach from the direction of hacking/
programming enthusiasm mindset, not from the AbstractFactoryFactory dogma-driven development practices of
corporate programming.

That's reason enough for me to be bummed out about the fact that TypeScript won't be able to track types
across the module boundary if the search part of the URL is used.

I'd like to help close this gap. Of course the best thing would be to support the URL ESM in TypeScript as
a whole. This is being tracked, but the scope that work is probably pretty ginormous, not a one person job.

https://github.com/microsoft/TypeScript/issues/41730

Also, this ticket suggests adding a new module resolution mode, which is not ideal, because I am using the
TypeScript language service in VS Code to enhance my type inferrence in JavaScript, so I don't actually
influence the TypeScript configuration used, and even if I could, I'd prefer this work out of the box.

So, maybe the right step here would be to settle for a hack-fix where only support for correctly resolving
paths with search query in them to paths would suffice.

The plan of attack is:

- [x] Get TypeScript debugging to work locally: https://blog.andrewbran.ch/debugging-the-type-script-codebase

`node --inspect-brk TypeScript/built/local/tsc.js -p .`

In VS Code: Cmd+Shift+P > Debug: Attach to Node Process

Compare with stock TypeScript: `npx typescript -p .` or `npm install --global typescript` followed by `tsc`

Check version: `tsc -v`

Run test: `tsc -p .` or `tsc index.ts` or `node TypeScript/built/local/tsc.js index.ts` to produce `index.js` (Git-ignored)

- [x] Create a TypeScript project with this module arrangement to feed to the instrumented compiler
- [x] Access a member of the imported object to cause an error if its type is not known - what we're fixing
- [x] Step through the various parts of the compiler to learn where the module resolution stuff happens
- [x] Try to identity the spot where to strip `\?.*$` from the import path if it matches anything

It is a good idea to open two integrated terminals:

- `node TypeScript/built/local/tsc.js index.ts` to test built changes
- `cd TypeScript` and then run `npm run build:compiler` after making changes

Use `(globalThis as any).console.log` where `console.log` won't work

The flow for `node TypeScript/built/local/tsc.js index.ts` is:

- TypeScript/src/tsc/tsc.ts:executeCommandLine
- TypeScript/src/executeCommandLine/executeCommandLine.ts:executeCommandLine
- TypeScript/src/executeCommandLine/executeCommandLine.ts:executeCommandLineWorker
- TypeScript/src/executeCommandLine/executeCommandLine.ts:performCompilation
- TypeScript/src/compiler/program.ts:loadWithLocalCache

The method is called for each module name to resolve. In here, we have a change
to look at what names end with a search query and strip it so that the follow-on
logic sees the module name without it.

Try both `node TypeScript/built/local/tsc.js index.ts` and with `-module esnext`
to test both resolvers.

- [x] Submit a patch to the TypeScript project and try to get this upstreamed and eventually used in VS Code

https://github.com/microsoft/TypeScript/pull/43082

Use the VS Code workspace configuration as is in this project and ensure to
accept the prompt to switch to the workspace TypeScript version. Alternatively,
Cmd+Shift+P and search for TypeScript: Switch TypeScript Version.

- [x] Figure out how to test `tsserver` which is the part actually used by VS Code

See updates to my fork which deal with logging.

Use Cmd+Shift+P and search for TypeScript: Restart TS Server and monitor the
Output pane TypeScript channel for TSServer messages.

Run `node TypeScript/built/local/tsserver.js` for CLI REPL

Commands VS Code sends (from the TSServer log, ServerMode: 1 [partial semantic]):

```json
{"seq":0,"type":"request","command":"configure","arguments":{"hostInfo":"vscode","preferences":{"providePrefixAndSuffixTextForRename":true,"allowRenameOfImportPath":true,"includePackageJsonAutoImports":"auto"},"watchOptions":{}}}

{"seq":1,"type":"request","command":"compilerOptionsForInferredProjects","arguments":{"options":{"module":"commonjs","target":"es2016","jsx":"preserve","strictFunctionTypes":true,"sourceMap":true,"allowJs":true,"allowSyntheticDefaultImports":true,"allowNonTsExtensions":true,"resolveJsonModule":true}}}

{"seq":2,"type":"request","command":"updateOpen","arguments":{"changedFiles":[],"closedFiles":[],"openFiles":[{"file":"ts-esm-search/index.ts","fileContent":…}],…}

{"seq":5,"type":"request","command":"geterr","arguments":{"delay":0,"files":["ts-esm-search/index.ts"]}}
{"seq":0,"type":"event","event":"semanticDiag","body":{"file":"ts-esm-search/index.ts","diagnostics":[{"start":{"line":2,"offset":18},"end":{"line":2,"offset":32},"text":"Cannot find module './mod2?test2' or its corresponding type declarations.","code":2307,"category":"error"}]}}

{"seq":7,"type":"request","command":"geterr","arguments":{"delay":0,"files":["ts-esm-search/index.ts"]}}
```

- `UpdateOpenRequest` + `UpdateOpenRequestArgs`
- TypeScript/src/server/session.ts:2552 (`[CommandNames.UpdateOpen]`)
- TypeScript/src/server/editorServices.ts:applyChangesInOpenFiles
- TypeScript/src/server/editorServices.ts:assignProjectToOpenedScriptInfo
- No idea how this actually talks to `tsc`

- [ ] Figure out how to make it so that this doesn't break the `\\?\` paths https://docs.microsoft.com/en-us/windows/win32/fileio/naming-a-file
  - [ ] Find out if this is needed to upstream to the TypeScript project or not
- [ ] Figure out if we need to worry about macOS paths which allow `?`
  - [ ] Decide if the way to tackle this and maybe `\\?\` is to strip and just check if the file exists
- [ ] Take a look at the TypeScript with extensions project and see if this could be offered
- [ ] Take a look at the diff between stock TS and this change and see if a patch could be used
- [ ] Rejoyce at the thought that I can now use my hacky pattern without breaking JS/TS type interference
