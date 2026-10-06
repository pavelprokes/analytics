# Nasazení na Vercel pro více domén

Jedna instance (jeden Vercel projekt + jedna DB) obsluhuje libovolný počet
webů. Každá doména = jeden „Website" v Umami s vlastním `website-id`.

## 1. Databáze
Vytvoř Postgres (Neon / Vercel Postgres / Supabase) a použij **pooled**
connection string jako `DATABASE_URL`. Funkce na Vercelu jsou serverless,
bez poolingu vyčerpáš spojení. Region funkcí nastav blízko DB
(Project → Settings → Functions → Region).

## 2. Vercel projekt
1. Import forku `pavelprokes/analytics` do Vercelu (Framework: Next.js).
2. Environment Variables: viz `.env.example` (min. `DATABASE_URL`, `APP_SECRET`).
3. Deploy. Build spustí migrace DB (`prisma migrate deploy`) a vytvoří
   uživatele `admin` / `umami` – **hned změň heslo**.
4. Vlastní doména pro analytiku, např. `stats.tvadomena.cz`
   (Settings → Domains). Doporučeno first-party subdoména.

## 3. Přidání domén
V Umami: Settings → Websites → Add website (doména + název), zkopíruj ID.
Do každého webu vlož:

```html
<script defer src="https://stats.tvadomena.cz/stats.js"
        data-website-id="UUID-WEBU"
        data-domains="web1.cz,www.web1.cz"></script>
```

- `data-domains` omezí odesílání jen na uvedené hostname (ochrana před
  zneužitím skriptu na cizích doménách).
- `TRACKER_SCRIPT_NAME` a `COLLECT_API_ENDPOINT` obcházejí základní
  adblock filtry (`script.js`, `/api/send`). Po změně je nutný redeploy.
- CORS pro tracker i collect endpoint je už povolený (`*`).

## 4. Preview deploye
Preview větve mají vlastní URL; nastav jim samostatnou DB
(Environment Variables → scope Preview), ať nezapisují do produkce.

## 5. Další krok: server-side (SSE) trackování
Eventy ze serveru půjdou na `POST /api/send` (nebo `COLLECT_API_ENDPOINT`)
s `website`, `hostname`, `url` a vlastním `User-Agent` / IP z původního
requestu. Postup nastavíme v dalším kroku.
