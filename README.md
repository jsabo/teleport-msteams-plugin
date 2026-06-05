# teleport-msteams-plugin

Setup guide for the [Teleport MS Teams access request plugin](https://goteleport.com/docs/identity-governance/access-requests/plugins/msteams/). Covers Azure Bot configuration, Cloud plugin enrollment, Teams app upload, and recipient routing via Access Monitoring Rules.

The Docker Compose setup at the bottom of this guide is a troubleshooting tool — use it if the Cloud plugin isn't delivering notifications and you need to isolate whether the issue is Azure permissions, the Teams app installation, or the plugin itself.

---

## Prerequisites

- Teleport Cloud cluster
- Azure subscription — someone with **Application Administrator** (or higher) to grant Graph API admin consent
- Microsoft Teams — someone with **Teams Administrator** (or higher) to upload the custom app

> **Two admin domains are involved and are often owned by different people:**
> - Steps 1–4 (Azure Bot, API permissions, admin consent) require an **Azure AD admin**
> - Steps 5–6 (Teams app upload, add to channel) require a **Teams admin**
>
> Identify both people before starting.

---

## Azure setup (required once)

### 1 — Create an Azure Bot

Go to [Create an Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot) in the Azure Portal and fill in the **Basics** tab:

- **Bot handle**: `teleport-msteams-plugin` (or any unique name)
- **Subscription** and **Resource group**: use your existing subscription; create a new resource group named after the bot handle
- **Data residency**: Global
- **Pricing tier**: Standard
- **Type of App**: Single Tenant
- **Creation type**: Create new Microsoft App ID

Click **Review + create**, then **Create**. Azure automatically creates an app registration alongside the bot — you do not need to create one separately.

### 2 — Note your App ID and Tenant ID

Once deployed, open the bot resource → **Settings** → **Configuration**:

- **Microsoft App ID** — this is your `<AZURE_APP_ID>`
- **App Tenant ID** — this is your `<AZURE_TENANT_ID>`

Save both values.

### 3 — Create a client secret

From the Configuration page, click **Manage Password** next to the Microsoft App ID. This opens the app registration's **Certificates & secrets** page.

Click **+ New client secret**:
- **Description**: `teleport-msteams-plugin`
- **Expires**: 730 days (24 months)

Click **Add**. Copy the **Value** column immediately — it is only shown once. Save it somewhere secure; you will need it when enrolling the plugin.

### 4 — API permissions

Still in the app registration (reached via **Manage Password**), click **API permissions** in the left sidebar under **Manage**.

Click **+ Add a permission** → **Microsoft Graph** → **Application permissions**.

Use the search box to find and add these four permissions one at a time, clicking **Add permissions** after each:

| Permission | Purpose |
|---|---|
| `AppCatalog.Read.All` | Used to list Teams apps and check if the app is installed |
| `User.Read.All` | Used to get notification recipients |
| `TeamsAppInstallation.ReadWriteSelfForUser.All` | Used to initiate communication with a user that never interacted with the Teams app before |
| `TeamsAppInstallation.ReadWriteSelfForTeam.All` | Used to discover if the app is installed in the team |

Once all four appear in the **Configured permissions** table, click **Grant admin consent for \<tenant\>**. This requires **Application Administrator**, **Cloud Application Administrator**, or **Global Admin**. All four should show a green **Granted for \<tenant\>** status.

> All four use the **Self** variant — the plugin can only manage its own app's installation, never any other app. This is the least-privilege set.

---

## Enable the Cloud-hosted plugin

> See the [official Teleport MS Teams plugin docs](https://goteleport.com/docs/identity-governance/access-requests/plugins/msteams/) for full reference.

**Teleport Web UI → Integrations → Microsoft Teams → Enroll**

Enter the values from the Azure setup:

| Field | Value |
|---|---|
| App ID | `<AZURE_APP_ID>` |
| Tenant ID | `<AZURE_TENANT_ID>` |
| App secret | your client secret value |
| TeamsApp ID | leave blank — Teleport generates one |
| Default Recipient | a Teams channel URL (Teams → right-click channel → **Get link to channel**) |

Click **Connect Microsoft Teams**. The plugin will show as **Failed** until the Teams app is uploaded in the next steps — that is expected.

---

## Upload the Teams app

### 5 — Download and patch the app package

After enrolling, go to **Integrations → Microsoft Teams → Options → Download app.zip**.

Teleport generates an `app.zip` containing the Teams app manifest. The manifest only includes `"scopes": ["team"]` — without adding `"personal"`, DM delivery silently fails regardless of permissions. Patch it before uploading:

```bash
unzip -q ~/Downloads/app.zip -d ~/Downloads/app-unpacked
```

Open `~/Downloads/app-unpacked/manifest.json` and make these changes:

1. Find `"scopes": ["team"]` inside the `bots` array and change it to `"scopes": ["team", "personal"]`
2. Bump the `"version"` field only if you are **re-uploading** an existing app — Teams Admin Center rejects uploads where the version hasn't changed. For a **first upload**, leave the version as-is.

Then repack:

```bash
cd ~/Downloads/app-unpacked && zip -j ../app-patched.zip color.png manifest.json outline.png
echo "Ready to upload: ~/Downloads/app-patched.zip"
```

### 6 — Upload to Teams Admin Center and add to a channel

This step requires a **Teams Administrator** or **Global Admin**.

1. [Teams Admin Center](https://admin.teams.microsoft.com) → Teams apps → Manage apps →
   **Upload new app** → select `~/Downloads/app-patched.zip`
2. The app appears as "TeleBot" in the org app catalog. If your org requires admin
   approval for custom apps, approve it from the same page.
3. In [Azure Portal](https://portal.azure.com), open the bot resource → **Settings** →
   **Channels** → add **Microsoft Teams**. Accept the terms of service. This connects the
   bot to the Teams Bot Framework so it can send cards. Once added, an **Open in Teams**
   link appears next to the Microsoft Teams channel.
4. Click **Open in Teams**. This opens TeleBot's app card directly in Teams. Click **Add**,
   then select the General channel of your target team.

> **Do not search for "TeleBot" in the Teams app store** — the search returns public apps
> only and will not find your org app. Use the **Open in Teams** link from the Azure Portal.

This installs TeleBot in the team, which is required before the plugin can post to any
standard channel in that team. General is just the installation point — notifications go to
whichever channel URL is configured in your recipients. Repeat for each team you want to
receive channel notifications.

> **Note:** Private channels are not supported. TeleBot installed at the team level can only post to standard channels.

Once TeleBot is in the org catalog and installed in a team, the plugin status will update.
The health check runs once at startup — if the plugin shows **Failed** after completing
these steps, trigger a recheck:

```bash
tctl get plugins --format text   # check current status
tctl edit plugin/msteams         # opens editor; save without changes to trigger restart
```

After saving, run `tctl get plugins --format text` again — status should show `RUNNING`.

---

### Routing with Access Monitoring Rules

The Cloud plugin routes all notifications to the Default Recipient unless an [Access Monitoring Rule](https://goteleport.com/docs/identity-governance/access-requests/plugins/msteams/) matches. Use AMRs for per-role routing to different channels or individual users via DM.

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

AMR recipients override the Default Recipient when the condition matches. The `contains` predicate requires an exact role name match.

> DM recipients must match the primary `mail` field in Azure AD — not proxy addresses or `.onmicrosoft.com` aliases.

---

## Troubleshooting with Docker Compose

If the Cloud plugin isn't delivering notifications, use this setup to validate your Azure configuration end-to-end. It runs the plugin locally so you can see exactly what's happening.
See the [official docs](https://goteleport.com/docs/identity-governance/access-requests/plugins/msteams/) for the full plugin reference.

### What you need

```bash
# Write your Azure app secret to a file (mounted into the plugin container)
echo -n '<client secret value>' > app-secret

# Create the bot identity in Teleport
tctl create -f rbac.yaml

# Generate a single-use join token for tbot
tctl bots instances add msteams-plugin --format=json | jq -r '.token_id' > token
```

> The token is single-use. Once tbot joins, it stores its identity in the `tbot-state`
> Docker volume and renews automatically. Generate a new token only after `docker compose down -v`.

### Generate app.zip and plugin.toml

The `configure` command generates a self-contained `app.zip` and TOML config. Use it when you want to run the plugin locally without enrolling via the Teleport UI:

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

On success:
```
[1] Created target directory: /workspace/assets
[2] Created /workspace/assets/app.zip

TeamsAppID: <generated-uuid>
```

Patch `assets/app.zip` the same way as the Cloud flow (add `"personal"` scope, bump version), then upload `assets/app-patched.zip` to Teams Admin Center.

Generate `plugin.toml` from the configure output:

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

> **Required:** Open `plugin.toml` and update `[role_to_recipients]` — replace `"*" = ["foo@example.com"]` with your Teams channel URL. The plugin will not deliver notifications without a valid recipient.

Also update `<TELEPORT_PROXY>` in `tbot.yaml` with your cluster address.

### Files

| File | Purpose |
|---|---|
| `docker-compose.yaml` | tbot + msteams plugin services |
| `tbot.yaml` | tbot machine identity config |
| `plugin.toml` | Plugin config — generated from configure output, gitignored |
| `rbac.yaml` | Teleport bot definition |
| `.env` | Image version (`TELEPORT_VERSION`) |
| `token` | Join token — gitignored, single-use |
| `app-secret` | Azure app client secret — gitignored |

### Run

```bash
docker compose up
```

tbot joins on first run and writes the identity to the shared `identity` volume. The msteams container restarts until the identity is available, then connects and logs "Plugin is ready".

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
