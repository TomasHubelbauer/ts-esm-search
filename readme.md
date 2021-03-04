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

Run test: `tsc -p .` or `tsc index.ts` or `node TypeScript/built/local/tsc.js index.ts ` to produce `index.js` (Git-ignored)

- [ ] Create a TypeScript project with this module arrangement to feed to the instrumented compiler
- [ ] Access a member of the imported object to cause an error if its type is not known - what we're fixing
- [ ] Step through the various parts of the compiler to learn where the module resolution stuff happens
- [ ] Try to identity the spot where to strip `\?.*$` from the import path if it matches anything
- [ ] Figure out how to make it so that this doesn't break the `\\?\` paths https://docs.microsoft.com/en-us/windows/win32/fileio/naming-a-file
  - [ ] Find out if this is needed to upstream to the TypeScript project or not
- [ ] Figure out if we need to worry about macOS paths which allow `?`
  - [ ] Decide if the way to tackle this and maybe `\\?\` is to strip and just check if the file exists
- [ ] Submit a patch to the TypeScript project and try to get this upstreamed and eventually used in VS Code
- [ ] Rejoyce at the thought that I can now use my hacky pattern without breaking JS/TS type interference
