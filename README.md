# Vlastní S3 úložiště (selfhost_s3 + Nginx Proxy Manager)

Self-hosted objektové úložiště, kompatibilní s S3 API, inspirované projektem [Notifuse/selfhost_s3](https://github.com/Notifuse/selfhost_s3). Přes n8n do něj nahráváš soubory a část z nich (prefix `public/`) je automaticky veřejně dostupná přes HTTPS, podobně jako u S3/Cloudflare R2 public bucketu.

Samotné úložiště (`selfhost_s3`) běží jako jeden Docker kontejner, jeden binární server v Go, žádná databáze. HTTPS a doménu řeší Nginx Proxy Manager (NPM), který už na serveru běží a má certifikáty.

## Co potřebuješ předem

- Server s Dockerem + Docker Compose pluginem
- Nginx Proxy Manager běžící na stejném hostu (mimo Docker)
- Doménu/subdoménu (např. `storage.tvoje-domena.cz`), kterou v NPM nasměruješ na tuto službu

## Nasazení

1. Zkopíruj tuto složku na server.
2. Vytvoř `.env` podle vzoru:

   ```
   cp .env.example .env
   ```

3. V `.env` uprav:
   - `S3_ACCESS_KEY` a `S3_SECRET_KEY` – vygeneruj náhodné hodnoty, např. `openssl rand -hex 16`
   - podle potřeby `S3_CORS_ORIGINS` (v produkci lepší omezit na konkrétní domény než `*`)

4. Spusť:

   ```
   docker compose up -d
   ```

   Kontejner naslouchá na `127.0.0.1:9000` (jen lokálně, ven nejde nic přímo).

5. V administraci Nginx Proxy Manageru přidej nový **Proxy Host**:
   - Domain Names: `storage.tvoje-domena.cz`
   - Scheme: `http`
   - Forward Hostname/IP: `127.0.0.1`
   - Forward Port: `9000`
   - SSL: přiřaď existující certifikát (nebo vyžádej nový přes Let's Encrypt), zapni **Force SSL**
   - V záložce *Advanced* nastav vyšší `client_max_body_size` (musí sedět s `S3_MAX_FILE_SIZE`, výchozí NPM limit je 1MB), např.:
     ```
     client_max_body_size 100M;
     ```
   - Pro streamování videa pomocí HTTP Range requests přidej do záložky *Advanced*:
     ```
     client_max_body_size 100M;
     proxy_set_header Range $http_range;
     proxy_set_header If-Range $http_if_range;
     proxy_set_header Accept-Encoding "";
     proxy_buffering off;
     proxy_request_buffering off;
     ```
     Nesmí se přidat druhý `client_max_body_size`; hodnota musí odpovídat `S3_MAX_FILE_SIZE`.

6. Po uložení by mělo jít:

   ```
   curl https://storage.tvoje-domena.cz/health
   ```

   a mělo by vrátit `{"status":"ok"}`.

Data se ukládají do `./data` vedle `docker-compose.yaml` – tuhle složku zálohuj.

## Jak to funguje

- Soubory se ukládají do bucketu nastaveného v `S3_BUCKET`, na cestě `{key}` v rámci bucketu.
- Vše pod prefixem `public/` (dá se změnit přes `S3_PUBLIC_PREFIX`) je čitelné bez přihlášení – ideální pro soubory, které chceš mít veřejně dostupné jako URL (obrázky, PDF, exporty apod.).
- Vše mimo `public/` vyžaduje AWS Signature V4 autentizaci (stejnou jako u AWS S3) – tam nahráváš přes n8n nebo AWS CLI, ne přes prohlížeč.
- Nahrávání (PUT) a mazání (DELETE) vždy vyžaduje autentizaci, i pro veřejné soubory – veřejný je jen čtecí přístup.

### Příklad veřejné URL

```
https://storage.tvoje-domena.cz/my-bucket/public/faktury/2026-07.pdf
```

Pro vynucení stažení místo zobrazení v prohlížeči přidej `?download=1`.

## Nahrávání z n8n

Server je S3-kompatibilní (path-style), takže v n8n stačí použít vestavěný **S3 node** (nebo HTTP Request node s AWS auth) s těmito údaji:

- Endpoint: `https://storage.tvoje-domena.cz`
- Bucket: hodnota z `S3_BUCKET`
- Access Key / Secret Key: hodnoty z `.env`
- Region: `us-east-1`
- Force path style: zapnuto (povinné)

Pro veřejně dostupný soubor nahrávej na klíč začínající `public/`, např. `public/faktury/2026-07.pdf`.

## Bezpečnostní doporučení

- V produkci nastav `S3_CORS_ORIGINS` na konkrétní domény místo `*`.
- Access/secret klíče drž jen v `.env` (needituj přímo `docker-compose.yaml`) a `.env` nikam nenahrávej (např. do gitu).
- Zvaž CDN před veřejnými soubory, pokud čekáš vyšší provoz.
- Pravidelně zálohuj složku `data/`.
- Port 9000 je publikovaný jen na `127.0.0.1`, takže je dostupný pouze skrz NPM – to je záměr, neotvírej ho navenek.

## Omezení tohoto řešení

Přebráno z omezení selfhost_s3: žádný multipart upload (soubory se nahrávají v jednom requestu, proto `S3_MAX_FILE_SIZE`, a musí mu odpovídat i `client_max_body_size` v NPM), žádné presigned URL, žádné verzování souborů, jeden kontejner = jeden bucket.
