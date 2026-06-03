# PRD â€” UNITEX
### Compilatore LaTeX 100% client-side, eseguito interamente nel browser

**Documento:** Product Requirements Document (v1.0)
**Prodotto:** UNITEX
**Categoria:** Developer / Academic Tooling â€” WebAssembly
**Stato:** Draft per validazione MVP

---

## 1. Vision & Elevator Pitch

**Elevator Pitch**
UNITEX Ã¨ il primo compilatore LaTeX *live* completamente gratuito che gira **direttamente nel browser**, senza alcun backend di compilazione. Il motore TeX in C Ã¨ compilato in WebAssembly (WASM) ed eseguito sul dispositivo dell'utente: niente installazioni, niente attese in coda, nessun dato che lascia la macchina. Si scrive, si compila, si ottiene il PDF â€” istantaneamente.

**Core Identity**
L'identitÃ  del prodotto poggia su tre pilastri non negoziabili:

- **Zero server di compilazione.** Non esiste infrastruttura che esegue TeX lato server. Questo azzera il principale costo variabile di ogni piattaforma concorrente (CPU/RAM per utente) e rende i costi marginali per utente praticamente nulli, abilitando un modello gratuito sostenibile su larga scala.
- **Privacy assoluta.** Documenti, sorgenti e PDF non vengono mai trasmessi. La compilazione Ã¨ locale per design, non per policy: Ã¨ architetturalmente impossibile per noi (o per terzi) leggere il contenuto degli utenti. Argomento decisivo per ricerca riservata, tesi, brevetti e materiale accademico sensibile.
- **Compilazione istantanea.** Eliminando il round-trip di rete verso un server e i timeout delle code condivise, il tempo percepito di compilazione Ã¨ limitato solo dalla CPU locale.

**Vision di lungo termine**
Diventare lo standard *de facto* per la scrittura LaTeX in ambito accademico e tecnico, costruendo un bacino utenti ampio e fedele che funga da *moat* tecnico. Gli obiettivi strategici di lungo periodo sono, alternativamente o in sequenza:

- **Monetizzazione** tramite native advertising mirato e non intrusivo (vedi Â§6);
- **Exit** â€” acquisizione o pivot B2B/enterprise una volta consolidata la base utenti.

---

## 2. Target Market & Competitor Analysis

### 2.1 Target di mercato

Il pubblico primario Ã¨ composto da utenti che producono documenti LaTeX *pesanti* e ricorrenti:

- **Studenti STEM universitari** (Informatica, Ingegneria, Matematica, Fisica) che compilano appunti densi, dispense ed esercizi con ambienti matematici complessi (es. requisiti tipici di un esame di **Analisi 1**).
- **Ricercatori e dottorandi** che redigono paper, preprint e materiale soggetto a vincoli di riservatezza.
- **Professori e docenti** che preparano slide, esercizi e dispense.
- **Autori di tesi** e **autori di libri tecnici** che lavorano su documenti lunghi e strutturati.

### 2.2 Analisi dei competitor

**Overleaf** â€” *editor cloud collaborativo*
- Piano a pagamento di fatto necessario per un uso serio (tempi di compilazione, progetti, collaboratori).
- Nessuna vera modalitÃ  offline.
- Assenza di template predefiniti per le **universitÃ  italiane**.
- Dati ospitati su server di terze parti â†’ nessun controllo reale sul dato e timeout di compilazione frequenti sui documenti complessi.

**SwiftLaTeX** â€” *engine WASM*
- Installazione/integrazione manuale e complessa.
- Nessuna UI definita: Ã¨ un motore, non un prodotto.
- Performance lente.
- Download iniziale molto pesante.
- Limitato ai pacchetti pre-caricati.

**TeXlyre** â€” *layer collaborativo su SwiftLaTeX*
- Problemi noti con il rendering SVG.
- Dipendenza diretta da SwiftLaTeX (ne eredita i limiti).
- Collaborazione lenta con file di grandi dimensioni.
- FunzionalitÃ  ancora sperimentali.
- La modalitÃ  offline non sincronizza finchÃ© non torna la connessione.

**TeX.js / engine JavaScript** â€” *porting JS del motore*
- Stack obsoleto (di fatto fermo dal 2023).
- JavaScript invece di WASM â†’ performance inferiori.
- Impossibile aggiungere nuovi pacchetti.
- Funzionano solo i pacchetti base; ricompilare dopo l'aggiunta di pacchetti Ã¨ impraticabile.

**Tectonic** â€” *engine moderno*
- Scritto in Rust invece di C/C++ (ecosistema TeX storicamente in C).
- Non Ã¨ un editor completo: Ã¨ solo un engine.
- Nessuna UI predefinita.
- Nessuna preview live.

### 2.3 Unique Value Proposition

UNITEX Ã¨ l'unica soluzione che combina **engine performante + prodotto completo + privacy by design + gratuitÃ  sostenibile**:

- **Nessun timeout** di compilazione.
- **Offline-first** reale.
- **100% client-side.**
- **Web UI moderna** e curata.
- **P2P efficiente** con ottimizzazione per file di grandi dimensioni.
- **WYSIWYG pronto all'uso**, zero configurazione.
- **Package on-demand intelligente** + caching e loading progressivo.
- **100% gratuito.**
- **Tanti layout disponibili**, incluso un **template dedicato alle universitÃ  italiane** â€” vuoto di mercato lasciato scoperto da tutti i competitor.

---

## 3. Technical Architecture (The Core)

L'intera proposta di valore dipende dalla soliditÃ  del motore eseguito nel browser. L'architettura Ã¨ vincolata da tre requisiti rigidi: stare nel modello di memoria del browser, caricare i pacchetti senza scaricare l'intera distribuzione TeX, e funzionare offline.

### 3.1 Engine TeX in C â†’ WebAssembly

- Il motore TeX (basato su codice C di basso livello) Ã¨ compilato in **WASM**, eseguito nel main thread *o*, preferibilmente, in un **Web Worker** per non bloccare la UI.
- WASM garantisce performance vicine al nativo, nettamente superiori a qualsiasi porting in JavaScript puro.

### 3.2 Gestione della memoria (vincolo critico)

La causa principale di crash dei compilatori WASM Ã¨ l'esaurimento di memoria. Linee guida obbligatorie:

- **Heap WASM con limiti espliciti** e politica di crescita controllata; evitare allocazioni illimitate che provocano il crash della tab.
- **Reset/teardown dell'istanza** dell'engine tra compilazioni pesanti per liberare memoria frammentata.
- **Monitoraggio del consumo** in runtime con soglie di guardia: oltre soglia, degradare con grazia e mostrare un errore gestito invece di far crashare il browser.
- **Streaming dell'I/O** dei file piÃ¹ grandi anzichÃ© caricarli interamente in memoria quando possibile.

### 3.3 Virtual File System (VFS) + lazy loading dei pacchetti

- Un **Virtual File System** in memoria emula la struttura `texmf` attesa dall'engine.
- I pacchetti LaTeX **non** vengono pre-scaricati in blocco. Sono caricati **on-demand (lazy loading)** da una **CDN**: quando il documento richiede un pacchetto (es. `tikz`, `pgfplots`, `amsmath`), il VFS intercetta la richiesta, scarica solo i file necessari e li monta.
- Questo abbatte il download iniziale (principale debolezza di SwiftLaTeX) mantenendo accesso all'intero ecosistema TeX.

### 3.4 Caching persistente & offline

- I pacchetti scaricati vengono salvati in **IndexedDB**, cosÃ¬ le compilazioni successive sono **istantanee** e non richiedono rete.
- Un **Service Worker** gestisce il caching dell'app shell e dell'engine WASM, abilitando l'**uso completamente offline** dopo il primo caricamento.
- Strategia di cache **intelligente con loading progressivo**: si serve subito ciÃ² che Ã¨ in cache, si aggiorna in background.

---

## 4. MVP Features & Stress Testing

### 4.1 Interfaccia (MVP)

- **Editor split-screen essenziale:** codice a sinistra, **PDF a destra**.
- **Sidebar laterale** per gestione file, template e stato dei pacchetti.
- Compilazione **live** con feedback immediato.

### 4.2 Requisiti di performance (criterio di accettazione)

L'engine deve essere affidabile sotto carico reale. L'MVP Ã¨ considerato valido solo se supera questi stress test:

- **Compilazione di file pesanti** (dispense lunghe, documenti con centinaia di pagine) senza crash della memoria.
- **Ambienti matematici complessi:** dimostrazioni, sistemi, allineamenti estesi (`amsmath`, `amsthm`, `mathtools`).
- **Grafica generata da librerie:** **TikZ** e **PGFPlots** con grafici complessi â€” benchmark di riferimento: il set tipico di un esame di **Analisi 1**.

> **Definition of Done dell'MVP:** un documento "Analisi 1" (testo denso + dimostrazioni + grafici TikZ/PGFPlots) compila correttamente, offline, dopo il primo caricamento dei pacchetti, senza timeout nÃ© crash.

---

## 5. UI/UX & Design Standards

L'obiettivo Ã¨ una percezione *premium*: UNITEX **non deve sembrare un tool open-source clunky**, ma un prodotto da agenzia web di alto livello.

- **Estetica minimalista e professionale:** gerarchia visiva pulita, tipografia curata, palette sobria, ampio uso di spazio negativo.
- **IdentitÃ  visiva coerente** e riconoscibile, che comunichi affidabilitÃ  e velocitÃ .
- **Feedback visivo chiaro** durante le operazioni in background â€” in particolare il **download dei pacchetti**: indicatori di progresso espliciti (es. "Scaricamento di `pgfplots`â€¦"), cosÃ¬ l'utente non percepisce il lazy loading come un blocco o un errore.
- **Stato della compilazione** sempre visibile: in corso, completata, errore (con log leggibile).
- Coerenza cross-device, layout responsive e interazioni fluide a 60fps.

---

## 6. Business Model & Roadmap

### 6.1 Modello di business

**Fase 1 â€” Adozione (100% gratuito).**
Nessun paywall. L'assenza di costi server rende sostenibile la gratuitÃ  totale, leva principale per la crescita rapida della base utenti.

**Fase 2 â€” Monetizzazione (native advertising mirato).**
Inserimento di **annunci nativi altamente curati e non intrusivi**, mirati alla nicchia STEM (es. cloud provider, tool per sviluppatori, tech recruiting). Vincolo di design: gli annunci **non devono mai intaccare lo spazio di scrittura** nÃ© l'esperienza di compilazione.

### 6.2 Roadmap

- **MVP** â€” Editor split-screen, engine WASM, VFS + lazy loading, caching offline, supporto matematica/TikZ/PGFPlots.
- **Crescita** â€” Libreria estesa di template (incluso il set per **universitÃ  italiane**), miglioramenti di performance, P2P per file grandi.
- **Feature future** â€” Supporto cloud opzionale (sync/backup *senza* compromettere il modello privacy-first) e **strumenti AI** integrati.
- **Lungo termine** â€” Costruzione del *moat* tecnico sulla base utenti, in vista di **monetizzazione, exit o pivot B2B/enterprise**.
