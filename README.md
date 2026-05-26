# require.kmi

`require.kmi` is a lightweight browser module loader that brings a small
CommonJS-style `require` workflow to client-side JavaScript. It loads modules
synchronously through `XMLHttpRequest`, supports extension-based resource
handlers, and exposes a global `$require` function for bootstrapping browser
applications.

The project is intentionally small: the loader lives in
[`src/require.kmi.js`](src/require.kmi.js), generated API documentation lives in
[`docs/`](docs/), and JSDoc configuration lives in [`jsdoc.json`](jsdoc.json).

## Features

- Browser global named `$require`.
- CommonJS-like JavaScript modules with `require`, `module`, `exports`, and
  `__dirname` arguments.
- Automatic `.js` extension fallback when a requested path has no extension.
- Relative, parent-relative, absolute, and module-root path resolution.
- Built-in handlers for JavaScript, CSS, HTML, text, JSON, images, and audio.
- Module caching by default, with opt-out support through `module.disposable`.
- Custom extension handlers and handler aliases.
- Simple entry-point discovery through `<meta kmi-init="...">`.

## Installation

Install dependencies only if you want to work on the repository or regenerate
documentation:

```bash
npm install
```

For browser usage, serve `src/require.kmi.js` with your application assets. The
default resolver expects bare module imports to live under `/kmi_modules`.

```html
<script src="/kmi_modules/require.kmi.js"></script>
<meta kmi-init="/app/main">
```

When the page finishes loading, the loader scans for `kmi-init` meta tags and
loads each entry path. Because `/app/main` has no extension, it resolves to
`/app/main.js`.

## Quick Start

Create an entry module:

```js
// /app/main.js
const config = require('./config.json');
const view = require('./view.html');

require('./styles.css');

document.body.appendChild(view);

exports.config = config;
```

Create a local dependency at `/app/config.json`:

```json
{
  "name": "Demo app"
}
```

Any module loaded through `require.kmi` receives a local `require` function that
resolves paths from that module's directory.

## API

### `$require(path[, callerFolder])`

Loads a resource and returns the handler's exported value.

```js
const app = $require('/app/main');
```

If `path` does not include an extension, `.js` is appended automatically. Loaded
modules are cached in `$require.modules` unless the handler sets
`module.disposable = true`.

### `$require.resolve(path, base[, parent])`

Resolves a dependency path before it is loaded.

| Input style | Example | Resolution behavior |
| --- | --- | --- |
| Absolute path | `/app/main` | Returned unchanged |
| URL | `https://example.com/app.js` | Returned unchanged |
| Relative path | `./view` | Resolved from `base` |
| Parent path | `../shared/util` | Walks up from `base` |
| Parent-relative path | `$component` | Resolved from `parent` |
| Bare module | `some-package` | Resolved from `/kmi_modules/some-package` |

### `$require.setHandler(ext, handler)`

Registers a resource handler for a file extension.

```js
$require.setHandler('svg', function (path, module) {
  const request = new XMLHttpRequest();
  request.open('GET', path, false);
  request.send(null);

  module.exports = request.responseText;
  module.disposable = true;
});
```

The handler receives the resolved path and a mutable `module` object. Set
`module.exports` to define the returned value. Set `module.disposable = true` to
skip caching.

You can also alias one extension to another handler:

```js
$require.setHandler('jpeg', 'img');
```

### `$require.getHandler(ext)`

Returns the handler registered for an extension. String aliases are followed
until the concrete handler function is found.

## Built-In Handlers

| Extension | Export | Cache behavior |
| --- | --- | --- |
| `js` | Executes a CommonJS-like module function | Cached |
| `css` | Appends a `<link rel="stylesheet">` and exports metadata | Cached |
| `html` | Fetches markup and exports the first DOM node | Cached |
| `txt` | Fetches and exports plain text | Not cached |
| `json` | Parses and exports JSON | Not cached |
| `img`, `bmp`, `gif`, `png`, `jpg` | Creates and exports an `Image` | Not cached |
| `audio`, `wav`, `mp3` | Creates and exports an `Audio` | Not cached |

For JavaScript modules, the loader first requests the resolved `.js` file. If it
is not found, it tries a `.min.js` variant, then an `index.js` file in a
directory with the requested module name.

## Bundles

`$require.bundles` can be populated with module functions to bypass network
loading:

```js
$require.bundles['/kmi_modules/example/index.js'] = function (
  require,
  __dirname,
  module,
  exports
) {
  exports.message = 'Loaded from bundle';
};
```

When a requested JavaScript path matches a bundle entry, the bundled function is
executed instead of fetching the file through `XMLHttpRequest`.

## Development

Generate JSDoc output:

```bash
npm run generate-docs
```

The generated documentation is written to `docs/`.

## Repository Layout

```text
.
|-- src/require.kmi.js    # Browser loader implementation
|-- docs/                 # Generated JSDoc site
|-- jsdoc.json            # Documentation generator configuration
|-- test.js               # Legacy local browser harness
|-- package.json          # Package metadata and npm scripts
`-- LICENSE               # GPL-3.0-or-later license text
```

## Limitations

- This is a browser loader, not a Node.js replacement for `require`.
- Loading is synchronous and can block the browser main thread.
- Requests are subject to normal browser same-origin, CORS, and local file
  restrictions.
- `package.json` currently points `main` to `index.js`, but this repository's
  loader implementation is `src/require.kmi.js`.
- `test.js` is a legacy harness rather than a complete automated test suite.

## License

`require.kmi` is licensed under the
[GNU General Public License v3.0 or later](LICENSE).
