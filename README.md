# MWW-Scanner

Een QR-scanner voor tablets in kioskmodus (MDM). Met deze webapp leveren leerlingen of gebruikers video's in via [file.kiwi](https://file.kiwi). De scanner opent alleen file.kiwi-links, en die gaan altijd via een eigen Cloudflare-proxy. Zo kun je op een MDM-tablet alleen deze scanner en het proxydomein toestaan en kan de tablet niet naar andere websites.

## Inhoud

- [Hoe het werkt](#hoe-het-werkt)
- [Bestanden](#bestanden)
- [Automatische updates en cache](#automatische-updates-en-cache)
- [Cloudflare Worker (proxy)](#cloudflare-worker-proxy)
  - [Wat de worker doet](#wat-de-worker-doet)
  - [Add File-knop aanpassingen](#add-file-knop-aanpassingen)
  - [Worker aanpassen en deployen](#worker-aanpassen-en-deployen)
  - [Volledige worker-code](#volledige-worker-code)
- [MDM / kiosk-configuratie](#mdm--kiosk-configuratie)
- [Lokaal testen](#lokaal-testen)
- [Aandachtspunten](#aandachtspunten)

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

> Dit geldt alleen voor de scanner. Wijzigingen aan de [Cloudflare Worker](#cloudflare-worker-proxy) werken direct na het deployen; daarvoor hoeft `version.json` niet omhoog.

## Cloudflare Worker (proxy)

De proxy draait als Cloudflare Worker op `mww-kiwi-proxy.mwwmarvin.workers.dev`.

- **Waar staat hij:** [Cloudflare dashboard](https://dash.cloudflare.com) → **Workers & Pages** → **`mww-kiwi-proxy`**
- **Gebruikt door:** `PROXY_DOMAIN` bovenaan het script in [`index.html`](index.html)
- **Code:** zie [Volledige worker-code](#volledige-worker-code) hieronder. De versie in Cloudflare is de versie die echt draait; houd deze README gelijk met Cloudflare.

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

De worker voegt direct na `<head>` een `<style>`- en `<script>`-blok toe:
- **Layout:** pagina-zoom `0.94` en geen verticaal scrollen, zodat alles in één scherm past op de tablet.
- **Taal:** zet `lang`, `locale` en `i18nextLng` op `en` in `localStorage` en overschrijft `navigator.language` en `navigator.languages`, zodat de interface Engels is.
- **Add File-knop:** zie hieronder.

### Add File-knop aanpassingen

De knop wordt herkend aan de selector `button[data-upload-more-trigger="true"]`.

| Aanpassing | Hoe |
|---|---|
| **Groter** | CSS: `font-size: 1.4rem`, `padding: 18px 36px`, `min-height: 64px`, `min-width: 260px`, groter icoon (`svg` 1.6em) |
| **Pulseren** | CSS-animatie `pulse-attention`: elke 1,5 s groeit de knop naar 104% met een groene gloed |
| **Nederlandse tekst** | Script vervangt "Add File(s)" door **"Bestand toevoegen"** (ook `aria-label`). Een `MutationObserver` houdt dit vast als de pagina de knop opnieuw tekent. De rest van de pagina blijft Engels. |

Waar pas je wat aan in de [worker-code](#volledige-worker-code):
- **Knopgrootte:** het CSS-blok onder `/* Add File knop: groter + pulseren */`
- **Animatie:** `@keyframes pulse-attention`
- **Knoptekst:** `NL_TEXT` (nieuwe tekst) en `PATTERN` (welke Engelse tekst vervangen wordt)

> ⚠️ Als file.kiwi het attribuut `data-upload-more-trigger` of de tekst "Add File" wijzigt, vallen deze aanpassingen stil weg. Controleer dan via *Inspecteren* in de browser wat de knop nu heet en pas de selector of `PATTERN` aan.

### Worker aanpassen en deployen

1. Ga naar [dash.cloudflare.com](https://dash.cloudflare.com) → **Workers & Pages** → **`mww-kiwi-proxy`**.
2. Klik op **Edit code**.
3. Bewaar eerst de huidige code (kopie in een tekstbestand) zodat je terug kunt.
4. Plak de nieuwe code en klik op **Deploy**. De wijziging is direct actief.
5. Werk de code in deze README bij, zodat GitHub en Cloudflare gelijk blijven.
6. Test: scan een file.kiwi-QR-code op een tablet en herlaad de pagina (wis zo nodig de cache).

### Volledige worker-code

```js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // 1. Doel-URL opbouwen naar file.kiwi met behoud van pad en parameters
    const targetUrl = new URL(url.pathname + url.search, "https://file.kiwi");

    // 2. Headers klonen en configureren
    const newReqHeaders = new Headers(request.headers);
    newReqHeaders.set("Host", "file.kiwi");
    newReqHeaders.set("Accept-Language", "en-US,en;q=0.9");

    // Maskeer Referer en Origin zodat file.kiwi de sessie en uploads accepteert
    newReqHeaders.set("Origin", "https://file.kiwi");
    newReqHeaders.set("Referer", "https://file.kiwi/");

    // Taalcookies meeleveren in het uitgaande verzoek
    const existingCookie = request.headers.get("Cookie") || "";
    newReqHeaders.set("Cookie", `${existingCookie}; lang=en; locale=en; NEXT_LOCALE=en`);

    // 3. Verzoek uitvoeren naar file.kiwi
    const response = await fetch(targetUrl.toString(), {
      method: request.method,
      headers: newReqHeaders,
      body: request.body,
      redirect: "follow"
    });

    // 4. Response headers opbouwen
    const newResHeaders = new Headers(response.headers);

    // Forceer cookies voor Engels aan de browserzijde
    newResHeaders.append("Set-Cookie", "lang=en; Path=/; Max-Age=31536000; SameSite=Lax");
    newResHeaders.append("Set-Cookie", "locale=en; Path=/; Max-Age=31536000; SameSite=Lax");

    // Beveiligingsheaders ontspannen voor embedded tablet-weergave
    newResHeaders.set("Access-Control-Allow-Origin", "*");
    newResHeaders.delete("X-Frame-Options");
    newResHeaders.delete("Content-Security-Policy");

    // 5. HTML aanpassen: CSS, pagina-zoom, taaloverride en Add File knop
    const contentType = response.headers.get("content-type") || "";
    if (contentType.includes("text/html")) {
      let html = await response.text();

      const injection = `
        <style>
          html, body {
            height: 100% !important;
            overflow-y: hidden !important;
            zoom: 0.94; /* Past net dat onderste randje direct in beeld */
          }

          /* Add File knop: groter + pulseren */
          button[data-upload-more-trigger="true"] {
            font-size: 1.4rem !important;
            font-weight: bold !important;
            padding: 18px 36px !important;
            min-height: 64px !important;
            min-width: 260px !important;
            border-radius: 14px !important;
            animation: pulse-attention 1.5s infinite ease-in-out !important;
          }

          /* Icoon in de knop meeschalen */
          button[data-upload-more-trigger="true"] svg {
            width: 1.6em !important;
            height: 1.6em !important;
          }

          @keyframes pulse-attention {
            0%   { transform: scale(1);    box-shadow: 0 0 0 0 rgba(100, 200, 50, 0.7); }
            50%  { transform: scale(1.04); box-shadow: 0 0 16px 4px rgba(100, 200, 50, 0.9); }
            100% { transform: scale(1);    box-shadow: 0 0 0 0 rgba(100, 200, 50, 0); }
          }
        </style>
        <script>
          try {
            localStorage.setItem('lang', 'en');
            localStorage.setItem('locale', 'en');
            localStorage.setItem('i18nextLng', 'en');
          } catch(e){}
          Object.defineProperty(navigator, 'language', { get: () => 'en-US' });
          Object.defineProperty(navigator, 'languages', { get: () => ['en-US', 'en'] });

          // Alleen de Add File knop vertalen naar het Nederlands
          (function () {
            var SELECTOR = 'button[data-upload-more-trigger="true"]';
            var NL_TEXT = 'Bestand toevoegen';
            var PATTERN = /add\\s*files?/i;

            function translateButton() {
              document.querySelectorAll(SELECTOR).forEach(function (btn) {
                var walker = document.createTreeWalker(btn, NodeFilter.SHOW_TEXT);
                var node;
                while ((node = walker.nextNode())) {
                  if (PATTERN.test(node.nodeValue)) {
                    node.nodeValue = node.nodeValue.replace(PATTERN, NL_TEXT);
                  }
                }
                if (btn.getAttribute('aria-label') !== NL_TEXT) {
                  btn.setAttribute('aria-label', NL_TEXT);
                }
              });
            }

            function start() {
              translateButton();
              new MutationObserver(translateButton).observe(document.body, {
                childList: true, subtree: true, characterData: true
              });
            }

            if (document.readyState === 'loading') {
              document.addEventListener('DOMContentLoaded', start);
            } else {
              start();
            }
          })();
        </script>
      `;

      html = html.replace("<head>", "<head>" + injection);

      return new Response(html, {
        status: response.status,
        statusText: response.statusText,
        headers: newResHeaders
      });
    }

    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: newResHeaders
    });
  }
};
```

[↑ Terug naar inhoud](#inhoud)

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
- Pas je de proxy-URL aan, werk dan ook `PROXY_DOMAIN` bovenaan het script in `index.html` bij.
- De worker voegt de code in met `html.replace("<head>", ...)`. Als file.kiwi ooit een `<head>` met attributen gebruikt, werkt dit niet meer; overstappen op Cloudflare `HTMLRewriter` lost dat op.
- De Add File-aanpassingen hangen af van de opbouw van file.kiwi; zie de waarschuwing bij [Add File-knop aanpassingen](#add-file-knop-aanpassingen).

[↑ Terug naar inhoud](#inhoud)
