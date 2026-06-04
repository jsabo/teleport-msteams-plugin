# teleport-msteams-plugin

Setup guide and local validation environment for the Teleport MS Teams access request
plugin. Walks through Azure and Teams configuration, then uses Docker Compose + tbot to
validate that everything is wired up correctly before enabling the Teleport Cloud-hosted
plugin.

**The production deployment is the Cloud-hosted plugin** — no self-hosting required. Once
the Azure setup is complete and the validate command succeeds, paste the same credentials
into Teleport Web UI (Integrations → Microsoft Teams) and it goes Healthy.

---

## How it works

The plugin watches Teleport access requests and posts adaptive cards to MS Teams via the
Microsoft Graph API. It can deliver to **channels** (via Teams channel URL) or to
**individual users** (via DM) depending on how recipients are configured.

Recipient routing is controlled by two mechanisms:
- **`role_to_recipients`** in `plugin.toml` — used by the Docker Compose validation setup
- **Access Monitoring Rules** — used by the Cloud-hosted plugin; take precedence over
  `role_to_recipients` when they match

For DM delivery, the plugin automatically installs TeleBot as a personal app for each
configured recipient at startup (via `preload = true`). Users do not need to install
anything themselves.

---

## Prerequisites

- Teleport Cloud cluster
- Azure subscription — someone with **Application Administrator** (or higher) access to
  grant Graph API admin consent
- Microsoft Teams — someone with **Teams Administrator** (or higher) access to upload
  the custom app
- Docker + Docker Compose (for local validation only)

> **Two admin domains are involved and are often owned by different people:**
> - Steps 1–3 (app registration, API permissions, admin consent) require an **Azure AD admin**
> - Steps 4–5 (Teams app upload, add to channel) require a **Teams admin**
>
> Identify both people before starting.

---

## Azure setup (required once)

### 1 — App registration

In [Azure Portal](https://portal.azure.com) → App registrations → New registration:

- Name: something like `teleport-msteams-plugin`
- Supported account types: Single tenant
- Redirect URI: leave blank

Note the **Application (client) ID** and **Directory (tenant) ID** from the Overview page.

Under **Certificates & secrets** → New client secret. Save the secret value immediately —
you cannot retrieve it again. Write it to `app-secret`:

```bash
echo -n '<client secret value>' > app-secret
```

### 2 — API permissions

Under **API permissions** → Add a permission → Microsoft Graph → Application permissions.

Add exactly these four permissions:

| Permission | Purpose |
|---|---|
| `AppCatalog.Read.All` | Find TeleBot in the org app catalog |
| `User.Read.All` | Resolve recipient email to Azure AD user ID |
| `TeamsAppInstallation.ReadWriteSelfForUser.All` | Auto-install TeleBot for DM recipients + get chat ID |
| `TeamsAppInstallation.ReadWriteSelfForTeam.All` | Check TeleBot installation in teams (for channel posting) |

After adding all four, click **Grant admin consent for \<tenant\>**. This requires
**Application Administrator**, **Cloud Application Administrator**, or **Global Admin**.
The status column should show green checkmarks for all four.

> All four use the **Self** variant — the plugin can only manage its own app's installation,
> never any other app. This is the least-privilege set.

### 3 — Azure Bot resource

In Azure Portal → Azure Bot → Create:

- Bot handle: any name (e.g. `teleport-msteams-bot`)
- Type: Single tenant
- Microsoft App ID: select **Use existing app registration** → enter the App ID from step 1

Once created, go to **Channels** → add **Microsoft Teams**. Accept the terms of service.

This wires the Bot Framework messaging endpoint to your app registration, which is what
allows the plugin to send cards via the Bot API.

### 4 — Generate and patch the Teams app package

Generate `app.zip` using the plugin's Docker image (no binary install needed):

```bash
mkdir assets
source .env
docker run --rm \
  -v "$(pwd)/assets:/tmp/assets" \
  public.ecr.aws/gravitational/teleport-plugin-msteams:${TELEPORT_VERSION} \
  configure /tmp/assets \
  --appID  "<AZURE_APP_ID>" \
  --tenantID "<AZURE_TENANT_ID>" \
  --appSecret "<client secret value>"
```

This outputs `assets/teleport-msteams.toml` (initial plugin config) and `assets/app.zip`
(the Teams app package). Note the `teams_app_id` value from the generated TOML — you will
need it later.

> **Not idempotent:** each run generates a new `teams_app_id` UUID. The config and zip
> are a matched pair. If you regenerate one, you must regenerate both and re-upload.

**Patch the manifest for personal scope (required for DM delivery):**

The generated manifest only includes `"scopes": ["team"]`, which enables channel posting
but not DMs. Without adding `"personal"`, DM delivery silently fails regardless of permissions.

```bash
cd assets
unzip app.zip -d app-unpacked

# Edit app-unpacked/manifest.json:
#   1. If updating an existing Teams app: set "id" to your existing teams_app_id
#      (so Teams Admin Center recognizes this as an update, not a new app)
#   2. Change "scopes": ["team"]  →  "scopes": ["team", "personal"]
#   3. Bump "version" (e.g. "1.0.0" → "1.0.1") so Teams Admin Center accepts the upload

cd app-unpacked
zip -j ../app-patched.zip color.png manifest.json outline.png
cd ../..
```

### 5 — Upload to Teams Admin Center and add to a channel

This step requires a **Teams Administrator** or **Global Admin**.

1. [Teams Admin Center](https://admin.teams.microsoft.com) → Teams apps → Manage apps →
   **Upload new app** → select `assets/app-patched.zip`
2. The app appears as "TeleBot" in the org app catalog. If your org requires admin
   approval for custom apps, approve it from the same page.
3. In Teams: open the target team → Apps → search "TeleBot" → **Set up a bot** → Add
   to the team's General channel.

Adding TeleBot to General grants it permission to post to all channels in that team.
Repeat step 3 for each team you want to receive channel notifications.

---

## Enable the Cloud-hosted plugin

With Azure and Teams configured, enable the plugin in your Teleport cluster:

1. Teleport Web UI → Integrations → Microsoft Teams → Enroll
2. Enter the values from the Azure setup:
   - Application (client) ID → `<AZURE_APP_ID>`
   - Directory (tenant) ID → `<AZURE_TENANT_ID>`
   - App secret → the value from `app-secret`
   - Teams app ID → `<TEAMS_APP_ID>` from the `configure` output
3. The plugin shows **Healthy** once it can reach the Graph API and find TeleBot in the
   org catalog.

The Cloud-hosted plugin manages its own identity and credentials. No tbot, no Docker, no
infrastructure to maintain.

### Routing with Access Monitoring Rules

The Cloud plugin supports a single default recipient. For per-role routing to multiple
channels or users, use Access Monitoring Rules. The plugin name is `msteams`.

```yaml
kind: access_monitoring_rule
version: v1
metadata:
  name: msteams-channel-example
spec:
  subjects: [access_request]
  condition: 'access_request.spec.roles.contains("<ROLE_NAME>")'
  notification:
    name: msteams
    recipients:
      - "<TEAMS_CHANNEL_URL>"   # channel
      - "user@example.com"      # user DM (primary mail only)
```

```bash
tctl create -f access-monitoring-rule.yaml
```

AMR recipients override the default recipient when the condition matches. The `contains`
predicate requires an exact role name match.

> DM recipients must match the primary `mail` field in Azure AD — not proxy addresses or
> `.onmicrosoft.com` aliases. Use the primary mail or the user's Azure AD object ID.

---

## Validate with Docker Compose

Use the Docker Compose setup to confirm your Azure configuration is correct before or
instead of using the Cloud plugin. This is also the easiest way to debug delivery issues.

### Teleport setup (validation only)

```bash
# Create the bot using the built-in access-plugin role — no custom role needed
tctl create -f rbac.yaml

# Generate a single-use join token
tctl bots instances add msteams-plugin --format=json | jq -r '.token_id' > token
```

> The token is single-use. tbot consumes it on first join and stores its identity in the
> `tbot-state` Docker volume. As long as that volume persists, no new token is needed.
> Generate a new one only after a volume wipe (`docker compose down -v`).

### Values to fill in

Collect these before editing config files:

| Placeholder | Where to find it |
|---|---|
| `<TELEPORT_PROXY>` | Your cluster address, e.g. `acme.teleport.sh:443` |
| `<AZURE_APP_ID>` | Azure Portal → App registrations → your app → Application (client) ID |
| `<AZURE_TENANT_ID>` | Azure Portal → App registrations → your app → Directory (tenant) ID |
| `<TEAMS_APP_ID>` | Output of the `configure` command in step 4 |
| `<TEAMS_CHANNEL_URL>` | Teams → right-click channel → Get link to channel |

To replace all placeholders at once:

```bash
export TELEPORT_PROXY="acme.teleport.sh:443"
export AZURE_APP_ID="..."
export AZURE_TENANT_ID="..."
export TEAMS_APP_ID="..."       # from assets/teleport-msteams.toml after step 4
export TEAMS_CHANNEL_URL="..."  # after uploading app in step 5

sed -i '' \
  -e "s|<TELEPORT_PROXY>|${TELEPORT_PROXY}|g" \
  -e "s|<AZURE_APP_ID>|${AZURE_APP_ID}|g" \
  -e "s|<AZURE_TENANT_ID>|${AZURE_TENANT_ID}|g" \
  -e "s|<TEAMS_APP_ID>|${TEAMS_APP_ID}|g" \
  -e "s|<TEAMS_CHANNEL_URL>|${TEAMS_CHANNEL_URL}|g" \
  plugin.toml tbot.yaml
```

> `sed -i ''` is macOS syntax. On Linux, use `sed -i` (no trailing `''`).

### Files

| File | Purpose |
|---|---|
| `docker-compose.yaml` | tbot + msteams plugin services |
| `tbot.yaml` | tbot machine identity config |
| `plugin.toml` | Plugin config (proxy addr, Azure credentials, recipients) |
| `rbac.yaml` | Teleport bot definition |
| `.env` | Image version (`TELEPORT_VERSION`) |
| `token` | Join token — gitignored, single-use |
| `app-secret` | Azure app client secret — gitignored |

### Run

```bash
docker compose up
```

tbot joins on first run and writes the identity file to the shared `identity` volume. The
msteams container restarts until the identity is available, then connects and logs
"Plugin is ready".

### Validate DM delivery

```bash
source .env
docker run --rm \
  -v "$(pwd)/plugin.toml:/etc/teleport-msteams.toml:ro" \
  -v "$(pwd)/app-secret:/etc/plugin/app-secret:ro" \
  public.ecr.aws/gravitational/teleport-plugin-msteams:${TELEPORT_VERSION} \
  validate --config=/etc/teleport-msteams.toml user@example.com
```

> If you see `Message sent, ID: ...` in the output, the DM was delivered successfully.
> The command may exit non-zero — check for that log line to confirm delivery.

### Upgrading

```bash
# Update TELEPORT_VERSION in .env, then:
docker compose pull && docker compose up -d
```
