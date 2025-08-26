# Svelte Monorepo Serve Error

In a monorepo, node_modules package may be a symlink to a local folder.

Svelte incorrectly configures Vite's dev server allow list in such cases,
leading to 403 Forbidden errors when a monorepo package tries to load an asset
(like a font).

## Reproduction

1. Clone this repo

   ```sh
   git clone https://github.com/maxpatiiuk/svelte-monorepo-serve-error
   cd svelte-monorepo-serve-error
   ```

2. Install dependencies

   ```sh
   # To keep reproduction size minimal, I hardcoded a minimal root-level node_modules
   # So, install Svelte dependencies only in the app folder:
   cd app
   npm install
   ```

3. Start the development server in the app folder

   ```sh
   npm run dev
   ```

   See this error in the browser console:

   ```
   (index):45  GET http://localhost:5174/@fs/Users/.../svelte-monorepo-serve-error/monorepo-utils/NotoSans.ttf net::ERR_ABORTED 403 (Forbidden)
   ```

Related details:

- Setting `server.fs.preserveSymlinks: true` in the Vite config does not help.
- This does not error for .svg assets because they are inlined (or maybe assets
  beyond size threshold are inlined?).
- Everything works correctly during build

## Related details

If you run the dev server with Vite config logging
(`DEBUG=vite:config npx ng serve`), you will see that the dev server was allowed
to serve only the package-level node_modules folder:

```yaml
  vite:config   server: {
  vite:config     ...
  vite:config     fs: {
  vite:config       ...
  vite:config       allow: [
  vite:config         '/Users/.../svelte-monorepo-serve-error/app/src/lib',
  vite:config         '/Users/.../svelte-monorepo-serve-error/app/src/routes',
  vite:config         '/Users/.../svelte-monorepo-serve-error/app/.svelte-kit',
  vite:config         '/Users/.../svelte-monorepo-serve-error/app/src',
  vite:config         '/Users/.../svelte-monorepo-serve-error/app/node_modules',
  vite:config         '/Users/.../svelte-monorepo-serve-error/node_modules'
  vite:config       ]
  vite:config     },
```

By default, in monorepo setups, Vite permits serving files from any folder in
the monorepo (`/Users/.../svelte-monorepo-serve-error/`). Svelte overrides that
behavior.

Vite's default behavior is implemented here:

https://github.com/vitejs/vite/blob/e899bc7c73a27cdf327875e5d696c50d396a7fc2/packages/vite/src/node/server/index.ts#L1126-L1127

Their default calls `searchForWorkspaceRoot()`, which is exposed by Vite and can
be called manually as documented in
https://vite.dev/config/server-options.html#server-fs-allow.
