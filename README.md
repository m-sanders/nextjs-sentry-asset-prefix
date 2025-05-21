# Reproduction

Using the experimental compile / generate mode results in an invalid server bundle (downgrading to `5.2.5` successfully builds).

Steps:

1. `npm run build -- --no-mangling --experimental-build-mode compile`
2. `npm run build -- --experimental-build-mode generate`

Results in the below error:

```
> next build --experimental-build-mode generate

   ▲ Next.js 15.3.2
   - Experiments (use with caution):
     · clientTraceMetadata

   Creating an optimized production build ...
 ✓ Collecting page data
   Inlining static env ...
Error occurred prerendering page "/_not-found". Read more: https://nextjs.org/docs/messages/prerender-error
/Users/me/code/nextjs-sentry-asset-prefix/.next/server/chunks/575.js:172413
    "/assets/" = userNextConfig.assetPrefix;
    ^^^^^^^^^^

SyntaxError: Invalid left-hand side in assignment
    at wrapSafe (node:internal/modules/cjs/loader:1486:18)
    at Module._compile (node:internal/modules/cjs/loader:1528:20)
    at Object..js (node:internal/modules/cjs/loader:1706:10)
    at Module.load (node:internal/modules/cjs/loader:1289:32)
    at Function._load (node:internal/modules/cjs/loader:1108:12)
    at TracingChannel.traceSync (node:diagnostics_channel:322:14)
    at wrapModuleLoad (node:internal/modules/cjs/loader:220:24)
    at Module.<anonymous> (node:internal/modules/cjs/loader:1311:12)
    at mod.require (/Users/me/code/nextjs-sentry-asset-prefix/node_modules/next/dist/server/require-hook.js:65:28)
    at require (node:internal/modules/helpers:136:16)
Export encountered an error on /_not-found/page: /_not-found, exiting the build.
 ⨯ Next.js build worker exited with code: 1 and signal: null
```

Where the source is invalid JavaScript:

```js
function setUpBuildTimeVariables(
  userNextConfig,
  userSentryOptions,
  releaseName,
) {
  const assetPrefix = userNextConfig.assetPrefix || userNextConfig.basePath || '';
  const basePath = userNextConfig.basePath ?? '';
  const rewritesTunnelPath =
    userSentryOptions.tunnelRoute !== undefined && userNextConfig.output !== 'export'
      ? `${basePath}${userSentryOptions.tunnelRoute}`
      : undefined;

  const buildTimeVariables = {
    // Make sure that if we have a windows path, the backslashes are interpreted as such (rather than as escape
    // characters)
    _sentryRewriteFramesDistDir: userNextConfig.distDir?.replace(/\\/g, '\\\\') || '.next',
    // Get the path part of `assetPrefix`, minus any trailing slash. (We use a placeholder for the origin if
    // `assetPrefix` doesn't include one. Since we only care about the path, it doesn't matter what it is.)
    _sentryRewriteFramesAssetPrefixPath: assetPrefix
      ? new URL(assetPrefix, 'http://dogs.are.great').pathname.replace(/\/$/, '')
      : '',
  };

  if (userNextConfig.assetPrefix) {
    "/assets/" = userNextConfig.assetPrefix;
  }

  if (userSentryOptions._experimental?.thirdPartyOriginStackFrames) {
    buildTimeVariables._experimentalThirdPartyOriginStackFrames = 'true';
  }

  if (rewritesTunnelPath) {
    buildTimeVariables._sentryRewritesTunnelPath = rewritesTunnelPath;
  }

  if (basePath) {
    buildTimeVariables._sentryBasePath = basePath;
  }

  if (userNextConfig.assetPrefix) {
    "/assets/" = userNextConfig.assetPrefix;
  }

  if (userSentryOptions._experimental?.thirdPartyOriginStackFrames) {
    buildTimeVariables._experimentalThirdPartyOriginStackFrames = 'true';
  }

  if (releaseName) {
    "e127d9a59a22b653b4c29453ad220721ff539e69" = releaseName;
  }

  if (typeof userNextConfig.env === 'object') {
    userNextConfig.env = { ...buildTimeVariables, ...userNextConfig.env };
  } else if (userNextConfig.env === undefined) {
    userNextConfig.env = buildTimeVariables;
  }
}
```
