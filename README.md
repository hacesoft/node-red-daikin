[🇨🇿 Česky](README_CZ.md) | [🇬🇧 **English**](README.md)

# Daikin ONECTA Module for Node-RED

A monitoring module that reads Daikin climate units through the public **ONECTA Cloud API**. It was developed as part of **LINEA / GridSight** and can also be used in a standalone Node-RED installation.

The module retrieves all units available to one ONECTA account, displays them in Dashboard 2.0, manages OAuth tokens, and prepares a restricted snapshot for LINEA's read-only API. The supplied flow **does not control the units**: it sends no command to switch them on, change their operating mode, or adjust the temperature.

## Contents

- [Displayed data](#displayed-data)
- [Requirements](#requirements)
- [Daikin Developer Portal registration](#daikin-developer-portal-registration)
- [Obtaining the first refresh token](#obtaining-the-first-refresh-token)
- [Installation in LINEA](#installation-in-linea)
- [Standalone installation](#standalone-installation)
- [Dashboard configuration](#dashboard-configuration)
- [How the module works](#how-the-module-works)
- [Files and credential protection](#files-and-credential-protection)
- [Data interface](#data-interface)
- [API limits and the correct interval](#api-limits-and-the-correct-interval)
- [Known limitations of this export](#known-limitations-of-this-export)
- [Troubleshooting](#troubleshooting)
- [Post-installation checklist](#post-installation-checklist)

## Displayed data

Depending on the data provided by the particular unit and its cloud adapter, the module displays:

- unit name, on/off state, and cloud availability;
- heating, cooling, automatic, dry, or fan-only operation mode;
- room temperature, outdoor temperature, and temperature setpoint;
- fan speed and horizontal or vertical louver position;
- Powerful and Holiday modes;
- the next scheduled action and a grouped weekly schedule;
- consumption data when available;
- IP address, firmware, serial number, time zone, and service information;
- error state and error code.

Sensitive and service data is reduced in LINEA's public snapshot. OAuth tokens, the Client Secret, IP address, serial number, and weekly schedule are not exported.

Available fields vary between air conditioners, heat pumps, and firmware versions. The public `onecta:basic.integration` scope may expose fewer features than the ONECTA mobile application.

## Requirements

| Component | Requirement |
| --- | --- |
| Daikin | A unit supported by ONECTA, connected to the Internet, and visible in the mobile application. |
| Account | A working Daikin/ONECTA account with access to the units. |
| Developer Portal | Your own application in the [Daikin Developer Portal](https://developer.cloud.daikineurope.com/). |
| Node-RED | Internet access to `idp.onecta.daikineurope.com` and `api.onecta.daikineurope.com`. |
| Dashboard | `@flowfuse/node-red-dashboard`; the export lists version `1.30.2`. |
| Storage | A writable, persistent directory for `daikin_config.json` and `daikin_tokens.json`. |

First verify that every required unit is visible and controllable in the ONECTA application. The Developer API cannot return a device that is not linked to the signed-in account.

## Daikin Developer Portal registration

The portal interface can change over time, but you need to create an OAuth application using the **Onecta OIDC** authentication strategy.

### 1. Sign in to the portal

1. Open the [Daikin Developer Portal](https://developer.cloud.daikineurope.com/).
2. Sign in using **the same method and email address** that you use in the ONECTA application. For example, preserve the same social sign-in method when one is used by ONECTA.
3. Complete developer account registration and accept the current terms of use.
4. If the ONECTA Cloud API page offers a **Register for v1** button, complete that registration first. Daikin controls the availability and wording of these items.

Authentication can succeed with another Daikin account while the device endpoint returns an empty list.

### 2. Create the application and first credentials

1. Open your account menu and select **My Apps**.
2. Select **New App** or **+ New App**.
3. Enter values such as:

   - **Application name:** `LINEA Node-RED`
   - **Authentication strategy:** `Onecta OIDC`
   - **Redirect URI:** `https://example.com/daikin-callback`

4. Create the application.
5. The portal displays:

   - **Client ID** – the public application identifier;
   - **Client Secret** – the application's secret key.

6. Store both values securely. The Client Secret may be shown only once. If it is lost or regenerated, save the new Secret in Node-RED and normally repeat authorization to obtain a new refresh token.

The Redirect URI does not have to serve a working page. After login, the browser may display an error from `example.com`; the required `code` parameter remains in the address bar. **The Redirect URI must match character for character** when creating the application, requesting authorization, and exchanging the code, including the scheme, path, slashes, and letter case.

## Obtaining the first refresh token

The Client ID and Client Secret are not sufficient. The user must authorize the application once and exchange the one-time authorization code for the initial token set.

The procedure below is intended for Windows PowerShell. The application secret is entered as a hidden value and is not placed directly in command history.

### 1. Start authorization

Open PowerShell and paste the complete block:

```powershell
$ClientId = Read-Host "Daikin Client ID"
$ClientSecretSecure = Read-Host "Daikin Client Secret" -AsSecureString
$SecretPtr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($ClientSecretSecure)
$ClientSecret = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($SecretPtr)
$RedirectUri = "https://example.com/daikin-callback"
$OAuthState = [guid]::NewGuid().ToString("N")
$Scope = "openid onecta:basic.integration"
$AuthorizeUrl = "https://idp.onecta.daikineurope.com/v1/oidc/authorize" +
    "?response_type=code" +
    "&client_id=$([uri]::EscapeDataString($ClientId))" +
    "&redirect_uri=$([uri]::EscapeDataString($RedirectUri))" +
    "&scope=$([uri]::EscapeDataString($Scope))" +
    "&state=$([uri]::EscapeDataString($OAuthState))"
Start-Process $AuthorizeUrl
```

Use exactly the same `$RedirectUri` value that you registered in the portal.

### 2. Grant consent in the browser

1. Sign in with the same ONECTA account.
2. Select the consent checkbox that grants the application access to the devices and confirm it.
3. The browser attempts to open an address such as:

```text
https://example.com/daikin-callback?code=...&state=...
```

4. Even if the page ends with an error, copy the **complete URL from the address bar**. The authorization code is short-lived and single-use, so continue immediately.

### 3. Exchange the code for tokens

In the same PowerShell window, run:

```powershell
$CallbackUrl = Read-Host "Paste the complete callback URL"
$CodePart = ($CallbackUrl -split "[?&]" | Where-Object { $_ -like "code=*" } | Select-Object -First 1)
$StatePart = ($CallbackUrl -split "[?&]" | Where-Object { $_ -like "state=*" } | Select-Object -First 1)
if (-not $CodePart) { throw "The callback URL does not contain a code parameter." }
$AuthCode = [uri]::UnescapeDataString(($CodePart -replace "^code=", ""))
$ReturnedState = if ($StatePart) { [uri]::UnescapeDataString(($StatePart -replace "^state=", "")) } else { "" }
if ($ReturnedState -ne $OAuthState) { throw "OAuth state does not match. Repeat authorization." }
$TokenResponse = Invoke-RestMethod -Method Post `
    -Uri "https://idp.onecta.daikineurope.com/v1/oidc/token" `
    -ContentType "application/x-www-form-urlencoded" `
    -Body @{
        grant_type    = "authorization_code"
        client_id     = $ClientId
        client_secret = $ClientSecret
        code          = $AuthCode
        redirect_uri  = $RedirectUri
    }
$TokenResponse | Select-Object token_type, expires_in, scope
$TokenResponse.refresh_token | Set-Clipboard
Write-Host "The refresh token has been copied to the clipboard."
[Runtime.InteropServices.Marshal]::ZeroFreeBSTR($SecretPtr)
$ClientSecret = $null
```

When the request succeeds, the refresh token is on the clipboard. Do not place token output in an issue, screenshot, GitHub repository, or public log.

### 4. Enter the values in Node-RED

Open the **Daikin Onecta** card in the Dashboard and enter:

1. Client ID;
2. Client Secret;
3. Refresh Token from the clipboard.

Select **Ulozit konfiguraci** (Save configuration). The Refresh Token field intentionally clears after sending. The flow saves the token to a file, forces the first refresh, and retrieves the devices when successful.

The **Otestovat token** (Test token) button does not create the first refresh token. It only tries to refresh an access token using a refresh token that must already be stored.

### Replacing a lost or revoked token

Repeat the complete authorization procedure after an `invalid_grant` error, a Client Secret change, revoked consent, or loss of `daikin_tokens.json`. Never reuse an authorization code.

## Installation in LINEA

1. Back up the current flow and configuration files before importing.
2. Check whether the `MODULE::DAIKIN::KLIMATIZACE` group already exists. It is already part of the supplied complete LINEA project.
3. When updating, replace the existing module. Do not run two copies because they would share tokens, global keys, and the API request budget.
4. Preserve the `RESET` connection from LINEA's main startup sequence to the Daikin link-in node.
5. Verify that LINEA has set `global.sValid_Patch` to a valid directory. The value must end with `/` because the module concatenates the path and filename directly.
6. Check that `Daikin Config UI` is assigned to the configuration page and `Klimatizace - karty` to the operational page.
7. Verify that the default polling interval is **8 minutes** and that the configuration does not contain a lower value. Details are in the rate limit section.
8. Complete registration, enter the credentials, and verify a Node-RED restart.

### LINEA API integration

The parser creates `global.lineaApiClimateState`. In the supplied LINEA project, `Build status API 1.0.0` includes it in the `GET /api/v1/status` response as:

```text
climate.available
climate.data.updatedAt
climate.data.devices[]
```

The standalone export contains no HTTP endpoint. The snapshot does not expose itself to the network.

## Standalone installation

### 1. Import and Dashboard

1. Install Dashboard 2.0 (`@flowfuse/node-red-dashboard`).
2. Import `daikin_flows_19092026_1853.json`.
3. The export does not include its own `tab`; nodes are inserted into the selected flow tab.
4. Check the imported Dashboard configuration. The export includes the `/dashboard` base, `/FVE` and `/config` pages, a theme, and groups. In an existing project, you can assign the templates to your own pages and remove unused duplicate configuration nodes.

### 2. Persistent directory and startup trigger

The standalone export depends on `global.sValid_Patch` and LINEA's external `RESET` link. Without changes, it may not load its files automatically after a restart.

Create a persistent directory such as `/data/daikin/`. In a container, place it on a persistent volume. Add an Inject configured as **once after 2 seconds** and connect it to a Function node containing:

```javascript
const sDaikinPath = "/data/daikin/";
global.set("sValid_Patch", sDaikinPath);
msg.payload = { reset: true, source: "standalone-startup" };
return msg;
```

Connect its output to the existing ten-second Delay node in the Daikin group. The Delay then triggers both file reads and populates the configuration UI.

The path must end with `/`. Without it, the module creates a path such as `/data/daikindaikin_tokens.json`.

On the first startup, the files do not exist yet, and the `file in` nodes may report an error. Saving credentials in the Dashboard creates them. Attach a Catch node to the file nodes if you need to distinguish an expected first startup from a real read error.

### 3. API interval

The corrected flow uses a default interval of **8 minutes** (`pollMinutes: 8`). The value is configured in the Dashboard and is used for subsequent scheduling after it is saved. Eight minutes is the permitted minimum; use 9 or 10 minutes for a larger API limit margin.

### 4. Completion

Open the Dashboard configuration, enter the OAuth credentials, save them, and watch the status below `Token manager`, `Uloz nove tokeny`, and `Parse klimatizace`. After the first success, restart Node-RED and verify automatic loading.

## Dashboard configuration

| Field / button | Meaning |
| --- | --- |
| Client ID | Application identifier from the Developer Portal. |
| Client Secret | Secret application key. Stored in `daikin_config.json`. |
| Refresh Token | Entered during initial setup or recovery from `invalid_grant`; it is not displayed again in the UI. |
| Poll interval (min) | Device polling interval. The default and minimum value is **8 minutes**; changes are stored and used by the scheduler. |
| Otestovat token | Forces a refresh of the stored token and, when successful, also performs a device GET. |
| Ulozit konfiguraci | Saves configuration; when a refresh token is entered, writes the token file and starts a refresh. |

The configuration UI uses Czech labels. The Client Secret and refresh token are not displayed as readable token status when the form is reopened; however, the saved Client Secret is sent back to the password input as its value.

## How the module works

### Startup

After the startup `RESET`, there is a ten-second delay. The flow then independently loads:

- `daikin_config.json` into `global.config.daikinConfig`;
- `daikin_tokens.json` into `global.daikinTokens`;
- current status into the configuration widget.

`Daikin startup sync` handles either file order. It starts a refresh only after the Client ID, Client Secret, and refresh token are available. A fifteen-second guard prevents a duplicate startup refresh.

### Tokens

`Token manager`:

1. uses an access token when more than 5 minutes of its lifetime remain;
2. otherwise sends a refresh request to `https://idp.onecta.daikineurope.com/v1/oidc/token`;
3. blocks a concurrent refresh for 60 seconds;
4. stores the new access token and any rotated refresh token;
5. writes the complete set to `daikin_tokens.json`;
6. sends one GET to `https://api.onecta.daikineurope.com/v1/gateway-devices`.

The code assumes an approximately three-hour access token lifetime when the server does not provide `expires_in`. A separate Inject calls the manager every 2.5 hours. When the access token is still valid, the tick does not refresh it but still performs a device GET.

### Device parsing

The parser looks for `climateControl`, `gateway`, `indoorUnit`, and `outdoorUnit` management points. A device without `climateControl` is not added to the Dashboard. It combines telemetry, schedule, energy, and service data into one object per unit.

On HTTP 401, it marks the access token as expired but does not retry immediately. The next poll refreshes the token. Other HTTP errors appear as an unexpected response.

## Files and credential protection

| File | Contents |
| --- | --- |
| `daikin_config.json` | `clientId`, **`clientSecret`**, and `pollMinutes`. |
| `daikin_tokens.json` | **`refresh_token`**, `access_token`, `access_expires`, and any other fields preserved from the token response. |

Both files contain plaintext secrets. Therefore:

- do not add them to a Git repository, release archive, or screenshot;
- add them to `.gitignore`;
- restrict permissions to the account running Node-RED;
- back them up only to protected storage;
- never share access or refresh tokens;
- after suspected disclosure, revoke the application or its Secret in the portal and authorize it again.

Recommended `.gitignore` entries:

```gitignore
daikin_config.json
daikin_tokens.json
```

The flow export itself contains no credential values unless they were manually written directly into a node.

## Data interface

### Context values

| Context | Meaning |
| --- | --- |
| `global.config.daikinConfig` | Client ID, Client Secret, and stored `pollMinutes`. |
| `global.daikinTokens` | Active token set. |
| `global.daikinTokenState` | UI state: refresh token availability, access token expiry, and latest error. |
| `global.daikinRefreshInFlight` | Timestamp lock preventing a concurrent refresh. |
| `global.lineaApiClimateState` | Restricted snapshot for the LINEA API. |
| `global.lineaApiClimateFwVersions` | Latest known firmware versions for change detection. |

### Dashboard output

Each item sent to `Klimatizace - karty` contains fields such as:

```json
{
  "id": "device-id",
  "device_name": "Living room",
  "cloud_up": true,
  "on_off": "on",
  "mode": "heating",
  "room_temp": 22.3,
  "outdoor_temp": 8.1,
  "setpoint": 23,
  "fan_speed": "auto",
  "error_code": "00-",
  "in_error": false,
  "updated": "18:30:00"
}
```

The object may also contain `schedule`, `energy`, `next_action`, `info`, setpoint limits, and louver directions.

### LINEA snapshot

`global.lineaApiClimateState` has the following structure:

```json
{
  "updatedAt": "2026-09-20T16:30:00.000Z",
  "devices": [
    {
      "name": "Living room",
      "cloudUp": true,
      "on": true,
      "operationMode": "heating",
      "roomTemperatureC": 22.3,
      "outdoorTemperatureC": 8.1,
      "setpointC": 23,
      "energy": null,
      "error": false,
      "errorCode": "00-",
      "firmwareVersion": "1.2.3",
      "firmwareChanged": false
    }
  ]
}
```

`operationMode` represents the configured operating mode. It does not prove that the compressor is physically running at that moment. `firmwareChanged` is not set during the first load after startup; it is set only after a change from a previously known version in the running context.

The snapshot remains stored after an API error. Consumers must check its age through `updatedAt`.

## API limits and the correct interval

Daikin states a default limit of **200 API requests per rolling 24 hours** and **20 requests per minute** for normal applications. One `GET /v1/gateway-devices` retrieves all units in the account, so the number of units does not by itself increase the polling request count. Always check the current limit and terms in the [Developer Portal](https://developer.cloud.daikineurope.com/).

The corrected flow uses a default polling interval of **8 minutes**:

- the main poll uses up to approximately 180 GET requests per 24 hours;
- the token tick runs every 2.5 hours and may also perform a GET when the token is valid;
- startup refreshes and manual tests add more requests;
- the interval can be configured through `pollMinutes`, with a minimum of 8 minutes.

Eight minutes is the intended default, but it leaves only a small margin against the daily limit. If Node-RED is restarted frequently, manual tests are used, or other software shares the same OAuth application, configure **9 to 10 minutes** instead. A ten-minute main poll uses 144 requests per 24 hours.

When the limit is reached, expect HTTP `429 Too Many Requests`. Repeated manual testing makes the situation worse; wait for capacity to return within the rolling window.

The cloud API is unsuitable for a fast PV or ESS control loop. Data has a multi-minute interval and cloud latency. Reactive control requires a local interface supported by the specific device.

## Known limitations of this export

1. **The module is read-only.** Despite wording in the UI help, it contains no request for changing temperature, mode, or power state.
2. **The eight-minute interval leaves only a small API limit margin.** Manual tests, restarts, or other software sharing the application can exhaust the daily budget.
3. **The standalone export has no complete automatic startup.** Its `RESET` waits for a link from LINEA, and the unnamed Inject is not configured to run automatically after startup.
4. **There is no freshness watchdog.** On an error, the Dashboard and `lineaApiClimateState` keep the latest successful data.
5. **HTTP 401 is not retried immediately.** The token is marked as expired and recovery occurs on the next trigger.
6. **Other HTTP errors have no detailed parser or automatic backoff.** The flow does not distinguish 403, 429, and 5xx errors in the user interface.
7. **Multiple copies are not isolated.** They share the same global keys, files, and OAuth application.
8. **Node-RED does not create the first refresh token.** It must be obtained through the manual OAuth authorization procedure above.
9. **Energy data is an interpretation of API fields.** Availability and the time meaning of the `d`, `w`, and `m` series depend on the model and Daikin response; values are not a substitute for a calibrated energy meter.

## Troubleshooting

| Symptom | Likely cause and check |
| --- | --- |
| Developer Portal does not show units | The portal does not normally list them. Verify them in ONECTA and use the same account and sign-in method. |
| API returns an empty `[]` | Different account/sign-in method, consent not granted, or devices are not linked to that account. |
| Redirect URI mismatch | The URI in the portal, authorization URL, and token request are not identical character for character. |
| Authorization page reports an error | Check Client ID, `response_type=code`, scope, registered URI, and any required Cloud API v1 registration. |
| `invalid_grant` while exchanging the code | The code expired, was used twice, or the Redirect URI differs. Start a new authorization. |
| `invalid_grant` while refreshing | The token was revoked, replaced by a rotated token, lost during concurrent refresh, or belongs to another Client ID/Secret. Obtain a new one. |
| `invalid_client` | Incorrect Client ID or Secret; check whitespace and whether the Secret was changed in the portal. |
| HTTP 401 from gateway-devices | Access token expired or is invalid; the next trigger attempts a refresh. |
| HTTP 403 | Account or application lacks permission, consent, or API registration. |
| HTTP 429 | Request budget exhausted. Configure 10 minutes and wait for the rolling 24-hour window to release capacity. |
| `ENOTFOUND`, timeout | DNS, Internet connection, or outbound HTTPS from the Node-RED container. |
| Files are written to the wrong location | `global.sValid_Patch` is empty or does not end with `/`. |
| Configuration is missing after restart | `RESET` did not run, the directory is not persistent, or files are unreadable. |
| Token works but cards are empty | The response contains no `climateControl`, the account has no units, or the parser encountered another format. Temporarily inspect `FULL_JSON`, but remove sensitive data before sharing it. |
| Dashboard shows old values | Check the `updated` time, `global.lineaApiClimateState.updatedAt`, HTTP node status, and API limit. |

Cloud service status is available at [Daikin Cloud Solutions Status](https://daikincloudsolutions.statuspage.io/).

## Post-installation checklist

- [ ] Units are visible in the ONECTA mobile application.
- [ ] The developer account uses the same account and sign-in method.
- [ ] The application uses Onecta OIDC and contains the exact Redirect URI.
- [ ] The Client Secret and tokens are absent from Git.
- [ ] The first authorization code was successfully exchanged for a refresh token.
- [ ] `global.sValid_Patch` points to a persistent directory and ends with `/`.
- [ ] `daikin_config.json` and `daikin_tokens.json` exist and Node-RED loads them after restart.
- [ ] Polling defaults to 8 minutes, and the stored `pollMinutes` value is actually used.
- [ ] `Token manager` reports a valid access token and the parser retrieves the expected number of units.
- [ ] Dashboard time and `global.lineaApiClimateState.updatedAt` update regularly.
- [ ] Restarting Node-RED does not cause `invalid_grant`, and automatic loading works.