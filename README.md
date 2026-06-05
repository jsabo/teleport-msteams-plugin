# teleport-msteams-plugin

Setup guide for the [Teleport MS Teams access request plugin](https://goteleport.com/docs/identity-governance/access-requests/plugins/msteams/). Covers Azure Bot and Teams app configuration, Cloud plugin enrollment, and recipient routing via Access Monitoring Rules.

The Docker Compose setup at the bottom of this guide is a troubleshooting tool — use it if the Cloud plugin isn't delivering notifications and you need to isolate whether the issue is Azure permissions, the Teams app installation, or something else.

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

## 5 — Generate and patch the Teams app package

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

**Copy the `TeamsAppID`** — you will need it when enrolling the plugin.

**Patch the manifest to enable DM delivery:**

The generated manifest only includes `"scopes": ["team"]`. Without adding `"personal"`, DM delivery silently fails regardless of permissions.

```bash
cd assets && unzip -q app.zip -d app-unpacked
```

Open `assets/app-unpacked/manifest.json` and make two changes:

1. Find `"scopes": ["team"]` inside the `bots` array and change it to `"scopes": ["team", "personal"]`
2. Bump the `"version"` field by one patch (e.g. `"1.0.0"` → `"1.0.1"`)

Then repack:

```bash
cd assets/app-unpacked && zip -j ../app-patched.zip color.png manifest.json outline.png && cd ../..
echo "Ready to upload: assets/app-patched.zip"
```

---

## 6 — Upload to Teams Admin Center

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

This installs TeleBot in the team, which is required before the plugin can post to any
standard channel in that team. General is just the installation point — notifications go to
whichever channel URL is configured in your recipients. Repeat for each team you want to
receive channel notifications.

> **Note:** Private channels are not supported. TeleBot installed at the team level can
> only post to standard channels.

---

## Enable the Cloud-hosted plugin

With Azure and Teams configured, enroll the plugin in your Teleport cluster:

**Teleport Web UI → Integrations → Microsoft Teams → Enroll**

Enter the values collected during setup:

| Field | Value |
|---|---|
| App ID | `<AZURE_APP_ID>` |
| Tenant ID | `<AZURE_TENANT_ID>` |
| TeamsApp ID | `<TeamsAppID>` from the `configure` output |
| App secret | your client secret value |
| Default Recipient | a Teams channel URL (Teams → right-click channel → Get link to channel) |

The plugin shows **Active** once it can reach the Graph API and find TeleBot in the org catalog. The Cloud-hosted plugin manages its own identity — no tbot, no Docker, no infrastructure to maintain.

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

> DM recipients must match the primary `mail` field in Azure AD — not proxy addresses or
> `.onmicrosoft.com` aliases.

---

## Troubleshooting with Docker Compose

If the Cloud plugin isn't delivering notifications, use this setup to validate your Azure configuration end-to-end. It runs the plugin locally against your cluster so you can see exactly what's happening.

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

### Generate plugin.toml

After running `configure` (step 5), generate `plugin.toml` from the output and fix the Docker paths:

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

### Files

| File | Purpose |
|---|---|
| `docker-compose.yaml` | tbot + msteams plugin services |
| `tbot.yaml` | tbot machine identity config — update `<TELEPORT_PROXY>` |
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
