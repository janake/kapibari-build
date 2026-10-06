# kapibari-build

Nyilvános fordító repó a Kapibari backend és gateway **GraalVM natív image**-éhez (KAP-773).

## Miért

A privát `janake/kapibari` repó GitHub-hosted runnere 2 vCPU / 7 GB RAM — a backend natív
fordítása ott OOM-ol (~8 GB heap kell). Nyilvános repóban a standard `ubuntu-latest` runner
4 vCPU / 16 GB és ingyenes, ezért a natív fordítás ide költözött.

## Mi van benne

**Nincs benne forráskód.** Csak a `.github/workflows/build-native.yml` workflow és ez a README.
A workflow futás közben checkoutolja a privát repót a kért commiton, natívan fordít, Docker
image-et épít (a privát repó `kapibari-backend/` ill. `kapibari-gateway/` Dockerfile-jából), és
a GHCR-be tolja:

- `ghcr.io/janake/kapibari-build/kapibari-backend:<sha>`
- `ghcr.io/janake/kapibari-build/kapibari-gateway:<sha>`

A checkout a futás végén törlődik. Tesztek itt nem futnak (a privát CI már lefuttatta őket).

## Indítás

Kizárólag `workflow_dispatch` (nincs `pull_request` / `pull_request_target` trigger). A privát
repó CI-ja (`ci.yml`, `backend-image` / `gateway-image` job) a REST API-n indítja, majd a
`native <component> <sha>` run-name alapján megkeresi és kivárja a futást.

Bemenetek: `sha` (40 karakteres hex, kötelező), `component` (`backend` | `gateway` | `both`),
`push` (alap: igen), `latest` (alap: nem; a privát CI csak main-pushnál kéri).

## Beállítás

- **Secret:** `SOURCE_READ_TOKEN` — fine-grained PAT, csak a `janake/kapibari` repóra,
  `Contents: Read-only` (+ az automatikus `Metadata: Read-only`).
- A privát repóban: `BUILD_DISPATCH_TOKEN` — fine-grained PAT, csak erre a repóra,
  `Actions: Read and write`.
- Opcionális repo-variable-ök: `NATIVE_BACKEND_XMX` (12g), `NATIVE_BACKEND_PARALLELISM` (4),
  `NATIVE_BACKEND_OPTS`, `NATIVE_GATEWAY_XMX` (8g), `NATIVE_GATEWAY_PARALLELISM` (4),
  `NATIVE_GATEWAY_OPTS`.

## Biztonság

- **Fork PR-ek:** Settings → Actions → General → „Require approval for all external
  contributors” (fork-PR workflow csak jóváhagyással fusson). A workflow egyetlen secretet sem
  ad ki PR-eseménynek, mert nincs PR-triggere.
- **A csomagok láthatósága PRIVÁT maradjon.** Az első push után ellenőrizd a
  `kapibari-backend` / `kapibari-gateway` csomag beállításait (Package settings → Danger Zone →
  visibility: Private). A workflow szándékosan nem tesz `org.opencontainers.image.source`
  labelt az image-re.
- A futási logok nyilvánosak: a Maven/native-image kimenetnek csak a vége kerül a logba,
  artifact-feltöltés nincs.
- A deploy-hostok a meglévő `GHCR_READ_TOKEN`-nel húznak — ennek olvasási joga kell legyen az
  új csomagokra.
