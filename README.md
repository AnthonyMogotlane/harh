# Hope Again Recovery Home

A full-stack web application developed for Hope Again Recovery Home, supporting its substance abuse recovery services through accessible information, screening appointment bookings, funding enquiries, and online donations.

Built collaboratively as a project for the CodeYourFuture Software Development Programme.

The application integrates Google Calendar for screening appointments, PostgreSQL for managing funding enquiries, and Yoco for processing online donations.

## Contents

- [Features](#features)
- [Technology stack](#technology-stack)
- [Local setup](#local-setup)
- [Configuration](#configuration)
- [Project structure](#project-structure)
- [Development and validation](#development-and-validation)
- [Deployment](#deployment)
- [Contributors](#contributors)

## Features

| Feature | Implementation |
| --- | --- |
| Recovery information | Programme, admissions, family days, projects, future plans, funding partners, and testimonials. |
| Screening appointments | Booking and caregiver information is submitted to Google Calendar. The backend accepts Wednesdays and Thursdays and creates one-hour events using the Africa/Johannesburg time zone. |
| Booking availability | An API endpoint returns event start times from the configured calendar. |
| Funding enquiries | Prospective funder and contribution details are stored in PostgreSQL, independently of donation payments. |
| Donations | The Yoco browser SDK obtains a payment token; Express submits the charge request. Currently configured for testing, with issues listed below. |

## Technology stack

| Area | Technologies |
| --- | --- |
| Frontend | React 18, React Router 6, Bootstrap 5, React Bootstrap, CSS. |
| Backend | Node.js, Express 4, express-validator. |
| Database | PostgreSQL through the `pg` connection pool. |
| Integrations | Google Calendar API with service-account JWT authentication; Yoco SDK and charge API. |
| HTTP and logging | Axios, Morgan, Winston. |
| Build and development | Babel, Webpack 5, webpack-dev-server, nodemon, concurrently. |
| Code quality | ESLint and Prettier. |

Dependency ranges and runtime constraints are recorded in [package.json](./package.json).

## Local setup

### Prerequisites

- Node.js and npm matching the `engines` entries in [package.json](./package.json). The declared ranges permit Node 14, 16, 18, or 24 and npm 6, 7, 8, or 11; they are not a verified compatibility matrix. [.npmrc](./.npmrc) enables strict engine checking.
- PostgreSQL and its command-line tools.
- For bookings: a Google Cloud project with Calendar API enabled and a service account.
- For donation testing: matching Yoco test credentials.

Runtime pins need alignment: [.nvmrc](./.nvmrc) selects Node 18.13.0, while CI uses Node 16. Select and validate a maintained runtime before deployment rather than treating these pins as current recommendations.

### 1. Clone and install

```sh
git clone https://github.com/AnthonyMogotlane/harh.git
cd harh
npm ci
```

### 2. Configure the environment

Create a `.env` file at the repository root using [Configuration](#configuration). Never commit credentials. PostgreSQL must be available even when browsing informational pages: the server connects to the database before it starts listening.

### 3. Initialize PostgreSQL

Create a database with a PostgreSQL role you control. For example, using the local `postgres` role:

```sh
createdb -U postgres harh
psql -U postgres -d harh
```

No schema or migrations are committed. This minimal development schema matches the insert in [funding.js](./server/funding.js); it is a suggested starting point, not an authoritative production schema:

```sql
CREATE TABLE funders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT,
    companyName TEXT,
    country TEXT,
    contactNumber TEXT,
    email TEXT,
    contribType TEXT,
    other TEXT
);
```

Run the SQL in `psql`, then exit with `\q`. PostgreSQL folds these unquoted column names to lowercase, matching the backend's unquoted query. Production constraints and migrations still need to be designed.

### 4. Start development

```sh
npm run dev
```

Open <http://localhost:3000>. Webpack serves the frontend on **3000** and proxies `/api` requests to Express on **3100**. Both processes watch for changes.

Check the backend directly:

```sh
curl http://localhost:3100/api
curl http://localhost:3100/health
```

The first returns `{"message":"Hello, world!"}`. The second returns HTTP 200 when reachable; it does not continuously verify database or integration connectivity.

## Configuration

The backend loads root-level dotenv configuration. Example for local development:

```dotenv
DATABASE_URL=postgres://postgres:replace_me@localhost:5432/harh
LOG_LEVEL=debug
NODE_ENV=development

# Google Calendar
CLIENT_EMAIL=service-account@your-project.iam.gserviceaccount.com
PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nreplace_me\n-----END PRIVATE KEY-----\n"
CALENDAR_ID=your_calendar_id

# Yoco server-side test credential
SECRET_KEY=replace_with_yoco_test_secret
```

| Variable | Purpose and default |
| --- | --- |
| `DATABASE_URL` | Full PostgreSQL URL; takes precedence over individual settings. URL-encode special characters in credentials. |
| `DB_HOST`, `DB_NAME` | Database host and name if no URL is supplied; default to empty strings. |
| `DB_USER` / `DB_USERNAME` | Username; `DB_USER` takes precedence. |
| `DB_PASS` / `DB_PASSWORD` | Password; `DB_PASS` takes precedence. |
| `DB_PORT` | Database port; default `5432`. |
| `PORT` | Server port; default `3000`. The development script overrides it to `3100`. |
| `LOG_LEVEL` | Logging level; default `info`. |
| `NODE_ENV` | Exact value `production` enables proxy trust and HTTPS redirection. |
| `CLIENT_EMAIL`, `PRIVATE_KEY` | Google service-account credentials, required for booking API calls. |
| `CALENDAR_ID` | Target Google Calendar identifier, required for booking API calls. |
| `SECRET_KEY` | Server-side Yoco secret used for charge requests. |

### Google Calendar setup

1. Enable the Google Calendar API in your Google Cloud project.
2. Create a service account and obtain its credentials securely.
3. Share the target calendar with the service-account email, granting permission to make changes to events.
4. Set `CLIENT_EMAIL`, `PRIVATE_KEY`, and `CALENDAR_ID`. Keep escaped PEM newlines inside a double-quoted value in the local dotenv file. Preserve actual PEM newlines when supplying hosted secrets.
5. Verify reading and creating events with synthetic data.

This authenticates the backend to Google; it does not provide visitor login.

## Project structure

```text
client/
  src/
    App.js             Client routes
    components/        Shared UI
    pages/             Pages, forms, and styles
    assets/            Images
  webpack/             Development and production build configuration
server/
  app.js               Middleware and route registration
  server.js            Database connection and HTTP lifecycle
  booking.js           Calendar integration
  bookingValidate.js   Booking validation
  funding.js           Funding persistence
  donate.js            Yoco charges
  db.js                PostgreSQL pool
  utils/               Configuration, logging, and middleware
.github/workflows/     CI
Dockerfile             Multi-stage container build
render.yaml            Render service and database configuration
manifest.yml           Cloud Foundry manifest
```

## Development and validation

Run commands from the repository root.

| Command | Purpose |
| --- | --- |
| `npm run dev` | Starts frontend and backend watchers. |
| `npm run build` | Cleans output and builds client and server into `dist`. |
| `npm start` | Starts the already-built server; does not build first. |
| `npm run serve` | Builds through the `preserve` lifecycle hook, then starts. Does not set `NODE_ENV`. |
| `npm run lint` | Runs ESLint and checks Prettier formatting. |
| `npm run lint:fix` | Applies ESLint fixes and rewrites files with Prettier. |

The development server exposes the Node inspector on **9229**. VS Code launch configurations are in [launch.json](./.vscode/launch.json).

[CI](./.github/workflows/pull_request.yml) runs dependency installation and linting on pull requests and manual dispatch. It does not build, run application tests, or verify deployments. There is no `npm test` script or application test suite.

Suggested checks before submitting changes:

```sh
npm run lint
npm run build
```

With services configured, manually verify navigation, invalid and valid bookings, calendar availability, and funding persistence. Exercise donations only with test credentials, checking the actual charge response and amount instead of relying on the confirmation UI. These instructions are not a claim that the current checkout passes those checks.

## Deployment

### Render

[render.yaml](./render.yaml) defines a Node web service, PostgreSQL database, and `/health` check. Its build command is:

```sh
npm ci --include dev; npm run build; npm prune --omit dev
```

Its start command is `npm start`. Create a Blueprint from your repository and review resource settings before applying it. The manifest contains free-plan settings; confirm their availability with the provider.

The Blueprint supplies `DATABASE_URL`, `LOG_LEVEL`, and `NODE_ENV=production`. Add Calendar credentials and the Yoco test secret separately, and initialize `funders` in the provisioned database. Confirm the branch and automatic deployment settings in Render; commits alone do not establish those settings.

### Local production build

```sh
npm run serve
```

This builds optimized frontend assets and starts Express, normally at <http://localhost:3000>. It does not set production runtime mode. For hosted production behavior, set `NODE_ENV=production` in the deployment environment and use a trusted HTTPS-terminating proxy: production mode redirects plain HTTP requests to HTTPS.

### Docker

[Dockerfile](./Dockerfile) requires `NODE_RELEASE` and defaults to Alpine `3.17`. Choose a published compatible Node/Alpine tag pair; CI does not validate one. Replace both placeholders:

```sh
docker build --build-arg NODE_RELEASE=<node-release> --build-arg ALPINE_RELEASE=<alpine-release> -t harh .
docker run --rm --env-file .env -p 3000:80 harh
```

The image defaults to port **80** and does not set `NODE_ENV=production`. Adjust the mapping if your environment overrides `PORT`. Use a database hostname reachable from the container; `localhost` inside the container refers to the container itself. Docker environment files do not interpret dotenv's quoted PEM newline escapes; supply the private key with its newlines preserved through appropriate secret injection.

## Contributors

The following contributors are identified in this repository's Git history, with author aliases grouped under one name:

- [Anthony Mogotlane](https://github.com/AnthonyMogotlane)
- [Tersia Koetzee](https://github.com/tersiakoetzee)
- [Nishka Kisten](https://github.com/Nishka-Kisten)
- [Chezlin Benson](https://github.com/chezlinbenson)