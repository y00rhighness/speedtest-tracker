# Speedtest Tracker — Failed Tests in Charts

This fork of [alexjustesen/speedtest-tracker](https://github.com/alexjustesen/speedtest-tracker)
adds failed speed tests to the dashboard charts. Speedtest Tracker monitors
internet performance and stores historical results using Ookla's Speedtest CLI.

Previously, the chart queries included only completed tests. Failed attempts
disappeared from the timeline, leaving gaps or connecting the successful results
on either side. This fork includes failed attempts at their original timestamps.
Recorded zero values are plotted as zero, and missing measurements on failed
tests are plotted as zero as well.

Fork: [y00rhighness/speedtest-tracker](https://github.com/y00rhighness/speedtest-tracker)

## Changes made

The original fix changes 27 lines across six existing widgets. It was based on
upstream branch `1.x`, commit `7ab10330f131a1841f7ed21917a56222365ec8a8`.
The chart changes were pushed to this fork as commit `8d96995`.

| File | Chart |
| --- | --- |
| [RecentDownloadChartWidget.php](app/Filament/Widgets/RecentDownloadChartWidget.php) | Download speed |
| [RecentUploadChartWidget.php](app/Filament/Widgets/RecentUploadChartWidget.php) | Upload speed |
| [RecentPingChartWidget.php](app/Filament/Widgets/RecentPingChartWidget.php) | Ping |
| [RecentDownloadLatencyChartWidget.php](app/Filament/Widgets/RecentDownloadLatencyChartWidget.php) | Download latency |
| [RecentUploadLatencyChartWidget.php](app/Filament/Widgets/RecentUploadLatencyChartWidget.php) | Upload latency |
| [RecentJitterChartWidget.php](app/Filament/Widgets/RecentJitterChartWidget.php) | Jitter |

Each widget now selects the result status and includes both terminal statuses:

```php
->whereIn('status', [ResultStatus::Completed, ResultStatus::Failed])
```

Missing measurements fall back to zero only for failed results. Existing numeric
measurements are preserved, including partial measurements from a failed test.
Download and upload retain their existing conversion to Mbps.

The download, upload, and ping average lines still use completed tests only.
Synthetic or real failures therefore do not lower these averages. Time-range
filters, chronological ordering, timezone formatting, and the existing chart
styling remain in place. No database migration or new application dependency
was added for this fix.

### Behavior and limits

- Existing failed records automatically appear within the selected chart range.
- Waiting, checking, started, running, benchmarking, and skipped results remain excluded.
- A zero substituted for missing ping, latency, or jitter represents a failed
  measurement; it does not mean that measured latency was zero.
- No records are invented for periods when the application was stopped or when
  a test attempt was never saved.
- Rebuilding a custom image is required after changing the PHP files. Pushing
  to GitHub alone does not update a running container.

## How to build the Docker container from scratch

These steps create a new installation with SQLite, port `8080`, and persistent
storage. For an existing installation, use the upgrade section instead so its
database and encryption key are retained.

Prerequisites: a Linux Docker host with Docker Compose v2, Git, OpenSSL, and
network access to GitHub and the LinuxServer image registry. Run Docker commands
as an account allowed to use Docker, or prefix them with `sudo` as appropriate.

The custom image layers only the six modified PHP files onto the LinuxServer
production image. It inherits the web server, PHP dependencies, Speedtest CLI,
scheduler, and startup behavior from that image. It does not replace the whole
application with the repository checkout. Review compatibility before applying
these widget files to substantially different upstream versions.

### 1. Clone the fork

On a fresh host:

```bash
git clone --branch 1.x https://github.com/y00rhighness/speedtest-tracker.git ~/src
cd ~/src
ls app/Filament/Widgets/Recent*ChartWidget.php
```

If this fork is already cloned, change into that checkout instead. All subsequent
build commands run from the repository root: the directory containing `app/`.
The Dockerfile paths below use `app/...`, not `src/app/...`.

### 2. Create `Dockerfile.custom`

Create this file in the repository root:

```dockerfile
ARG BASE_IMAGE=lscr.io/linuxserver/speedtest-tracker:latest
FROM ${BASE_IMAGE}

COPY app/Filament/Widgets/RecentDownloadChartWidget.php /app/www/app/Filament/Widgets/
COPY app/Filament/Widgets/RecentUploadChartWidget.php /app/www/app/Filament/Widgets/
COPY app/Filament/Widgets/RecentPingChartWidget.php /app/www/app/Filament/Widgets/
COPY app/Filament/Widgets/RecentDownloadLatencyChartWidget.php /app/www/app/Filament/Widgets/
COPY app/Filament/Widgets/RecentUploadLatencyChartWidget.php /app/www/app/Filament/Widgets/
COPY app/Filament/Widgets/RecentJitterChartWidget.php /app/www/app/Filament/Widgets/

RUN set -eu; \
    for file in /app/www/app/Filament/Widgets/Recent*ChartWidget.php; do \
        php -l "$file"; \
    done

RUN php -r 'require "/app/www/vendor/autoload.php"; foreach (glob("/app/www/app/Filament/Widgets/Recent*ChartWidget.php") as $file) { require_once $file; }'
```

The build checks PHP syntax and loads the widget classes to catch basic class
compatibility errors. These checks do not replace a dashboard or database test.
For reproducible future builds, set `BASE_IMAGE` to a tested version tag or
digest; `latest` can change between builds.

### 3. Create `Dockerfile.custom.dockerignore`

This Dockerfile-specific ignore file avoids changing any existing `.dockerignore`
used by other builds. Save the following in the repository root:

```text
.git
.env
.env.*
vendor
node_modules
data
ssl-keys
backups
*.tar
*.tar.gz
```

### 4. Create `compose.custom.yaml`

Save this alongside `Dockerfile.custom`. The separate name preserves the upstream
repository's development `compose.yaml`.

```yaml
services:
  speedtest-tracker:
    build:
      context: .
      dockerfile: Dockerfile.custom
    image: speedtest-tracker-custom:local
    container_name: speedtest-tracker
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      PUID: "${PUID:?Set PUID in .env}"
      PGID: "${PGID:?Set PGID in .env}"
      APP_KEY: "${APP_KEY:?Set APP_KEY in .env}"
      APP_URL: "${APP_URL:?Set APP_URL in .env}"
      TZ: America/Boise
      DISPLAY_TIMEZONE: America/Boise
      DB_CONNECTION: sqlite
      SPEEDTEST_SCHEDULE: "3-59/10 * * * *"
    volumes:
      - speedtest-config:/config

volumes:
  speedtest-config:
```

The image tag belongs in `image:`. The container name must not contain a colon.
The `speedtest-config` named volume stores the database and configuration.

### 5. Generate a configuration file

For a fresh installation only, generate `.env` once. This command refuses to
overwrite an existing `.env`:

```bash
if [ -e .env ]; then
  echo '.env already exists; preserve it and review its settings.'
else
  (
    umask 077
    cat > .env <<EOF
PUID=$(id -u)
PGID=$(id -g)
APP_KEY=base64:$(openssl rand -base64 32)
APP_URL=http://REPLACE_WITH_HOST_IP:8080
EOF
  )
fi

nano .env
```

Replace `REPLACE_WITH_HOST_IP` with the Docker host's LAN IP address or hostname.
If using a reverse proxy, set `APP_URL` to the URL users will actually access.
Preserve this encryption key through future upgrades and restarts. The upstream
`.gitignore` excludes `.env`; never commit keys, passwords, databases, or backups.

### 6. Build and start

```bash
docker compose -f compose.custom.yaml build --pull && \
docker compose -f compose.custom.yaml up -d --no-build --pull never

docker compose -f compose.custom.yaml ps
docker compose -f compose.custom.yaml logs --tail=100 speedtest-tracker
```

Start runs only if the build succeeds. The container initializes the application
and applies its database migrations during startup.

Open `http://HOST_IP:8080/admin`. Unless overridden during initial setup, the
default email is `admin@example.com` and the password is `password`. Change the
login details through the profile page after signing in.

### 7. Apply later chart-code changes

From the same checkout and Compose project directory:

```bash
git pull --ff-only
docker compose -f compose.custom.yaml build && \
docker compose -f compose.custom.yaml up -d --no-build --pull never
```

The named volume survives container recreation. Keep the same Compose project
identity so it continues to use the same volume. Back up before upgrades that
change the underlying application. Add `--pull` to the build only when you also
intend to update the base image.

If you used the original deployment filenames `Dockerfile` and `compose.yaml`,
use your existing filenames and omit `-f compose.custom.yaml` from the commands.

## Test scheduling

The configured schedule runs every ten minutes, offset by three minutes:

```yaml
SPEEDTEST_SCHEDULE: "3-59/10 * * * *"
```

It runs at `:03`, `:13`, `:23`, `:33`, `:43`, and `:53` every hour. In the reviewed
application code, the scheduler evaluates the expression using
`DISPLAY_TIMEZONE`, set here to `America/Boise`.

| Frequency | Cron expression |
| --- | --- |
| Every ten minutes on the hour | `*/10 * * * *` |
| Every ten minutes, offset by three minutes | `3-59/10 * * * *` |
| Every fifteen minutes | `*/15 * * * *` |
| Every thirty minutes | `*/30 * * * *` |
| Hourly at minute zero | `0 * * * *` |
| Daily at 7:00 AM | `0 7 * * *` |

After editing the schedule, recreate the service with:

```bash
docker compose -f compose.custom.yaml up -d --no-build --pull never speedtest-tracker
```

An image rebuild is not needed for an environment-only change.

## Verify failed results plot at zero

The following command inserts one clearly marked synthetic failed result. It
does not run a speed test, interrupt connectivity, or dispatch model events.
The download, upload, and ping values are explicitly zero. Missing latency and
jitter measurements exercise the failed-result fallback as well.

```bash
docker exec -i --user abc speedtest-tracker php <<'PHP'
<?php
chdir('/app/www');
require 'vendor/autoload.php';

$app = require 'bootstrap/app.php';
$app->make(Illuminate\Contracts\Console\Kernel::class)->bootstrap();

$result = App\Models\Result::withoutEvents(fn () => App\Models\Result::create([
    'service' => App\Enums\ResultService::Faker,
    'status' => App\Enums\ResultStatus::Failed,
    'scheduled' => false,
    'download' => 0,
    'upload' => 0,
    'ping' => 0,
    'comments' => 'Manual zero chart test',
    'data' => [
        'type' => 'log',
        'level' => 'error',
        'message' => 'Synthetic failure to verify zero charting',
    ],
]));

echo "Created test result ID: {$result->id}\n";
PHP
```

Refresh the dashboard and choose a chart range covering the current time. All
six charts should include the failed record and plot its missing or zero values
at zero. Confirm that the average lines still reflect completed results only.

Each invocation creates a separate record. To check several points, rerun it a
minute or two apart and run a real test afterward to see the line rise again.
Delete the records labeled `Manual zero chart test` from the Results page when
finished.

### Validation performed

- `git diff --check` passed for the six modified widgets.
- PHP syntax parsing passed for all six files during development.
- The user reported that the Docker deployment and zero-result graphing worked.
- Full Laravel/Pest tests were not run in the development environment, which
  did not have a PHP runtime or installed application dependencies.

## Transfer the custom image to another host

Export and import the image using Docker's image archive commands. This moves
the application image, not the source host's stored test results.

On the build host:

```bash
docker image save -o ~/speedtest-tracker-custom.tar speedtest-tracker-custom:local
scp ~/speedtest-tracker-custom.tar USER@DESTINATION_HOST:~/
```

On the destination:

```bash
docker image load -i ~/speedtest-tracker-custom.tar
docker image inspect speedtest-tracker-custom:local --format '{{.Os}}/{{.Architecture}}'
docker info --format '{{.Architecture}}'
```

Replace `USER` and `DESTINATION_HOST`. The destination must support the image
platform. The deployed image was `linux/amd64`; Linux hosts reporting `x86_64`
match that architecture. An ARM host needs an image built for ARM.

## Upgrade an existing installation without losing its results

Use the destination's existing Compose project, environment, database, and
volumes. Do not replace them with the fresh-install example above. In particular,
retain its `APP_KEY`, database settings, and mounted storage.

### Identify the current deployment

```bash
docker inspect speedtest-tracker --format 'Current image: {{.Config.Image}}'
docker inspect speedtest-tracker --format '{{range .Mounts}}{{println .Type .Source "->" .Destination}}{{end}}'
docker exec speedtest-tracker printenv DB_CONNECTION
docker inspect speedtest-tracker --format \
'Compose directory: {{index .Config.Labels "com.docker.compose.project.working_dir"}}
Compose files: {{index .Config.Labels "com.docker.compose.project.config_files"}}'
```

If Compose labels are absent, identify how the container was created before
recreating it. If environment output is blank, check the application's existing
configuration rather than assuming the database type.

### Back up the SQLite deployment

The existing host inspected during this setup used SQLite and these bind mounts:

| Host path | Container path |
| --- | --- |
| `/home/labadmin/data` | `/config` |
| `/home/labadmin/ssl-keys` | `/config/keys` |

The following backup commands apply to that layout and the standard SQLite
database stored under `/config`. If `DB_DATABASE` points somewhere else, back up
that location too. For MariaDB, MySQL, or PostgreSQL, take a database backup using
that engine's tools; backing up `/config` alone does not back up an external DB.

Run from the existing Compose project directory after loading the custom image:

```bash
docker image tag \
  "$(docker inspect --format '{{.Image}}' speedtest-tracker)" \
  speedtest-tracker-before-upgrade:local

backup_dir="/home/labadmin/backups/speedtest-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"
chmod 700 "$backup_dir"

docker stop speedtest-tracker

sudo tar -czpf "$backup_dir/config-and-keys.tar.gz" \
  -C /home/labadmin data ssl-keys

sudo tar -tzf "$backup_dir/config-and-keys.tar.gz" >/dev/null \
  && echo "Backup archive readable: $backup_dir"
```

Stopping the application before copying SQLite avoids writes during the backup.
The archive check verifies that the archive is readable, not that a database
restore has been tested. If a backup command fails, do not upgrade; restart the
old container with `docker start speedtest-tracker` and resolve the backup issue.

Also copy the actual Compose file and `.env`, if present, to the private backup
directory. Adjust paths and the container name for other installations.

### Switch only the image

Edit the existing Compose service so these lines are exactly:

```yaml
image: speedtest-tracker-custom:local
container_name: speedtest-tracker
```

Remove any `build:` block for that service when using the imported image. Retain
all existing environment, database, port, network, and volume settings. In the
layout above, retain both existing bind mounts:

```yaml
volumes:
  - /home/labadmin/data:/config
  - /home/labadmin/ssl-keys:/config/keys
```

From the existing Compose project directory, using its original Compose filename:

```bash
docker compose up -d --no-build --pull never
docker logs --tail=100 speedtest-tracker
```

If the deployment uses a nondefault Compose filename or multiple files, include
the original `-f` options. Verify login with the existing credentials, historical
result counts, charts, and the next scheduled test. Keep the backup and old image
until these checks pass. Do not run `docker compose down -v`, which removes named
volumes.

The newer application may apply database migrations automatically. A rollback
can require restoring the pre-upgrade database as well as the previous image;
switching to the old image alone is not a reliable database rollback. Restore
only while the application is stopped and preserve the original encryption key.

## Troubleshooting

| Error or symptom | Fix |
| --- | --- |
| `COPY ... not found` | Build from the repo root containing `app/`. Use `COPY app/...`, not `COPY src/app/...`, and check the build context and ignore file. |
| `Invalid container name (speedtest-tracker-custom:local)` | Put the tag in `image:`; use `container_name: speedtest-tracker`. |
| `No such image: lscr.io/linuxserver/speedtest-tracker-custom:local` | Remove the registry prefix. The imported local tag is `speedtest-tracker-custom:local`. |
| Local image is missing | Build it on this host or load `speedtest-tracker-custom.tar`. Confirm with `docker image ls speedtest-tracker-custom`. |
| New empty database after upgrade | Check the Compose project identity and `/config` mount against the original deployment before continuing. |
| Chart changes disappear after recreation | Use the custom image; copying files into a running container alone is not persistent. |
| `git push -u origin main` fails because `main` is absent | This fork uses branch `1.x`; use `git push -u origin 1.x`. |
| Fresh-install login fails with `admin@admin` | Use `admin@example.com` and `password`, unless overridden at setup. Existing installations retain their existing accounts. |

## Add this documentation to the fork

Place this file at the repository root as `README.md`. After creating the optional
Docker deployment files described above, add only the files intended for Git:

```bash
git add README.md Dockerfile.custom Dockerfile.custom.dockerignore compose.custom.yaml
git commit -m "Document failed-test charts and custom Docker deployment"
git push -u origin 1.x
```

If only the README was created, use `git add README.md` instead. Do not include
`.env`, host data directories, SSL keys, image archives, or backup archives.

## Upstream and reference documentation

- [Upstream project](https://github.com/alexjustesen/speedtest-tracker)
- [Speedtest Tracker Docker Compose installation](https://docs.speedtest-tracker.dev/getting-started/installation/using-docker-compose)
- [Environment variables](https://docs.speedtest-tracker.dev/getting-started/environment-variables)
- [Authentication](https://docs.speedtest-tracker.dev/security/authentication)
- [LinuxServer Docker image source](https://github.com/linuxserver/docker-speedtest-tracker)
- [Docker build contexts and ignore files](https://docs.docker.com/build/concepts/context/)
- [Docker image save](https://docs.docker.com/reference/cli/docker/image/save/)
- [Docker image load](https://docs.docker.com/reference/cli/docker/image/load/)

Speedtest Tracker and this fork retain the upstream [MIT license](LICENSE.md).

