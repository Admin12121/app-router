<div align="center">

# app-router

A minimal Flask app router for server-rendered Python web apps with route-based
templates, nested layouts, partial navigation, route-local assets, API handlers,
CSRF protection, and secure defaults.

<p>
  <img alt="Release v0.1.0" src="https://img.shields.io/badge/RELEASE-v0.1.0-e6b8b8?labelColor=2f2d42&style=flat-square">
  <img alt="Stars 0" src="https://img.shields.io/badge/STARS-0-b9b7ee?labelColor=2f2d42&style=flat-square">
  <img alt="Issues 0 open" src="https://img.shields.io/badge/ISSUES-0%20OPEN-f4a77b?labelColor=2f2d42&style=flat-square">
  <img alt="Contributors 1" src="https://img.shields.io/badge/CONTRIBUTORS-1-aee3a2?labelColor=2f2d42&style=flat-square">
</p>

</div>

## Overview

`app-router` is a lightweight Flask extension that brings file-style page
routing patterns to traditional Flask applications. It lets you register routes
in Python while rendering predictable Jinja templates such as `page.html`,
`dashboard/page.html`, and nested `layout.html` files.

The router is designed for server-rendered apps that want a smoother navigation
experience without becoming a full frontend framework. Normal requests render
complete HTML pages. Internal link clicks can be upgraded by the bundled client
script into partial JSON requests, replacing only the route boundary that
changed.

## Features

- Page routes registered with `@router.page(...)`.
- JSON/API routes registered with `@router.api(...)`.
- Nested layouts with route boundaries for partial page updates.
- Built-in client script served from `/_app/router.js`.
- Route-local asset rewriting for `./asset.ext` and `@/asset.ext` imports.
- Manifest-backed asset serving through `/_app/assets`.
- Signed, time-limited, session-bound CSRF tokens for unsafe methods.
- Safe redirect helper with same-origin validation.
- Default security headers including CSP, `nosniff`, and referrer policy.
- Small Jinja helpers for class names, HTML attributes, and CSRF inputs.

## Requirements

- Python 3.10+
- Flask
- Jinja2
- Werkzeug
- MarkupSafe
- itsdangerous

This directory currently contains the package source only. It does not include
a `pyproject.toml`, `setup.py`, or dependency lock file.

## Installation

For local development, keep this package on your Python path or package it with
your application. The Python import path used by the current source is
`flask_app_router`.

```python
from flask_app_router import AppRouter
```

If this project is later published as `app-router`, the distribution name can
be `app-router` while the import package remains `flask_app_router`.

## Quick Start

```python
from flask import Flask, request
from flask_app_router import AppRouter, router_redirect

app = Flask(__name__)
app.secret_key = "replace-with-a-strong-secret"

router = AppRouter(app)


@router.page("/")
def home():
    return {
        "name": "Student",
        "_meta": {
            "title": "Home",
            "description": "Welcome page rendered by app-router",
        },
    }


@router.page("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        return router_redirect("/")
    return {}


@router.api("/api/status")
def status():
    return {"ok": True}
```

Create the matching template:

```html
<!-- templates/page.html -->
<h1>Hello {{ name }}</h1>
```

Add CSRF protection to forms that use unsafe methods:

```html
<form method="post">
  {{ csrf_input() }}
  <button type="submit">Submit</button>
</form>
```

## Routing Model

Routes are declared in Python and mapped to templates automatically unless a
template is provided explicitly.

```python
@router.page("/dashboard")
def dashboard():
    return {"stats": load_stats()}
```

By default, `/dashboard` maps to:

```text
templates/dashboard/page.html
```

Dynamic Flask route segments map to bracket-style template folders:

```python
@router.page("/users/<user_id>")
def user_detail(user_id):
    return {"user": get_user(user_id)}
```

This maps to:

```text
templates/users/[user_id]/page.html
```

Catch-all `path` converters are intentionally rejected for page routes because
they make template ownership and asset resolution ambiguous.

## Layouts

`layout.html` files wrap pages and create boundaries for partial navigation.

```text
templates/
  layout.html
  page.html
  dashboard/
    layout.html
    page.html
```

The root `layout.html` wraps every page. Nested layouts wrap only the routes
below their folder. Layout templates receive `children`, which should be
rendered where the child route content belongs.

```html
<!-- templates/layout.html -->
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
  </head>
  <body>
    {{ children }}
  </body>
</html>
```

## Page Loader Return Values

Page loaders may return:

- `dict`: template context data.
- `None`: empty template context.
- `RedirectResult`: redirect generated with `router_redirect(...)`.
- `flask.Response`: direct custom response.

Special dictionary keys:

- `_meta`: optional `title`, `description`, and `image` metadata.
- `_redirect`: redirect URL.
- `_status`: redirect status code, defaulting to `303`.
- `_message`: optional redirect message for partial navigation.
- `_cache`: enables public page caching when truthy.
- `_ttl`: cache lifetime in seconds when `_cache` is enabled.

## API Routes

API loaders are converted to JSON responses unless they return a direct Flask
`Response`.

```python
@router.api("/api/profile", methods=["GET"])
def profile():
    return {"name": "Ada"}
```

Tuple-style responses are supported:

```python
return {"created": True}, 201, {"X-App": "app-router"}
```

API responses use `Cache-Control: no-store` by default.

## Assets

Templates can reference local assets with either relative or alias-style paths:

```html
<link rel="stylesheet" href="./page.css">
<script type="module" src="./page.js"></script>
<img src="@/shared/logo.svg" alt="Logo">
```

Supported extensions are:

```text
.js, .css, .png, .jpg, .jpeg, .webp, .svg, .woff, .woff2
```

During rendering or manifest generation, assets are resolved, hashed, and served
through the configured asset URL prefix. Parent directory traversal, unsupported
extensions, missing files, and symlink escapes are rejected.

## Build Manifest

Generate route and asset metadata with the CLI:

```bash
flask-app-router build --app app:app --output .flask-app-router
```

For an app factory:

```bash
flask-app-router build --app 'app:create_app()'
```

The build writes:

- `routes.json`: route, layout, and rendering metadata.
- `manifest.json`: asset metadata and route asset indexes.
- `assets/`: copied build assets.

## Configuration

```python
router = AppRouter(
    app,
    asset_url_path="/_app/assets",
    client_url_path="/_app/router.js",
    security_headers=True,
    csrf=True,
    build_dir=".flask-app-router",
)
```

Flask configuration:

- `SECRET_KEY`: required for CSRF signing.
- `FLASK_APP_ROUTER_CSRF_MAX_AGE`: CSRF token lifetime in seconds.
- `FLASK_APP_ROUTER_BUILD_DIR`: alternate manifest directory.

## Security Assessment

No critical vulnerability was identified in the reviewed source.

Implemented controls:

- CSRF tokens are signed with Flask `SECRET_KEY`, time-limited, tied to the
  current session, and compared with `hmac.compare_digest`.
- Redirects are limited to local paths or same-origin absolute URLs.
- Asset serving is manifest-backed and restricts extensions, traversal, and
  symlink escapes.
- Private assets receive `Cache-Control: no-store`; public assets use immutable
  cache headers.
- API and partial-navigation responses use `Cache-Control: no-store`.
- Default response hardening includes `Content-Security-Policy`,
  `X-Content-Type-Options: nosniff`, and `Referrer-Policy`.

Known risks and limitations:

- The default CSP includes `style-src 'unsafe-inline'`.
- Partial navigation uses `innerHTML` to apply trusted server-rendered HTML.
  Application templates must rely on Jinja escaping and avoid unsafe `Markup`
  usage with untrusted input.
- CSRF protection is only as strong as the Flask `SECRET_KEY`.
- This folder does not include dependency metadata or a lock file, so dependency
  versions are not reproducible from this source tree alone.
- No automated tests are included in this directory.

Recommended hardening:

- Add `pyproject.toml` with package metadata, dependency ranges, and the CLI
  entry point.
- Add tests for CSRF validation, redirect safety, asset traversal rejection,
  manifest loading, error rendering, and partial navigation.
- Use a stricter CSP if inline styles are not required by consuming apps.
- Document template trust boundaries for applications that pass user-generated
  HTML into templates.

## Project Structure

```text
app-router/
  __init__.py          Public API exports
  router.py            Core router, rendering, headers, and internal routes
  assets.py            Asset resolver, rewriter, manifest builder, and server
  csrf.py              CSRF token generation and validation
  helpers.py           Jinja and routing helpers
  responses.py         Redirect response helper
  exceptions.py        Package-specific exceptions
  static/router.js     Partial-navigation browser client
  templates/           Built-in 404 and 500 templates
  README.md            Project documentation
```

## Verification

The Python source was parsed successfully with `ast`. A normal `compileall`
check could not complete in this environment because the filesystem is
read-only and bytecode writes to `__pycache__` failed.
