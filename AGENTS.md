# AGENTS.md

## Cursor Cloud specific instructions

This repo is **Goof** — Snyk's intentionally vulnerable Node.js/Express TODO demo app. There is a single web service (`goof`) at the repo root. It stores TODOs and users in **MongoDB** (required) and has an optional **MySQL** (TypeORM) backend used only by the `/users` routes.

### Services

| Service | Required? | How to run | Notes |
|---|---|---|---|
| `goof` web app | yes | `npm run dev` (nodemon) or `npm start` — listens on `http://localhost:3001` | Scripts already set `NODE_OPTIONS=--openssl-legacy-provider`; do not drop this flag (needed by the old dependency stack on modern Node). |
| MongoDB | yes | container `goof-mongo` on port 27017 (see below) | App connects at boot to `mongodb://localhost/express-todo` and seeds `admin@snyk.io` / `SuperSecretPassword`. Use **Mongo 3.x/4.2** (node `mongodb` driver is `3.5.9`); Mongo 5+ is incompatible. |
| MySQL | optional | container `goof-mysql` on port 3306 | Only needed for `/users` routes. Creds are hard-coded in `typeorm-db.js` (`root`/`root`, db `acme`). If it is down the app still boots (connection error is caught). |

### Non-obvious startup caveats (important)

- **Docker uses a non-default config here.** This VM's kernel does not support `overlay2` and `iptables`/`nftables` are not installable (Ubuntu archive egress is blocked), so `/etc/docker/daemon.json` is set to `storage-driver: vfs`, `iptables: false`, `bridge: none`. Because of `bridge: none`, **containers must use `--network host`** (port publishing with `-p` does not work). `vfs` is slow/disk-heavy but functional.
- **Start MongoDB before the app.** The MySQL/TypeORM connection is attempted only once at startup; if you want `/users` to work, start `goof-mysql` and wait for it to be ready **before** starting the app (otherwise restart the app once MySQL is up).
- Start dockerd (no systemd in this container) if it is not already running: `sudo dockerd &` (logs to your redirect). Then:
  - `sudo docker run -d --name goof-mongo --network host mongo:3`
  - `sudo docker run -d --name goof-mysql --network host -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=acme mysql:5`
- Reset TODOs: `npm run cleanup` (requires a local `mongo` shell; alternatively use the container).

### Lint / test / build

- **Lint:** none configured (no ESLint/Prettier).
- **Test:** `npm test` runs `snyk test` — a security/dependency scan that needs a Snyk account (`snyk auth`) and network to snyk.io; it is not a functional unit-test suite. The file under `tests/` is a demo spec and is not wired to a runnable test runner.
- **Build (optional):** `npm run build` (browserify) regenerates `public/js/bundle.js`; not required for the server to boot.

### Quick end-to-end sanity check

```bash
# login (NoSQL-auth demo) then create a todo
curl -s -X POST -c cj.txt -H 'Content-Type: application/json' \
  --data-binary '{"username":"admin@snyk.io","password":"SuperSecretPassword"}' \
  http://localhost:3001/login -o /dev/null
curl -s -X POST -b cj.txt --data 'content=hello' http://localhost:3001/create -o /dev/null
curl -s http://localhost:3001/ | grep hello        # todo shows on homepage
curl -s http://localhost:3001/users/               # MySQL-backed users route
```
