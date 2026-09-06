# Markdown plans

## Serve the files locally

Install [Caddy](https://caddyserver.com/docs/install), then run this directory as the
working directory:

```sh
caddy run --config Caddyfile
```

Open <http://127.0.0.1:8000/>. Caddy provides the directory index and serves the Markdown
files directly. Stop it with `Ctrl-C`.

For a one-off port or bind address, use Caddy's file server directly:

```sh
caddy file-server --root . --browse --listen :9000
caddy file-server --root . --browse --listen 0.0.0.0:8000
```
