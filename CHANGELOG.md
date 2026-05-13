# Changelog

All notable changes to EverShelf will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.7.12] - 2026-05-13

### Fixed
- **Banner "Usa prima" con data calcolata confusa** — `_renderUseExpiryHint` mostrava una data di scadenza *calcolata* (shelf life dopo apertura) anziché la data reale. Ora, se il prodotto ha `opened_at`, il banner mostra "Quella [nel frigo], aperta da X giorni — usala prima!" usando la nuova chiave `use.expiry_warning_opened`.
- **"Usa TUTTO / Finito" nelle ricette cancellava la riga** — `submitRecipeUse(true)` inviava `use_all: true` all'API che eseguiva un `DELETE` diretto sulla riga di inventario senza conferma. La funzione ora calcola la quantità esatta dagli item disponibili (`_recipeUseContext.items`) e invia un normale `inventory_use` con quantità esplicita.
- **Ricette: `qty_number` in grammi per prodotti `pz`** — Il prompt AI e la post-elaborazione PHP ora istruiscono Gemini a esprimere `qty_number` come pezzi interi per ingredienti con unità `pz` (Pan bauletto, fette biscottate, ecc.). La lista ingredienti nel prompt include `[usa PEZZI interi]` per ogni prodotto `pz`. Il fallback PHP per `pz` senza `default_quantity` non divide più per 100 (trattando grammi come pezzi), ma usa il `qty_number` restituito dall'AI se sembra un conteggio plausibile, altrimenti 1.

### Added
- **Traduzione `use.expiry_warning_opened`** — Nuova chiave in `it.json`, `en.json`, `de.json` con placeholder `{loc}` (posizione) e `{when}` (giorni dall'apertura).

## [1.7.11] - 2026-05-12

### Added
- **Scan page redesign** — La pagina di scansione è stata completamente ridisegnata per tablet e mobile:
  - **2× zoom fisso** — zoom hardware se disponibile, altrimenti CSS `scale(2)` automatico.
  - **Torcia** — bottone nel viewport con feedback toast e stato visivo.
  - **Flip fotocamera** — switch front/back con persistenza in settings.
  - **3 tab input** — Barcode / Nome / AI per un accesso rapido a ciascuna modalità.
  - **Prodotti recenti** — chip degli ultimi 6 prodotti scansionati (localStorage), con icona categoria.
  - **Live code overlay** — codice barcode rilevato parzialmente mostrato in sovrimpressione nel viewport.
  - **Confirm overlay** — checkmark + nome prodotto per 900ms al riconoscimento avvenuto.
  - **Angoli guida** — frame visivo per inquadrare il barcode.
  - **AI Number OCR** — dopo 4s senza scansione, compare il bottone "Leggi numeri con AI": Gemini analizza l'immagine e legge le cifre del barcode anche se non viene letto otticamente.
- **PHP `gemini_number_ocr`** — Nuovo endpoint POST; accetta un'immagine JPEG base64, chiede a Gemini di individuare il codice EAN-13 / EAN-8 stampato sul prodotto, e restituisce le cifre o `not_found`.

### Fixed
- **Falsi positivi anomalia consumo "Mozzarella 3 pezzi"** — Rimossa la direzione `untracked` (consumo maggiore degli acquisti registrati) che generava banner su ogni prodotto con acquisti non tracciati. Ora vengono segnalate solo le anomalie `phantom` e `missing`.
- **Predizione "~0g/settimana"** — Il modello richiedeva ora min 5 transazioni (era 3) e un arco temporale di almeno 7 giorni; se il consumo predetto è < 15% della baseline viene saltato, eliminando i falsi positivi su prodotti con poche transazioni ravvicinate.
- **Menu a tendina suggerimenti sul campo Nome (scan)** — Rimosso `list="common-products"` dal campo di input, il datalist non viene più aperto su tablet.

## [1.7.10] - 2026-05-11

### Fixed
- **Banner "Imposta scadenza" non faceva nulla** — `editBannerNoExpiry()` chiamava `openEditInventoryModal()` che non esiste. Corretto in `editInventoryItem()` (la funzione corretta usata da tutti gli altri handler banner). Aggiunto anche il fetch preventivo di `inventory_list` perché `currentInventory` è vuoto sulla dashboard.
- **"Prodotto non trovato" aprendo modal da banner** — `currentInventory` è sempre vuoto sulla dashboard; il fetch dell'inventario ora avviene prima di aprire la modal (stesso pattern di `editReviewItem` e `weighBannerItem`).
- **Banner scaduto su latte UHT aperto** — Il testo mostrava "Scaduto!" invece di "Aperto da troppo tempo". Ora i prodotti con `opened_at` mostrano "Aperto da N giorni in [posizione]" sia nel titolo che nel dettaglio del banner.
- **Shelf life latte generico 4 → 7 giorni** — Il latte senza qualificatori (es. "Latte") veniva trattato come fresco (4 giorni). Il latte fresco è già gestito esplicitamente (`latte fresco/intero/parzial/scremato` → 3gg); il generico ora vale 7 giorni (default UHT). Fix applicato sia in PHP (`database.php`) che in JS (`app.js`).
- **`opened_at` stale sulle confezioni intere dopo split** — Quando un uso splitta la riga in "confezioni intere + frazione aperta", la riga delle intere non azzerava `opened_at`. Ora tutti e 3 i percorsi di split eseguono `opened_at = NULL` sulla riga sigillata.
- **`inventory_update` non registrava transazioni** — La modal di modifica quantità aggiornava l'inventario senza creare transazioni. La differenza viene ora registrata automaticamente come `'in'` o `'out'` con nota `[Correzione manuale]`, evitando falsi positivi nel rilevatore di anomalie.
- **False anomalie di consumo dopo la spesa** — La baseline della prediction usava solo la quantità del rifornimento (`restockQty`), ignorando le scorte preesistenti → `actual > expected` sistematicamente. Nuova baseline: `qty_attuale + consumato_da_ultimo_rifornimento`, che riflette correttamente la realtà indipendentemente dalle scorte pregresse.
- **Banner "consumo anomalo" su quasi tutti i prodotti** — Due fix:
  1. `expected = 0` non genera più anomalia "more" (il modello pensa che dovresti aver finito, ma hai ricomprato).
  2. Soglia "more than expected" alzata al 400% (era 30%); "less than expected" rimane al 30%.
- **Sezione scaduti mostra prodotti già buttati** — La query `expired` mancava di `AND i.quantity > 0`; i prodotti buttati (qty=0) con scadenza passata continuavano ad apparire. Corretta la query + pulizia righe orfane nel DB.
- **Hardcoded `scade il` in banner** — Stringa italiana hardcodata nel dettaglio del banner scaduti rimossa.
- **Docker: `SQLSTATE[HY000][14] unable to open database file`** — Aggiunta `_ensureDataDir()` in `database.php` che crea la directory se mancante e tenta `chmod(0775)` se non scrivibile.

### Added
- **i18n completa** — Aggiunti ~25 chiavi di traduzione mancanti per UI kiosk, gemini, banner, scanner, shopping, appliances in tutti e 3 i file (`it.json`, `en.json`, `de.json`). Totale: 934 chiavi per lingua.


### Added
- **Category badge on inventory items** — Every product in the inventory now displays a macro-category badge (icon + label) next to the location badge. Badges showing `altro` are asynchronously refined via the new `guess_category` AI endpoint (Gemini + `data/category_ai_cache.json` cache) so the correct category appears automatically after the page loads.
- **Category search** — The inventory search bar now matches items by category. Typing "biscotti" returns every cookie/biscuit regardless of brand or exact name; the match uses both the direct category key and the translated label.
- **Brand map in `guessCategoryFromName`** — A fast-path brand table (Oreo, Ringo, Uno, Barilla, De Cecco, Galbani, Mutti, Lavazza, etc.) provides instant category resolution before any regex evaluation.
- **PHP `guess_category` endpoint** — New server-side action that calls Gemini to classify a product name into a local category key, with file-based caching (`data/category_ai_cache.json`). Returns `altro` immediately when no Gemini API key is configured.

### Fixed
- **Duplicate banner alerts** — `loadBannerAlerts()` was occasionally enqueuing the same item multiple times when called concurrently. Fixed with a `_bannerLoading` re-entrancy guard and a `_queuedItemIds` Set that prevents any item from being pushed more than once per refresh cycle.
- **`mapToLocalCategory` with `en:dairies` / `en:dairies-and-eggs`** — The dairy regex was not matching OpenFoodFacts tags that use the `dairi` stem; extended to cover the full range of dairy tags.
- **`mapToLocalCategory` always returning `altro`** — When the input category was already `altro`, the function exited the direct-match loop before attempting any fallback, losing all name-based guesses. The loop now skips the `altro` key for the early-return and falls back to `guessCategoryFromName(productName)` at the end.
- **"Tonno all'olio" → condimenti** — `tonno\b` was matched after `olio\b` (condimenti) due to regex ordering. Moved the conserve block before the condimenti block so tuna products resolve correctly.

### Security
- **AI function guards** — All Gemini-powered functions now check `_geminiAvailable` (JS) or the presence of `GEMINI_API_KEY` (PHP) before executing. Affected functions: `_refineCategoryBadgesAsync`, `fetchAllPrices`, `getShoppingPrice`. The PHP endpoint returns `{"success":false,"error":"no_api_key"}` instead of silently returning empty results, making the missing-key state explicit and diagnosable.

## [1.7.8] - 2026-05-10

### Added
- **Trasferisci a Ricette dalla chat** — Quando la chat con Gemini Chef genera una ricetta, compare il bottone "📥 Trasferisci a Ricette". Premendolo, Gemini converte il testo in JSON strutturato completo (titolo, pasti, ingredienti, passi), il backend arricchisce ogni ingrediente con product_id e location via fuzzy-match (identico a generateRecipe), la ricetta viene salvata in archivio e si apre direttamente nella sezione Ricette con tutti i pulsanti "Usa" e la modalità cottura completa.
- **Bottone "Apri la ricetta"** — Dopo un trasferimento riuscito, il bottone "📥 Trasferisci a Ricette" si trasforma direttamente in "📖 Apri la ricetta" (stesso elemento DOM), evitando problemi di sovrapposizione.
- **Crea una ricetta per ingrediente** — Nel pannello azione di ogni alimento in inventario compare il bottone "👨‍🍳 Crea una ricetta con questo" (teal, larghezza piena). Premendolo, Gemini genera una ricetta italiana usando quell'alimento come protagonista (stesso pipeline di chatToRecipe: arricchimento fuzzy-match inventario, meal=null, 8192 token max).
- **meal non auto-categorizzato** — Le ricette generate da chat o da ingrediente non vengono più auto-categorizzate (meal rimane null); il tag pasto nell'UI viene mostrato solo se valorizzato.

### Fixed
- **Smart shopping: falso positivo "quasi finito"** — Se un prodotto in grammi/ml era quasi esaurito (es. Burro 30g = 12%) ma lo stesso prodotto era disponibile anche come confezione (Burro 1 conf = 99%), il sistema segnalava ugualmente "sta finendo". Ora verifica se la famiglia `shopping_name` ha scorte da altri prodotti: se sì, l'alert viene soppresso. (Esempio: 30g di Burro + 1 conf di Burro → nessun alert.)
- **Traduzioni JSON corrotte** — La sezione `action` era duplicata nei file `de.json`, `en.json` e `it.json`, causando errori di parsing che bloccavano la CI/CD. Rimossa la sezione spuria.

## [1.7.7] - 2026-05-10

### Fixed
- **Smart shopping family suppression** — La logica `recentlyExhausted` (prodotti terminati < 14gg) bypassava erroneamente anche la suppression per `shopping_name` family, causando falsi positivi: prodotti come Yaourt Vanille apparivano come urgenti anche con 2kg di Yogurt in stock, Salame Paesano con 1kg di Affettato in stock, Gran bauletto rustico con più pani in stock. Ora `recentlyExhausted` bypassa solo il check token-based (match lasco), mentre la family suppression per `shopping_name` si applica sempre.
- **Shelf life pre-warming nel cron** — Il cron ora chiama `prewarmShelfLifeCache()` ogni 5 minuti, precaricando via Gemini AI la shelf life degli item aperti in inventario (max 5 item per ciclo) prima che l'utente li visualizzi. Questo elimina il delay percepibile al primo click su "Aperto il...".

## [1.7.6] - 2026-05-10

### Fixed
- **`shopping_name` troncato (Piadina)** — Il prodotto "Piadine medie" aveva `shopping_name='Pi'` (troncato), non veniva aggruppato correttamente nella famiglia. Corretto in `Piadina`.
- **Family merges DB** — Grana Padano ora sotto `Formaggio` (era `Grana` singleton), Prosciutto cotto ora sotto `Affettato`, Panna acida ora sotto `Panna`.
- **`daily_rate` su periodo effettivo** — Il tasso di consumo giornaliero usava `first_in → now` come finestra, diluendo il rate con periodi in cui il prodotto era già esaurito (es. aglio esaurito a 34gg veniva calcolato su 60+). Ora usa `first_in → last_activity` (ultimo acquisto o ultimo uso), più preciso per le previsioni di riordino.
- **Anomaly dismiss key stabile** — La chiave di dismiss usava `product_id + round(expected)` che cambiava ad ogni nuova transazione, causando la ricomparsa delle anomalie già chiuse. Ora usa `product_id + direction` (phantom/missing/untracked) — stabile finché la direzione non cambia.
- **Smart shopping: prodotti esauriti < 14 giorni** — Prodotti terminati negli ultimi 14 giorni non vengono più soppressi dal check token-coverage o shopping_name-family: se li hai appena finiti, è probabile tu voglia ricomprarli indipendentemente dalla presenza di equivalenti in stock.
- **Chat pruning** — `chatSave()` ora esegue `DELETE` dei messaggi oltre i 200 più recenti dopo ogni salvataggio, evitando crescita illimitata della tabella `chat_messages`.
- **`getStats()` query consolidate** — Le 5 query separate (COUNT products, SUM inventory, COUNT locations, COUNT recent_in, COUNT recent_out) sono ora una sola query con subselect, riducendo i round-trip SQLite da 5 a 1.
- **Bring! cleanup rate-limiting** — Aggiunto `usleep(300ms)` tra le rimozioni multiple per evitare di sovraccaricare l'API Bring! in burst.
- **Indici compositi su `transactions`** — Aggiunti `idx_transactions_type_date(type, created_at)` (per `getStats`) e `idx_transactions_pid_type_undone(product_id, type, undone)` (per `smartShopping`), con migration automatica per DB esistenti.

### Security
- **CSRF protection** — Le action di scrittura (inventory_add, bring_add, product_save, ecc.) richiedono ora `X-EverShelf-Request: 1` oppure `Content-Type: application/json`. Il frontend `api()` invia sempre il header su POST. Questo previene attacchi CSRF cross-site tramite form HTML.

## [1.7.5] - 2026-05-10

### Added
- **Vacuum sealed prompt on item use** — After using a conf/weighted-unit item that still has remaining stock, a sliding popup asks "🔒 Messo sotto vuoto?" with Sì/No buttons and an 8-second auto-dismiss countdown bar. Default is Sì if the item was previously sealed, No otherwise. Works for all container units (conf, g, kg, ml, l) and any item previously marked as vacuum sealed.
- **Multi-function appliance awareness in recipes** — When the user sets a multi-function appliance (Cookeo, Bimby, Thermomix, Monsieur Cuisine, Instant Pot, Multicooker, Robot da cucina) in Settings, all Gemini recipe prompts (chat, recipe generation, weekly meal plan) now explicitly instruct the AI to consolidate as many cooking steps as possible into that single machine. Each appliance's available functions (rosolare, tritare, vapore, cuocere a pressione, etc.) are listed and the AI is required to indicate the specific mode/program at each step.
- **Server-side Bring! cleanup in cron** — `bringCleanupObsolete()` now runs every 5 minutes via cron without requiring any client page load. Items auto-added by the app (identified by `⚡`/`🟠`/`🛒` markers in their Bring! spec) are automatically removed when the smart shopping engine no longer flags them as needed. Works across all devices/clients.
- **`shopping_name` in `inventory_list` API** — The `inventory_list` endpoint now returns the `shopping_name` field from the products table, enabling family-based stock matching in the client-side cleanup fallback.

### Fixed
- **Bring! cleanup: false token match (Succo/Frutta)** — `bringCleanupObsolete` previously indexed smart items by product name tokens. "Pera Italiana **Succo** e polpa **frutta**" (shopping_name: "Pere") caused "Succo" and "Frutta" to be retained on Bring! indefinitely even when fully stocked. Now indexes **only** by `shopping_name` tokens.
- **Bring! cleanup: expired items with fresh family stock (Verdure)** — When a product is expired but its `shopping_name` family has ≥50% fresh stock from other products (e.g. Minestrone tradizione scaduto 01/05 but 590g fresh Verdure in freezer/pantry), it is no longer flagged as `critical` and is removed from the shopping list.
- **Bring! remove: catalog items not removed (Formaggio/Käse)** — `bringRemoveItem()` and `bringCleanupObsolete()` now try both the Italian display name and the Bring! internal German catalog key (e.g. `Käse` for `Formaggio`). Previously, catalog items with a German key were silently not removed.
- **Barcode scanner: EAN auto-submit on manual input** — Typing or pasting a valid 8/13-digit EAN in the manual barcode field now auto-submits immediately without needing to press a button. Checksum validation gives a warning toast for invalid codes without blocking entry.
- **Shopping list: `isExpiringSoon` false positives** — Products bought in bulk that expire naturally in 3 days (e.g. fresh produce) were flagged `medium` urgency on the shopping list despite having 100%+ stock. Now requires `pctLeft < 50%` before triggering.
- **Shopping list: expired batch with fresh restock suppressed** — Products with an expired batch AND a recent fresh restock (≥50% fresh stock) are no longer flagged `critical` for shopping. The expired-batch UI banner on the dashboard handles the disposal prompt instead.
- **Shopping list: cross-device cleanup** — Client-side `cleanupObsoleteBringItems()` now detects app-added items by their spec markers (`⚡`/`🟠`/`🛒`) instead of a per-device localStorage map, making cleanup work correctly on all clients including newly logged-in devices. Throttle reduced from 30 minutes to 3 minutes.
- **API fetch caching disabled** — All `api()` calls in the frontend now set `cache: 'no-store'` to prevent stale data from browser cache.
- **Shopping page multi-client sync** — Added 45-second polling on the shopping page so changes made on another device are reflected automatically.



### Added
- **AI price estimation for shopping list** — Each item on the Bring! shopping list now shows an estimated retail price badge (per unit and total). Prices are fetched from Gemini AI and cached server-side for 3 months (`PRICE_UPDATE_MONTHS`). The running estimated total is displayed both in the shopping tab and as a green pill badge on the dashboard stat card.
- **Dashboard price total badge** — The shopping stat card on the dashboard shows a green `ca. €X.XX` badge (top-right, same position as the old urgency badge). It updates in real-time as prices are calculated and persists across navigation via `sessionStorage`.
- **Background price refresh** — Prices are fetched silently every 2 minutes even when not on the shopping tab, keeping the dashboard badge current without user interaction.
- **Smart quantity estimation** — The price payload uses `smart_shopping` data (consumption patterns) to send the correct buy quantity per item; falls back to Bring! spec parsing, then to `qty=1, unit=conf` for manually-added items.

### Fixed
- **`stat-price-total` not visible on dashboard** — The total was only computed when `shoppingItems` was populated (i.e. shopping tab had been visited). Now uses `sessionStorage._pricetotal` as fallback so the badge is visible immediately on any page.
- **Price bar reloading on every tab switch** — `renderShoppingItems` now checks if ALL items are already cached with matching qty/unit; if so, it applies prices from cache instantly with no loading bar or API call.
- **`stat-price-total` real-time update** — Dashboard stat now increments as each individual item is priced (not only after the entire fetch completes).
- **Broken emoji in `log.title`** — Corrupted `\uFFFD` character in `it.json` and `de.json` replaced with `📒`.
- **`PRICE_CACHE_PATH` undefined crash** — Server-side constant was used inside functions that were called before the define; moved define to the very top of `api/index.php` (line 19). Affected: all `get_shopping_price` and `get_all_shopping_prices` calls from 16:33–16:40 on 2026-05-07.

## [1.7.1] - 2026-05-04

### Fixed
- **Destructive actions now require confirmation** — "Butta tutto" (`throwAll`) and "Finisci tutto" (`submitUseAll`) now display a confirmation modal before executing. The modal features a 5-second auto-confirm countdown bar (red) with an "Annulla" cancel button, matching the scale auto-confirm UX pattern already in use.
- **History undo button visibility** — The ↩ undo button in the transaction log was using `color: var(--text-muted)` making it nearly invisible. It now uses a red tint background + border (`#f87171`) with larger font size (1rem) for easy tap targeting.
- **History undo uses custom modal** — `undoTransactionEntry()` previously used the native browser `confirm()` dialog (broken in Android WebView kiosk mode). It now uses the same `_showDestructiveConfirm()` modal with countdown.



### Added
- **Demo mode (JS frontend)** — Full client-side demo experience: Gemini is treated as available, Bring! write operations silently no-op, and a mock pantry + shopping list is shown; activated via `?demo=1` URL param or `.env` `DEMO_MODE=true`; a "DEMO" badge is injected in the header and Settings is hidden to prevent accidental writes
- **Graceful Bring! no-key state** — When Bring! credentials are not configured the shopping tab shows a friendly localised message with a direct link to the Settings page instead of a raw API error
- **Use-quantity guard** — Consuming more than the quantity stocked at the selected location is now blocked before the API call; the quantity input shakes (CSS `input-shake` animation) and a toast shows `use.error_exceeds_stock`
- **Kiosk: smart auto-discovery rewrite** — `autoDiscover()` now uses `ExecutorCompletionService` + `NetworkInterface` (replaces deprecated `WifiManager`), 60 parallel threads, 600 ms TCP pre-check per host, real-time UI feedback every 120 ms, ports `[443, 80, 8080, 8443]`; VPN/cellular interfaces (tun, ppp, rmnet, pdp, ccmni, etc.) are filtered out and `wlan*`/`eth*` interfaces are prioritised
- **Kiosk: permissions button transform** — After permissions are granted, the button changes to "✅ Permessi concessi — Continua →" (green background, dark text) and advances to step 3 on tap, replacing the separate "permissions granted" card
- **Kiosk: gateway auto-pre-configuration** — On successful gateway install `finishSetup()` POSTs `scale_enabled=true` + `scale_gateway_url=ws://127.0.0.1:8765` to the server's `save_settings` endpoint so the webapp is scale-ready immediately after setup
- **Kiosk: ErrorReporter init at setup start** — `SetupActivity.onCreate()` now calls `ErrorReporter.init()` with any previously saved URL, ensuring errors in step 4 (gateway install) are reported even before the user confirms the server URL

### Fixed
- **Kiosk: wrong subnet scanned** — The previous implementation picked up VPN/tun interfaces and scanned a 10.x.x.x range instead of the device's actual Wi-Fi LAN; fixed by filtering interface names and preferring `wlan`/`eth`
- **Kiosk: port 443 missing from discovery** — HTTPS servers were never reachable during auto-discovery; ports list extended to `[443, 80, 8080, 8443]`
- **Kiosk: gateway install status=1 silent failure** — `PackageInstaller.STATUS_FAILURE` (status 1) showed an error card but never called `ErrorReporter`; `ErrorReporter.reportMessage()` is now called with status code, message, and package name
- **Screensaver toggle in web settings** — The screensaver row was missing a `<span class="toggle-slider">` inside the `<span class="toggle-switch">` wrapper, so no slider was rendered; corrected to use the same `toggle-row` / `toggle-switch` / `toggle-slider` structure as all other settings toggles
- **antiwaste.title translation** — IT and DE locale files were missing the `antiwaste.title` key, causing a raw key string to appear in the anti-waste section header; added to both `it.json` and `de.json`

### Kiosk (v1.4.0 → v1.5.0)
- `autoDiscover()` fully rewritten (CompletionService, NetworkInterface, TCP pre-check, real-time feedback, correct LAN subnet)
- Port 443 added to discovery scan
- Permissions button transforms after grant (`onPermissionsGranted()`)
- `ErrorReporter.init()` called at `SetupActivity.onCreate()`
- `ErrorReporter.reportMessage()` called on gateway install failure
- `finishSetup()` pre-configures gateway via `save_settings` API call

## [1.6.0] - 2026-05-03

### Added
- **Dashboard skeleton loading** — Stat cards (Dispensa / Frigo / Freezer) show an animated shimmer placeholder (`…`) instead of the jarring `0` flash that appeared for 3–5 seconds before data loaded; the loading class is applied before the API call and removed atomically when data arrives
- **Webapp startup preloader** — Full-screen spinner overlay during initial app load, fades out after the dashboard is ready
- **Webapp update notification** — A dismissible top banner alerts the user when a newer GitHub release is available (checked once every 6 hours, comparison based on `published_at`)
- **Native Android update banners** — Both Kiosk (v1.4.0) and Scale Gateway (v2.1.0) show a native top bar when a newer APK is available, with one-tap download and install

### Fixed
- **APK install conflict** — Replaced `ACTION_VIEW`-based APK install with the `PackageInstaller.Session` API (API 21+) in both Kiosk and Scale Gateway; the session-based approach correctly handles:
  - `STATUS_PENDING_USER_ACTION` → automatically launches the system confirmation dialog
  - `STATUS_SUCCESS` → success toast
  - `STATUS_FAILURE_CONFLICT` / `STATUS_FAILURE_INCOMPATIBLE` → `AlertDialog` offering to uninstall the old app (signature mismatch) before reinstalling
- **Cooking mode z-index** — Update banner and app header are now hidden when `body.cooking-mode-active` is set, and the cooking overlay z-index was raised to `99998` so it can no longer be obscured by UI chrome
- **Version-aware error reporting** — GitHub Issues are only created when the client is running the latest released version, avoiding noise from stale deployments; non-semver tag names (e.g. `"latest"`) are treated as "always up-to-date"
- **XOR-obfuscated GitHub token** — The PAT used for GitHub API calls is stored as an XOR-encoded hex string in both the PHP backend and Kotlin apps to prevent accidental exposure via secret scanning

### Kiosk (v1.3.0 → v1.4.0)
- FileProvider + `REQUEST_INSTALL_PACKAGES` permission added
- APK download destination moved to `getExternalFilesDir(null)` (no storage permission needed)
- `PackageInstaller` self-update with signature-conflict recovery
- BLE scale gateway update banner with download + install flow

### Scale Gateway (v2.0.0 → v2.1.0)
- Same FileProvider + permission + `PackageInstaller` changes as Kiosk
- Update banner for self-update
- CI workflow now triggers on `develop` branch (in addition to `main`)

## [Unreleased] - 2026-04-30

### Fixed
- **Low-qty banner false positive** — A "suspiciously low quantity" review alert is now suppressed for a partially-used inventory entry when one or more sibling entries for the same product (identified by barcode, or name+brand as fallback) exist in other locations with stock > 0. Prevents noise like "191 ml of milk" when 11 sealed packages are stored in the pantry.

### Changed
- **Non-alarmist expired banner** — Banner icon, CSS class, and title suffix now adapt to the `getExpiredSafety()` level:
  - `ok` (long-life products, freezer within margin): green banner, ✅ icon, "— Scaduto (ancora ok)"
  - `warning` (items that should be inspected): amber/yellow banner, 👀 icon, "— Scaduto (controlla)"
  - `danger` (raw meat, dairy, fish, etc.): unchanged red 🚫 banner and "— Scaduto!" title
- Added `expiry.expired_suffix_ok` and `expiry.expired_suffix_warning` i18n keys to all three language files (IT/EN/DE)
- Added `banner-expired-ok` and `banner-expired-warning` CSS variants (green / amber) in `style.css`

## [1.5.0] - 2026-04-28

### Added
- **Expired banner for opened products** — Products whose opened-product shelf-life has passed (e.g. fridge cream opened 6 days ago) now appear in the top notification banner, not just the dashboard list
- **Safety-aware expired banner** — Each expired banner item shows a contextual safety tip (from `getExpiredSafety()`); danger-level items (fridge dairy/meat/fish) get an intense red banner and "L'ho buttato" as the primary button; safe/warning items keep the original button order
- **AI model fallback** — All Gemini API endpoints (expiry scan, product identification, chat, recipe non-streaming, shopping name classifier) now try `gemini-2.5-flash` first and fall back to `gemini-2.0-flash` automatically, matching the resilience already in place for recipe streaming
- **Friendly AI quota message** — When the AI returns a quota/rate-limit error the user sees "Quota AI esaurita. Riprova tra qualche minuto." instead of the raw API error string
- **Cooking TTS auto-read** — Each recipe step is read aloud automatically when navigating forward or backward; the first step is also read when entering cooking mode
- **Cooking timer 10-second warning** — When a cooking timer reaches 10 seconds the TTS announces "Attenzione! [label]: mancano 10 secondi!"
- **Cooking recipe completion announcement** — "Ricetta completata! Buon appetito!" is spoken via TTS when the last step is confirmed

### Fixed
- **Cooking TTS gate** — `speakCookingStep()` was blocked by the global `tts_enabled` setting; the `_cookingTTS` toggle (🔊/🔇 button) is now the only gate; browser Web Speech API is used by default without requiring TTS configuration in Settings
- **Anomaly dismiss label** — The "La quantità è giusta" button now appends the current inventory quantity, e.g. "La quantità è giusta (2 pz)", so the action is unambiguous
- **i18n sync** — Added `timer_warning_tts`, `recipe_done_tts`, `error.ai_quota` keys to all three language files (IT/EN/DE)


### Added
- **Generic shopping names** — Products are grouped by type ("Latte", "Affettato", "Pasta") rather than brand; computed via an expanded keyword map with Google Gemini AI as fallback for unknown products
- **Bring! auto-migration** — Existing list items with old specific names are silently migrated to generic names on every list load, throttled to once per 10 minutes
- **Bring! catalog coverage** — All 93 shopping_name values now resolve to a German Bring! catalog key (icons and categories in the Bring! app); 24 aliases added to cover previously unmatched names
- **Auto-add to Bring! on depletion** — When a product reaches zero the app adds it to Bring! automatically using the generic shopping name, with the specific product name and brand in the specification field
- **Finished-product confirmation banner** — Instead of silently deleting zero-stock entries, a banner prompts the user to confirm; banner title includes the last 3 digits of the product barcode for easier identification
- **Anomaly detection banner** — Dashboard notifications for suspicious inventory/transaction mismatches and consumption prediction errors, with one-tap inline correction
- **SSE recipe streaming** — Recipe generation streams live via Server-Sent Events; Gemini agent feedback is shown in real time as it is generated
- **Smart alert banners** — Configurable expired-only mode with explanatory messages; banner buttons are fully internationalized

### Fixed
- **Scale double-deduction** — Multiple BLE stable readings of the same weight no longer fire duplicate `inventory_use` events; JS preserves the confirmation sentinel on submit and PHP rejects a second `out` transaction for the same product within 12 seconds
- **Kiosk native TTS** — CI workflow now builds the APK on `develop` branch too; the native Android `TextToSpeech` bridge bypasses Web Speech API voice-availability issues without requiring offline voice packs
- **TTS voice loading** — Retries for up to 10 seconds on page load; shows a message if no voices are available and offers a manual refresh button
- **Bring! migration** — Corrected two bugs: wrong removal API (`DELETE /item` → `PUT remove=item`) and wrong purchase key sent to Bring! (Italian shopping name → German catalog key), which previously created Italian/German duplicate entries
- **Gemini 429 rate limiting** — API calls are retried with exponential backoff; recipe requests are capped at 5 per minute with a dedicated rate-limit bucket

### Performance
- **Gemini calls centralized** — All Gemini API requests go through a single `callGemini()` helper with intelligent backoff; Gemini removed from the product-selection and bringSuggest flows in favour of fast offline logic

## [1.3.0] - 2026-04-18

### Added
- **Expired product banner** — Dashboard notifications for expired products with use, throw away, edit, and dismiss actions
- **Expiring soon banner** — Dashboard notifications for products expiring within 3 days with use, edit, and dismiss actions
- **Priority-sorted notifications** — Banner alerts sorted by urgency: expired > expiring > suspicious quantities > consumption predictions
- **Swipe navigation** — Touch swipe left/right to browse banner notifications, with dot indicators and arrow buttons
- **Quick-access buttons** — Inventory page shows 4 recently used and up to 8 most popular products for quick selection
- **Recent & popular products API** — New `recent_popular_products` endpoint
- **Auto-refresh** — Banner notifications refresh every 5 minutes while on the dashboard
- **Edit from expiry banner** — Correct expiry dates directly from expired/expiring notifications

### Fixed
- **Negative scale values** — BLE scale readings with negative weight are now ignored
- **Banner re-appearing after edit** — Editing from a banner now persists the confirmation so it doesn't reappear on dashboard reload
- **False consumption predictions** — Manual inventory edits (updated_at > last restock) now use the correct baseline for prediction calculations
- **Kiosk overlay blocking header** — Removed injected exit/refresh buttons from the web app header in kiosk mode

## [1.2.0] - 2026-04-13

### Changed
- **Project renamed** from "Dispensa Manager" to **EverShelf**
- Contact email updated to `evershelfproject@gmail.com`
- Docker service, container, and volume renamed to `evershelf`
- SQLite database renamed from `dispensa.db` to `evershelf.db`
- All localStorage keys migrated: `dispensa_*` → `evershelf_*`
- Apache config file renamed to `evershelf.conf`
- CI workflow Docker image/container names updated
- App name updated in all translations (it, en, de)
- Navigation title updated to EverShelf across all languages

### Added
- Version badge (`v1.2.0`) in the app header

### Fixed
- JS file truncation caused by `sed` in-place edit on large files
- Browser cache invalidation via bumped asset version strings (`?v=20260413a`)

## [1.0.0] - 2026-04-10

### Added
- Complete pantry inventory management (Pantry, Fridge, Freezer, Other)
- Barcode scanning with QuaggaJS
- Open Food Facts barcode lookup
- Google Gemini AI integration (product identification, expiry reading, recipes, chat)
- Bring! shopping list integration
- Smart shopping predictions with cron-based caching
- Cooking mode with step-by-step guidance and TTS support
- Opened product tracking with reduced shelf-life calculation
- Vacuum-sealed product support with extended expiry
- Waste vs. consumption tracking (30-day chart)
- Expired product safety assessment by category
- Weekly meal plan configuration
- DupliClick online grocery ordering integration
- PWA support (installable, mobile-first)
- Local database backup script
- Multi-device settings sync via SQLite

### Security
- Centralized `.env` configuration (secrets never in code)
- Removed all hardcoded credentials and personal data
- Input validation on inventory operations
- Parameterized SQL queries throughout
