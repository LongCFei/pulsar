# Building a custom web-tree-sitter

Tree-sitter parsers often use external C scanners, and those scanners sometimes use functions in the C standard library. For this to work in a WASM environment, `web-tree-sitter` needs to have anticipated which stdlib functions will need to be available. If a Tree-sitter parser uses stdlib function X, but X is not included in [this list of symbols](https://github.com/tree-sitter/tree-sitter/blob/master/lib/src/wasm/stdlib-symbols.txt), the parser will fail to work and will throw an error whenever it hits a code path that uses the rogue function.

For this reason, Pulsar builds a custom `web-tree-sitter`. Every time someone tries to integrate a new tree-sitter parser into a Pulsar grammar, they might find that the parser relies on some stdlib function we haven’t included yet — in which case they can let us know and we’ll be able to update our `web-tree-sitter` build so that it can export that function.

The need to do this will decrease over time as C++ scanners are deprecated and as parsers are increasingly encouraged to use a fixed subset of possible stdlib exports, but it’s still necessary right now.

We also take advantage of the custom build by adding a check for a common failure scenario — a parser trying to use a stdlib function that _hasn’t_ been exported — so that we can log a helpful error message to the console when it happens.

## Check out the modified branch for the version we’re targeting

At time of writing, Pulsar was targeting `web-tree-sitter` version **0.23.0**, so a branch exists [on our fork](https://github.com/pulsar-edit/tree-sitter/tree/v0-23-0-modified) called `v0-23-0-modified`. That branch contains a modified `stdlib-symbols.txt` file and a modified script for building `web-tree-sitter`.

When we target a newer version of `web-tree-sitter`, a similar branch should be created against the corresponding upstream tag. The commits that were applied on the previous modified branch should be able to be cherry-picked onto the new one rather easily.

## Add whatever methods are needed to `stdlib-symbols.txt`

For instance, one of the parsers we use depends on the C stdlib function `isalnum`, and `web-tree-sitter` doesn’t export that one by default. So we can add the line

```
  "isalnum",
```

in an appropriate place in `stdlib-symbols.txt`, then rebuild `web-tree-sitter` so that the WASM-built version of that parser has that function available to it.

If a third-party tree-sitter grammar needs something more esoteric, we should encourage them to follow current best practices for Tree-sitter parsers. But we may still want to add that dependency to the build.

## Build `web-tree-sitter`

To build `web-tree-sitter` for a particular version, make sure you’re using the appropriate version of Emscripten. [This document](https://github.com/sogaiu/ts-questions/blob/master/questions/which-version-of-emscripten-should-be-used-for-the-playground/README.md) is useful at matching up tree-sitter versions with Emscripten versions.

Switch to the `lib/binding_web` directory, then run:

```sh
npm install
CJS=true npm run build
```

This generates a CommonJS version of `web-tree-sitter`, along with a `.wasm` file and a `.d.ts` file.

## Copy it to `vendor`

All the artifacts of the build command should be named `web-tree-sitter` (with various extensions) and will be put into the current directory. Copy them all into `vendor/web-tree-sitter`, then rename `web-tree-sitter.cjs` and `web-tree-sitter.cjs.map` to have a `js` extension.

## Add a warning message

When a parser tries to use a stdlib function that isn’t exported by `web-tree-sitter`, the error that’s thrown is not very useful. So we try to detect when that scenario is going to happen and insert a warning in the console to help users that might otherwise be befuddled.

This may be automated in the future, but for now you can modify `web-tree-sitter.js` so that this function…

```js
function resolveSymbol(sym) {
  var resolved = resolveGlobalSymbol(sym).sym;
  if (!resolved && localScope) {
    resolved = localScope[sym];
  }
  if (!resolved) {
    resolved = moduleExports[sym];
  }
  return resolved;
}

```

…has an extra check at the end:

```js
function resolveSymbol(sym) {
  var resolved = resolveGlobalSymbol(sym).sym;
  if (!resolved && localScope) {
    resolved = localScope[sym];
  }
  if (!resolved) {
    resolved = moduleExports[sym];
  }
  if (!resolved) {
    console.warn(`Warning: parser wants to call function ${sym}, but it is not defined. If parsing fails, this is probably the reason why. Please report this to the Pulsar team so that this parser can be supported properly.`);
  }
  return resolved;
}
```


The function in question is [generated by emscripten](https://github.com/emscripten-core/emscripten/blob/127fb03dad7288a71f51bd46be49f1da8bdb0fa8/src/library_dylink.js#L665-L677) and is the rough equivalent of what we’d get if we built with assertions enabled (though less generic and more tailored to Pulsar). If the implementation changes on the emscripten side, you should still be able to find the equivalent logic.

## TEMPORARY: Fix incompatible syntax

Technically, the new build process for `web-tree-sitter` generates some syntax that’s incompatible with Pulsar running on Electron 12/Node 14. [Static initialization blocks](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Static_initialization_blocks) are a syntactic enhancement that doesn’t arrive until Node v16.11.0, but they’re easy to convert to a more broadly compatible syntax.

Find a block that looks like this:

```js
// src/lookahead_iterator.ts
var LookaheadIterator = class {
  static {
    __name(this, "LookaheadIterator");
  }
  // …
};
```

It’s lucky for us that the entire `static {` block can simply be removed without anything breaking; it’s mainly for friendlier stack traces. But we can go further and simply move it below the class declaration:

```js
// src/lookahead_iterator.ts
var LookaheadIterator = class {
  // …
};
__name(LookaheadIterator, "LookaheadIterator");
```

This needs to be done in about a half-dozen places.

Once we upgrade to a more recent version of Electron, this step won’t be necessary.

## Test it

Relaunch Pulsar and do a smoke test with a couple of existing grammars to make sure you didn’t break anything.

## Commit it

Be sure to mention the version you’re upgrading to **in the commit message** so grammar authors have some way of discerning the version of `web-tree-sitter` they should target.

(It’s a stretch goal to include this information in a more structured format so that it can be inspected at runtime.)
