# Run DQOps on Railway

This guide deploys DQOps to [Railway](https://railway.app) from this repository using the existing multi-stage
`Dockerfile`. The `railway.json` file in the repository root tells Railway to build the Dockerfile and to start the
container in server mode (`run` command, no interactive shell).

## 1. Create the service

1. In Railway, click **New Project** → **Deploy from GitHub repo** and pick this repository (branch `develop` or `main`).
2. Railway detects `railway.json` and builds the `Dockerfile`. The first build compiles the Java backend and the React
   frontend and takes roughly 15–25 minutes; later builds reuse cached layers.

## 2. Add a persistent volume (recommended)

The DQOps User Home (`/dqo/userhome`) stores connections, check definitions, sensor readouts and check results.
Without a volume everything is lost on every redeploy.

1. Right-click the service → **Add Volume**.
2. Mount path: `/dqo/userhome`.

If you deliberately skip the volume, set `DQO_DOCKER_USER_HOME_ALLOW_UNMOUNTED=true`, otherwise DQOps exits with code
101 on start-up.

## 3. Configure variables

Open the service → **Variables** and add:

| Variable                               | Value                          | Why                                                                                   |
|----------------------------------------|--------------------------------|---------------------------------------------------------------------------------------|
| `PORT`                                 | `8888`                         | DQOps binds to `${port:8888}`; pinning it makes Railway route traffic to the right port |
| `DQO_CLOUD_START_WITHOUT_API_KEY`      | `true`                         | Start in offline mode without prompting for a DQOps Cloud API key                     |
| `DQO_USER_INITIALIZE_USER_HOME`        | `true`                         | Create the User Home on an empty volume without an interactive prompt                 |
| `DQO_DOCKER_USER_HOME_ALLOW_UNMOUNTED` | `true` (only without a volume) | Allow an ephemeral User Home inside the container                                     |
| `DQO_JAVA_OPTS`                        | `-XX:MaxRAMPercentage=70.0`    | Optional; JVM heap as a share of the container memory                                 |
| `DQO_CLOUD_API_KEY`                    | your key                       | Optional; enables DQOps Cloud dashboards and synchronization                          |

Allocate at least 2 GB of memory to the service (Settings → Resources); 4 GB is comfortable.

## 4. Expose the service

Settings → **Networking** → **Generate Domain** and choose port `8888`. Open the generated URL; the DQOps UI loads
at `/`, the REST API at `/api`, and Swagger at `/swagger-ui.html`.

## 5. Load your data

DQOps reads local CSV/Parquet/JSON files through DuckDB. To analyse a file on Railway, put it on the volume, e.g.
copy it into `/dqo/userhome/data/` with `railway ssh` (or `railway run` + `scp`-style tooling), then register a
DuckDB connection pointing at `/dqo/userhome/data`. Any database connection (PostgreSQL, MySQL, BigQuery, Snowflake,
...) works the same way as on a local install; Railway-hosted databases can be referenced with their private
`*.railway.internal` host names.

## Security note

The open-source edition of DQOps has no built-in authentication. A public Railway domain therefore exposes the UI and
REST API to anyone who knows the URL. Keep the domain private, front it with an authenticating proxy, or use
Railway's private networking and access it through another authenticated service.

## Deploying with the Railway CLI

```
npm i -g @railway/cli
railway login
railway init            # or: railway link  (existing project)
railway up              # builds the Dockerfile and deploys
railway variables --set PORT=8888 --set DQO_CLOUD_START_WITHOUT_API_KEY=true --set DQO_USER_INITIALIZE_USER_HOME=true
railway volume add --mount-path /dqo/userhome
railway domain
```
