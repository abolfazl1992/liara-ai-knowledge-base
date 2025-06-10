#Liara

#### **1. Core Platform (PaaS) Fundamentals**

*   **`File System`**:
    *   **Default State**: `ReadOnly`. Improves security. Code changes are made locally and redeployed.
    *   **Writable Exception**: `/tmp` directory. 100MB temporary storage. Used for logs, temp files.
    *   **Persistent Storage**: Achieved via **Disks**. The `/tmp` directory can be mounted to a larger disk for more space.
    *   **Making Writable**: Possible via Console settings. **Warning**: All changes are ephemeral and will be lost on restart/redeploy. The writable space is 0.5% of the plan's total disk space.
    *   **Special Platforms**: `Docker` and `NextJS` platforms default to a writable file system.

*   **`Networking`**:
    *   **Private Network**: Enables secure, high-speed, and protocol-agnostic communication between apps and databases within the same account/team.
        *   **Connection Method**: `http://<app-name>:<port>`. Use the app's unique ID as the hostname.
        *   **Isolation**: Apps in different private networks cannot communicate.
    *   **Static IP**: Shared static IP available for whitelisting on external services (e.g., payment gateways). Enabled via App Settings. **Note**: This IP is for outgoing requests, not for direct access to the app.

*   **`Deployment Lifecycle & Build Process`**:
    *   **Zero-Downtime Deployment**: Default for all apps. Liara starts the new version, waits for it to become healthy, then switches traffic before stopping the old version.
    *   **Health Check**: Essential for zero-downtime deployment. Configured in `liara.json`.
        *   **Mechanism**: Liara runs a specified command at intervals. Success (exit code 0) means the deployment is healthy.
        *   **Failure Action**: If a new deployment fails health checks, traffic is not switched. If a running app fails checks, it is automatically restarted.
    *   **Build Location**: `iran` (default, faster push) vs. `germany` (no package download restrictions). Configurable via `liara.json` or CLI.
    *   **Cache**: Build cache is enabled by default to speed up subsequent deployments. Can be disabled (`--no-cache`).

*   **`Environment Variables (Envs)`**:
    *   **Purpose**: Configure app for different environments (dev, production). Used for secrets (DB credentials, API keys).
    *   **Management**:
        *   **Console**: UI for adding, editing, deleting. Supports bulk copy-paste and `.env` file upload.
        *   **CLI**: `liara env set`, `liara env ls`, `liara env unset`.
    *   **Access in Code**:
        *   Node.js: `process.env.VAR_NAME`
        *   Python: `os.getenv('VAR_NAME')`
        *   PHP: `getenv('VAR_NAME')`
        *   Go: `os.Getenv("VAR_NAME")`
    *   **Special Envs**: `LIARA_URL`, `PORT`.

---

#### **2. `liara.json` - The Core Configuration File**

The central point for defining deployment settings. Overrides console settings and CLI flags.

*   **Global Fields**:
    *   `"platform"`: (string) Specifies the app platform. e.g., `"node"`, `"docker"`, `"laravel"`. **Note**: Optional, as Liara can detect it, but recommended for clarity. Not used in GitHub deployments.
    *   `"app"`: (string) The app's unique ID. Pre-selects the app for `liara deploy`. Not used in GitHub deployments.
    *   `"port"`: (number) The port your application listens on. e.g., `3000`, `8080`.
    *   `"team-id"`: (string) Deploys the project to a specific team account.

*   **`build` Block**:
    *   `"location"`: (string) `"iran"` (default) or `"germany"`.
    *   `"cache"`: (boolean) `true` (default) or `false`.
    *   **Docker-specific**:
        *   `"dockerfile"`: (string) Path to the Dockerfile if not at the root. e.g., `"path/to/Dockerfile"`.
        *   `"args"`: (array of strings) Build-time arguments for `Dockerfile`. e.g., `["APP_VERSION=2.0.0"]`.

*   **`disks` Array**:
    *   Mounts persistent storage. Each object has:
        *   `"name"`: (string) The name of the disk created in the Liara console.
        *   `"mountTo"`: (string) The path inside the container to mount the disk. e.g., `"uploads"`, `"/var/www/html/storage"`.

*   **`healthCheck` Object**:
    *   `"command"`: (string) The command to run. Must exit with 0 on success. e.g., `"CMD curl --fail http://localhost:3000 || exit 1"`.
    *   `"interval"`: (number) Seconds between checks.
    *   `"timeout"`: (number) Seconds to wait for the command to complete.
    *   `"retries"`: (number) Number of retries on failure.
    *   `"startPeriod"`: (number) Seconds to wait after app start before the first check.

*   **Platform-Specific Blocks** (e.g., `"node"`, `"laravel"`, `"docker"`):
    *   `"version"` / `"phpVersion"` / `"pythonVersion"` / `"nodeVersion"`: Specifies the runtime version. e.g., `"22"`, `"8.3"`, `"3.12"`.
    *   `"timezone"`: Sets the container timezone. e.g., `"Asia/Tehran"`, `"UTC"`.
    *   `"mirror"` / `"composerMirror"`: (boolean) `true` (default) or `false`. Toggles usage of Liara's package mirror.
    *   `"args"` (Docker): (array of strings) Overrides the image's `ENTRYPOINT`.
    *   **Full Example Structure**:
        ```json
        {
          "platform": "docker",
          "app": "my-deno-app",
          "port": 8000,
          "build": {
            "location": "germany",
            "dockerfile": "path/to/Dockerfile"
          },
          "disks": [
            { "name": "my-data-disk", "mountTo": "/app/data" }
          ],
          "healthCheck": {
            "command": "CMD curl --fail http://localhost:8000/health || exit 1",
            "interval": 30,
            "timeout": 10,
            "retries": 3
          }
        }
        ```

---

#### **3. Core CLI Commands (`liara`)**

*   **`liara login`**: Authenticates the CLI. Can use browser (default) or interactive (`-i`).
*   **`liara deploy`**: The main deployment command.
    *   `--platform <value>`: e.g., `docker`, `node`.
    *   `--port <value>`: e.g., `8000`.
    *   `--app <id>`: Specifies the target app.
    *   `--path <path>`: Specifies the project directory.
    *   `--disks <name>:<mountTo>`: Mounts a disk. e.g., `--disks data:/app/data`.
    *   `--image <name>`: Deploys a public Docker image directly.
    *   `--no-cache`: Disables build cache for a fresh build.
    *   `--team-id <id>`: Deploys to a team.

*   **App Management**:
    *   `liara app:create` (or `liara create`): Creates a new application.
    *   `liara app:list` (or `liara ls`): Lists all applications.
    *   `liara restart <app-id>`: Restarts an app.
    *   `liara stop <app-id>`: Stops an app.
    *   `liara delete <app-id>`: Deletes an app.

*   **Service Management**:
    *   `liara shell <app-id> [/bin/sh]`: Connects to the running app's container. **Crucial for debugging.** Use `/bin/sh` for Alpine-based images.
    *   `liara env set --app <id> KEY=VALUE`: Sets environment variables.
    *   `liara env ls --app <id>`: Lists environment variables.
    *   `liara logs [-f] <app-id>`: Shows app logs. `-f` for real-time streaming.
    *   `liara disk:create`: Creates a new persistent disk.

---

#### **4. Database as a Service (DBaaS) - Core Concepts**

*   **Provisioning**: Create managed databases via Console or `liara db create`.
*   **Plans**:
    *   **Hardware Plans**: Tiers based on CPU/RAM/Disk (e.g., Mercury (Free), Venus, Earth, Mars...).
    *   **Software Plans**: Tiers based on features (Base, Bronze, Silver, Gold). Affects backups, monitoring retention, public network access. **Key Point**: Free plans often lack public network access.
*   **Networking**:
    *   **Private Network**: The recommended connection method for apps within the same Liara network. Faster and more secure. Connection details use internal hostnames (e.g., `postgres-db:5432`).
    *   **Public Network**: Required for connecting from local machine or external services. Can be enabled/disabled in settings.
*   **Connection Info**:
    *   Found in the "How to Connect" (نحوه اتصال) tab of the database dashboard.
    *   Provides both Public and Private network connection details (URIs, host, port, user, etc.).
*   **Connection Pooling**: A mechanism to manage and reuse database connections efficiently, reducing overhead. Liara's documentation provides examples for various languages/frameworks.
*   **Backups**: Automated daily/weekly/monthly backups (depending on software plan). Manual backups are also possible.

This first response covers the absolute essentials: the foundational concepts of the PaaS, the critical `liara.json` file for configuration-as-code, the most important CLI commands for interaction, and the basic principles of the managed database service. This knowledge is the prerequisite for successfully deploying and managing any application, including the Deno project.

Of course. I will continue summarizing the provided documentation, focusing on the technical details and prioritizing the information most relevant to your project.

Here is the second part of the knowledge base.



#### **5. PaaS Deep Dive: Docker Platform**

This is the most flexible platform, allowing deployment of any application that can be containerized.

*   **Core Deployment Methods**:
    1.  **From Source (Dockerfile)**:
        *   **Process**: Liara detects `Dockerfile` at the project root, builds the image, and deploys the container.
        *   **`liara.json` Configuration**:
            *   `"platform": "docker"`
            *   `"port"`: **Required**. The port your application `EXPOSE`s inside the container.
            *   `"build"` (object):
                *   `"dockerfile"`: (string) Relative path to your `Dockerfile` if not at the root.
                *   `"args"`: (array of strings) Build-time variables (`--build-arg`). Example: `["APP_VERSION=1.2.3"]`.
                *   `"cache"`: (boolean) `false` to disable layer caching during build.
            *   `"args"` (top-level array): Overrides the Docker image's `ENTRYPOINT` or `CMD`. Example: `["sh", "-c", "sleep 10 && /entrypoint.sh"]`.
    2.  **From Public Registry (DockerHub, GHCR, etc.)**:
        *   **Process**: Pulls a pre-built public image instead of building from source.
        *   **CLI Flag**: `liara deploy --image <IMAGE_NAME>:<TAG>`
        *   **`liara.json` Field**: `"image": "<IMAGE_NAME>:<TAG>"`.
        *   **Example**: `{"app": "meili-search", "platform": "docker", "port": 7700, "image": "getmeili/meilisearch:v1.7"}`

*   **Persistent Storage (Disks vs. Volumes)**:
    *   Docker `VOLUME` instruction is **not** directly used for persistent storage on Liara.
    *   **Solution**: Use Liara's **Disks**.
    *   **Mechanism**:
        1.  Create a disk via Console or `liara disk:create`.
        2.  Mount it to an **absolute path** inside the container.
    *   **`liara.json` Example**:
        ```json
        {
          "disks": [
            { "name": "app-data-disk", "mountTo": "/app/data" },
            { "name": "config-disk", "mountTo": "/etc/my-app/config" }
          ]
        }
        ```
    *   **CLI Flag**: `liara deploy --disks data:/app/data`

*   **Advanced Patterns**:
    *   **Cron Jobs with Supercronic**:
        1.  In `Dockerfile`, copy the `supercronic` binary: `COPY --from=liaracloud/supercronic:v0.1.11 /usr/local/bin/supercronic /usr/local/bin/supercronic`.
        2.  Create a `crontab` file with your job schedule: `* * * * * cd $ROOT && echo "job running" >> /tmp/log.txt`.
        3.  Use an `entrypoint.sh` to start both services:
            ```sh
            #!/bin/bash
            supercronic /path/to/crontab &
            exec /path/to/your/main/app
            ```
    *   **Nginx as a Reverse Proxy**:
        1.  Create a Docker app for Nginx.
        2.  In `nginx.conf`, use Liara's internal DNS resolver to proxy requests to other apps in the same private network.
        3.  **Key `nginx.conf` directives**:
            ```nginx
            resolver 127.0.0.11 valid=5s;

            upstream backend_app {
              server <target-app-name>:<target-app-port> resolve;
            }

            server {
              location / {
                proxy_pass http://backend_app;
              }
            }
            ```

*   **Deno Specific Deployment (`related-apps/deno.mdx`)**:
    *   `Dockerfile` skeleton:
        ```dockerfile
        FROM denoland/deno:alpine-1.30.0
        WORKDIR /app
        COPY . .
        RUN deno cache server.ts
        CMD ["deno", "run", "--allow-net", "server.ts"]
        ```
    *   Port is specified in the `liara deploy` command or `liara.json`.

---

#### **6. DBaaS Deep Dive: PostgreSQL**

*   **Quick Setup**:
    *   Create via Console or `liara db:create --type postgresql`.
    *   Specify version (e.g., 16.3, 15.4).
    *   A default database `postgres` and user `root` are created.

*   **Connection**:
    *   **Tools**:
        *   CLI: `psql` - Direct connection string is provided in the dashboard.
        *   GUI: `pgAdmin` (Web-based, enabled with one click), `DBeaver`.
    *   **Connection Strings**:
        *   `postgresql://<user>:<password>@<host>:<port>/<database>`
        *   Always prefer the **private network URI** for connections from other Liara apps.
    *   **Code Example (Go)**:
        *   Package: `github.com/lib/pq`
        *   Connection: `sql.Open("postgres", os.Getenv("DB_URI"))`

*   **Backup & Restore**:
    *   **Backup**: Automated daily backups. Manual backups via Console (`.dump` format).
    *   **Restore**:
        *   CLI Tool: `pg_restore`. Example command: `pg_restore -h DB_HOST -p DB_PORT -U DB_USERNAME -d postgres /path/to/backup.dump`
        *   GUI Tool: `pgAdmin` has a restore feature. **Warning**: Not suitable for large backups.

*   **User Management (via `psql`)**:
    *   `CREATE USER readonly_user WITH PASSWORD 'secret';`
    *   `GRANT CONNECT ON DATABASE my_db TO readonly_user;`
    *   `GRANT USAGE ON SCHEMA public TO readonly_user;`
    *   `GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;`
    *   `REVOKE SELECT ON ... FROM ...;`
    *   `DROP ROLE readonly_user;`
    *   `\du`: List all users and roles.

---

#### **7. Object Storage (S3-Compatible) Service**

*   **Core Concept**: S3-compatible storage. Any AWS S3 SDK or S3-compatible tool can be used.

*   **Terminology**:
    *   **Bucket**: A container for objects. Names must be globally unique.
    *   **Object**: A file and its metadata.
    *   **Endpoint**: The base URL for the storage service (e.g., `https://storage.iran.liara.space`).
    *   **Access Key & Secret Key**: Credentials for authentication. **Critical**: Secret Key is only shown once upon creation.

*   **Bucket Management**:
    *   **Creation**: `liara bucket create` or via Console.
    *   **Access Level**:
        *   `private` (default): Objects are not publicly accessible. Access via **pre-signed URLs**.
        *   `public-read`: Objects are accessible via a permanent, public URL (`<endpoint>/<bucket-name>/<object-key>`).
    *   **Custom Domain**: A CNAME record can point a custom domain/subdomain to the bucket.

*   **SDK Usage (General Pattern)**:
    1.  **Installation**: Install the official AWS SDK for your language (e.g., `npm install @aws-sdk/client-s3`, `pip install boto3`).
    2.  **Client Initialization**:
        ```javascript
        const s3Client = new S3Client({
          endpoint: "https://storage.iran.liara.space",
          region: "us-east-1", // This is arbitrary but required
          credentials: {
            accessKeyId: process.env.LIARA_ACCESS_KEY,
            secretAccessKey: process.env.LIARA_SECRET_KEY,
          },
        });
        ```
    3.  **Common Commands**:
        *   **Upload**: `PutObjectCommand`
        *   **List**: `ListObjectsV2Command`
        *   **Delete**: `DeleteObjectCommand`
        *   **Generate Temporary URL**: `getSignedUrl` with `GetObjectCommand`.

*   **Third-Party Tools**:
    *   **`rclone`**: Powerful CLI tool for syncing, copying, and moving files between local storage and S3/other cloud providers.
    *   **`S3 Browser`**: Windows GUI client for managing S3 buckets.

*   **Use Case: Direct Download**: To force a browser to download a file instead of displaying it, set the `Content-Disposition: 'attachment'` header during the upload (`PutObjectCommand`).

---

This completes the second summary, focusing on the practical application of core services. The next part will delve into specific platform runtimes like NodeJS, PHP, and Python, exploring their unique configurations and common use cases on Liara.

Of course. Here is the third part of the technical knowledge base, continuing with the same structured, keyword-focused format.

---



#### **8. PaaS Deep Dive: NodeJS Platform**

*   **Core Concepts**:
    *   **Detection**: Liara detects a NodeJS project by the presence of a `package.json` file.
    *   **Build Process**:
        1.  `npm install`: Installs dependencies from both `dependencies` and `devDependencies`.
        2.  `npm run build`: Executes the `build` script if it exists in `package.json`.
        3.  `npm run start`: Executes the `start` script to run the application. **This script is mandatory.**
    *   **Default Port**: Assumes port `3000` if not specified.

*   **Configuration (`liara.json`)**:
    *   `"platform": "node"`
    *   `"node"` (object):
        *   `"version"`: (string) Set Node.js version (e.g., `"18"`, `"20"`, `"22"` (default)).
        *   `"mirror"`: (boolean) `false` to disable Liara's npm mirror.
        *   `"timezone"`: (string) e.g., `"UTC"`.

*   **Framework Deployment Examples**:
    *   **`Express.js`**: Standard setup. Define `start` script: `"start": "node server.js"`.
    *   **`NestJS`**:
        *   `"build"` script is essential: `"build": "nest build"`.
        *   `"start"` script should run the compiled output: `"start": "node dist/main"`.
        *   Main application must listen on `0.0.0.0`: `app.listen(3000, '0.0.0.0');`.
    *   **`AdonisJS`**:
        *   Requires specific environment variables: `HOST=0.0.0.0`, `PORT=3000`, `NODE_ENV=production`, and `APP_KEY`.
        *   `"start"` script: `"start": "node server.js"`.
    *   **`Hono`**:
        *   Use `@hono/node-server` adapter.
        *   Main file: `serve({ fetch: app.fetch, port: PORT })`.
    *   **`SvelteKit`**:
        *   Requires `@sveltejs/adapter-node`.
        *   `svelte.config.js`: `kit: { adapter: adapter() }`.
        *   `"start"` script: `"start": "node build/index.js"`.

*   **Common Errors & Fixes**:
    *   **CORS**: Use the `cors` npm package: `app.use(cors())`. Specific frameworks like AdonisJS and NestJS have built-in CORS configuration.
    *   **Trusted Proxies**: Since all traffic comes through Liara's reverse proxy, the application needs to trust these proxies to get the real user IP.
        *   **Express**: `app.set('trust proxy', 1)`
        *   **Koa**: `app.proxy = true`
        *   **Hapi**: Requires a middleware to read `x-forwarded-for` header.

---

#### **9. PaaS Deep Dive: PHP Platform**

*   **Core Concepts**:
    *   **Web Server**: Apache.
    *   **Detection**: Liara detects a PHP project if no other platform is recognized and a `.php` file exists.
    *   **Dependency Management**: If `composer.json` is present, Liara runs `composer install`.
        *   **Important**: It's recommended to remove `composer.lock` before deploying to avoid conflicts.

*   **Configuration (`liara.json`)**:
    *   `"platform": "php"`
    *   `"php"` (object):
        *   `"version"`: (string) e.g., `"8.2"`, `"8.3"`.
        *   `"composerMirror"`: (boolean) `false` to disable Liara's Composer mirror.
        *   `"timezone"`: (string) e.g., `"Asia/Tehran"`.

*   **Custom Configurations**:
    *   **`liara_php.ini`**: Place at the root to override default `php.ini` settings. Example:
        ```ini
        file_uploads = On
        upload_max_filesize = 64M
        post_max_size = 128M
        ```
    *   **`.htaccess`**: Supported for custom Apache rewrite rules and headers.
        ```apache
        RewriteEngine on
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteCond %{REQUEST_FILENAME}\.php -f
        RewriteRule ^(.*)$ $1.php [NC,L]
        ```
    *   **`liara_nginx.conf`** (for Laravel): For advanced Nginx configuration.

*   **Framework Deployment**:
    *   **`Laravel`**:
        *   Recognized as its own platform (`"platform": "laravel"`).
        *   Automatically runs `php artisan storage:link`.
        *   `liara.json` `"laravel"` block supports:
            *   `"configCache": true` -> `php artisan config:cache`
            *   `"routeCache": true` -> `php artisan route:cache`
            *   `"buildAssets": false` -> Disables `npm run production`.
            *   `"ssr": true` -> Enables Inertia.js SSR support.
    *   **`Yii`**:
        *   Requires `APACHE_DOCUMENT_ROOT=web/` environment variable.
        *   The `web/assets` directory often needs to be mounted to a persistent disk.

*   **Hooks (`liara_pre_build.sh`, `liara_post_build.sh`, `liara_pre_start.sh`)**:
    *   **`liara_pre_build.sh`**: Runs before `composer install`. Ideal for installing system dependencies (`apt-get install ...`).
    *   **`liara_pre_start.sh`**: Runs before the app starts. Ideal for database migrations: `php artisan migrate --force`.

---

#### **10. PaaS Deep Dive: Python Platform**

*   **Core Concepts**:
    *   **Detection**: Presence of `requirements.txt` or a `.py` file.
    *   **Dependency Management**: Liara runs `pip install -r requirements.txt`.
    *   **Web Server**: Gunicorn is used by default for web applications.

*   **Configuration (`liara.json`)**:
    *   `"platform": "python"` (or `"django"`, `"flask"`)
    *   `"python"` (object):
        *   `"version"`: (string) e.g., `"3.11"` (default), `"3.12"`.
        *   `"mirror"`: (boolean) `false` to disable Liara's PyPI mirror.
        *   `"timezone"`: (string) e.g., `"UTC"`.

*   **Framework Deployment**:
    *   **`Django`** (`"platform": "django"`):
        *   Liara automatically detects `wsgi.py` or `asgi.py`.
        *   Runs `python manage.py collectstatic` by default. Can be disabled in `liara.json`: `"collectStatic": false`.
        *   **Gunicorn**: Can be configured with environment variables:
            *   `GUNICORN_WORKERS`: Number of worker processes. Default: `3`.
            *   `GUNICORN_TIMEOUT`: Worker timeout in seconds. Default: `30`.
            *   `GUNICORN_MAX_REQUESTS`: Restarts a worker after a certain number of requests. Default: `10000`.
    *   **`Flask`** (`"platform": "flask"`):
        *   Looks for an application object named `app` in a file named `app.py` or `wsgi.py` by default.
        *   Can be configured in `liara.json`: `"flask": { "appModule": "server:my_flask_app" }`.

*   **SQLite Usage**:
    1.  Create a disk in Liara Console (e.g., named `database-disk`).
    2.  Mount the disk to a directory in your project (e.g., `db`) via `liara.json`:
        ```json
        {
          "disks": [{ "name": "database-disk", "mountTo": "db" }]
        }
        ```
    3.  In your code, reference the database file inside that directory: `DATABASE = 'db/database.sqlite'`

*   **Hooks**:
    *   `liara_pre_start.sh`: Perfect for running migrations.
        ```sh
        #!/bin/bash
        python manage.py migrate
        ```

This concludes the third part of the summary. The knowledge is becoming more specific, detailing how to deploy common application types and solve platform-specific problems like Gunicorn timeouts or SQLite persistence. The next section will cover frontend frameworks and static sites.

Of course. Here is the fourth part of the technical knowledge base, following the same structured and keyword-heavy format.

---



#### **11. PaaS Deep Dive: Frontend & Static Platforms**

Liara offers optimized platforms for modern frontend frameworks, which are built into static assets and served by Nginx.

*   **`Static` Platform**:
    *   **Purpose**: For deploying pure HTML, CSS, and JavaScript projects, or the build output of any Static Site Generator (SSG).
    *   **Deployment**: Deploy the contents of the final build directory (e.g., `dist`, `build`, `public`).
        *   `liara deploy --platform=static --path=./dist`
    *   **Nginx Configuration**:
        *   Default serves `index.html`.
        *   Handles Single Page Application (SPA) routing with `try_files $uri $uri/ /index.html =404;`.
        *   Customizable via a `liara_nginx.conf` file at the root of the deployed directory.
    *   **SSG Examples**:
        *   **Hugo**: Run `hugo`, then deploy the `public` directory.
        *   **Jekyll**: Run `jekyll build`, then deploy the `_site` directory.
        *   **Eleventy**: Run `npx @11ty/eleventy`, then deploy the `_site` directory.
        *   **Gatsby**: Run `npm run build`, then deploy the `public` directory.
        *   **Astro**: Run `npm run build`, then deploy the `dist` directory.

*   **`React` Platform**:
    *   **Detection**: Based on `create-react-app` or `Vite` (with React plugin) structure.
    *   **Build Process**:
        1.  `npm install`
        2.  `npm run build`
        3.  The resulting `build` (or `dist` for Vite) directory is served by Nginx.
    *   **Configuration (`liara.json`)**:
        *   `"platform": "react"`
        *   `"react"` (object):
            *   `"sourceMap"`: (boolean) `false` by default to not expose source maps in production.
            *   `"mirror"`: (boolean) Toggle Liara's npm mirror.
    *   **Vite Specifics**: Must configure `vite.config.js` to output to `build`: `build: { outDir: 'build' }`.

*   **`Vue.js` Platform**:
    *   **Detection**: Based on `vue-cli` or `Vite` (with Vue plugin) structure.
    *   **Build Process**: Similar to React: `npm install` -> `npm run build` -> serve `dist` with Nginx.
    *   **Configuration (`liara.json`)**:
        *   `"platform": "vue"`
        *   `"vue"` (object): `"mirror": false`.

*   **`Next.js` Platform**:
    *   **This is a NodeJS platform, not a static one.** It runs a Node.js server.
    *   **Detection**: Standard Next.js project structure.
    *   **Build Process**:
        1.  `npm install`
        2.  `npm run build`
        3.  `npm run start` (Runs `next start`).
    *   **Key Consideration: `output: 'standalone'`**: Liara automatically handles this for optimal deployment. If you encounter `Could not modify config file` error, you must add it manually to `next.config.js`.
    *   **ISR (Incremental Static Regeneration)**:
        *   To persist ISR cache across deployments, you need a **Disk**.
        *   Mount the disk to the cache path:
            *   Pages Router: `/.next/server/pages`
            *   App Router: `/.next/server/app`
        *   This prevents the cache from being wiped on every new deployment.
    *   **Static HTML Export (`output: 'export'`)**: If you use this feature, you should deploy the output (`out` directory) to the **`Static`** platform, not the Next.js platform.

---

#### **12. Domain Management**

*   **Default Subdomain**: Every app gets a free `https://<app-id>.liara.run` URL. Can be disabled in app settings to prevent duplicate content issues for SEO.
*   **Custom Domains**:
    1.  **Add Domain**: In the app's "Domains" tab, add your custom domain (e.g., `example.com`).
    2.  **Configure DNS**: Liara provides DNS records (usually `A` or `CNAME`) that you must configure at your domain registrar (e.g., Cloudflare, GoDaddy).
    3.  **Check Status**: Liara's dashboard verifies that the DNS records are correctly pointing to its servers.
*   **SSL Certificates**:
    *   **Automatic & Free**: Liara provides free SSL certificates for all custom domains.
    *   **Activation**: One-click activation from the domain management page.
    *   **CDN Interaction**: If using a CDN like Cloudflare, set SSL mode to **"Flexible"**. For Liara to issue a certificate, you might need to temporarily disable the CDN proxy (grey-cloud it), activate SSL in Liara, and then re-enable the CDN proxy (orange-cloud).
*   **Subdomains**:
    *   `www`: Can be added easily and will redirect to the root domain by default.
    *   Other subdomains (e.g., `api.example.com`): Added just like a root domain.
*   **Liara DNS Service**: You can delegate your domain's nameservers to Liara (`ns1.liara.ir`, `ns2.liara.ir`). This allows Liara to manage all DNS records for you automatically, simplifying the process.

---

#### **13. CI/CD Integration**

Automate deployments from your Git repositories.

*   **Core Principle**: Use a CI/CD provider (like GitHub Actions or GitLab CI) to run the `liara deploy` command.

*   **Authentication**:
    *   **API Token**: The primary method for authenticating in a CI/CD environment.
    *   **Generation**: Create a token from the "API" section of your Liara account settings.
    *   **Security**: Store the token as a **secret** in your CI/CD provider's settings (e.g., GitHub Secrets, GitLab CI/CD Variables). **Never hardcode it in your CI file.**

*   **`GitHub Actions` Workflow (`.github/workflows/liara.yaml`)**:
    ```yaml
    name: Liara CD
    on:
      push:
        branches: [ main ] # Trigger on push to the main branch
    jobs:
      deploy:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v3
          - uses: actions/setup-node@v3 # To get npm
            with:
              node-version: '20'
          - name: Install Liara CLI and Deploy
            env:
              LIARA_API_TOKEN: ${{ secrets.LIARA_API_TOKEN }}
            run: |
              npm i -g @liara/cli
              liara deploy --app="my-app" --api-token="$LIARA_API_TOKEN" --detach
    ```
    *   `--detach` or `--no-app-logs`: Recommended to prevent the CI job from hanging while tailing logs.

*   **`GitLab CI` Workflow (`.gitlab-ci.yml`)**:
    ```yaml
    image: node:20-alpine
    stages:
      - deploy
    deploy_production:
      stage: deploy
      only:
        - master
      script:
        - npm i -g @liara/cli
        - export http_proxy=http://proxy.liara.ir:6666 # IMPORTANT: Proxy for GitLab runners
        - liara deploy --app="my-app" --api-token="$LIARA_TOKEN" --no-app-logs
    ```
    *   **`$LIARA_TOKEN`**: This is a CI/CD variable you must set in your GitLab project's settings.
    *   **Proxy**: GitLab's shared runners are often outside Iran and may be blocked. The provided proxy is necessary for them to reach Liara's API. This is not needed for self-hosted runners.

*   **Multi-Branch Deployment**: Use conditional logic in your CI file to deploy different branches to different Liara apps (e.g., `main` branch to `my-prod-app`, `develop` branch to `my-staging-app`).

This section covers the final steps of getting an application live and maintainable: serving it to users via custom domains and automating its deployment process. This knowledge builds upon the core concepts to form a complete picture of the development and deployment lifecycle on Liara.


Of course. Here is the fifth part of the technical knowledge base. This section focuses on the `references` directory, which contains details about the CLI commands and the Console UI, providing a practical guide for interacting with and managing the Liara platform.

---



#### **14. CLI Reference: Account & Authentication (`liara account` & `liara login`)**

*   **`liara login`**:
    *   **Primary Action**: Authenticates the CLI with your Liara account.
    *   **Default Method**: Opens a web browser for authentication.
    *   **Flags**:
        *   `-i, --interactive`: Login using email/password directly in the terminal (no browser).
        *   `-e, --email <value>`: Pre-fills the email address.
        *   `-p, --password <value>`: Pre-fills the password.
        *   `--api-token <value>`: **(CI/CD)** Logs in non-interactively using a token.
        *   `--account <name>`: Specifies a named account context to use for this command.

*   **`liara account:add`**: Adds another Liara account to the CLI config, allowing easy switching.
    *   `-a, --account <name>`: Assigns a short, memorable name to the account (e.g., `work`, `personal`).

*   **`liara account:list` (or `ls`)**: Lists all configured accounts.

*   **`liara account:use`**: Sets the default account for all subsequent commands.

*   **`liara account:remove` (or `rm`)**: Removes a configured account from the CLI.

---

#### **15. CLI Reference: Application Management (`liara app` or command alias)**

*   **`liara create` / `liara app:create`**:
    *   **Action**: Creates a new application. Prompts for name, platform, plan, etc., if not provided via flags.
    *   **Key Flags**:
        *   `-a, --app <name>`: Sets the app's unique ID.
        *   `-P, --platform <type>`: Sets the platform (e.g., `node`, `docker`, `laravel`).
        *   `--plan <plan-id>`: Sets the hardware plan (e.g., `free`, `small-g2`).
        *   `--feature-plan <plan-id>`: Sets the software plan (e.g., `standard`, `pro`).
        *   `-n, --network <name>`: Assigns the app to a specific private network.
        *   `-r, --read-only <true|false>`: Sets the file system state.

*   **`liara deploy`**:
    *   **Action**: Deploys the project from the current directory.
    *   **Key Flags**:
        *   `--path <path>`: Specifies the project root directory.
        *   `--port <port>`: The port the application listens on.
        *   `-i, --image <name:tag>`: Deploys a public Docker image.
        *   `-m, --message <string>`: Adds a descriptive message to the deployment.
        *   `--disks <disk-name>:<mount-path>`: Mounts a disk. Can be used multiple times.
        *   `--no-cache`: Forces a build without using the Docker layer cache.
        *   `--detach`: Exits the command immediately after starting the deployment, without streaming logs.

*   **`liara app:list` / `liara ls`**:
    *   **Action**: Lists all applications in the current account/team.
    *   **Key Flags**:
        *   `-x, --extended`: Shows more columns of information.
        *   `--columns <c1,c2,...>`: Shows only specified columns.
        *   `--output <format>`: Output in `json`, `yaml`, or `csv`.
        *   `--sort <column>`: Sorts the output. Prepend `-` for descending order (e.g., `--sort=-plan`).

*   **`liara logs <app-id>`**:
    *   **Action**: Streams application logs.
    *   **Key Flags**:
        *   `-f, --follow`: Streams logs in real-time.
        *   `-s, --since <duration>`: Shows logs from a specific time (e.g., `"1h"`, `"10m"`).
        *   `-c, --colorize`: Adds color to the output.

*   **`liara shell <app-id> [shell_path]`**:
    *   **Action**: Opens an interactive shell inside the running application container.
    *   **Argument `[shell_path]`**: The shell to execute. **Crucial**: Use `/bin/sh` for Alpine-based images. Default is `/bin/bash`.
    *   **Flag `-c, --command <cmd>`**: Executes a non-interactive command inside the container.

---

#### **16. CLI Reference: Database & Service Management (`liara db`, `liara mail`, etc.)**

*   **`liara db create`**: Creates a managed database.
    *   `-t, --type <db-type>`: e.g., `mysql`, `postgresql`, `mongodb`.
    *   `-v, --version <version>`: e.g., `"16.1"` for PostgreSQL.
    *   `-n, --name <db-id>`: Unique ID for the database.
    *   `--plan <plan-id>`: Hardware plan.
    *   `--public-network`: Enables public network access on creation.

*   **`liara db list` / `ls`**: Lists all databases.

*   **`liara db delete` / `rm`**: Deletes a database.
    *   `-y, --yes`: Skips the confirmation prompt.

*   **`liara db start|stop|resize`**: Manages the state and plan of a database.

*   **`liara bucket create|delete|list`**: Manages Object Storage buckets.

*   **`liara mail create|delete|list`**: Manages Mail Servers.

*   **`liara zone create|delete|list|check|get`**: Manages DNS zones.

---

#### **17. Console UI (console.liara.ir) - Key Sections**

A visual representation of the CLI's capabilities.

*   **Dashboard/Homepage**: Overview of all applications, databases, and services.
*   **Application View**:
    *   **Overview**: App status, URL, plan details, restart/stop buttons.
    *   **Events (رویدادها)**: A log of all platform-level actions on the app (e.g., `start`, `stop`, `deploy`, `crash`, `RAM limit exceeded`). **Crucial for diagnosing deployment failures and crashes.**
    *   **Logs (لاگ‌ها)**: The application's `stdout` and `stderr`. Searchable and filterable by time.
    *   **Deploy (استقرار)**:
        *   Methods: Drag-and-drop source code (ZIP), deploy from DockerHub, or connect to a GitHub repository.
        *   Configuration: Set platform-specific options (Node version, build settings), port, and mount disks during deployment.
    *   **Disks (دیسک‌ها)**: Create, delete, and manage persistent disks. Create FTP access credentials here.
    *   **Domains (دامنه‌ها)**: Add custom domains, manage `www` subdomains, and activate SSL.
    *   **Settings (تنظیمات)**:
        *   **Environment Variables (متغیرها)**: GUI for managing env vars. Includes bulk edit and `.env` upload.
        *   **File System**: Toggle `ReadOnly` mode.
        *   **Static IP**: Enable/disable the static IP feature.
        *   **Change Plan**: Upgrade/downgrade app plan.

*   **Database View**:
    *   **How to Connect (نحوه اتصال)**: **Most important tab**. Provides connection strings (URIs) and individual credentials for both **private** and **public** networks.
    *   **Backups (پشتیبان‌گیری)**: View automated backups and create manual ones.
    *   **Settings**: Enable/disable public access, change plan.
    *   **One-Click Panels**: For databases like MySQL and PostgreSQL, provides a one-click button to open `phpMyAdmin` or `pgAdmin`.

This section provides the practical "how-to" knowledge for interacting with Liara. Understanding the CLI is vital for automation and CI/CD, while familiarity with the console is key for quick checks, debugging via logs/events, and managing services that are less frequently changed (like domains and disks).

Of course. Here is the sixth part of the technical knowledge base. This section transitions from the "how-to" of platform and service management to the specifics of individual "One-Click Apps", which are pre-configured Docker images for popular open-source software.

---


#### **18. One-Click Apps: Core Concepts**

*   **What they are**: Pre-configured, ready-to-deploy Docker images of popular open-source tools. Liara manages the underlying `liara.json` or Docker Compose logic for the user.
*   **Deployment Methods**:
    1.  **Quick Install**: Simplest method. You only provide an app name. Liara automatically provisions the app, a new private network, and a default plan.
    2.  **Advanced Install (via Liara Compose)**: The UI presents a `liara-compose.yaml` file. This allows you to:
        *   Customize the app ID and plan.
        *   Change the Docker image tag (version).
        *   Define environment variables.
        *   Mount disks.
        *   Deploy multiple interconnected services (e.g., WordPress + a MySQL database) in a single operation.
*   **Updating Versions**: There's no one-click "update" button. To change the version of a one-click app, you must redeploy it from its Docker Hub image with a new tag.
    *   **Method**: Go to the app's "Deploy" tab, select "Docker Hub", and enter the new image name and tag (e.g., `getmeili/meilisearch:v1.8`).
    *   **Persistence**: If the app uses a disk for data (like Meilisearch or N8N), the data will persist across version updates as long as the disk is re-attached.

---

#### **19. Key One-Click App: `Imgproxy`**

*   **Purpose**: A fast, secure, on-the-fly image processing server. Used for resizing, cropping, watermarking, and optimizing images without modifying the original.
*   **Liara Setup**:
    *   Deployed as a one-click app.
    *   Port: `80`.
*   **Core Usage**:
    *   Construct a URL in the format: `https://<imgproxy-app-url>/<signature>/<processing-options>/<encoded-source-image-url>`
    *   **`<processing-options>`**: e.g., `resize:fill:300:400:0/gravity:sm`
    *   **`<encoded-source-image-url>`**: The URL of the original image, Base64-encoded. Use `plain/` prefix for non-encoded URLs (less secure).
*   **Security Configuration (Environment Variables)**:
    *   `IMGPROXY_KEY` & `IMGPROXY_SALT`: A hex-encoded key/salt pair for generating secure URL signatures. **Highly recommended for production**.
        *   Generation: `echo $(xxd -g 2 -l 64 -p /dev/random)`
    *   `IMGPROXY_SECRET`: A simpler alternative for limiting access (acts like a master password).
*   **Integration with Object Storage**:
    *   Imgproxy can directly process images stored in an S3-compatible bucket.
    *   The `plain/` URL would be: `plain/https://<bucket-name>.<endpoint-host>/<image-key.jpg>`
    *   Example: `https://my-proxy.liara.run/_/resize:fill:300:400/plain/https://my-bucket.storage.iran.liara.space/cat.jpg`

---

#### **20. Key One-Click App: `Soketi`**

*   **Purpose**: An open-source, Pusher-compatible WebSocket server. Used for building real-time features like live chats, notifications, and collaborative tools.
*   **Liara Setup**:
    *   Deployed as a one-click app.
    *   Port: `6001`.
*   **Environment Variables (Auto-configured)**:
    *   `SOKETI_APP_ID`, `SOKETI_APP_KEY`, `SOKETI_APP_SECRET`: Credentials for your application to connect to the Soketi server.
*   **Client-Side Connection (JavaScript)**:
    *   Use `laravel-echo` and `pusher-js`.
    *   **Key Configuration**:
        ```javascript
        import Echo from 'laravel-echo';
        window.Pusher = require('pusher-js');

        window.Echo = new Echo({
          broadcaster: 'pusher',
          key: 'YOUR_SOKETI_APP_KEY', // From Soketi app envs
          wsHost: '<soketi-app-id>.liara.run',
          wsPort: 443, // Always use 443 for wss
          wssPort: 443, // Also 443
          forceTLS: true,
          encrypted: true,
          disableStats: true,
          enabledTransports: ['ws', 'wss'],
        });
        ```
    *   **`forceTLS: true`** is critical for connecting via `wss://`.

*   **Backend Integration (`Laravel`)**:
    *   Set `BROADCAST_DRIVER=pusher` in your Laravel app's `.env`.
    *   In `config/broadcasting.php`, configure the `pusher` connection to point to your Soketi app:
        ```php
        'pusher' => [
            'driver' => 'pusher',
            'key' => env('PUSHER_APP_KEY'), // Matches SOKETI_APP_KEY
            'secret' => env('PUSHER_APP_SECRET'),
            'app_id' => env('PUSHER_APP_ID'),
            'options' => [
                'host' => '<soketi-app-id>.liara.run',
                'port' => 443,
                'scheme' => 'https'
            ],
        ],
        ```

---

#### **21. Key One-Click App: `Meilisearch`**

*   **Purpose**: A fast, open-source search engine. An alternative to Elasticsearch or Algolia for full-text search capabilities.
*   **Liara Setup**:
    *   Deployed as a one-click app.
    *   Port: `7700`.
    *   **Requires a Disk**: A disk named `data` must be created and mounted to `/meili_data` to persist the search index. This is configured in the `liara.json` or `liara-compose.yaml`.
*   **Authentication**:
    *   `MEILI_MASTER_KEY`: An environment variable is automatically generated on first startup. This key is required for all API operations (creating indexes, adding documents, searching).
*   **Core API Usage (Python Example)**:
    *   Package: `meilisearch`
    *   Client Initialization: `client = meilisearch.Client('<your-meili-app-url>', '<your-master-key>')`
    *   **Indexing**:
        ```python
        index = client.index('movies')
        index.add_documents([{'id': 1, 'title': 'The Matrix'}])
        ```
    *   **Searching**:
        ```python
        result = index.search('matrix')
        ```

---

#### **22. Key One-Click App: `Headless Chrome`**

*   **Purpose**: Runs a version of Google Chrome without a graphical user interface. Used for automated testing, web scraping, and generating PDFs/screenshots of web pages.
*   **Liara Setup**:
    *   Deployed as a one-click app.
    *   Port: `3000`.
    *   `TOKEN`: An environment variable is auto-generated to secure access to the instance.
*   **Connection Methods**:
    *   **`Puppeteer` (Node.js)**:
        ```javascript
        const browser = await puppeteer.connect({
            browserWSEndpoint: `wss://<app-id>.liara.run?token=${process.env.TOKEN}`,
        });
        ```
    *   **`Playwright` (Node.js)**:
        ```javascript
        const browser = await playwright.chromium.connect({
            wsEndpoint: `wss://<app-id>.liara.run/playwright?token=${process.env.TOKEN}`,
        });
        ```
    *   **`Selenium` (Python/Node.js)**: Connect to the WebDriver endpoint.
        ```python
        driver = webdriver.Remote(
            command_executor='https://<app-id>.liara.run/webdriver',
            options=chrome_options
        )
        # The token is passed via a capability:
        # chrome_options.set_capability('browserless:token', '<your-token>')
        ```

This section covers the most relevant one-click apps that a project like SimpleDraft might eventually use to add advanced features, providing a solid foundation for building a more complex, microservice-oriented architecture on Liara.

Of course. Here is the seventh part of the technical knowledge base, continuing the deep dive into platform specifics and advanced configuration options.

---



#### **23. PaaS Deep Dive: Go Platform**

*   **Detection**: Presence of `go.mod` file.
*   **Dependency Management**: Automatically runs `go mod download` if `go.mod` and `go.sum` exist. It's crucial to have a correct `go.sum` file (`go mod tidy` locally is recommended).
*   **Build Process**: Compiles the project using `go build`.
*   **Execution**:
    *   By default, Liara looks for and runs `main.go`.
    *   If the main file has a different name, it must be specified in `liara.json`.
*   **Configuration (`liara.json`)**:
    *   `"platform": "go"`
    *   `"go"` (object):
        *   `"version"`: (string) Go version, e.g., `"1.22"`.
        *   `"procfile"`: (string) The entrypoint file if not `main.go`. e.g., `"app.go"`.
        *   `"cgoEnabled"`: (boolean string) ` "true"` or `"false"`. **Required** for packages that use C bindings, like `go-sqlite3`.
        *   `"timezone"`: (string) e.g., `"UTC"`.
    *   **Example `liara.json`**:
        ```json
        {
          "platform": "go",
          "port": 8080,
          "go": {
            "version": "1.22",
            "cgoEnabled": "true"
          }
        }
        ```

---

#### **24. PaaS Deep Dive: .NET Platform**

*   **Detection**: Presence of a `.sln` or `.csproj` file.
*   **Build Process**:
    1.  `dotnet restore`
    2.  `dotnet publish` (in `Release` configuration)
*   **Execution**: Runs the published DLL.
*   **Configuration (`liara.json`)**:
    *   `"platform": "dotnet"`
    *   `"dotnet"` (object):
        *   `"version"`: (string) .NET SDK version, e.g., `"8.0"`.
        *   `"finalDllName"`: (string) Name of the final DLL if it differs from the project name.
        *   `"csprojectFile"`: (string) Path to the `.csproj` file if it's not in the root.
        *   `"timezone"`: (string) e.g., `"UTC"`.
*   **Database Connection (MSSQL Example)**:
    *   Connection string in `appsettings.json`.
    *   **Key Parameters**: `Encrypt=False;` and `TrustServerCertificate=True;` are often required when connecting to managed SQL Server instances.
    *   **Code**:
        ```csharp
        // Program.cs
        builder.Services.AddDbContext<MyDbContext>(options =>
          options.UseSqlServer(builder.Configuration.GetConnectionString("Production")));
        ```

---

#### **25. Advanced Feature: Hooks**

Hooks are user-defined shell scripts that run at specific points in the deployment lifecycle. They allow for powerful customization of the build and run process.

*   **File Names & Execution Order**:
    1.  **`liara_pre_build.sh`**:
        *   **Runs**: Before any dependency installation (`npm install`, `composer install`, `pip install`).
        *   **Env Access**: No.
        *   **Use Case**: Installing system-level packages needed for building. e.g., `apt-get update && apt-get install -y build-essential`.
    2.  **`liara_post_build.sh`**:
        *   **Runs**: After the build script (`npm run build`).
        *   **Env Access**: No.
        *   **Use Case**: Post-build asset optimization, cleanup, or moving files.
    3.  **`liara_pre_start.sh`**:
        *   **Runs**: After the build is complete, just before the application's `start` command is executed.
        *   **Env Access**: **Yes**. Can access all application environment variables.
        *   **Use Case**: **Database migrations**. This is the recommended way to run migrations. e.g., `php artisan migrate --force`, `npx prisma migrate deploy`.

*   **Example `liara_pre_start.sh` for a Django app**:
    ```sh
    #!/bin/bash
    echo "Running pre-start script..."
    python manage.py migrate
    echo "Migrations complete."
    ```

---

#### **26. Advanced Feature: `supervisor.conf`**

*   **Purpose**: To run and manage background worker processes alongside the main web application. It ensures that worker processes are always running and restarts them if they crash.
*   **Supported Platforms**: PHP, Python (Django, Flask).
*   **Setup**:
    1.  Create a `supervisor.conf` file at the root of your project.
    2.  Define one or more `[program:your-worker-name]` sections.
*   **Key Directives**:
    *   `command`: The full command to run the worker. e.g., `php /var/www/html/artisan queue:work`, `celery -A proj worker -l INFO`.
    *   `autostart=true`: Starts the process when the app starts.
    *   `autorestart=true`: Restarts the process if it exits unexpectedly.
    *   `numprocs=1`: Number of instances of this process to run.
    *   `stdout_logfile=/tmp/worker.log`: Redirects worker logs to a file in the temporary directory. **Essential for debugging workers**.
*   **Example (Laravel Queue Worker)**:
    ```ini
    [program:laravel-worker]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/html/artisan queue:work redis --sleep=3 --tries=3
    autostart=true
    autorestart=true
    user=www-data
    numprocs=2
    redirect_stderr=true
    stdout_logfile=/tmp/laravel-worker.log
    ```
*   **Monitoring**: Check the status of supervisor-managed processes by running `supervisorctl status` in the app's shell (`liara shell`).

---

#### **27. Advanced Feature: Custom Nginx Configuration (`liara_nginx.conf`)**

*   **Purpose**: To extend the default Nginx configuration for static-serving platforms (Static, React, Vue, Angular) and some backend platforms (PHP, Django, Flask).
*   **Mechanism**: The contents of your `liara_nginx.conf` file are **included** inside the main `server { ... }` block of Liara's default Nginx configuration. You cannot overwrite the entire server block, only add to it.
*   **Common Use Cases**:
    *   **Custom Headers**: Add security headers.
        ```nginx
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options nosniff;
        add_header Strict-Transport-Security "max-age=63072000" always;
        ```
    *   **Custom `location` blocks**: Define custom routing for specific paths.
        ```nginx
        location /api {
          # proxy_pass settings for an API backend
        }
        ```
    *   **Enabling Gzip Compression**:
        ```nginx
        gzip on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_types text/plain text/css application/json application/javascript;
        ```
    *   **Browser Caching Rules**:
        ```nginx
        location ~* \.(?:jpg|jpeg|gif|png|ico)$ {
          expires 1M;
          add_header Cache-Control "public";
        }
        ```

This section covers the remaining platform-specific details and advanced features that provide deep control over the deployment and runtime environment, enabling complex application architectures.

Of course. Here is the ninth part of the technical knowledge base. This section covers **Object Storage** in detail, a critical component for modern applications that need to handle user-uploaded content, media files, or large static assets.

---


#### **31. Object Storage: Core Concepts**

*   **Technology**: S3-Compatible Object Storage. This means any tool, SDK, or library designed for Amazon S3 can work with Liara's Object Storage with minimal configuration changes.
*   **Key Terminology**:
    *   **Bucket**: A container for your files (objects). Bucket names are globally unique across Liara.
    *   **Object**: A file and its associated metadata.
    *   **Endpoint**: The base URL for the storage service (e.g., `https://storage.iran.liara.space`). This is the "server" address you point your SDK to.
    *   **Access Key & Secret Key**: Your credentials for programmatic access. **Crucial**: The Secret Key is only shown **once** upon creation. It must be stored securely.
*   **Access Control**:
    *   **`private` (Default)**: Objects are not publicly accessible via a direct URL. They can only be accessed programmatically using signed credentials or by generating a **pre-signed URL**. This is the recommended mode for user data and sensitive files.
    *   **`public-read`**: All objects in the bucket are publicly accessible via their permanent URL (`<endpoint>/<bucket-name>/<object-key>`). Suitable for public assets like images, CSS, or public downloads.
    *   **Changing Access Level**: Can be done anytime from the bucket's settings in the Liara console.

---

#### **32. Programmatic Access with AWS SDKs**

This is the primary way applications interact with Object Storage.

*   **General Workflow**:
    1.  **Install SDK**: `npm install @aws-sdk/client-s3` (for Node.js) or `pip install boto3` (for Python).
    2.  **Configure Environment Variables**: Your application needs the following environment variables, which are provided in the Liara console:
        *   `LIARA_ENDPOINT_URL`
        *   `LIARA_ACCESS_KEY`
        *   `LIARA_SECRET_KEY`
        *   `LIARA_BUCKET_NAME`
    3.  **Initialize S3 Client**: Create an S3 client instance, pointing it to Liara's endpoint. **This is the most important step.**
        *   **Node.js (`@aws-sdk/client-s3`) Example**:
            ```javascript
            import { S3Client } from "@aws-sdk/client-s3";

            const s3Client = new S3Client({
              endpoint: process.env.LIARA_ENDPOINT_URL,
              region: "us-east-1", // A required but arbitrary value for S3-compatible services
              credentials: {
                accessKeyId: process.env.LIARA_ACCESS_KEY,
                secretAccessKey: process.env.LIARA_SECRET_KEY,
              },
            });
            ```
        *   **Python (`boto3`) Example**:
            ```python
            import boto3
            
            s3_client = boto3.client(
                "s3",
                endpoint_url=os.getenv("LIARA_ENDPOINT_URL"),
                aws_access_key_id=os.getenv("LIARA_ACCESS_KEY"),
                aws_secret_access_key=os.getenv("LIARA_SECRET_KEY")
            )
            ```

*   **Common Operations**:
    *   **Upload File**: `PutObjectCommand` (Node.js) or `upload_fileobj` (Python).
        *   **Key Parameter**: You can set `ContentDisposition: 'attachment'` to force browser download.
    *   **Download File**: `GetObjectCommand` (Node.js) or `download_file` (Python).
    *   **Delete File**: `DeleteObjectCommand` (Node.js) or `delete_object` (Python).
    *   **List Files**: `ListObjectsV2Command` (Node.js) or `list_objects_v2` (Python).
    *   **Generate Pre-signed URL**: Create a temporary, secure URL for accessing private objects.
        *   **Node.js**: Use `@aws-sdk/s3-request-presigner` package with `getSignedUrl`.
        *   **Python**: Use `generate_presigned_url` method on the client.

---

#### **33. Integration with Frameworks & Tools**

*   **Laravel**:
    *   Uses the `league/flysystem-aws-s3-v3` package.
    *   Configure `config/filesystems.php` to use the `s3` driver and set the endpoint via environment variables (`AWS_ENDPOINT`, `AWS_BUCKET`, etc.).
    *   Set `AWS_USE_PATH_STYLE_ENDPOINT=true`.

*   **Django**:
    *   Uses the `boto3` and `django-storages` packages.
    *   Configure `settings.py` with `AWS_STORAGE_BUCKET_NAME`, `AWS_S3_ENDPOINT_URL`, etc.
    *   Set `DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'`.

*   **Strapi**:
    *   Uses the `@strapi/provider-upload-aws-s3` provider.
    *   Configure `config/plugins.js` to set up the `aws-s3` provider with the correct endpoint and credentials from environment variables.

*   **Rclone (CLI Tool)**:
    *   A powerful tool for syncing local directories with a bucket, or transferring files between different cloud providers.
    *   Configure a new "remote" for S3, selecting "Other" as the provider and manually entering Liara's endpoint URL and credentials.

*   **S3 Browser (GUI Tool)**: A Windows desktop client for managing S3-compatible storage. Add a new account, select "S3 Compatible Storage", and input Liara's endpoint and credentials.

This section covers the essential knowledge for managing files and user-generated content, a fundamental requirement for most web applications beyond a simple static page. Understanding how to use the S3-compatible service is key to building scalable and robust applications on Liara. The next and final response will cover account management, teams, and billing.

Of course. Here is the final part of the technical knowledge base, wrapping up with information on account management, teamwork features, and general platform administration.

---


#### **34. Account Management & Security**

*   **Authentication**:
    *   Primary method is email/password or social login (Google, GitHub).
    *   **Two-Factor Authentication (2FA)**: **Strongly recommended**.
        *   Setup is done in Console -> Profile -> Security.
        *   Uses standard TOTP apps (e.g., Google Authenticator).
        *   **Crucial**: Recovery codes must be saved securely. Losing both the 2FA device and recovery codes will result in permanent account lockout.
*   **API Keys**:
    *   Generated in Console -> API.
    *   Used for authenticating `liara-cli` and any other programmatic access.
    *   Treat them like passwords; store them securely and do not expose them in client-side code or public repositories.
*   **Billing & Invoices**:
    *   **Credit System**: Liara operates on a pre-paid credit system. Services consume credit on an hourly basis.
    *   **Increasing Credit**: Can be done through the console.
    *   **Cost Estimation**: The console provides a tool to estimate monthly costs based on running services.
    *   **Invoices**: Can be generated for payments after filling out the necessary business information in the billing section.

---

#### **35. Team Features**

*   **Purpose**: Allows multiple users to collaborate on a shared set of applications and services under a single billing account.
*   **Creation**: A personal account can be converted into a team, or a new team can be created.
*   **Roles & Permissions**:
    *   Liara has a Role-Based Access Control (RBAC) system.
    *   **Default Roles**: Owner, Admin, Developer, etc.
    *   **Custom Roles**: You can create custom roles with granular permissions.
    *   **Permissions**: Control access to specific actions like deploying apps, managing billing, creating databases, viewing logs, etc.
*   **Team ID (`team-id`)**:
    *   A unique identifier for the team.
    *   **Usage**: Required when using `liara-cli` to perform actions within a team's context.
    *   **CLI Flag**: `liara deploy --team-id=<your-team-id>`
    *   **`liara.json`**: `{ "team-id": "<your-team-id>" }`

---

#### **36. General Platform Administration**

*   **Moving Services**:
    *   **Between Accounts/Teams**: There is no direct "transfer" button. The process is manual:
        1.  **Download Source**: Download the latest successful deployment source code from the original app's "History" tab.
        2.  **Backup Data**: Manually back up any databases or disk contents.
        3.  **Create New Services**: Create a new, identical app and database(s) in the target account/team.
        4.  **Configure**: Set up all environment variables in the new app.
        5.  **Deploy & Restore**: Deploy the source code to the new app and restore the data to the new databases/disks.
        6.  **Switch DNS**: Update DNS records to point to the new app.
        7.  **Delete Old Services**: Once confirmed working, delete the original services.
*   **Notifications**:
    *   Can be configured in Console -> Profile -> Notifications.
    *   Events that can trigger notifications: deployments, low credit warnings, service crashes, etc.
    *   Channels: Email, SMS.
*   **Tickets**:
    *   The primary channel for official technical and billing support.
    *   Accessed through the "Support" section of the console.

---


### **The Definitive Liara Playbook for Modern Web Applications (AI Edition)**

This guide synthesizes all previous knowledge into a practical, step-by-step workflow with best practices and essential commands for deploying and managing modern web applications (Backend API, Frontend, Database, Storage) on Liara.

---

### **Phase 1: Project Setup & Local Configuration (`liara.json`)**

This is the most critical phase. A well-structured `liara.json` prevents most common deployment issues.

#### **1.1. The Ultimate `liara.json` Template**

Create a `liara.json` at the root of your project. This is your source of truth.

```json
{
  // App Identification (Set once, then commit)
  "app": "َapp-name",
  "platform": "docker",
  "port": 8000,
  "team-id": "your-team-id-if-applicable",

  // Build Configuration (Optimize for speed & correctness)
  "build": {
    "location": "germany", // Use 'germany' for access to global packages/dependencies
    "cache": true, // Keep 'true' for faster subsequent deploys
    "dockerfile": "./Dockerfile" // Explicitly define path
  },

  // Runtime Health (Crucial for Zero-Downtime)
  "healthCheck": {
    "command": "CMD curl --fail http://localhost:8000/health || exit 1",
    "interval": 30,
    "timeout": 10,
    "retries": 3,
    "startPeriod": 15 // Give the app 15s to boot before first health check
  },

  // Persistent Storage (Essential for databases, user uploads, etc.)
  "disks": [
    {
      "name": "app-data", // The name of the disk you created in Liara console
      "mountTo": "/app/data" // The absolute path inside your container
    }
  ]
}
```

**Best Practice Checklist for `liara.json`**:
-   [ ] **`platform`**: Always specify `docker` for Deno/Bun or custom runtimes. It gives you maximum control.
-   [ ] **`port`**: Must match the port your application listens on (`app.listen(PORT)`).
-   [ ] **`build.location`**: Default to `"germany"` unless all your dependencies are reliably available from Iranian mirrors.
-   [ ] **`healthCheck`**: Always implement a dedicated `/health` or `/status` endpoint in your app that returns a `200 OK` status. This is non-negotiable for reliable production deployments.
-   [ ] **`disks`**: Pre-plan your persistent data needs. Any data not on a disk **will be lost** on redeploy.

---

### **Phase 2: Dockerizing for Liara (`Dockerfile`)**

A minimal, multi-stage, and secure `Dockerfile` is key.

#### **2.1. The Ultimate `Dockerfile` for Deno**

```dockerfile
# === Stage 1: Build & Cache ===
FROM denoland/deno:alpine-1.44.4 as builder
# Use a specific, stable version. Avoid 'latest'.

WORKDIR /app

# First, copy only dependency manifests
COPY deno.json deno.lock ./

# Cache dependencies in a separate layer. This layer is only rebuilt if
# deno.json or deno.lock changes, dramatically speeding up builds.
RUN deno cache --lock=deno.lock server.ts

# Then, copy the rest of the source code
COPY . .

# (Optional) If you have a build step (e.g., for frontend assets), run it here
# RUN deno task build


# === Stage 2: Production ===
# Use the same Alpine base for a smaller final image
FROM denoland/deno:alpine-1.44.4

WORKDIR /app

# Copy only the cached dependencies and the essential built files from the builder stage
COPY --from=builder /deno-dir/ /deno-dir/
COPY --from=builder /app .

# Expose the port (good practice, though Liara uses the 'port' from liara.json)
EXPOSE 8000

# The final command to run the application.
# Use --no-check to avoid type-checking at startup for faster boot times.
# All necessary permissions should be listed explicitly.
CMD ["deno", "run", "--allow-net", "--allow-env", "--allow-read=/app/data", "--no-check", "server.ts"]
```

**Best Practice Checklist for `Dockerfile`**:
-   [ ] **Use Specific Versions**: Pin the Deno version (e.g., `denoland/deno:alpine-1.44.4`) instead of `latest`.
-   [ ] **Multi-Stage Builds**: Use a `builder` stage to create dependencies and a final, clean stage to run the app. This results in a smaller, more secure production image.
-   [ ] **Layer Caching**: Copy `deno.json`/`deno.lock` and run `deno cache` *before* copying the rest of your code.
-   [ ] **Minimal Permissions in `CMD`**: Only grant the permissions your application absolutely needs at runtime.
-   [ ] **`--no-check`**: Use this flag in your production `CMD` to skip type-checking on startup, leading to faster boot times.

---

### **Phase 3: Service Provisioning & First Deployment (CLI)**

This is the sequence of commands to get your environment up and running from scratch.

#### **3.1. The Initial Setup Workflow (Run Once)**

```bash
# 1. Login to Liara (if not already)
liara login

# 2. Create the managed PostgreSQL database
# Use the CLI to see available plans and versions first
# liara plan:ls -t postgresql
liara db:create --name "simpledraft-postgres" --type "postgresql" --plan "small-g2" --version "16" --team-id <your-team-id>

# 3. Create a persistent disk for the application
# Use a meaningful name related to the app and its purpose.
liara disk:create --name "simpledraft-api-data" --app "simpledraft-api" --size 1 --team-id <your-team-id>

# 4. Create an Object Storage bucket for file uploads
liara bucket create --name "simpledraft-media" --permission "private" --plan "2.5GB" --team-id <your-team-id>

# 5. Create API keys for the bucket
# IMPORTANT: Save the Access Key and Secret Key immediately. The Secret Key will not be shown again.
liara bucket:keys --name "simpledraft-media"

# 6. Create the application itself on Liara
liara app:create --name "simpledraft-api" --platform "docker" --plan "small-g2" --network <your-network> --team-id <your-team-id>

# 7. Set environment variables from your Neon DB and S3 keys
# It's best to do this in one go using a local .env file and a script, or manually in the console.
# For CI/CD, these should be secrets.
liara env:set --app "simpledraft-api" \
  DATABASE_URL="postgresql://..." \
  JWT_SECRET="a-very-strong-random-string" \
  LIARA_ENDPOINT_URL="https://storage.iran.liara.space" \
  LIARA_ACCESS_KEY="..." \
  LIARA_SECRET_KEY="..." \
  LIARA_BUCKET_NAME="simpledraft-media"

# 8. Perform the first deployment
# The --path flag points to your project's root.
# All other settings are read from liara.json.
liara deploy --path . --app "simpledraft-api"
```

---

### **Phase 4: Ongoing Development & Debugging**

These are the commands you will use daily.

#### **4.1. The Day-to-Day Workflow**

1.  **Develop Locally**: Make your code changes.
2.  **Deploy**:
    ```bash
    # Simply run deploy. It will read liara.json for all settings.
    liara deploy
    ```
3.  **Monitor & Debug**:
    ```bash
    # Watch logs in real-time immediately after deploying
    liara logs -f simpledraft-api

    # If something is wrong, shell into the container
    # This is your most powerful debugging tool.
    liara shell simpledraft-api /bin/sh

    # Inside the shell, check:
    # 1. Are my files here?
    /app # ls -la
    
    # 2. Are my environment variables set correctly?
    /app # printenv | grep DATABASE
    
    # 3. Can I connect to my database from inside the container?
    # (Requires a network tool like 'nc' or 'ping', may need to be added to Dockerfile)
    /app # nc -zv <postgres-private-host> 5432
    
    # 4. Check the contents of the persistent disk
    /app # ls -la /app/data
    ```

#### **4.2. Database Migrations**

Migrations should be automated. The best practice is to use a `pre-start` hook.

1.  **Create `liara_pre_start.sh`**:
    ```sh
    #!/bin/sh
    
    # This script runs *inside the container* before the main app starts.
    echo "Running database migrations..."
    
    # For Deno with Drizzle ORM (example)
    deno run -A ./src/db/migrate.ts
    
    echo "Migrations finished."
    ```
2.  **Make it executable**: `chmod +x liara_pre_start.sh`
3.  **Commit and deploy**. Liara will automatically execute this script on every deployment before starting your application server. This ensures your database schema is always in sync with your code.




#### **37. Best Practices Recap**

*   **Use Specific Versions**: Pin dependencies in `Dockerfile` and `deno.json`/`package.json`. Prevents unexpected updates.
*   **Multi-Stage Builds**: Minimizes image size and improves security.
*   **Declarative Configuration**: Use `liara.json` for nearly all deployment settings. This version-controls your infrastructure.
*   **Private Networks**: Use them extensively for app-to-database and microservice-to-microservice communication.
*   **Persistent Disks**: Understand where your data needs to be persisted and mount disks appropriately.
*   **Automated Migrations**: Use `liara_pre_start.sh` and a migration tool to keep database schema up-to-date.
*   **Health Checks**: The most important part of a reliable deployment.
*   **Monitor Logs & Events**: Be familiar with `liara logs -f` and the Liara console's events panel for quick debugging.
*   **Secure Your Keys**: Use the CI/CD system to store API tokens and Secrets as enviroment virable.

#### **38. Example Project Structure (Modern Web App with API)**

This is a template to visualize how to organize code. It is not a requirement but a recommende way to organize your project's files.

```
simpledraft-project/
├── .gitignore             # Git ignore file (crucial!)
├── liara.json             # Liara project configuration
├── Dockerfile             # Dockerfile for API
├── frontend/              # Directory for frontend code (if using a separate frontend)
│   ├── package.json
│   ├── next.config.js     # or vite.config.ts/vue.config.js etc.
│   ├── src/
│   └── ...
├── backend/               # Directory for backend API code
│   ├── deno.json          #or package.json for Node js
│   ├── server.ts          # Main API file
│   ├── src/               # Source code directory
│   ├── db/
│   │   ├── schema.ts    # Drizzle ORM schema definition
│   │   └── migrations/  # Database migrations
│   └── ...
├── database/               # DDL and DML files or seed files
├── README.md
└── LICENSE
```

*   **Key Directory Breakdown**:
    *   **`.` (Root)**:
        *   `liara.json`: Contains top-level application configuration.
        *   `Dockerfile`: Describes how to build your API container.
        *   `.gitignore`: Specifies files/folders to exclude from the build process.
    *   **`frontend/`**:
        *   `package.json` (and lockfile): Frontend dependencies (React, Vue, Svelte, etc.).
        *   `src/`: Source code for the frontend application.
        *   Config file (e.g. `next.config.js`): Framework-specific configuration.
    *   **`backend/`**:
        *   `deno.json` or `package.json`: Lists the application's backend dependencies.
        *   `server.ts` (or similar): The main file for API.
        *   `db/`: Stores the ORM definitions and migrations.

#### **39. Example  and Full Content of `deno.json`**

```json
{
  "tasks": {
    "start": "deno run --allow-net --allow-read --allow-write --allow-env server.ts"
  },
  "imports": {
    "hono": "https://deno.land/x/hono@v3.12.6/mod.ts",
    "hono/cors": "https://deno.land/x/hono@v3.12.6/middleware/cors/index.ts",
    "hono/jwt": "https://deno.land/x/hono@v3.12.6/middleware/jwt/index.ts",
    "sqlite": "https://deno.land/x/sqlite/mod.ts",
    "bcrypt": "https://deno.land/x/bcrypt@v0.4.1/mod.ts",
    "zod": "https://deno.land/x/zod@v3.22.4/mod.ts",
    "postgres": "https://deno.land/x/postgres@v0.19.3/mod.ts",
    "jsr:@std/io/buffer": "jsr:@std/io@0.19/buffer.ts",
    "jsr:@std/io/types": "jsr:@std/io@0.19/types.ts",
    "jsr:@std/io/copy": "jsr:@std/io@0.19/copy.ts",
    "jsr:@std/io/read-all": "jsr:@std/io@0.19/read_all.ts",
    "jsr:@std/io/write-all": "jsr:@std/io@0.19/write_all.ts",
    "jsr:@std/io/iterate-reader": "jsr:@std/io@0.19/iterate_reader.ts",
    "jsr:@luca/esbuild-deno-loader": "jsr:@luca/esbuild-deno-loader@0.7.0/mod.ts"
  },
  "compilerOptions": {
    "lib": ["dom", "esnext"],
    "target": "esnext",
    "jsx": "react-jsx",
    "jsxImportSource": "react"
  }
}
```

#### **40. Request for Future Projects**

To further refine my knowledge, when you embark on new projects with a similar architecture, I would greatly benefit from the following information (if possible):

1.  **`liara.json`**: The complete `liara.json` file, including all platform-specific configurations.
2.  **`Dockerfile`**: The multi-stage `Dockerfile`, especially the `CMD` used to run the application.
3.  **Database connection code snippet**: Extract the code you're using to connect to your Neon PostgreSQL in `server.ts`.


/* ======== LIARA DOCs STRUCTURE (if you need more info you can ask me, I'll provide you with the file or links) ======== */

└── pages/
    ├── ai/
    │   ├── connect-to-service/
    │   │   ├── chat-gpt-copilot.mdx
    │   │   ├── n8n.mdx
    │   │   └── openweb.mdx
    │   ├── foundations/
    │   │   ├── overview.mdx
    │   │   ├── prompts.mdx
    │   │   └── tools.mdx
    │   ├── references/
    │   │   └── ai-sdk-core/
    │   │       ├── jsonschema.mdx
    │   │       └── zodschema.mdx
    │   ├── about.mdx
    │   ├── anthropic-claude.mdx
    │   ├── deepseek.mdx
    │   ├── google-gemini.mdx
    │   ├── grok-x-ai.mdx
    │   ├── meta-llama.mdx
    │   ├── mistral-nemo.mdx
    │   ├── openai.mdx
    │   ├── quick-start.mdx
    │   └── teams.mdx
    ├── dbaas/
    │   ├── details/
    │   │   ├── plans/
    │   │   │   ├── about.mdx
    │   │   │   ├── hardware-plans.mdx
    │   │   │   └── software-plans.mdx
    │   │   ├── about.mdx
    │   │   ├── change-plan.mdx
    │   │   ├── connection-links.mdx
    │   │   ├── connection-pool.mdx
    │   │   ├── customizing-db-parameters.mdx
    │   │   ├── delete-database.mdx
    │   │   ├── events.mdx
    │   │   ├── observations.mdx
    │   │   └── private-network.mdx
    │   ├── elastic-search/
    │   │   ├── how-tos/
    │   │   │   └── connect-via-platform/
    │   │   │       ├── about.mdx
    │   │   │       ├── django.mdx
    │   │   │       ├── dotnet.mdx
    │   │   │       ├── flask.mdx
    │   │   │       ├── go.mdx
    │   │   │       ├── laravel.mdx
    │   │   │       ├── nextjs.mdx
    │   │   │       ├── nodejs.mdx
    │   │   │       ├── php.mdx
    │   │   │       └── python.mdx
    │   │   ├── choose-version.mdx
    │   │   └── quick-setup.mdx
    │   ├── mariadb/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-cli/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── mariadb.mdx
    │   │   │   │   └── mysql.mdx
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── dbeaver.mdx
    │   │   │   │   └── phpmyadmin.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── flask.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── mongodb/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-cli/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mongosh.mdx
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mongodb-compass.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── flask.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── mssql/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-cli/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── sqlcmd.mdx
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── azure-data-studio.mdx
    │   │   │   │   ├── dbeaver.mdx
    │   │   │   │   └── mssql-server-studio.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── mysql/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-cli/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mysql.mdx
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── dbeaver.mdx
    │   │   │   │   ├── phpmyadmin.mdx
    │   │   │   │   └── workbench.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── flask.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── postgresql/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-cli/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── psql.mdx
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── dbeaver.mdx
    │   │   │   │   └── pgadmin.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── flask.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── rabbitmq/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── rabbitmq-management.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── flask-nodejs.mdx
    │   │   │   │   ├── flask.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── redis/
    │   │   ├── how-tos/
    │   │   │   ├── connect-via-cli/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── redis-cli.mdx
    │   │   │   ├── connect-via-gui/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── phpredisadmin.mdx
    │   │   │   ├── connect-via-platform/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── django.mdx
    │   │   │   │   ├── dotnet.mdx
    │   │   │   │   ├── flask.mdx
    │   │   │   │   ├── go.mdx
    │   │   │   │   ├── laravel.mdx
    │   │   │   │   ├── nextjs.mdx
    │   │   │   │   ├── nodejs.mdx
    │   │   │   │   ├── php.mdx
    │   │   │   │   └── python.mdx
    │   │   │   ├── create-backup.mdx
    │   │   │   └── restore-backup.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── create-user.mdx
    │   │   └── quick-setup.mdx
    │   ├── about.mdx
    │   └── move.mdx
    ├── dns-management-system/
    │   ├── details/
    │   │   ├── about.mdx
    │   │   ├── delete-dns-management-system.mdx
    │   │   ├── supported-records.mdx
    │   │   └── wildcard-dns-records.mdx
    │   ├── how-tos/
    │   │   ├── add-records.mdx
    │   │   └── manage-records.mdx
    │   ├── about.mdx
    │   └── quick-setup.mdx
    ├── email-server/
    │   ├── details/
    │   │   ├── about.mdx
    │   │   ├── change-plan.mdx
    │   │   ├── common-errors.mdx
    │   │   ├── delete-email-server.mdx
    │   │   ├── dns-records.mdx
    │   │   ├── mail-id.mdx
    │   │   ├── observations.mdx
    │   │   ├── plans.mdx
    │   │   └── ports.mdx
    │   ├── how-tos/
    │   │   ├── connect-via-platform/
    │   │   │   ├── about.mdx
    │   │   │   ├── django.mdx
    │   │   │   ├── dotnet.mdx
    │   │   │   ├── flask.mdx
    │   │   │   ├── ghost.mdx
    │   │   │   ├── go.mdx
    │   │   │   ├── laravel.mdx
    │   │   │   ├── nextjs.mdx
    │   │   │   ├── nodejs.mdx
    │   │   │   ├── php.mdx
    │   │   │   ├── python.mdx
    │   │   │   └── wordpress.mdx
    │   │   ├── add-account.mdx
    │   │   ├── add-smtp-user.mdx
    │   │   ├── block-emails.mdx
    │   │   ├── change-sending-mode.mdx
    │   │   ├── control-spam.mdx
    │   │   ├── manage-incoming-emails.mdx
    │   │   ├── manage-limitations.mdx
    │   │   ├── manage-sent-emails.mdx
    │   │   ├── send-email-via-console.mdx
    │   │   ├── set-forward.mdx
    │   │   ├── set-spam.mdx
    │   │   ├── set-subscription-header.mdx
    │   │   └── use-tags.mdx
    │   ├── about.mdx
    │   └── quick-setup.mdx
    ├── iaas/
    │   ├── api/
    │   │   ├── about.mdx
    │   │   ├── create-vm.mdx
    │   │   ├── see-vm-events.mdx
    │   │   └── send-vm-signals.mdx
    │   ├── backups/
    │   │   ├── about.mdx
    │   │   └── take-full-backup.mdx
    │   ├── debian/
    │   │   ├── how-tos/
    │   │   │   ├── deploy-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mongodb.mdx
    │   │   │   ├── deploy-stack/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mongodb.mdx
    │   │   │   ├── change-ssh-port.mdx
    │   │   │   ├── connect-domain.mdx
    │   │   │   ├── connect-to-server-using-ssh.mdx
    │   │   │   ├── create-new-user.mdx
    │   │   │   ├── grant-privileges-to-user.mdx
    │   │   │   ├── set-dns.mdx
    │   │   │   └── set-firewall.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── details/
    │   │   ├── about.mdx
    │   │   ├── change-plan.mdx
    │   │   ├── delete-server.mdx
    │   │   ├── events.mdx
    │   │   ├── hardware-plans.mdx
    │   │   ├── ip.mdx
    │   │   ├── limitations.mdx
    │   │   ├── network-observation.mdx
    │   │   ├── openssh-package.mdx
    │   │   ├── reset-root-password.mdx
    │   │   ├── signals.mdx
    │   │   ├── ssh-key.mdx
    │   │   └── vm-id.mdx
    │   ├── disks/
    │   │   ├── about.mdx
    │   │   ├── create.mdx
    │   │   ├── mount.mdx
    │   │   ├── see-disk-details.mdx
    │   │   ├── see-disks.mdx
    │   │   └── unmount.mdx
    │   ├── ubuntu/
    │   │   ├── how-tos/
    │   │   │   ├── deploy-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mongodb.mdx
    │   │   │   ├── deploy-stack/
    │   │   │   │   ├── about.mdx
    │   │   │   │   └── mongodb.mdx
    │   │   │   ├── setup-stack/
    │   │   │   │   ├── lamp.mdx
    │   │   │   │   ├── lemp.mdx
    │   │   │   │   └── mean.mdx
    │   │   │   ├── change-ssh-port.mdx
    │   │   │   ├── connect-domain.mdx
    │   │   │   ├── connect-to-server-using-ssh.mdx
    │   │   │   ├── create-new-user.mdx
    │   │   │   ├── grant-privileges-to-user.mdx
    │   │   │   ├── set-dns.mdx
    │   │   │   └── set-firewall.mdx
    │   │   ├── choose-version.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   └── about.mdx
    ├── object-storage/
    │   ├── details/
    │   │   ├── about.mdx
    │   │   ├── change-plan.mdx
    │   │   ├── delete-object-storage.mdx
    │   │   ├── observations.mdx
    │   │   └── plans.mdx
    │   ├── how-tos/
    │   │   ├── connect-via-platform/
    │   │   │   ├── about.mdx
    │   │   │   ├── django.mdx
    │   │   │   ├── dotnet.mdx
    │   │   │   ├── flask.mdx
    │   │   │   ├── go.mdx
    │   │   │   ├── imgproxy.mdx
    │   │   │   ├── laravel.mdx
    │   │   │   ├── nextjs.mdx
    │   │   │   ├── nodejs.mdx
    │   │   │   ├── php.mdx
    │   │   │   ├── python.mdx
    │   │   │   ├── strapi.mdx
    │   │   │   └── wordpress.mdx
    │   │   ├── change-access-level.mdx
    │   │   ├── create-backup-using-rclone.mdx
    │   │   ├── create-backup-using-s3-browser.mdx
    │   │   ├── create-key.mdx
    │   │   ├── delete-file.mdx
    │   │   ├── delete-key.mdx
    │   │   ├── direct-download.mdx
    │   │   ├── download-file.mdx
    │   │   ├── edit-key.mdx
    │   │   ├── generate-new-key.mdx
    │   │   ├── move-bucket.mdx
    │   │   ├── see-file.mdx
    │   │   ├── share-file.mdx
    │   │   └── upload-file.mdx
    │   ├── about.mdx
    │   ├── add-domain.mdx
    │   └── quick-setup.mdx
    ├── one-click-apps/
    │   ├── ackee/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── activepieces/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── actual/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── affine/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── apache-answer/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── connect-to-db.mdx
    │   │   └── quick-start.mdx
    │   ├── appsmith/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── configure-vars.mdx
    │   │   └── quick-start.mdx
    │   ├── automatisch/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── chatwoot/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── chroma/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── cyberchef/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── directus/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── discourse/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── docmost/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── docuseal/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── drupal/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── etherpad/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── flowise/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── focalboard/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── formbricks/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── freescout/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── ghost/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── gitea/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── grafana/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── grist/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── harness/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── headless-chrome/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── connect-by-nodejs-and-playwright.mdx
    │   │   │   ├── connect-by-nodejs-and-puppeteer.mdx
    │   │   │   ├── connect-by-nodejs-and-selenium.mdx
    │   │   │   ├── connect-by-python-and-pyppeteer.mdx
    │   │   │   └── connect-by-python-and-selenium.mdx
    │   │   └── quick-start.mdx
    │   ├── imgproxy/
    │   │   ├── how-tos/
    │   │   │   ├── add-url-signature.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── limit-access.mdx
    │   │   │   ├── use-in-django.mdx
    │   │   │   ├── use-in-laravel.mdx
    │   │   │   └── use-in-nodejs.mdx
    │   │   └── quick-start.mdx
    │   ├── infisical/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── jupyter-notebook/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── keila/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── kibana/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── kutt/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── liara-compose/
    │   │   ├── about.mdx
    │   │   ├── envs.mdx
    │   │   ├── fields-tables.mdx
    │   │   └── quick-start.mdx
    │   ├── linkstack/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── listmonk/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── matomo/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── mattermost/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── maybe/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── meilisearch/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── connect.mdx
    │   │   └── quick-start.mdx
    │   ├── metabase/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── mixpost/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── moodle/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── n8n/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── nextcloud/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── nocobase/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── nocodb/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── ntfy/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── odoo/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── onedev/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── openproject/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── openwebui/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── paperless/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── passbolt/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── create-superuser.mdx
    │   │   └── quick-start.mdx
    │   ├── pghero/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── pocketbase/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── postiz/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── prestashop/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── rallly/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── rocketchat/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── soketi/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── connect-by-laravel.mdx
    │   │   └── quick-start.mdx
    │   ├── strapi/
    │   │   └── quick-start.mdx
    │   ├── supertokens/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── connect-to-app.mdx
    │   │   └── quick-start.mdx
    │   ├── teable/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── typesense/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── umami/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── unleash/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── uptime-kuma/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── set-trusted-proxies.mdx
    │   │   └── quick-start.mdx
    │   ├── varnish/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   └── configure-vars.mdx
    │   │   └── quick-start.mdx
    │   ├── vikunja/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── vscode/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── vvveb/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── wikijs/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   ├── wordpress/
    │   │   ├── fix-common-errors/
    │   │   │   ├── about.mdx
    │   │   │   ├── css-not-loading-error.mdx
    │   │   │   ├── file-access-errors.mdx
    │   │   │   └── too-many-redirects-error.mdx
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── customize-php-ini.mdx
    │   │   │   ├── duplicator.mdx
    │   │   │   ├── migrate-from-cpanel.mdx
    │   │   │   ├── see-extensions.mdx
    │   │   │   └── set-reverse-proxy.mdx
    │   │   └── quick-start.mdx
    │   ├── zitadel/
    │   │   ├── how-tos/
    │   │   │   └── choose-version.mdx
    │   │   └── quick-start.mdx
    │   └── about.mdx
    ├── overview/
    │   ├── about.mdx
    │   └── data-centers.mdx
    ├── paas/
    │   ├── angular/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   └── set-http-security-headers.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── cicd/
    │   │   ├── about.mdx
    │   │   ├── github.mdx
    │   │   └── gitlab.mdx
    │   ├── details/
    │   │   ├── observations/
    │   │   │   ├── about.mdx
    │   │   │   ├── hardware.mdx
    │   │   │   └── software.mdx
    │   │   ├── plans/
    │   │   │   ├── about.mdx
    │   │   │   ├── hardware-plans.mdx
    │   │   │   └── software-plans.mdx
    │   │   ├── about.mdx
    │   │   ├── basic-auth.mdx
    │   │   ├── build-location.mdx
    │   │   ├── change-plan.mdx
    │   │   ├── console-shell.mdx
    │   │   ├── delete-app.mdx
    │   │   ├── dns-server-settings.mdx
    │   │   ├── envs.mdx
    │   │   ├── events.mdx
    │   │   ├── file-system.mdx
    │   │   ├── health-check.mdx
    │   │   ├── ignoring-files.mdx
    │   │   ├── private-network.mdx
    │   │   ├── private-registry.mdx
    │   │   ├── reverse-proxy.mdx
    │   │   ├── static-ip.mdx
    │   │   └── zero-downtime-deployment.mdx
    │   ├── disks/
    │   │   ├── about.mdx
    │   │   ├── create-backup.mdx
    │   │   ├── create.mdx
    │   │   ├── decrease-value.mdx
    │   │   ├── delete.mdx
    │   │   ├── ftp-access.mdx
    │   │   ├── increase-value.mdx
    │   │   ├── move-files-to-bucket.mdx
    │   │   ├── move-files-to-other-disk.mdx
    │   │   ├── restore-backup-using-ftp.mdx
    │   │   ├── restore-backup-using-wget.mdx
    │   │   └── route.mdx
    │   ├── django/
    │   │   ├── fix-common-errors/
    │   │   │   ├── about.mdx
    │   │   │   ├── cors-media.mdx
    │   │   │   ├── cors.mdx
    │   │   │   ├── multiple-settings-files.mdx
    │   │   │   ├── upload-limit-size.mdx
    │   │   │   └── worker-timeout.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elastic-search.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-gunicorn-maxrequest.mdx
    │   │   │   ├── set-gunicorn-workers.mdx
    │   │   │   ├── set-http-security-headers.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── use-asgi.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-ffmpeg-module.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   ├── use-supervisord.mdx
    │   │   │   └── use-websocket.mdx
    │   │   ├── related-apps/
    │   │   │   └── celery.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── docker/
    │   │   ├── how-tos/
    │   │   │   ├── configure-supercronic.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── deploy-docker-compose.mdx
    │   │   │   ├── deploy-image-from-dockerhub.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   └── use-disk.mdx
    │   │   ├── related-apps/
    │   │   │   ├── arangodb.mdx
    │   │   │   ├── bun.mdx
    │   │   │   ├── deno.mdx
    │   │   │   ├── fastapi-mssql.mdx
    │   │   │   ├── flutter.mdx
    │   │   │   ├── nginx.mdx
    │   │   │   ├── seq.mdx
    │   │   │   └── streamlit.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── domains/
    │   │   ├── about.mdx
    │   │   ├── add-domain.mdx
    │   │   ├── add-subdomains.mdx
    │   │   ├── add-www-subdomain.mdx
    │   │   ├── default-subdomain.mdx
    │   │   ├── delete-domain.mdx
    │   │   ├── enable-ssl.mdx
    │   │   ├── move.mdx
    │   │   ├── supported-tlds.mdx
    │   │   └── use-cdn.mdx
    │   ├── dotnet/
    │   │   ├── fix-common-errors/
    │   │   │   ├── 502-bad-gateway.mdx
    │   │   │   ├── about.mdx
    │   │   │   └── cors.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── deploy-dll-files.mdx
    │   │   │   ├── deploy-solution-directory.mdx
    │   │   │   ├── manage-logs.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-ffmpeg-module.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   └── use-websocket.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── flask/
    │   │   ├── fix-common-errors/
    │   │   │   ├── about.mdx
    │   │   │   ├── cors.mdx
    │   │   │   ├── module-not-found.mdx
    │   │   │   ├── upload-limit-size.mdx
    │   │   │   └── worker-timeout.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elastic-search.mdx
    │   │   │   │   ├── mongodb.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-gunicorn-workers.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── set-trusted-proxies.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-ffmpeg-module.mdx
    │   │   │   └── use-hooks.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── go/
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elastic-search.mdx
    │   │   │   │   ├── mongodb.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── use-cgo.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   └── use-websocket.mdx
    │   │   ├── related-apps/
    │   │   │   ├── beego.mdx
    │   │   │   ├── echo.mdx
    │   │   │   ├── fiber.mdx
    │   │   │   └── gin.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── laravel/
    │   │   ├── fix-common-errors/
    │   │   │   ├── 419-page-expired.mdx
    │   │   │   ├── about.mdx
    │   │   │   ├── cors.mdx
    │   │   │   └── upload-limit-size.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elastic-search.mdx
    │   │   │   │   ├── mariadb.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── configure-livewire-trusted-proxy.mdx
    │   │   │   ├── configure-trustedproxies.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-php-ini.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip-and-caching.mdx
    │   │   │   ├── enable-ssr-using-inertia.mdx
    │   │   │   ├── install-new-extension.mdx
    │   │   │   ├── manage-laravel-logs.mdx
    │   │   │   ├── see-extension.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-http-security-headers.mdx
    │   │   │   ├── set-status-page.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-ffmpeg-module.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   ├── use-queues.mdx
    │   │   │   └── use-ziggy.mdx
    │   │   ├── related-apps/
    │   │   │   ├── laravel-octane.mdx
    │   │   │   ├── lumen.mdx
    │   │   │   └── voyager.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── nextjs/
    │   │   ├── fix-common-errors/
    │   │   │   ├── about.mdx
    │   │   │   ├── econnreset.mdx
    │   │   │   └── modify-config-file.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elasticsearch.mdx
    │   │   │   │   ├── mariadb.mdx
    │   │   │   │   ├── mongodb.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── increase-next-cache.mdx
    │   │   │   ├── reach-static-files.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   ├── use-isr.mdx
    │   │   │   ├── use-static-html-export.mdx
    │   │   │   ├── use-type-script.mdx
    │   │   │   └── use-websocket.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── nodejs/
    │   │   ├── fix-common-errors/
    │   │   │   ├── cors-error/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── adonisjs.mdx
    │   │   │   │   ├── expressjs.mdx
    │   │   │   │   ├── fastify.mdx
    │   │   │   │   ├── hapi.mdx
    │   │   │   │   ├── koa.mdx
    │   │   │   │   ├── nestjs.mdx
    │   │   │   │   ├── nuxtjs.mdx
    │   │   │   │   └── strapi.mdx
    │   │   │   ├── about.mdx
    │   │   │   └── graphql-error.mdx
    │   │   ├── how-tos/
    │   │   │   ├── configure-trusted-proxy/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── expressjs.mdx
    │   │   │   │   ├── hapi.mdx
    │   │   │   │   └── koa.mdx
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── drizzle/
    │   │   │   │   │   ├── about.mdx
    │   │   │   │   │   ├── deploy-apps.mdx
    │   │   │   │   │   └── todo-project.mdx
    │   │   │   │   ├── sequelize/
    │   │   │   │   │   ├── about.mdx
    │   │   │   │   │   ├── mariadb.mdx
    │   │   │   │   │   ├── mssql.mdx
    │   │   │   │   │   ├── mysql.mdx
    │   │   │   │   │   ├── postgresql.mdx
    │   │   │   │   │   └── sqlite.mdx
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── mongodb.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── prisma.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── build-and-use-es6.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-ffmpeg-module.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   ├── use-type-script.mdx
    │   │   │   └── use-websocket.mdx
    │   │   ├── related-apps/
    │   │   │   ├── strapi/
    │   │   │   │   ├── starter.mdx
    │   │   │   │   ├── strapi-mongodb.mdx
    │   │   │   │   └── strapi-sqlite.mdx
    │   │   │   ├── adonisjs.mdx
    │   │   │   ├── blitzjs.mdx
    │   │   │   ├── fastify.mdx
    │   │   │   ├── hapi.mdx
    │   │   │   ├── hono.mdx
    │   │   │   ├── json-server.mdx
    │   │   │   ├── nestjs.mdx
    │   │   │   ├── nitro.mdx
    │   │   │   ├── nuxtjs.mdx
    │   │   │   ├── qwik.mdx
    │   │   │   ├── remix.mdx
    │   │   │   ├── svelte-kit.mdx
    │   │   │   └── svelte.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── php/
    │   │   ├── fix-common-errors/
    │   │   │   ├── about.mdx
    │   │   │   ├── cors.mdx
    │   │   │   └── upload-limit-size.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elastic-search.mdx
    │   │   │   │   ├── mongodb.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-htaccess.mdx
    │   │   │   ├── customize-php-ini.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── install-new-extension.mdx
    │   │   │   ├── see-extension.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-http-security-headers.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   └── use-queues.mdx
    │   │   ├── related-apps/
    │   │   │   └── yii.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── python/
    │   │   ├── fix-common-errors/
    │   │   │   ├── about.mdx
    │   │   │   ├── cors-media.mdx
    │   │   │   ├── cors.mdx
    │   │   │   ├── multiple-settings-files.mdx
    │   │   │   ├── upload-limit-size.mdx
    │   │   │   └── worker-timeout.mdx
    │   │   ├── how-tos/
    │   │   │   ├── connect-to-db/
    │   │   │   │   ├── about.mdx
    │   │   │   │   ├── elastic-search.mdx
    │   │   │   │   ├── mongodb.mdx
    │   │   │   │   ├── mssql.mdx
    │   │   │   │   ├── mysql.mdx
    │   │   │   │   ├── postgresql.mdx
    │   │   │   │   ├── redis.mdx
    │   │   │   │   └── sqlite.mdx
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   ├── set-cron-job.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   ├── set-gunicorn-maxrequest.mdx
    │   │   │   ├── set-gunicorn-workers.mdx
    │   │   │   ├── set-http-security-headers.mdx
    │   │   │   ├── set-logs.mdx
    │   │   │   ├── use-asgi.mdx
    │   │   │   ├── use-disk.mdx
    │   │   │   ├── use-ffmpeg-module.mdx
    │   │   │   ├── use-hooks.mdx
    │   │   │   ├── use-supervisord.mdx
    │   │   │   └── use-websocket.mdx
    │   │   ├── related-apps/
    │   │   │   ├── celery.mdx
    │   │   │   └── fastapi.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── react/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   ├── set-envs.mdx
    │   │   │   └── set-http-security-headers.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── static/
    │   │   ├── how-tos/
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   └── set-http-security-headers.mdx
    │   │   ├── related-apps/
    │   │   │   ├── astro.mdx
    │   │   │   ├── eleventy.mdx
    │   │   │   ├── gatsby.mdx
    │   │   │   ├── gridsome.mdx
    │   │   │   ├── hugo.mdx
    │   │   │   ├── jekyll.mdx
    │   │   │   ├── nuxtjs.mdx
    │   │   │   └── solidjs.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── vue/
    │   │   ├── how-tos/
    │   │   │   ├── choose-version.mdx
    │   │   │   ├── create-app.mdx
    │   │   │   ├── customize-nginx.mdx
    │   │   │   ├── deploy-app.mdx
    │   │   │   ├── enable-gzip.mdx
    │   │   │   └── set-http-security-headers.mdx
    │   │   ├── getting-started.mdx
    │   │   ├── quick-start.mdx
    │   │   └── related-links.mdx
    │   ├── about.mdx
    │   ├── liarajson.mdx
    │   ├── move.mdx
    │   └── update.mdx
    ├── references/
    │   ├── api/
    │   │   ├── about.mdx
    │   │   └── get-info.mdx
    │   ├── cli/
    │   │   ├── about.mdx
    │   │   ├── add-account.mdx
    │   │   ├── add-or-edit-envs.mdx
    │   │   ├── autocomplete.mdx
    │   │   ├── check-domain-status.mdx
    │   │   ├── choose-default-account.mdx
    │   │   ├── connect-to-app-shell.mdx
    │   │   ├── create-app.mdx
    │   │   ├── create-bucket.mdx
    │   │   ├── create-db.mdx
    │   │   ├── create-domain.mdx
    │   │   ├── create-email-server.mdx
    │   │   ├── create-liara-json.mdx
    │   │   ├── delete-app.mdx
    │   │   ├── delete-bucket.mdx
    │   │   ├── delete-db.mdx
    │   │   ├── delete-domain.mdx
    │   │   ├── delete-email-server.mdx
    │   │   ├── deploy-app.mdx
    │   │   ├── get-domain-details.mdx
    │   │   ├── install.mdx
    │   │   ├── list-accounts.mdx
    │   │   ├── list-apps.mdx
    │   │   ├── list-buckets.mdx
    │   │   ├── list-dbs.mdx
    │   │   ├── list-domains.mdx
    │   │   ├── list-email-servers.mdx
    │   │   ├── list-envs.mdx
    │   │   ├── login.mdx
    │   │   ├── manage-disks.mdx
    │   │   ├── remove-account.mdx
    │   │   ├── remove-env.mdx
    │   │   ├── resize-db.mdx
    │   │   ├── restart-app.mdx
    │   │   ├── see-app-logs.mdx
    │   │   ├── see-platform-plans.mdx
    │   │   ├── start-app.mdx
    │   │   ├── start-db.mdx
    │   │   ├── stop-app.mdx
    │   │   └── stop-db.mdx
    │   ├── console/
    │   │   ├── about.mdx
    │   │   ├── change-password.mdx
    │   │   ├── cost-estimation.mdx
    │   │   ├── create-and-see-tickets.mdx
    │   │   ├── enable-two-factor-authentication.mdx
    │   │   ├── increase-credit.mdx
    │   │   ├── manage-accounts.mdx
    │   │   ├── receive-invoice.mdx
    │   │   ├── see-notifs.mdx
    │   │   ├── see-user-info.mdx
    │   │   ├── set-notifs.mdx
    │   │   └── upgrade-user-level.mdx
    │   └── team/
    │       ├── about.mdx
    │       ├── convert-personal-account-to-team.mdx
    │       ├── create-new-role.mdx
    │       ├── create.mdx
    │       ├── delete-member.mdx
    │       ├── delete-team.mdx
    │       ├── grant-access.mdx
    │       ├── invite-members.mdx
    │       ├── permissions.mdx
    │       ├── roles.mdx
    │       └── transfer-ownership.mdx
    ├── tv/
    │   ├── courses/
    │   │   ├── django.js
    │   │   ├── dotnet.js
    │   │   ├── flask.js
    │   │   ├── go.js
    │   │   ├── laravel.js
    │   │   ├── nextjs.js
    │   │   ├── node.js
    │   │   └── php.js
    │   └── index.js
    ├── _app.js
    ├── _document.js
    ├── 404.js
    └── index.js