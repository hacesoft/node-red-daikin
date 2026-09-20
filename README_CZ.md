[🇨🇿 **Česky**](README_CZ.md) | [🇬🇧 English](README.md) | [Linea project](https://github.com/hacesoft/Linea)

# Daikin ONECTA modul pro Node-RED

Monitorovací modul pro čtení klimatizačních jednotek Daikin z veřejného **ONECTA Cloud API**. Vznikl jako součást projektu **LINEA / GridSight**, ale lze jej použít také samostatně v Node-RED.

Modul načítá všechny jednotky dostupné pod jedním účtem ONECTA, zobrazuje je v Dashboardu 2.0, spravuje OAuth tokeny a připravuje omezený snapshot pro read-only API LINEA. Dodaný flow **klimatizaci neovládá**: neposílá příkazy pro zapnutí, změnu režimu ani teploty.

<img width="411" height="363" alt="image" src="https://github.com/user-attachments/assets/4cc1602b-d676-455a-bf89-f708542664d0" />

## Obsah

- [Co modul zobrazuje](#co-modul-zobrazuje)
- [Požadavky](#požadavky)
- [Registrace v Daikin Developer Portalu](#registrace-v-daikin-developer-portalu)
- [Získání prvního refresh tokenu](#získání-prvního-refresh-tokenu)
- [Instalace v LINEA](#instalace-v-linea)
- [Samostatná instalace](#samostatná-instalace)
- [Nastavení v Dashboardu](#nastavení-v-dashboardu)
- [Jak modul pracuje](#jak-modul-pracuje)
- [Soubory a ochrana přístupových údajů](#soubory-a-ochrana-přístupových-údajů)
- [Datové rozhraní](#datové-rozhraní)
- [Limity API a správný interval](#limity-api-a-správný-interval)
- [Známá omezení tohoto exportu](#známá-omezení-tohoto-exportu)
- [Řešení problémů](#řešení-problémů)
- [Kontrola po instalaci](#kontrola-po-instalaci)

## Co modul zobrazuje

Podle toho, co konkrétní jednotka a její cloudový adaptér poskytují, modul zobrazuje:

- název jednotky, stav zapnuto/vypnuto a dostupnost v cloudu;
- režim topení, chlazení, automatiku, odvlhčování nebo ventilátor;
- pokojovou a venkovní teplotu a požadovanou teplotu;
- rychlost ventilátoru a polohu vodorovných a svislých lamel;
- režimy Powerful a Holiday;
- další plánovanou akci a seskupený týdenní plán;
- údaje spotřeby, pokud je jednotka poskytuje;
- IP adresu, firmware, sériové číslo, časovou zónu a servisní údaje;
- chybový stav a kód chyby.

Ve veřejném snapshotu LINEA jsou citlivé a servisní údaje omezeny. Neexportují se OAuth tokeny, Client Secret, IP adresa, sériové číslo ani týdenní plán.

Dostupná pole se mezi klimatizacemi, tepelnými čerpadly a verzemi firmwaru liší. Veřejný scope `onecta:basic.integration` může poskytovat méně funkcí než mobilní aplikace ONECTA.

## Požadavky

| Součást | Požadavek |
| --- | --- |
| Daikin | Jednotka podporovaná aplikací ONECTA, připojená k internetu a viditelná v mobilní aplikaci. |
| Účet | Funkční Daikin/ONECTA účet, pod kterým jsou jednotky viditelné. |
| Developer Portal | Vlastní aplikace v [Daikin Developer Portalu](https://developer.cloud.daikineurope.com/). |
| Node-RED | Přístup k internetu na `idp.onecta.daikineurope.com` a `api.onecta.daikineurope.com`. |
| Dashboard | `@flowfuse/node-red-dashboard`; export uvádí verzi `1.30.2`. |
| Úložiště | Zapisovatelný a trvalý adresář pro `daikin_config.json` a `daikin_tokens.json`. |

Nejprve zkontrolujte, že všechny požadované jednotky vidíte a ovládáte v aplikaci ONECTA. Developer API nemůže vrátit zařízení, které není spojené s přihlášeným účtem.

## Registrace v Daikin Developer Portalu

Rozhraní portálu se může časem mírně změnit, ale potřebujete vytvořit OAuth aplikaci se strategií **Onecta OIDC**.

### 1. Přihlášení do portálu

1. Otevřete [Daikin Developer Portal](https://developer.cloud.daikineurope.com/).
2. Přihlaste se **stejným způsobem a stejnou e-mailovou adresou**, jakou používáte v aplikaci ONECTA. Pokud například v ONECTA používáte přihlášení přes sociální účet, zachovejte stejnou metodu.
3. Dokončete registraci vývojářského účtu a potvrďte aktuální podmínky používání.
4. Pokud portál u ONECTA Cloud API nabízí tlačítko **Register for v1**, dokončete nejprve tuto registraci. Dostupnost a názvy položek řídí Daikin.

Při použití jiného Daikin účtu může autentizace fungovat, ale endpoint vrátí prázdný seznam zařízení.

### 2. Vytvoření aplikace a prvního klíče

1. Otevřete menu svého účtu a zvolte **My Apps**.
2. Klikněte na **New App** nebo **+ New App**.
3. Zadejte například:

   - **Application name:** `LINEA Node-RED`
   - **Authentication strategy:** `Onecta OIDC`
   - **Redirect URI:** `https://example.com/daikin-callback`

4. Aplikaci vytvořte.
5. Portál zobrazí:

   - **Client ID** – veřejný identifikátor aplikace;
   - **Client Secret** – tajný klíč aplikace.

6. Obě hodnoty bezpečně uložte. Client Secret se může zobrazit pouze jednou. Když jej ztratíte nebo obnovíte, musíte v Node-RED uložit nový Secret a obvykle znovu provést autorizaci pro nový refresh token.

Redirect URI nemusí hostovat funkční stránku. Po přihlášení může prohlížeč zobrazit chybu webu `example.com`; potřebný parametr `code` zůstane v adresním řádku. **Redirect URI se musí při vytvoření aplikace, autorizaci i výměně kódu shodovat znak po znaku**, včetně `https`, cesty, lomítek a velikosti písmen.

## Získání prvního refresh tokenu

Client ID a Client Secret nestačí. Uživatel musí aplikaci jednou autorizovat a jednorázový autorizační kód vyměnit za první sadu tokenů.

Níže uvedený postup je určen pro Windows PowerShell. Heslo aplikace se zadává skrytě a nevkládá se přímo do historie příkazů.

### 1. Spuštění autorizace

Otevřete PowerShell a vložte celý blok:

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

Použijte stejnou hodnotu `$RedirectUri`, jakou jste zadali v portálu.

### 2. Souhlas v prohlížeči

1. Přihlaste se stejným ONECTA účtem.
2. Zaškrtněte souhlas s přístupem aplikace k zařízením a potvrďte jej.
3. Prohlížeč se pokusí otevřít například:

```text
https://example.com/daikin-callback?code=...&state=...
```

4. I když stránka skončí chybou, zkopírujte **celou URL z adresního řádku**. Autorizační kód je krátkodobý a jednorázový, proto pokračujte hned.

### 3. Výměna kódu za tokeny

Ve stejném okně PowerShellu spusťte:

```powershell
$CallbackUrl = Read-Host "Vlozte celou navratovou URL"
$CodePart = ($CallbackUrl -split "[?&]" | Where-Object { $_ -like "code=*" } | Select-Object -First 1)
$StatePart = ($CallbackUrl -split "[?&]" | Where-Object { $_ -like "state=*" } | Select-Object -First 1)
if (-not $CodePart) { throw "Navratova URL neobsahuje parametr code." }
$AuthCode = [uri]::UnescapeDataString(($CodePart -replace "^code=", ""))
$ReturnedState = if ($StatePart) { [uri]::UnescapeDataString(($StatePart -replace "^state=", "")) } else { "" }
if ($ReturnedState -ne $OAuthState) { throw "Nesouhlasi OAuth state. Autorizaci opakujte." }
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
Write-Host "Refresh token byl zkopirovan do schranky."
[Runtime.InteropServices.Marshal]::ZeroFreeBSTR($SecretPtr)
$ClientSecret = $null
```

Pokud příkaz uspěje, refresh token je ve schránce. Nevkládejte výstup tokenu do issue, screenshotu, GitHubu ani veřejného logu.

### 4. Vložení do Node-RED

V Dashboardu otevřete kartu **Daikin Onecta** a vložte:

1. Client ID;
2. Client Secret;
3. Refresh Token ze schránky.

Stiskněte **Uložit konfiguraci**. Pole Refresh Token se po odeslání vyprázdní úmyslně. Flow uloží token do souboru, vynutí první refresh a při úspěchu načte zařízení.

Tlačítko **Otestovat token** nevytváří první refresh token. Pouze se pokusí obnovit access token z refresh tokenu, který už musí být uložen.

### Obnovení ztraceného nebo zneplatněného tokenu

Při chybě `invalid_grant`, změně Client Secretu, odvolání souhlasu nebo ztrátě `daikin_tokens.json` zopakujte celou autorizaci. Nový autorizační kód nepoužívejte podruhé.

## Instalace v LINEA

1. Před importem zálohujte aktuální flow a konfigurační soubory.
2. Zkontrolujte, zda už existuje skupina `MODULE::DAIKIN::KLIMATIZACE`. V dodaném celkovém projektu LINEA již je.
3. Při aktualizaci nahraďte existující modul; nespouštějte dvě kopie, protože by sdílely tokeny, globální klíče a API limit.
4. Zachovejte propojení `RESET` z hlavní startovní sekvence LINEA do link-in uzlu Daikin.
5. Ověřte, že LINEA nastavila `global.sValid_Patch` na platný adresář. Hodnota musí končit `/`, protože modul spojuje cestu přímo s názvem souboru.
6. Zkontrolujte přiřazení `Daikin Config UI` na konfigurační stránku a `Klimatizace - karty` na provozní stránku.
7. Ověřte, že výchozí interval pollingu je nastaven na **8 minut** a v konfiguraci není zadána nižší hodnota. Podrobnosti jsou v části o limitech.
8. Proveďte registraci, vložte údaje a ověřte restart Node-RED.

### Vazba na API LINEA

Parser vytváří `global.lineaApiClimateState`. V dodané LINEA jej uzel `Build status API 1.0.0` vkládá do odpovědi `GET /api/v1/status` jako:

```text
climate.available
climate.data.updatedAt
climate.data.devices[]
```

Samostatný export neobsahuje HTTP endpoint. Snapshot sám o sobě nic nezpřístupní do sítě.

## Samostatná instalace

### 1. Import a Dashboard

1. Nainstalujte Dashboard 2.0 (`@flowfuse/node-red-dashboard`).
2. Importujte `daikin_flows_19092026_1853.json`.
3. Export neobsahuje vlastní `tab`; uzly se vloží do zvolené karty flow.
4. Zkontrolujte importované Dashboard konfigurace. Export přináší `/dashboard`, stránky `/FVE` a `/config`, téma a skupiny. Ve stávajícím projektu můžete šablony přiřadit vlastním stránkám a odstranit nepoužívané duplicitní konfigurace.

### 2. Trvalý adresář a startovací impuls

Samostatný export spoléhá na `global.sValid_Patch` a externí link `RESET` z LINEA. Beze změny se po restartu nemusí automaticky načíst soubory.

Vytvořte trvalý adresář, například `/data/daikin/`. V kontejneru musí být na persistentním svazku. Potom přidejte Inject nastavený na **once after 2 seconds**, připojte jej do Function uzlu:

```javascript
const sDaikinPath = "/data/daikin/";
global.set("sValid_Patch", sDaikinPath);
msg.payload = { reset: true, source: "standalone-startup" };
return msg;
```

Jeho výstup připojte do existujícího desetisekundového Delay uzlu ve skupině Daikin. Delay následně spustí načtení obou souborů a naplnění konfigurace UI.

Cesta musí končit `/`. Bez něj modul vytvoří například `/data/daikindaikin_tokens.json`.

Při prvním startu soubory ještě neexistují a `file in` uzly mohou hlásit chybu. Po uložení údajů v Dashboardu je modul vytvoří. Připojte Catch uzel k souborovým uzlům, pokud chcete rozlišit očekávaný první start od skutečné chyby čtení.

### 3. Interval API

Opravený flow používá výchozí interval **8 minut** (`pollMinutes: 8`). Hodnota se nastavuje v Dashboardu a po uložení se použije pro další plánování dotazů. Osm minut je povolené minimum; pro větší rezervu API limitu lze nastavit 9 nebo 10 minut.

### 4. Dokončení

Otevřete konfiguraci Dashboardu, vložte OAuth údaje, uložte je a sledujte stav pod uzly `Token manager`, `Uloz nove tokeny` a `Parse klimatizace`. Po prvním úspěchu restartujte Node-RED a ověřte automatické načtení.

## Nastavení v Dashboardu

| Pole / tlačítko | Význam |
| --- | --- |
| Client ID | Identifikátor aplikace z Developer Portalu. |
| Client Secret | Tajný klíč aplikace. Ukládá se do `daikin_config.json`. |
| Refresh Token | Vkládá se při prvním nastavení nebo opravě `invalid_grant`; po uložení se v UI nezobrazuje. |
| Poll interval (min) | Interval načítání zařízení. Výchozí a minimální hodnota je **8 minut**; změna se uloží do konfigurace a použije plánovačem. |
| Otestovat token | Vynutí refresh uloženého tokenu a při úspěchu také GET zařízení. |
| Uložit konfiguraci | Uloží konfiguraci; při vloženém refresh tokenu zapíše tokenový soubor a spustí refresh. |

Konfigurační UI používá české texty. Client Secret ani refresh token se po opětovném otevření formuláře nezobrazují jako čitelný tokenový stav; uložený Client Secret se však do formuláře posílá jako hodnota heslového pole.

## Jak modul pracuje

### Spuštění

Po startovním `RESET` následuje desetisekundové zpoždění. Potom flow nezávisle načte:

- `daikin_config.json` do `global.config.daikinConfig`;
- `daikin_tokens.json` do `global.daikinTokens`;
- stav do konfiguračního widgetu.

`Daikin startup sync` řeší libovolné pořadí souborů. Refresh spustí až po dostupnosti Client ID, Client Secretu a refresh tokenu. Patnáctisekundová pojistka brání dvojímu startovnímu refreshi.

### Tokeny

`Token manager`:

1. použije platný access token, pokud do jeho expirace zbývá více než 5 minut;
2. jinak pošle refresh request na `https://idp.onecta.daikineurope.com/v1/oidc/token`;
3. brání souběhu dvou refreshů po dobu 60 sekund;
4. uloží nový access token a případný rotovaný refresh token;
5. zapíše kompletní sadu do `daikin_tokens.json`;
6. odešle jeden GET na `https://api.onecta.daikineurope.com/v1/gateway-devices`.

Access token se v kódu považuje za přibližně tříhodinový, pokud server neposkytne `expires_in`. Samostatný Inject volá manager každých 2,5 hodiny. Pokud je access token stále platný, tick jej neobnoví, ale přesto provede GET zařízení.

### Parsování zařízení

Parser hledá management pointy `climateControl`, `gateway`, `indoorUnit` a `outdoorUnit`. Jednotka bez `climateControl` se do Dashboardu nepřidá. Zpracuje telemetrii, plán, spotřebu a servisní údaje do jednoho objektu na jednotku.

Při HTTP 401 označí access token jako expirovaný, ale neprovede okamžitý opakovaný request. Další poll token obnoví. Jiné chybové odpovědi zobrazí jako neočekávanou odpověď.

## Soubory a ochrana přístupových údajů

| Soubor | Obsah |
| --- | --- |
| `daikin_config.json` | `clientId`, **`clientSecret`**, `pollMinutes`. |
| `daikin_tokens.json` | **`refresh_token`**, `access_token`, `access_expires` a případná další pole zachovaná z tokenové odpovědi. |

Oba soubory obsahují tajné údaje v čitelné podobě. Proto:

- neukládejte je do Git repozitáře, release ZIPu ani screenshotu;
- přidejte je do `.gitignore`;
- omezte práva na účet, pod kterým běží Node-RED;
- zálohujte je pouze do chráněného úložiště;
- nesdílejte access ani refresh tokeny;
- při podezření na únik zrušte aplikaci nebo její Secret v portálu a proveďte novou autorizaci.

Doporučený `.gitignore`:

```gitignore
daikin_config.json
daikin_tokens.json
```

Flow export samotné hodnoty neobsahuje, pokud jste je ručně nezapsali přímo do uzlů.

## Datové rozhraní

### Kontexty

| Kontext | Význam |
| --- | --- |
| `global.config.daikinConfig` | Client ID, Client Secret a uložené `pollMinutes`. |
| `global.daikinTokens` | Aktivní tokenová sada. |
| `global.daikinTokenState` | Stav pro UI: existence refresh tokenu, expirace access tokenu a poslední chyba. |
| `global.daikinRefreshInFlight` | Časový zámek souběžného refreshu. |
| `global.lineaApiClimateState` | Omezený snapshot pro API LINEA. |
| `global.lineaApiClimateFwVersions` | Poslední známé verze firmwaru pro detekci změny. |

### Výstup do Dashboardu

Každý prvek pole poslaného do `Klimatizace - karty` obsahuje mimo jiné:

```json
{
  "id": "device-id",
  "device_name": "Obyvak",
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

Objekt může dále obsahovat `schedule`, `energy`, `next_action`, `info`, rozsah setpointu a směry lamel.

### Snapshot LINEA

`global.lineaApiClimateState` má tvar:

```json
{
  "updatedAt": "2026-09-20T16:30:00.000Z",
  "devices": [
    {
      "name": "Obyvak",
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

`operationMode` vyjadřuje nastavený režim jednotky. Neprokazuje, že kompresor v dané sekundě fyzicky běží. `firmwareChanged` se při prvním načtení po startu nenastaví; objeví se až při změně vůči dříve známé verzi v běžícím kontextu.

Snapshot zůstává po chybě API uložený. Konzument musí podle `updatedAt` kontrolovat stáří.

## Limity API a správný interval

Daikin u běžných aplikací uvádí výchozí limit **200 API requestů za pohyblivých 24 hodin** a **20 requestů za minutu**. Jeden `GET /v1/gateway-devices` vrátí všechny jednotky účtu, takže počet klimatizací sám o sobě nezvyšuje počet poll requestů. Aktuální limit a podmínky vždy ověřte v [Developer Portalu](https://developer.cloud.daikineurope.com/).

Opravený flow používá výchozí polling **každých 8 minut**:

- hlavní poll spotřebuje nejvýše přibližně 180 GET requestů za 24 hodin;
- tokenový tick běží každé 2,5 hodiny a při platném tokenu může také spustit GET;
- startovní refresh a ruční testy přidávají další požadavky;
- interval lze nastavit pomocí `pollMinutes`, přičemž minimum je 8 minut.

Osm minut odpovídá zamýšlenému výchozímu nastavení, ale rezerva vůči dennímu limitu je malá. Pokud často restartujete Node-RED, používáte ruční testy nebo stejnou OAuth aplikaci využívá další software, nastavte raději **9 až 10 minut**. Desetiminutový hlavní poll spotřebuje 144 requestů za 24 hodin.

Při dosažení limitu očekávejte HTTP `429 Too Many Requests`. Opakované ruční testování situaci zhorší; vyčkejte na uvolnění pohyblivého okna.

Cloudové API není vhodné pro rychlou regulační smyčku FVE nebo ESS. Data mají několikaminutový interval a cloudovou latenci. Pro reaktivní řízení je nutné místní rozhraní podporované konkrétním zařízením.

## Známá omezení tohoto exportu

1. **Modul je read-only.** Navzdory textu v nápovědě UI neobsahuje žádné volání pro změnu teploty, režimu ani zapnutí.
2. **Osmiminutový interval má malou rezervu API limitu.** Další ruční testy, restarty nebo jiný software používající stejnou aplikaci mohou denní limit vyčerpat.
3. **Samostatný export nemá plnohodnotný automatický start.** Jeho `RESET` čeká na link z LINEA a nepojmenovaný Inject není nastavený jako automatický po startu.
4. **Chybí watchdog čerstvosti.** Při chybě zůstávají v Dashboardu a `lineaApiClimateState` poslední úspěšná data.
5. **HTTP 401 se neopakuje ihned.** Token se označí jako expirovaný a oprava proběhne při dalším triggeru.
6. **Jiné HTTP chyby nemají podrobný parser ani automatický backoff.** Flow nerozlišuje 403, 429 a chyby 5xx v uživatelském rozhraní.
7. **Více kopií není izolováno.** Sdílejí stejné globální klíče, soubory a OAuth aplikaci.
8. **První refresh token nevytvoří Node-RED.** Musí se získat ruční OAuth autorizací popsanou výše.
9. **Údaje spotřeby jsou interpretací polí API.** Dostupnost a časový význam řad `d`, `w` a `m` závisí na modelu a odpovědi Daikin; hodnoty nejsou náhradou kalibrovaného elektroměru.

## Řešení problémů

| Projev | Pravděpodobná příčina a kontrola |
| --- | --- |
| Developer Portal neukazuje jednotky | Portal jednotky běžně nezobrazuje; ověřte je v ONECTA a použijte stejný účet i způsob přihlášení. |
| API vrací prázdné `[]` | Jiný účet/metoda přihlášení, neudělený souhlas nebo zařízení není pod tímto účtem. |
| Redirect URI mismatch | URI v portálu, autorizační URL a token requestu není znak po znaku stejné. |
| Autorizační stránka hlásí chybu | Zkontrolujte Client ID, `response_type=code`, scope, registrované URI a případnou registraci Cloud API v1. |
| `invalid_grant` při výměně kódu | Kód expiroval, byl použit podruhé nebo se neshoduje Redirect URI. Spusťte novou autorizaci. |
| `invalid_grant` při refreshi | Refresh token byl odvolán, nahrazen rotovaným tokenem, ztracen při souběhu nebo patří k jinému Client ID/Secret. Získejte nový. |
| `invalid_client` | Nesprávný Client ID nebo Secret; zkontrolujte mezery a zda Secret nebyl v portálu změněn. |
| HTTP 401 z gateway-devices | Access token vypršel nebo je neplatný; další trigger se pokusí o refresh. |
| HTTP 403 | Účet/aplikace nemá oprávnění, chybí souhlas nebo API registrace. |
| HTTP 429 | Vyčerpaný limit. Nastavte 10 minut a vyčkejte na uvolnění pohyblivého 24h okna. |
| `ENOTFOUND`, timeout | DNS, internet nebo odchozí HTTPS z kontejneru Node-RED. |
| Soubory se zapisují do špatného místa | `global.sValid_Patch` je prázdný nebo nekončí `/`. |
| Po restartu chybí konfigurace | Nespustil se `RESET`, adresář není persistentní nebo soubory nejsou čitelné. |
| Token funguje, ale karty jsou prázdné | Odpověď neobsahuje `climateControl`, účet nemá jednotky nebo parser narazil na jiný formát. Zapněte dočasně `FULL_JSON`, ale výstup nesdílejte bez odstranění citlivých údajů. |
| Dashboard ukazuje staré hodnoty | Zkontrolujte čas `updated`, `global.lineaApiClimateState.updatedAt`, stav HTTP uzlu a API limit. |

Stav cloudu lze ověřit na [Daikin Cloud Solutions Status](https://daikincloudsolutions.statuspage.io/).

## Kontrola po instalaci

- [ ] Jednotky jsou viditelné v mobilní aplikaci ONECTA.
- [ ] Developer účet používá stejný účet a metodu přihlášení.
- [ ] Aplikace má strategii Onecta OIDC a přesně zapsanou Redirect URI.
- [ ] Client Secret a tokeny nejsou v Git repozitáři.
- [ ] První autorizační kód byl úspěšně vyměněn za refresh token.
- [ ] `global.sValid_Patch` ukazuje na trvalý adresář a končí `/`.
- [ ] Existují `daikin_config.json` a `daikin_tokens.json` a Node-RED je po restartu načte.
- [ ] Polling má výchozí interval 8 minut a uložené `pollMinutes` se skutečně používá.
- [ ] `Token manager` hlásí platný access token a parser načte očekávaný počet jednotek.
- [ ] Čas v Dashboardu a `global.lineaApiClimateState.updatedAt` se pravidelně obnovuje.
- [ ] Restart Node-RED nezpůsobí `invalid_grant` a automatické načtení funguje.