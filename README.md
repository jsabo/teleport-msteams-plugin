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
> - Steps 1–4 (Azure Bot, API permissions, admin consent) require an **Azure AD admin**
> - Steps 5–6 (Teams app upload, add to channel) require a **Teams admin**
>
> Identify both people before starting.

---

## Azure setup (required once)

### 1 — Create an Azure Bot

Go to [Create an Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot) in the Azure Portal and fill in the **Basics** tab:

- **Bot handle**: `teleport-msteams-plugin` (or any unique name — this is just a label)
- **Subscription** and **Resource group**: use your existing subscription; create a new resource group named after the bot handle
- **Data residency**: Global
- **Pricing tier**: Standard
- **Type of App**: Single Tenant
- **Creation type**: Create new Microsoft App ID

Click **Review + create**, then **Create**. Azure automatically creates an app registration alongside the bot resource — you do not need to create one separately.

### 2 — Note your App ID and Tenant ID

Once deployed, open the bot resource → **Settings** → **Configuration**:

- **Microsoft App ID** — this is your `<AZURE_APP_ID>`
- **App Tenant ID** — this is your `<AZURE_TENANT_ID>`

Save both values. You will need them for the plugin config and the `configure` command.

### 3 — Create a client secret

From the Configuration page, click **Manage Password** next to the Microsoft App ID. This opens the app registration's **Certificates & secrets** page.

Click **+ New client secret**:
- **Description**: `teleport-msteams-plugin`
- **Expires**: 730 days (24 months)

Click **Add**. Copy the **Value** column immediately — it is only shown once and cannot be retrieved again. Save it somewhere secure (password manager, notes); you will need it in several places later.

### 4 — API permissions

Still in the app registration (reached via **Manage Password** in the previous step), click
**API permissions** in the left sidebar under **Manage**.

Click **+ Add a permission** → **Microsoft Graph** → **Application permissions**.

A search box appears at the top of the permissions panel — use it, the full list is enormous.
Add these four permissions one at a time, clicking **Add permissions** after each:

| Permission | Purpose |
|---|---|
| `AppCatalog.Read.All` | Used to list Teams apps and check if the app is installed |
| `User.Read.All` | Used to get notification recipients |
| `TeamsAppInstallation.ReadWriteSelfForUser.All` | Used to initiate communication with a user that never interacted with the Teams app before |
| `TeamsAppInstallation.ReadWriteSelfForTeam.All` | Used to discover if the app is installed in the team |

Once all four appear in the **Configured permissions** table, click
**Grant admin consent for \<tenant\>** at the top of the table. This requires
**Application Administrator**, **Cloud Application Administrator**, or **Global Admin**.

All four should show a green **Granted for \<tenant\>** status. If the button is greyed
out, you do not have sufficient privileges — a tenant admin needs to complete this step.

> All four use the **Self** variant — the plugin can only manage its own app's installation,
> never any other app. This is the least-privilege set.

### 5 — Generate and patch the Teams app package

Generate `app.zip` using the plugin's Docker image (no binary install needed):

```bash
source .env
docker run --rm \
  -v "$(pwd):/workspace" \
  public.ecr.aws/gravitational/teleport-plugin-msteams:${TELEPORT_VERSION} \
  configure /workspace/assets \
  --appID  "<AZURE_APP_ID>" \
  --tenantID "<AZURE_TENANT_ID>" \
  --appSecret "<client secret value>"
```

> If you need to re-run `configure`, remove the output directory first: `rm -rf assets`

On success you will see:

```
[1] Created target directory: /workspace/assets
[2] Created /workspace/assets/app.zip

TeamsAppID: <generated-uuid>
```

Two files are written to `assets/`. Generate `plugin.toml` from the output and fix the Docker paths in one step:

**macOS:**
```bash
cp assets/teleport-msteams.toml plugin.toml
sed -i '' \
  -e 's|addr = "localhost:3025"|addr = "<TELEPORT_PROXY>"|' \
  -e 's|identity = "identity"|identity = "/identity/identity"|' \
  -e 's|# refresh_identity = true.*$|refresh_identity = true|' \
  -e 's|app_secret = ".*"|app_secret = "/etc/plugin/app-secret"|' \
  plugin.toml
```

**Linux:**
```bash
cp assets/teleport-msteams.toml plugin.toml
sed -i \
  -e 's|addr = "localhost:3025"|addr = "<TELEPORT_PROXY>"|' \
  -e 's|identity = "identity"|identity = "/identity/identity"|' \
  -e 's|# refresh_identity = true.*$|refresh_identity = true|' \
  -e 's|app_secret = ".*"|app_secret = "/etc/plugin/app-secret"|' \
  plugin.toml
```

Then open `plugin.toml` and replace `"*" = ["foo@example.com"]` under `[role_to_recipients]` with your Teams channel URL.

> **Not idempotent:** each run generates a new `TeamsAppID`. If you re-run `configure`, run `rm -rf assets` first and repeat this step.

**Patch the manifest to enable DM delivery:**

The generated manifest only includes `"scopes": ["team"]`. Without adding `"personal"`, DM delivery silently fails regardless of permissions.

```bash
cd assets && unzip -q app.zip -d app-unpacked
```

Open `assets/app-unpacked/manifest.json` in a text editor and make two changes:

1. Find `"scopes": ["team"]` inside the `bots` array and change it to `"scopes": ["team", "personal"]`
2. Bump the `"version"` field by one patch (e.g. `"1.0.0"` → `"1.0.1"`)

Then repack:

```bash
cd assets/app-unpacked && zip -j ../app-patched.zip color.png manifest.json outline.png && cd ../..
echo "Ready to upload: assets/app-patched.zip"
```

### 6 — Upload to Teams Admin Center

This step requires a **Teams Administrator** or **Global Admin**.

1. [Teams Admin Center](https://admin.teams.microsoft.com) → Teams apps → Manage apps →
   **Upload new app** → select `assets/app-patched.zip`
2. The app appears as "TeleBot" in the org app catalog. If your org requires admin
   approval for custom apps, approve it from the same page.
3. Back in [Azure Portal](https://portal.azure.com), open the bot resource → **Settings** →
   **Channels** → add **Microsoft Teams**. Accept the terms of service. This connects the
   bot to the Teams Bot Framework so it can send cards.
4. In Teams: open the target team → Apps → search "TeleBot" → **Set up a bot** → Add
   to the team's General channel.

Adding TeleBot to General grants it permission to post to all channels in that team.
Repeat step 4 for each team you want to receive channel notifications.

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

Write your Azure app secret to the `app-secret` file (this is mounted into the plugin container):

```bash
echo -n '<client secret value>' > app-secret
```

```bash
# Create the bot using the built-in access-plugin role — no custom role needed
tctl create -f rbac.yaml

# Generate a single-use join token
tctl bots instances add msteams-plugin --format=json | jq -r '.token_id' > token
```

> The token is single-use. tbot consumes it on first join and stores its identity in the
> `tbot-state` Docker volume. As long as that volume persists, no new token is needed.
> Generate a new one only after a volume wipe (`docker compose down -v`).

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
