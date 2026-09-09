
## Watch the Tutorial for docker-compose install:
[https://m.youtube.com/watch?v=A6CjAmJOWvA&t=5s](https://m.youtube.com/watch?v=A6CjAmJOWvA&t=5s)

## Warning
If you are upgrading from Postiz old version, please make sure you update your docker compose, you can read more here:
https://docs.postiz.com/installation/migration

### Configuration uses environment variables

The docker containers for Postiz are entirely configured with environment variables.

- **Option A** - environment variables in your `docker-compose.yml` file
- **Option B** - environment variables in a `postiz.env` file mounted in `/config` for the Postiz container only
- **Option C** - environment variables in a `.env` file next to your `docker-compose.yml` file (not recommended).

... or a mixture of the above options!

There is a [configuration reference](https://docs.postiz.com/configuration/reference) page with a list
of configuration settings.

Setup:
```
git clone https://github.com/gitroomhq/postiz-docker-compose
```

Then run:
```
docker compose -f docker-compose.yaml up
```

Wait for it to load:

Open your website on http://localhost:4007

## Dokploy import with Cloudflare R2

The standalone [docker-compose.yml](docker-compose.yml) and
[template.toml](template.toml) are packaged in [import.json](import.json) and
[postiz.base64](postiz.base64). The `.yaml` file remains the upstream local stack;
the `.yml` file is the Dokploy stack. Always specify the filename in local commands.

### Import and deploy

1. Create a new Compose service in Dokploy. Open Advanced → Import / Import Base64
   and paste the complete contents of `postiz.base64`.
2. Inspect the preview and import. Dokploy creates the environment entries, generates
   `JWT_SECRET`, `DB_PASSWORD` and `TEMPORAL_PASSWORD`, and creates the Temporal
   configuration file mount. No production secret is embedded in the payload.
3. The generated hostname routes to service `postiz`, internal port `5000`, path `/`.
   Enable HTTPS and Let's Encrypt in Dokploy Domains before using the app: the
   current Base64 importer creates domains with `certificateType=none`.
   `POSTIZ_HOST` contains only the hostname, without a scheme, path or trailing slash;
   the application URLs use `https://` automatically. If you change the domain,
   update both the domain entry and `POSTIZ_HOST`.
4. Fill the R2 entries described below in Environment. Social platform credentials,
   including Facebook, Instagram, Threads, TikTok, X, YouTube and Reddit, are empty
   placeholders. Fill only the providers you use; other optional API credentials
   also remain empty.
5. Deploy. No Compose structure edits or repository checkout are required. Redeploy
   after changing environment values. Registration defaults to `false` for
   `DISABLE_REGISTRATION`, matching upstream; change it in Environment when needed.

Use a new Compose service for a fresh installation. Import generates new secrets
and replaces the service's environment, mounts and domains. When upgrading an
existing installation, preserve its JWT secret, database credentials and volumes;
changing an environment password does not change an existing PostgreSQL account.

The stack publishes no host ports. Postiz is exposed through Dokploy; Temporal UI,
admin tools and the optional `debug` profile remain internal. The upstream service
images, persistent volumes, healthchecks and dependency ordering are preserved.
`BACKEND_INTERNAL_URL` uses `127.0.0.1:3000` inside the Postiz container.

### Cloudflare R2

`STORAGE_PROVIDER=cloudflare` and `CLOUDFLARE_REGION=auto` are set by the template.
Fill these entries before deploying:

- `CLOUDFLARE_ACCOUNT_ID`
- `CLOUDFLARE_ACCESS_KEY`
- `CLOUDFLARE_SECRET_ACCESS_KEY`
- `CLOUDFLARE_BUCKETNAME`
- `CLOUDFLARE_BUCKET_URL`

The bucket must already exist. Scope credentials to Object Read & Write for that
bucket. `CLOUDFLARE_BUCKET_URL` is its publicly readable HTTPS media origin (a custom
domain or enabled public R2 domain), not the authenticated S3 API endpoint; omit
any trailing slash. Social platforms must be able to fetch the uploaded media.
Configure CORS for `https://<POSTIZ_HOST>` using the
[Postiz R2 guide](https://docs.postiz.com/self-host/configuration/r2).
The upstream uploads volume is retained; existing local uploads are not migrated.

### Regenerate the import payload

Run from this repository after editing Compose or TOML:

```bash
jq -n \
  --rawfile compose docker-compose.yml \
  --rawfile config template.toml \
  '{compose: $compose, config: $config}' > import.json

base64 < import.json | tr -d '\n' > postiz.base64
```

`postiz.base64` is a single line. Base64 is an encoding, not encryption: keep actual
credentials in Dokploy Environment, never in these four artifacts.

### Local validation

[.env.example](.env.example) mirrors the template inputs. Copy it to a temporary
file outside the repository and populate the hostname and temporary generated
secrets for syntax validation. Do not use or print production secrets.

```bash
docker compose --env-file /path/to/validation.env -f docker-compose.yml config --quiet
```

Exported shell variables override an env file; validate in a clean environment.
A successful config check validates Compose, not R2 access or container startup.
Dokploy materializes `../files/temporal/development-sql.yaml` from the template;
a local `up` would additionally require that file and a reverse proxy with HTTPS.

The import format follows the [official template repository](https://github.com/Dokploy/templates),
its [Postiz template](https://github.com/Dokploy/templates/tree/canary/blueprints/postiz),
and the [Dokploy import implementation](https://github.com/Dokploy/dokploy/blob/canary/apps/dokploy/server/api/routers/compose.ts).
