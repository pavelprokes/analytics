# Prompt: přidání Umami trackingu do projektu

Zkopíruj do Claude Code v cílovém projektu a vyplň hodnoty v sekci „Vstupy".
Vychází z toho, co tento fork přijímá na `/api/send` (viz `src/app/api/send/route.ts`).

````text
Přidej do tohoto projektu analytics tracking do mé self-hosted Umami instance.

## Vstupy
- Doména webu: {{DOMAIN}}                     (např. mujweb.cz, případně i www varianta)
- Umami URL: https://analytics.pavelprokes.cz   (tracking doména, HTTPS; bez lomítka na konci)
- Website ID: {{WEBSITE_ID}}                  (UUID z Umami → Settings → Websites)
- Název skriptu: {{SCRIPT_NAME}}              (např. stats.js; výchozí je script.js)
- Collect endpoint: {{COLLECT_ENDPOINT}}      (např. /api/e; výchozí je /api/send)

## Postup
1. Nejdřív zjisti framework a verzi (package.json, struktura app/ vs pages/). Pokud
   to není Next.js, přizpůsob řešení (Vite/Astro/čisté HTML = jen client script,
   server část podle runtime). Neměň nic nesouvisejícího.
2. Ověř, jaký tracking v projektu už je (grep: umami, gtag, googletagmanager,
   google-analytics, fbq/facebook pixel, hotjar, clarity, linkedin insight, tiktok
   pixel, ads/remarketing tagy, vložená videa a widgety třetích stran), ať nevznikne
   duplicita.
   - Umami samo (cookieless, bez osobních údajů) cookie banner nevyžaduje; načítej
     ho vždy, nezávisle na souhlasu.
   - Pokud najdeš GA, reklamní nebo jiné trackery, které ukládají cookies nebo
     sdílejí data s třetími stranami, musí na webu být cookie lišta / consent.
     Zjisti, jestli už existuje (grep: cookie banner, consent, cookiebot, onetrust,
     klaro, osano, vlastní komponenta).
     - Existuje → v pořádku, nech ji být a nic neměň.
     - Neexistuje → NEDĚLEJ ji a nezastavuj kvůli tomu práci. Pokračuj s Umami
       a na konci reportu uveď varování: „nalezeny trackery X, Y bez cookie
       souhlasu, je potřeba doplnit consent".
   - Pokud trackery nenajdeš, nic neřeš a banner nenavrhuj.

### A) Pageviews (client)
- Nastav env: NEXT_PUBLIC_UMAMI_URL, NEXT_PUBLIC_UMAMI_WEBSITE_ID,
  NEXT_PUBLIC_UMAMI_SCRIPT (název skriptu) a doplň je do .env.example.
- V root layoutu (Next.js App Router: app/layout.tsx; Pages Router: _app.tsx)
  přidej `next/script` se strategy="afterInteractive":
  `src={`${URL}/${SCRIPT}`}`, `data-website-id`, `data-domains="{{DOMAIN}}"`.
- Vykresli skript jen v produkci / když je website ID nastavené, ať se dev a
  preview neznečišťují.
- Navigace v Next.js (History API) se trackuje automaticky, žádné ruční
  `umami.track` na změnu route nepřidávej.
- Pokud má projekt CSP, povol https://analytics.pavelprokes.cz v script-src a connect-src.

### B) Server events (SSE) – bez blokování odpovědi
Vytvoř `lib/umami.ts` (server-only) s funkcí
`trackServerEvent({ name, data, request })`:
- POST na `${UMAMI_URL}${COLLECT_ENDPOINT}` s hlavičkou `Content-Type: application/json`
  a tělem:
  { "type": "event",
    "payload": {
      "website": WEBSITE_ID,
      "hostname": "{{DOMAIN}}",
      "url": cesta aktuálního requestu (např. "/checkout"),
      "name": "nazev-udalosti",
      "data": { ...jen jednoduché hodnoty... },
      "ip": IP návštěvníka,
      "userAgent": User-Agent návštěvníka,
      "referrer": volitelně,
      "language": volitelně z Accept-Language
    } }
- `ip` a `userAgent` BERE Z PŮVODNÍHO REQUESTU návštěvníka (x-forwarded-for → první
  položka, jinak x-real-ip; user-agent). Bez nich by všechny události vypadaly
  jako jeden návštěvník ze serveru.
- Funkce nikdy nehází výjimku a nikdy neblokuje odpověď: použij `after()` z
  `next/server` (Next 15+) nebo `waitUntil` z `@vercel/functions`, plus timeout
  (AbortSignal.timeout(3000)). Chyby jen zaloguj.
- Bez requestu (cron, webhook): vynech ip/userAgent a pošli stabilní `id`
  (distinct id, např. interní ID uživatele/objednávky), ať se události spojí.
- Název události: krátký, kebab-case, nesmí začínat znaky = + - @.
  `data`: žádné osobní údaje (e-mail, jméno, celé adresy), jen identifikátory/čísla/flagy.
- Zavolej ji na místech, kde dává smysl serverová událost (route handlery,
  server actions, webhooky: registrace, objednávka, platba, odeslání formuláře).
  Navrhni seznam míst a názvů, počkej na moje potvrzení a pak je doplň.
- Server eventy se nepoužívají na pageviews, ty řeší client skript.

### C) Ověření
- Spusť lint/typecheck/build projektu.
- Dej mi curl příkaz pro ruční test:
  curl -i -X POST "https://analytics.pavelprokes.cz{{COLLECT_ENDPOINT}}" -H "Content-Type: application/json" \
    -d '{"type":"event","payload":{"website":"{{WEBSITE_ID}}","hostname":"{{DOMAIN}}","url":"/test","name":"smoke-test","userAgent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/124 Safari/537.36","ip":"203.0.113.10"}}'
  (očekávaná odpověď je 200 s JSON; `{"beep":"boop"}` znamená, že UA vyhodnotil
  bot filtr jako bota).
- Napiš, co zkontrolovat v Umami (Realtime / Events) po nasazení.
- V reportu uveď i nález z kroku 2 (nalezené trackery a stav cookie lišty).

## Pravidla
- Žádná tajemství v kódu; Umami URL a ID jen přes env.
- Nepřidávej závislosti, pokud to nejde bez nich (stačí fetch).
- Na konci vypiš seznam změněných souborů a env proměnných, které musím nastavit ve Vercelu.
````

## Poznámka k serverovým událostem na Vercelu

Když v payloadu pošleš `ip`, Umami přeskočí geo hlavičky Vercelu a hledá zemi
v GeoLite databázi (`geo/GeoLite2-City.mmdb`). Databáze se do repa nedává (licence
MaxMind to zakazuje), stahuje se při buildu: ve Vercelu nastav `BUILD_GEO=1`
a `MAXMIND_LICENSE_KEY` (zdarma, účet na maxmind.com). `next.config.ts` ji přibalí
do funkcí `/api/send`, `/api/batch` a `/api/record`. Bez databáze už lookup
nespadne, jen se země u server eventů neuloží.
