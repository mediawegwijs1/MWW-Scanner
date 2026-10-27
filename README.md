# MWW-Scanner

Een QR-scanner voor tablets in kioskmodus (MDM). Met deze webapp leveren leerlingen of gebruikers video's in via [file.kiwi](https://file.kiwi). De scanner opent alleen file.kiwi-links, en die gaan altijd via een eigen Cloudflare-proxy. Zo kun je op een MDM-tablet alleen deze scanner en het proxydomein toestaan en kan de tablet niet naar andere websites.

## Hoe het werkt

```
┌────────────┐   scan QR    ┌──────────────────┐   herschrijft URL   ┌──────────────────────────────────────┐        ┌───────────┐
│ Gebruiker  │ ───────────▶ │  MWW-Scanner     │ ──────────────────▶ │ mww-kiwi-proxy.mwwmarvin.workers.dev │ ─────▶ │ file.kiwi │
│ (tablet)   │              │  (GitHub Pages)  │                     │ (Cloudflare Worker)                  │ ◀───── │           │
└────────────┘              └──────────────────┘                     └──────────────────────────────────────┘        └───────────┘
```

1. De gebruiker opent de scanner en tikt op **"Start de Camera"**.
2. De achtercamera scant een QR-code.
3. **Controle:** alleen codes die beginnen met `https://file.kiwi/` of `http://file.kiwi/` worden geaccepteerd. Bij andere codes verschijnt de melding *"Let op: Dit is geen file.kiwi inlever-link."*
4. Bij een geldige code stopt de camera en ziet de gebruiker de gescande link.
5. De knop **"Open inlever-link"** opent dezelfde link, maar met `file.kiwi` vervangen door het proxydomein. Pad, parameters en hash blijven hetzelfde.
   - Voorbeeld: `https://file.kiwi/abc123#key` wordt `https://mww-kiwi-proxy.mwwmarvin.workers.dev/abc123#key`
6. Met **"Opnieuw scannen"** herlaadt de pagina en kun je een nieuwe code scannen.

## Bestanden

| Bestand | Doel |
|---|---|
| `index.html` | De hele scanner-app (HTML, CSS en JavaScript). Gebruikt [`html5-qrcode`](https://github.com/mebjas/html5-qrcode) voor het scannen. |
| `version.json` | Het huidige versienummer van de app. Wordt gebruikt voor automatische updates. |
| `.nojekyll` | Zorgt dat GitHub Pages de bestanden ongewijzigd publiceert. |

## Automatische updates en cache

Kiosk-tablets houden pagina's vaak lang in de cache. Daarom doet de app het volgende:

- Bij elke keer laden haalt hij `version.json` op, zonder cache.
- Is de versie anders dan de opgeslagen versie (`localStorage` → `mww_app_version`), dan worden `localStorage` en `sessionStorage` gewist en herlaadt de pagina met een cache-busting URL (`?v=<versie>&t=<timestamp>`).
- De huidige versie staat linksonder in beeld (bijv. `v0.9`).

**Een nieuwe versie uitrollen:** pas de code aan en verhoog het nummer in `version.json`. Alle tablets halen de nieuwe versie dan automatisch op zodra de pagina opnieuw laadt.

## Cloudflare Worker (proxy)

De proxy draait als Cloudflare Worker op `mww-kiwi-proxy.mwwmarvin.workers.dev`. De code staat **niet** in deze repository en wordt beheerd in Cloudflare.

### Wat de worker doet

**Uitgaand verzoek (tablet → file.kiwi)**
- Stuurt elk verzoek door naar `https://file.kiwi`, met hetzelfde pad en dezelfde parameters.
- Zet de headers `Host`, `Origin` en `Referer` op `file.kiwi`, zodat file.kiwi sessies en uploads accepteert.
- Forceert Engels via `Accept-Language` en de cookies `lang=en`, `locale=en` en `NEXT_LOCALE=en`.
- Stuurt method en body (uploads) ongewijzigd door en volgt redirects.

**Antwoord (file.kiwi → tablet)**
- Zet de cookies `lang=en` en `locale=en` in de browser (1 jaar geldig).
- Voegt `Access-Control-Allow-Origin: *` toe.
- Verwijdert `X-Frame-Options` en `Content-Security-Policy`, zodat de pagina ook ingesloten op de tablet werkt.

**HTML-aanpassingen** (alleen bij `text/html`)

De worker voegt direct na `<head>` het volgende toe:
- **Layout:** pagina-zoom `0.94` en geen verticaal scrollen, zodat alles in één scherm past op de tablet.
- **Animatie:** een groen pulserend effect op de knop *Add File* (`button[data-upload-more-trigger="true"]`), zodat gebruikers zien waar ze moeten tikken.
- **Taal:** zet `lang`, `locale` en `i18nextLng` op `en` in `localStorage` en overschrijft `navigator.language` en `navigator.languages`, zodat de interface altijd Engels is.

## MDM / kiosk-configuratie

Aanbevolen instellingen voor de tablet:

- Kioskmodus met de browser vergrendeld op de URL van deze scanner (GitHub Pages).
- Webfilter/allowlist met alleen:
  - het GitHub Pages-domein van deze scanner
  - `mww-kiwi-proxy.mwwmarvin.workers.dev`
  - `unpkg.com` (voor de QR-bibliotheek)
- Cameratoegang toestaan voor de browser.

## Lokaal testen

Camera-toegang werkt alleen via HTTPS of `localhost`:

```bash
npx serve .
# of
python3 -m http.server 8000
```

Open daarna `http://localhost:8000` en scan een file.kiwi-QR-code.

## Aandachtspunten

- `html5-qrcode` wordt van `unpkg.com` geladen zonder vaste versie. Zet er een vaste versie in (bijv. `html5-qrcode@2.3.8`) als je zeker wilt zijn van stabiel gedrag.
- Pas je de proxy aan, werk dan ook `PROXY_DOMAIN` bovenaan het script in `index.html` bij.
