# Handoff — Report recensioni app Vittoria Assicurazioni

**File di lavoro:** `report-vittoria-app.html` (report HTML standalone, 1.229 righe)
**Branch:** `claude/vittoria-app-review-report-y7csiw`
**Commit:** `af6927e` (modifiche richieste) → `a19c631` (documentazione app analizzate)
**Stato:** 6 richieste su 6 implementate. 2 sono in attesa di dati esterni che questa sessione non è riuscita a recuperare.

---

## 1. Cosa è stato chiesto e cosa è stato fatto

| # | Richiesta | Stato |
|---|---|---|
| 1 | Indicazione specifica del periodo di analisi | ✅ Completo |
| 2 | Valutazione "totale" (incluse le sole stelline) | ⚠️ UI pronta, **manca il dato** |
| 3 | Confronto totale vs media recensioni con testo | ⚠️ UI e calcoli pronti, dipende da #2 |
| 4 | Sentiment: indicare la % non interpretabile | ✅ Completo |
| 5 | Filtrare criticità per Apple / Android | ⚠️ Filtro funzionante, **manca lo split** |
| 6 | Extra: focus ultimo anno | ✅ Completo |

### Dettaglio di quanto già implementato

**#1 Periodo** — Fascia dedicata sotto l'hero: finestra osservata (1 mag 2023 → 24 mar 2026, 35 mesi), data di estrazione, base con testo (2.284), base datata (988). Periodo replicato in header, hero, titolo del grafico trend e footer.

**#2/#3 Valutazione totale** — KPI in panoramica + sezione "Valutazione totale vs recensioni con testo": tre card (totale / con testo / scostamento), tabella per store con scostamento riga per riga, nota metodologica sul perché le due letture divergono. Tutti i valori derivati si calcolano da soli: voto complessivo ponderato sul numero di valutazioni, quota di valutazioni silenziose, scostamenti per store e complessivo.

**#4 Sentiment** — Aggiunta la quarta classe **Non interpretabile 5,0% (≈114 recensioni)** con nota che spiega perché 43,5 + 11,5 + 40,0 = 95 e non 100. Corretta anche la barra del negativo, disegnata a `flex:45` mentre la legenda diceva 40,0%.

**#5 Filtro store** — Segmented control Tutti / Android / Apple che filtra aree tematiche e matrice priorità, con rendering store-aware, ordinamento per volume e barre relative al massimo della vista corrente. Finché `STORE_SPLIT` è `null`, selezionando un singolo store compare un pannello esplicito con i KPI reali di quello store, invece di numeri aggregati che verrebbero letti come dati di store.

**#6 Focus ultimo anno** — Sezione apr 2025 / mar 2026: 336 recensioni, ★ 3,87, −0,05 sui 12 mesi precedenti, ultimo semestre ★ 3,74 (−0,23). Grafico combinato rating (linea) + volume (barre) e due insight sulla dinamica del trimestre.

---

## 2. I due dati che mancano

### `STORE_TOTALS` — riga 660 di `report-vittoria-app.html`

```js
const STORE_TOTALS = {
  android: { rating: null, count: null },   // Google Play Store
  apple:   { rating: null, count: null }    // Apple App Store
};
```

- `rating` → voto medio mostrato sulla scheda dello store (es. `4.1`)
- `count` → numero totale di valutazioni (es. `18500`)

È il voto pubblico, quello che comprende anche le valutazioni con sole stelline. **Non è ricavabile dalle 2.284 recensioni analizzate**: va letto dalla scheda store o da uno scraper. Compilato questo oggetto, si popolano da soli: KPI in panoramica, le tre card di confronto, la tabella per store e tutti gli scostamenti. La nota gialla "dato da completare" scompare automaticamente.

### `STORE_SPLIT` — riga 665

```js
const STORE_SPLIT = null;
```

Struttura attesa (le chiavi devono corrispondere **esattamente** al campo `name` degli array `topics` e `frictions`):

```js
const STORE_SPLIT = {
  topics: {
    'Pagamento Polizza': {
      android: {count: 300, avg: 2.80, neg: 50.1},
      apple:   {count: 107, avg: 2.30, neg: 60.4}
    },
    // ... una entry per ciascuna delle 10 aree tematiche
  },
  frictions: {
    'App si blocca al login': {
      android: {count: 70, avg: 1.70, neg: 80.0},
      apple:   {count: 27, avg: 1.40, neg: 88.9}
    },
    // ... una entry per ciascuna delle 10 frizioni
  }
};
```

`count` = menzioni, `avg` = rating medio, `neg` = % di valutazioni negative. Le entry mancanti non rompono nulla: un topic senza dati per uno store semplicemente non compare in quella vista.

**Alternativa migliore:** il dataset grezzo delle recensioni con il campo store. Da quello si ricava lo split per store con una riclassificazione, senza dover ricalcolare a mano 40 terne.

> Nomi esatti delle chiavi: gli array `topics` e `frictions` sono nello script del file, subito dopo `trendData` (riga 677 e seguenti).

---

## 3. Perché questa sessione non ce l'ha fatta

Tre blocchi indipendenti, tutti verificati:

1. **Tool Apify assenti.** `ListConnectors` restituisce Ahrefs, ECDB, Figma, Gmail, Google Calendar, Google Drive, mcp-ga4-remote, Meta MCP, Notion, Pinterest, Twist-MCP, Vertex, vidIQ. Nessun Apify. Ricerca per nome dei tool tipici (`call-actor`, `search-actors`, `get-dataset-items`) → nessun risultato. Config MCP di progetto vuota (`mcpServers: {}`).
2. **Rete bloccata.** `api.apify.com`, `apify.com`, `mcp.apify.com`, `console.apify.com`, `itunes.apple.com`, `apps.apple.com`, `amp-api.apps.apple.com`, `play.google.com`, `cdnjs.cloudflare.com` → tutti **403 al CONNECT** dal gateway. `api.apify.com` e `apify.com` sono stati aggiunti all'allowlist *durante* la sessione, ma la network policy è fissata all'avvio del container: la modifica non si applica a una sessione già in corso.
3. **Nessun token Apify** nell'ambiente, quindi nemmeno la REST API diretta era autenticabile.

Provate anche le alternative: ECDB non copre Vittoria (è un database e-commerce, `Not Found` su tutte le varianti), e gli host Apple alternativi sono bloccati come gli altri.

---

## 4. Da fare nella nuova sessione

### Prerequisiti da verificare *prima* di iniziare

- [ ] **Allowlist domini** già salvata e sessione **nuova** (non riavviata: nuova, così la policy viene applicata). Domini utili: `api.apify.com`, `apify.com`, `mcp.apify.com` — e, se si vuole poter verificare il rendering dei grafici in locale, anche `cdnjs.cloudflare.com`.
- [ ] **Connettore Apify attivo in questa chat.** Verificare con `ListConnectors`: serve `connected: true` **e** `enabledInChat: true`. Se è `connected: true` ma `enabledInChat: false`, va abilitato nelle impostazioni connettori della chat.
- [ ] In alternativa al connettore: un `APIFY_TOKEN` nelle variabili d'ambiente, per usare la REST API.

### App da analizzare

| Store | Identificativo | URL |
|---|---|---|
| Apple App Store | MyVittoria, id `444690713`, storefront IT | https://apps.apple.com/it/app/myvittoria/id444690713 |
| Google Play Store | `it.vittoria.assicurazioni`, storefront IT | https://play.google.com/store/apps/details?id=it.vittoria.assicurazioni&hl=it |

### Sequenza operativa

1. **Cerca gli actor** nello store Apify (via il tool di ricerca actor del connettore MCP, non per nome a memoria: i nomi degli actor cambiano e non sono stati verificabili da questa sessione). Servono due cose: uno scraper delle recensioni Google Play e uno delle recensioni App Store. Molti actor restituiscono anche i metadati di scheda, che includono già il rating aggregato e il numero totale di valutazioni.
2. **Estrai i metadati di scheda** delle due app → compila `STORE_TOTALS`. Questo da solo chiude le richieste #2 e #3.
3. **Estrai le recensioni** con, per ciascuna: store, data, rating, testo. Copertura obiettivo: dal 2023-05 a oggi, per poter confermare o aggiornare la serie mensile.
4. **Riclassifica per store** le 10 aree tematiche e le 10 frizioni → compila `STORE_SPLIT`. Questo chiude la richiesta #5, che è il pezzo di maggior valore.
5. **Aggiorna la data di estrazione** in tutti i punti dove compare (vedi §6): oggi il report è fermo al 24 marzo 2026.

### Opportunità: con il dataset grezzo si possono correggere due limiti noti

- La serie mensile copre **988** delle 2.284 recensioni (le altre non hanno data attribuibile al mese). Con dati freschi si può ricostruire la serie completa ed eliminare la nota "il trend va letto come andamento relativo, non come livello assoluto".
- Il **5,0% di sentiment non interpretabile** è oggi una quota residua dedotta (100 − 95). Con i testi a disposizione si può classificarla esplicitamente.

---

## 5. Numeri già derivati (da non ricalcolare)

Tutti verificati con i dati presenti nel file. Le medie di periodo sono **ponderate per il numero di recensioni di ciascun mese**.

**Base e periodo**
- Recensioni con testo: 2.284 — Play Store 1.757 (76,9%, ★ 3,44) · App Store 527 (23,1%, ★ 2,22)
- Rating complessivo con testo: ★ 3,16 — coerente con la media ponderata dei due store
- Serie mensile: 35 mesi, mag 2023 → mar 2026, **988** recensioni datate, media ponderata ★ 4,01

**Distribuzione stelle** (somma esattamente 2.284)
- 4-5★ = 1.240 → **54,3%** · 3★ = 131 → **5,7%** · 1-2★ = 913 → **40,0%**

**Sentiment su base 2.284**
- Positivo 43,5% (≈994) · Neutro 11,5% (≈263) · Negativo 40,0% (≈914) · **Non interpretabile 5,0% (≈114)**

**Focus ultimo anno**
- Ultimi 12 mesi (apr 2025 – mar 2026): **336** recensioni, **★ 3,87**
- 12 mesi precedenti (apr 2024 – mar 2025): 302 recensioni, ★ 3,92 → **Δ −0,05**
- Semestre apr–set 2025: 188 rec., ★ 3,97 · semestre ott 2025 – mar 2026: 148 rec., **★ 3,74** → **Δ −0,23**
- Q1 2026: 73 rec., **★ 3,43** (gen 3,5 · feb 3,4 · mar 3,3 — minimo della serie, tre mesi consecutivi in discesa)
- 7 mesi su 12 sotto ★ 4,0 · volume +11% sull'anno precedente

---

## 6. Incongruenze del report originale

### Già corrette in `af6927e`

- **KPI sentiment con conteggi errati.** Le card mostravano "Sentiment Positivo 54,3% — 757 recensioni" e "Sentiment Negativo 40,0% — 763". 54,3% e 40,0% sono in realtà le quote **4-5★** e **1-2★** (1.240 e 913 su 2.284, verificato), mentre 757 e 763 non corrispondono ad alcuna base calcolabile. Le card sono state rietichettate per fascia di stelle con i conteggi corretti: i numeri che il cliente già conosce restano, ma ora tornano.
- **Barra sentiment negativa** disegnata a `flex:45` con legenda a 40,0%.

### Da sapere, non risolvibili senza dati nuovi

- **Media stelle vs media dichiarata.** La media pesata sulla distribuzione stelle dà ★ 3,25, il report dichiara ★ 3,16. Lo scarto è compatibile con l'arrotondamento delle medie per store (3,44 e 2,22), ma non è stato possibile verificarlo sui dati grezzi.
- **Serie mensile più "generosa" della base.** ★ 4,01 sulle 988 datate contro ★ 3,16 sulle 2.284. Il sottoinsieme datato non è rappresentativo del totale. Segnalato nel report con due note esplicite (sezione trend e focus ultimo anno).

---

## 7. Come verificare le modifiche

Chart.js arriva da `cdnjs.cloudflare.com`, che era bloccato: per testare in locale conviene sostituire il tag script con uno stub e sondare il DOM renderizzato.

```bash
# Sintassi JS senza browser
python3 -c "import re;print(re.findall(r'<script>(.*?)</script>',open('report-vittoria-app.html').read(),re.S)[0])" > /tmp/c.js
node --check /tmp/c.js

# Rendering: stub di Chart + screenshot / dump-dom
/opt/pw-browsers/chromium-1194/chrome-linux/chrome --headless --disable-gpu --no-sandbox \
  --hide-scrollbars --virtual-time-budget=4000 --window-size=1200,7200 \
  --screenshot=full.png file:///percorso/view.html
```

Casi da coprire, tutti verificati funzionanti nello stato attuale:
- `STORE_TOTALS` vuoto → `—` con badge "dato da inserire" e nota gialla
- `STORE_TOTALS` compilato → voto complessivo ponderato, scostamenti, tabella, nota gialla nascosta
- Filtro su Android/Apple **senza** `STORE_SPLIT` → pannello esplicativo, dati aggregati nascosti
- Filtro su Android/Apple **con** `STORE_SPLIT` → solo gli elementi con dati, metriche per store, base aggiornata
- Ritorno su "Tutti gli store" → ripristino completo dei 10 topic e delle 10 frizioni

---

## 8. Vincolo da rispettare

I dati non ricavabili non vanno stimati né riempiti con valori plausibili: è un report che va al cliente. Le due costanti sono documentate in testa allo script proprio perché lo stato "manca il dato" sia esplicito nell'output invece di essere mascherato da un numero inventato. Se un dato resta irrecuperabile, meglio lasciare il placeholder visibile e dirlo.
