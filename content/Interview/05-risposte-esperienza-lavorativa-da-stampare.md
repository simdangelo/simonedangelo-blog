# Risposte — Ramo 1, versione senior

**Che cos'è questo documento.** È una simulazione di colloquio in prima persona, scritta per essere raccontata a voce e non recitata. Rispetto alla versione precedente cambia il livello: non c'è la risposta breve da manuale, c'è il ragionamento che un engineer con esperienza farebbe davanti a un intervistatore che scava. Ogni argomento è diviso in cinque blocchi: una risposta principale da tre-cinque minuti, i follow-up probabili con risposte distese, una prova di padronanza fatta di dettagli concreti, le cose da non dire e una formula onesta per quando non so.

**Come sono trattati i fatti.** Tutto ciò che racconto viene dalle fonti del Ramo 1: il CV, le schede, il codice dei tre repository. Non invento metriche, nomi di progetti, volumi o risultati. Dove un dato non è noto dico che non è noto; dove parlo di ordini di grandezza lo dichiaro; dove una cosa l'ha fatta il team, il cliente o l'architetto lo dico. La distinzione tra ciò che ho fatto io e ciò che hanno fatto gli altri non è una formalità: è la parte della risposta che regge quando l'intervistatore chiede "e tu, esattamente, cosa hai fatto?".

**Come sono trattati i confini.** L'infrastruttura CDK, la pipeline CI/CD, IAM e il Docker degli ambienti erano del team di piattaforma del cliente. L'architettura iniziale del repricing l'ha progettata un collega senior. L'architettura della migrazione SAS, comprese Step Functions e il database di stato, era dell'architetto. Il presentation layer dello store orchestrator l'ho fatto a metà con uno sviluppatore del cliente. Il modello dati SAS non l'ho progettato. Questi confini tornano in ogni argomento, perché sono la spina dorsale della credibilità.

**Come è scritto.** Parlo come Simone D'Angelo, software engineer backend Python in JAKALA. Uso il "dipende" quando dipende davvero, il "a questa scala" quando la scala è il criterio, il "non l'ho misurato" quando non ho un numero. Non elenco classi e file: l'intervistatore non vede il codice. Quando un dettaglio tecnico serve ad ancorare un fatto, lo cito in una riga e torno al ragionamento.

**Una nota sul tono.** Una risposta senior non è una risposta più lunga: è una risposta che dichiara il problema, il vincolo, la scelta, il costo della scelta, il failure mode e cosa cambierebbe se il contesto cambiasse. Se una frase non aggiunge nessuno di questi elementi, nel colloquio si può tagliare. In questo documento la tengo perché serve a costruire il ragionamento, ma a voce va asciugata.

**Nota di revisione.** I fatti sul codice sono stati ricontrollati sui tre repository. Le domande comportamentali — perché cambio ruolo, un errore, un conflitto, il punto debole, le priorità, le domande per l'intervistatore — sono state spostate in `08-risposte-comportamentali.md`, che richiama questo documento dove serve. Per ogni argomento c'è una versione da 60 secondi: è quella che si porta a voce, il resto serve a reggere i follow-up.

## Indice

- 1.1 — JAKALA: racconto, ruolo, behavioral, uso dell'AI
- 1.2 — Repricing: racconto e pattern reali
- 1.3 — Cash flow: esecuzione e job asincroni
- 1.4 — Cash flow dati: S3, Parquet, Athena
- 1.5 — DynamoDB e SQL vs NoSQL
- 1.6 — Store orchestrator: racconto e MVP
- 1.7 — Migrazione SAS → AWS: racconto onesto, ETL, Glue
- 1.8 — Apache Spark: architettura, partizioni, join, skew
- 1.9 — Repricing: estendibilità, limiti, flussi post-run
- 1.10 — Store orchestrator: DuckDB, cache da S3, Streamlit
- 1.11 — Step Functions e qualità/test delle pipeline dati
- 1.12 — Data modeling e livelli di elaborazione
- 1.13 — IaC e CI/CD del cliente
- 1.14 — Validazione input con Pandera
- Come usare questa versione

---

## 1.1 — JAKALA: racconto, ruolo, behavioral, uso dell'AI

### Risposta principale (3–5 minuti)

Parto dal vincolo, non dal curriculum. In JAKALA ho lavorato come software engineer backend Python su un cliente del settore energy, su **tre applicazioni web che girano su AWS**. Il punto di partenza non erano tre progetti nuovi da zero: erano strumenti legacy che il cliente usava davvero:
* Uno era un'applicazione desktop con un database locale sulla macchina di ogni utente;
* l'altro era un notebook con qualche script, che un utente finale non poteva usare.

Il problema non era il carico: gli utenti sono pochi. Era la **distribuzione e l'affidabilità del dato**: un eseguibile da consegnare a ogni rilascio, database che divergevano da una macchina all'altra, nessuno storico condiviso e input in Excel e CSV facili da corrompere.

Il vincolo vero non era "riscrivere in Python". Era mantenere il comportamento di un calcolo di business mentre cambiava il contenitore: la logica di dominio esisteva già e non spettava a me reinventarla. Nel desktop c'erano un eseguibile da distribuire, un database locale che divergeva e risultati che vivevano sul computer di chi aveva lanciato il calcolo. Nel notebook c'era un flusso utilizzabile solo da chi conosceva il codice. Un porting riga per riga avrebbe portato su web gli stessi problemi, con in più la rete e la concorrenza.

La risposta architetturale — che non ho progettato io, ma che ho contribuito a realizzare — è stata:
* dati e risultati centralizzati su S3
* calcoli onerosi spostati fuori dalla macchina dell'utente e fuori dalla richiesta HTTP
* stato delle esecuzioni tracciato
* un'unica applicazione web multiutente al posto di eseguibili distribuiti.

Il **valore non è "cloud"**: è che **due persone che lavorano sulla stessa wave vedono gli stessi dati, e che un calcolo lungo non blocca la loro giornata**. Questo è il tipo di motivazione che, in un colloquio, vale più dell'elenco dei servizi AWS.

Nel repricing il problema era diverso: un algoritmo di ottimizzazione dei prezzi di listino, con una pipeline di step che trasformano uno snapshot di dati. L'architettura della pipeline l'ha progettata un collega senior; io l'ho estesa. "Estesa" vuol dire che ho letto il codice prima di toccarlo, ho aggiunto e modificato step, ho lavorato sulle condizioni e sulle strategie di caricamento e salvataggio, e ho capito dove il contratto dell'architettura veniva rispettato e dove no. Non rivendico il disegno iniziale: rivendico di saperlo spiegare e di averci lavorato dentro senza romperlo.

Nello store orchestrator ho lavorato al **presentation layer** a metà con uno sviluppatore del cliente. Lì il vincolo era Streamlit: lo script viene rieseguito a ogni interazione, quindi lo stato e le query costose vanno gestiti in modo esplicito. Il risultato è una separazione in View, ViewManager e DataRepository, che chiamiamo "ispirata a MVP" senza pretendere che sia MVP puro. Anche qui: non è un'architettura che ho deciso da solo, è un lavoro fatto insieme, e so spiegare perché quella separazione esiste e dove fa acqua.

I confini li dico subito, perché sono parte della risposta. L'infrastruttura CDK, la pipeline CI/CD, IAM e il Docker degli ambienti erano del team di piattaforma del cliente. Nella migrazione SAS l'architettura — Step Functions, database di stato, scelte di piattaforma — era dell'architetto; io ero junior e ho tradotto flussi in job Glue/PySpark. Il modello dati SAS non l'ho progettato. Non lo dico per modestia: lo dico perché se l'intervistatore scava e scopre che ho gonfiato il ruolo, perde fiducia su tutto il resto, comprese le cose che ho fatto davvero.

Sull'AI devo essere preciso, perché è un tema dove è facile raccontarsi una storia. La uso come acceleratore: esplorare soluzioni, generare test, scrivere documentazione, fare refactoring guidato. Su Pynance e su Beaver i test sono AI-generated; il frontend di Pynance è vibecodato; la configurazione di Caddy l'ho fatta con l'AI; gli ADR, la wiki e il journal li ho scritti con l'AI e li uso come materiale di studio. Il codice lo leggo, lo eseguo e lo verifico. Il rischio che vedo non è "l'AI scrive codice sbagliato", perché quello si vede: è "l'AI scrive un test che sembra giusto e testa l'implementazione invece del comportamento", e allora hai una copertura che dà falsa sicurezza. Per questo i test generati li considero materiale da studiare e da riscrivere, non un mio risultato: riscriverne alcuni a mano è il passo che ho davanti, e lo dico come tale.

Quello che porto è un mix abbastanza concreto: Python in produzione, integrazione AWS via SDK, lettura di codice altrui, capacità di lavorare con un cliente senza nascondere i limiti, e l'abitudine a distinguere ciò che so da ciò che sto imparando. Quello che cerco è un team che discuta le scelte e non solo i task. Se devo riassumere in una frase: so già lavorare su sistemi reali, e voglio imparare a progettarli meglio, con qualcuno che mi corregge.

### Versione da 60 secondi

Sono un software engineer backend Python in JAKALA. Ho lavorato su tre applicazioni web su AWS per un cliente energy: due ricostruzioni di strumenti legacy, un desktop e un notebook, diventate applicazioni multiutente con dati e storico centralizzati, e una dashboard di analytics commerciale. Il problema non era il carico ma la distribuzione e l'affidabilità del dato. Nel repricing ho esteso un algoritmo di ottimizzazione dei prezzi costruito come pipeline di step; nella dashboard ho lavorato al presentation layer insieme a uno sviluppatore del cliente. L'infrastruttura e la pipeline di rilascio erano del team di piattaforma, e l'architettura del repricing l'ha progettata un collega senior: le conosco e le so spiegare, ma non le rivendico.

### Follow-up

**D. Cosa hai fatto tu, concretamente, e cosa hanno fatto gli altri?**

R. Sulle due ricostruzioni ho lavorato sul codice applicativo: la logica di calcolo portata in Python, l'integrazione con S3 e Athena per salvare e leggere i risultati, l'uso dello stato delle esecuzioni su DynamoDB. Il disegno complessivo dell'architettura web non l'ho deciso io: c'erano un senior e un team, e il meccanismo che lancia i job (payload, stato, dispatcher) fa parte della libreria del cliente. Sul repricing ho esteso la pipeline: step, condizioni, strategie di caricamento e salvataggio. L'architettura iniziale è del collega senior. Sullo store orchestrator ho fatto il presentation layer a metà con lo sviluppatore del cliente. L'infrastruttura, la pipeline, IAM e il Docker degli ambienti erano del team di piattaforma.

La distinzione che faccio non è tra "coding" e "architettura" in astratto: è tra ciò che ho scritto e capito e ciò che ho usato senza averlo progettato. Per esempio, so spiegare perché il task ECS ha 8 vCPU e 40 GB e perché la Lambda ha un timeout di 15 minuti, ma non ho deciso io quelle dimensioni: sono scritte nel progetto e nell'infrastruttura. So spiegare perché il job è asincrono e come viene tracciato lo stato perché ho letto e usato quel codice, non perché l'abbia progettato. So spiegare perché CDK fa synth e deploy, ma non ho scritto lo stack.

Se l'intervistatore vuole verificare, il modo è chiedermi il "perché" delle scelte e il "cosa sarebbe successo se". Se rispondo solo sul "cosa", posso averlo letto; se rispondo sul "perché" e sui failure mode, allora ci ho lavorato. È una distinzione che applico anche a me stesso quando preparo un colloquio: non voglio imparare a memoria un elenco di cose che non ho fatto.

**D. Come usi l'AI, e come verifichi quello che produce?**

R. La uso in modo dichiarato e in parti diverse del lavoro. Sul codice: esplorazione di soluzioni, bozze di refactoring, generazione di test. Sulla documentazione: ADR, wiki, journal. Sull'infrastruttura personale: la configurazione di Caddy. Sul frontend di Pynance: è vibecodato, cioè costruito in modo rapido con l'AI, e lo dico senza girarci intorno. La regola che mi sono dato è: l'AI può scrivere la prima versione, ma la responsabilità di ciò che resta è mia.

La verifica dipende dal tipo di output. Sul codice: leggo, eseguo, provo a romperlo con un caso limite, guardo se le scelte sono coerenti con il resto del repository. Sui test il rischio principale è che testino l'implementazione invece del comportamento: un test generato che mocka tutto e verifica che una funzione ne chiami un'altra è rumore, un test che prende un input, esegue la trasformazione e verifica l'output è utile. I test generati li uso come materiale di studio e come base da riscrivere.

Sulla documentazione: gli ADR scritti con l'AI vanno riletti, perché tendono a essere generici e a non dire cosa è stato scartato. Un ADR senza alternative e senza conseguenze è un riassunto, non una decisione; per me sono anche una fonte di studio. Su Caddy sono onesto: l'ho configurato con l'AI e non l'ho ancora studiato a fondo. So cosa fa un reverse proxy e perché sta davanti all'applicazione, ma non rivendico quella configurazione.

La cosa che non delego è il giudizio su cosa va costruito e perché. L'AI è molto brava a produrre una soluzione plausibile a un problema mal posto; se non hai chiaro il problema, ti convince. In un colloquio questo lo dico apertamente: non vendo l'AI come moltiplicatore di produttività senza dire che sposta il lavoro sulla verifica. Il beneficio è reale, il costo è la revisione, e la revisione richiede competenza.

**Le domande comportamentali** — perché cambio ruolo, un errore, un conflitto, il punto debole, le priorità, perché backend, le domande per l'intervistatore — sono in `08-risposte-comportamentali.md`, con le storie da cui nascono. Qui restano i fatti: contesto, confini e uso dell'AI.

### Prova di padronanza

- So raccontare il contesto partendo dal vincolo — legacy desktop e notebook, dati locali divergenti, eseguibile da distribuire, utenti pochi ma dato poco affidabile — e non dall'elenco delle tecnologie.
- Distinguo in ogni frase ciò che ho fatto io da ciò che hanno fatto il senior, il team di piattaforma, il cliente e l'architetto, e so dire perché questa distinzione è credibilità e non modestia.
- Ho un inventario preciso dell'uso dell'AI: test AI-generated su Pynance e Beaver, frontend di Pynance vibecodato, Caddy configurato con AI, ADR/wiki/journal scritti con AI e usati come materiale di studio. Non lo nascondo e non lo esagero.
- So spiegare il rischio specifico dell'AI sui test — test che verificano l'implementazione invece del comportamento, copertura che dà falsa sicurezza — e cosa faccio per correggerlo.
- So dire "non lo so" senza bloccarmi, e so trasformarlo in un ragionamento su come lo verificherei.
- So dove trovare le risposte comportamentali (`08`): errore, conflitto, punto debole, motivazioni, domande per l'intervistatore.

### Cosa non dire

- "Abbiamo fatto" quando l'intervistatore vuole sapere cosa hai fatto tu: è il modo più rapido per perdere credibilità.
- "Ho progettato io l'architettura del repricing" o "ho fatto io CDK e la pipeline": non è vero, e l'intervistatore può scavare in tre domande.
- Numeri inventati su volumi, durate, utenti o risparmi: meglio "non l'ho misurato" che una cifra che non regge.
- "Me ne vado perché il progetto era gestito male": anche se fosse vero, ti fa sembrare una persona che scarica le colpe.
- "L'AI ha scritto tutto" oppure "l'AI non la uso": entrambe le versioni sono false e si sentono lontano un miglio.
- Un elenco di tecnologie senza il problema che risolvevano: AWS, Python, Spark, DuckDB non sono una storia.
- "Sono ancora junior, sto imparando" detto come scusa: puoi dire cosa non sai senza sminuire quello che sai.
- Risposte da due minuti e mezzo quando ne bastano trenta secondi: il tempo dell'intervistatore è un vincolo, non un suggerimento.

### Se non lo so

Se mi chiedi un dettaglio che non ricordo — una configurazione, un numero, una scelta fatta prima che arrivassi — preferisco dirlo subito e ragionare su come lo verificherei, invece di costruire una risposta plausibile. Se mi chiedi un giudizio su una decisione che non ho preso, dico che non l'ho presa io e provo a ricostruire il contesto. Se mi chiedi cosa avrei fatto al posto di chi l'ha presa, rispondo, ma lo dichiaro come ipotesi e non come storia.

---

## 1.2 — Repricing: racconto e pattern reali

### Risposta principale (3–5 minuti)

Il **repricing** è un problema di ottimizzazione dei prezzi di listino per le offerte di un operatore energetico. La parte che conta raccontare non è la matematica: è il disegno del software. Il punto di partenza è uno snapshot di dati — contratti, caratteristiche dell'offerta, volumi, marginalità, probabilità di churn, limiti di prezzo — e l'obiettivo è trovare prezzi che rispettino i vincoli e si avvicinino a un target di marginalità. Non è un modello che "indovina" il prezzo: è una ricerca sotto vincoli, fatta di regole e calcoli, con un risultato che va spiegato e verificato.

Il vincolo di design è che il **flusso non è lineare**. Ci sono sequenze, condizioni e cicli: uno step si esegue solo se una certa condizione è vera, un altro si ripete finché l'ottimizzazione non converge o finché un limite non viene raggiunto. **Se metti tutto in un'unica funzione, non puoi cambiare una parte senza toccare il resto**, non puoi testare un passo in isolamento e non puoi spiegare a un altro perché l'ordine è quello. La **pipeline** nasce da qui: **ogni step riceve uno snapshot, lo trasforma e restituisce uno snapshot**; il flusso è composto, non monolitico.

La forma è una pipeline di step, e la pipeline è a sua volta uno step. Questo è il punto che fa la differenza ed è il pattern **Composite**: il codice che esegue gli step non deve sapere se sta eseguendo un passo elementare o una sotto-pipeline. La conseguenza positiva è che puoi annidare e comporre; la conseguenza negativa è che il debug diventa ricorsivo e che l'ordine e i confini delle sotto-pipeline vanno conosciuti. A questa scala il beneficio supera il costo, ma non è gratis: una pipeline annidata è più difficile da seguire di una lista piatta.

Il secondo pattern è il Template Method. Il metodo pubblico `process` definisce lo scheletro dell'esecuzione: logga il nome dello step, delega al metodo interno `_process` e, se lo step lo richiede, libera i dati a fine esecuzione. Non gestisce gli errori: se uno step solleva un'eccezione, la pipeline si interrompe. Le sottoclassi implementano solo il passo specifico, e così tutti gli step condividono lo stesso comportamento comune senza duplicarlo. Il punto da conoscere è che il contratto non è blindato: una sotto-pipeline ridefinisce `process` per misurare marginalità e colonne prima e dopo, e chiama direttamente `_process`, quindi salta il log del nome dello step e il cleanup opzionale. L'intenzione è legittima, ma la forma è fragile: il modo pulito sarebbe un hook previsto dal template, o un decorator, così lo scheletro resta intero.

Il terzo pattern è Strategy, usato per caricamento e salvataggio: le strategie cambiano in base al contesto, locale o S3, e chi le usa non conosce l'implementazione. Poi c'è la costruzione delle sotto-pipeline tramite funzioni factory, così la composizione è centralizzata e il chiamante non deve conoscere l'ordine degli step; sono semplici funzioni di costruzione, non il pattern Factory Method del libro. E c'è un punto che non va raccontato come una virtù: la scelta della strategia non passa da una factory, è un `if` sul `RunMode` ripetuto in cinque funzioni di avvio. È il primo punto in cui estrarrei una funzione che costruisce le strategie. Strategy è il comportamento intercambiabile, factory è la costruzione: non sono sinonimi.

Il runner è il contesto condiviso. Non è un Observer: non ci sono osservatori che si registrano e vengono notificati quando il soggetto cambia stato. Gli step leggono parametri dal runner e, al massimo, scrivono una flag come "ottimizzazione terminata". È un oggetto di esecuzione condiviso, non un pub/sub. La differenza non è accademica: in Observer l'ordine e la reazione dipendono dalla registrazione e dalla notifica; qui l'ordine è determinato dalla composizione della pipeline e lo stato è letto quando serve. Se l'intervistatore vuole la frase secca: Observer è "dimmi quando cambia", qui è "leggo lo stato condiviso quando mi serve".

Il costo di questa comodità è l'accoppiamento. Il runner ha molti parametri, e ogni step può leggere qualunque cosa: il contratto è implicito. All'inizio è comodo perché aggiungere un parametro è veloce; dopo un po' il runner diventa un contenitore di tutto e gli step diventano difficili da leggere e da testare. La soluzione che proporrei non è "togliere tutto", è separare configurazione, stato di esecuzione e dipendenze: un oggetto di configurazione per area, passato solo a chi serve, e un contesto di esecuzione più piccolo. È un refactoring incrementale, non un big rewrite.

C'è un secondo difetto, più piccolo ma istruttivo, e va descritto per quello che è: un difetto latente. Un passo ricorrente registra il suo passo di preparazione anche quando è `None`. Il tipo è `Optional`, quindi il contratto ammette l'assenza, ma il codice la tratta come obbligatoria e andrebbe in errore. Oggi non si manifesta, perché tutte le composizioni esistenti passano una preparazione. Il problema è il contratto: il tipo promette una cosa, il codice ne assume un'altra. La correzione è banale, registrare solo se esiste, ma il test che manca, "preparazione assente", è proprio quello che nessuno scrive finché la composizione non cambia.

Una cosa che conviene dire prima che la chieda l'intervistatore: nel ciclo c'è una stima di churn prodotta da un modello, gestito da `ChurnModelManager`, che gli step chiamano come componente, e il clustering dei prezzi usa KMeans. Non li ho addestrati né li presento come lavoro mio: il mio lavoro è il codice attorno, cioè come gli step li usano e come il flusso si compone.

Il mio ruolo: l'architettura iniziale l'ha progettata un collega senior. Io l'ho estesa, ho aggiunto e modificato step, ho lavorato sulle condizioni e sulle strategie, e ho letto il codice abbastanza da trovare i due difetti. Non ho scritto test automatici su questa pipeline: nei progetti di lavoro non c'era quella cultura, e questo è un limite che dichiaro. So spiegare le scelte e i loro limiti, ma non rivendico il design iniziale. È una distinzione che mi serve anche per capire cosa dovrei imparare: la prossima volta voglio essere io a scegliere la forma, non solo a estenderla.

### Versione da 60 secondi

Il repricing è un algoritmo di ottimizzazione dei prezzi di listino: parte da uno snapshot di contratti, vincoli e marginalità e cerca prezzi che rispettino i limiti e si avvicinino a un target. È costruito come pipeline di step: la pipeline è a sua volta uno step (Composite), ogni step ha uno scheletro comune (Template Method), caricamento e salvataggio sono strategie scelte dal contesto, locale o S3. Il runner è un contesto condiviso, non un Observer: gli step ne leggono i parametri. L'architettura l'ha disegnata un collega senior; io l'ho estesa e ne conosco i limiti: un runner con troppi parametri, la scelta delle strategie ripetuta in cinque funzioni, uno scavalcamento del template in una sotto-pipeline e un difetto latente su un `Optional`.

### Follow-up

**D. Perché non è Observer? Spiegami la differenza con un esempio.**

R. Observer è un pattern in cui un soggetto mantiene una lista di osservatori e, quando cambia stato, li notifica. Gli osservatori si registrano, il soggetto non sa cosa fanno, e c'è un'inversione di controllo: è il soggetto che chiama. Nel repricing non c'è niente di tutto questo. Gli step non si registrano, non c'è una lista di sottoscrittori, non c'è una notifica. Il runner è un oggetto condiviso che contiene parametri e qualche flag; gli step lo ricevono e leggono ciò che serve.

Un esempio concreto: quando un passo scrive "ottimizzazione terminata", non sta notificando nessuno. Il passo successivo, se è un ciclo, legge quella flag e decide se continuare. Non c'è un evento, non c'è un handler, non c'è un ordine di notifica. Se domani aggiungessi un osservatore, dovrei inventarmi un meccanismo che oggi non esiste. Chiamarlo Observer sarebbe un'etichetta messa a posteriori, e l'intervistatore bravo se ne accorge.

La differenza pratica si vede nel testing e nel debug. Con Observer devi testare la registrazione, l'ordine di notifica e il caso di un osservatore che fallisce; qui testi la composizione e lo stato dello snapshot. Con Observer puoi aggiungere un comportamento senza toccare il soggetto; qui aggiungi uno step alla pipeline. Sono due modelli di estensione diversi. Se il flusso avesse bisogno di reagire a eventi esterni in modo disaccoppiato, Observer o un event bus avrebbero senso; per una pipeline di trasformazioni, no.

**D. Perché non una macchina a stati o un workflow engine?**

R. Una macchina a stati sarebbe più adatta se il flusso fosse dominato da transizioni esplicite e stati persistenti, con regole diverse per ogni stato e attese tra un passo e l'altro. Qui il flusso è una sequenza di trasformazioni con condizioni e cicli: la pipeline è più semplice da leggere e da comporre. La macchina a stati aggiungerebbe un livello di indirezione — stati, transizioni, guardie — che a questa scala non compra nulla. È il tipo di astrazione che sembra più "architetturale" e in realtà sposta la complessità senza ridurla.

Un workflow engine avrebbe senso se ci fossero retry, attese, integrazioni esterne, ripresa da un punto, più job distribuiti. Nel repricing il calcolo è locale alla run, l'ordine è noto e il fallimento si gestisce con il log e il rerun. Se il processo diventasse un sistema con molti servizi e attese, allora un orchestratore sarebbe la scelta giusta. La domanda che faccio sempre è: la complessità è nel dominio o nel modo in cui l'ho modellato? Se è nel dominio, l'astrazione aiuta; se è nel mio modello, l'astrazione la nasconde.

C'è anche un costo di adozione: un workflow engine va configurato, monitorato, versionato, e chi ci lavora deve conoscerlo. Per un team piccolo e una pipeline che gira in un processo, è un investimento che non si ripaga. Quando lo cambierei: se il job dovesse durare ore, se dovesse essere ripreso dopo un fallimento senza rifare tutto, se dovesse coordinare passi su macchine diverse. A quel punto la pipeline attuale non basta, e non è un difetto: è un cambio di requisiti.

**D. Perché non una clean architecture o un'architettura esagonale?**

R. Perché sarebbe indirezione senza un beneficio concreto. Clean architecture ed esagonale hanno senso quando il dominio deve vivere senza framework, quando ci sono più adapter da scambiare, quando vuoi testare il core senza database o senza UI. Qui il dominio è un calcolo su DataFrame, c'è un solo tipo di input e output, non ci sono più database da scambiare e il team è piccolo. Aggiungere porte e adapter significherebbe scrivere più codice per fare la stessa cosa, con più punti da capire.

La pipeline dà già i confini che servono: ogni step ha un contratto, le strategie isolano l'I/O, le factory centralizzano la composizione. Non è la stessa cosa di una clean architecture, ma risolve lo stesso problema di fondo — separare il calcolo dall'infrastruttura — con meno cerimonia. Se il progetto dovesse avere più fonti, più output, un'API pubblica e un dominio che cambia indipendentemente dai framework, allora rivaluterei. Non applico un'architettura perché è nel libro: la applico se il costo si ripaga.

Il rischio opposto è giustificare qualsiasi cosa con "è pragmatico". La differenza tra pragmatico e improvvisato è che il pragmatico sa quali confini ha e perché; l'improvvisato no. Qui i confini ci sono e sono spiegabili: step, condizioni, strategie, factory, runner. Il punto debole è il runner, non l'assenza di clean architecture. Se dovessi investire, investirei lì, non in uno strato di porte e adapter che nessuno usa.

**D. Cosa serve per aggiungere uno step? E per aggiungere un parametro?**

R. Per aggiungere uno step: capire se è davvero una responsabilità nuova, definire il contratto — cosa legge dallo snapshot, cosa scrive, quali invarianti rispetta, quali errori può lanciare — implementare il passo, inserirlo nella composizione giusta e verificarlo con un caso piccolo. Se lo step ha bisogno di un parametro, il parametro va dichiarato e documentato, non letto dal runner in modo implicito. Il criterio è: la pipeline deve restare leggibile a chi arriva dopo, e l'ordine degli step deve avere un senso spiegabile.

Per aggiungere un parametro: oggi lo aggiungo al runner e lo leggo dove serve. È veloce, ma aumenta la superficie del runner e rende il contratto implicito. La direzione giusta è un oggetto di configurazione per area — prezzo, clustering, output — passato solo agli step che ne hanno bisogno, con un default esplicito e una validazione. La differenza tra un parametro e uno step è questa: il parametro configura la stessa responsabilità, lo step aggiunge una responsabilità nuova. Se ogni eccezione diventa un parametro, il modello si dissolve.

Il rischio pratico di aggiungere un parametro è la regressione silenziosa: un default sbagliato cambia il comportamento di tutte le run senza che nessuno se ne accorga. Per questo, se il parametro è nuovo, lo tratterei come una modifica al contratto: default esplicito, validazione, un test che copra il caso in cui manca, e una nota su cosa cambia nell'output. Non è burocrazia: è il modo per non scoprire l'effetto tre mesi dopo.

**D. Il runner è un God Object? Come lo refattorizzeresti senza rompere tutto?**

R. È un contenitore con una superficie ampia: molti parametri, flag e stato di esecuzione. Non è un God Object nel senso che fa tutto — non calcola, non salva, non decide il flusso — ma è un punto di accoppiamento: ogni step può leggere qualsiasi cosa e il contratto è implicito. La diagnosi è questa, non "il file è lungo". Il problema non è la quantità di parametri in sé, è che gli step dipendono da lui per cose che non sempre riguardano il loro compito.

Il refactoring che farei è incrementale. Primo: test di caratterizzazione su un caso reale, per avere un output di riferimento. Secondo: estrarre un gruppo di parametri coerente — per esempio la configurazione del prezzo — in un oggetto esplicito, mantenendo il runner come facciata per compatibilità. Terzo: migrare gli step che usano quel gruppo, uno alla volta. Quarto: rimuovere i parametri vecchi dal runner solo quando nessuno li legge. Non mescolerei l'estrazione della configurazione con la correzione del template: sono due cambiamenti diversi e vanno verificati separatamente.

Il costo è il tempo e il rischio di toccare codice che funziona. Il beneficio è che gli step diventano testabili con un oggetto piccolo, e che il contratto diventa leggibile. Se non avessi test, il primo passo sarebbe salvare un output di riferimento e confrontarlo, non "fare il refactor e sperare". Il refactoring senza rete è solo movimento.

**D. Come testeresti uno step e la composizione?**

R. Uno step è una funzione di trasformazione: input piccolo e deterministico, output atteso, asserzioni sui valori calcolati, sulle flag e sulle invarianti. Non mockerei tutto: se lo step legge tre parametri, gli passo tre parametri; se ha una dipendenza esterna, la inietto. I casi che coprirei sono: il caso felice, il caso limite — zero, nullo, valore massimo — e il caso in cui un input opzionale manca. Quest'ultimo è proprio il caso che il passo ricorrente non gestisce, ed è la dimostrazione che i casi "noiosi" sono quelli che contano.

Per la composizione: una pipeline con due o tre step, una condizione vera e una falsa, un ciclo che termina e uno che non parte, una sotto-pipeline annidata. Verifico l'ordine, lo stato finale e che il logging e il cleanup siano stati eseguiti. Il test del template è importante: se una sotto-pipeline scavalca il metodo pubblico, il test deve accorgersene. Come? Con uno step finto che registra le chiamate, o verificando che il cleanup sia avvenuto anche in caso di errore. Non è un test complicato, è un test che nessuno aveva scritto.

Per la pipeline completa, un test di regressione su un caso reale: stesso input, stesso output atteso. Non serve un dataset enorme; serve un caso che eserciti le condizioni e i cicli. Se l'output cambia, il test fallisce e qualcuno guarda. È il tipo di rete di sicurezza che permette di fare refactoring senza paura. Oggi non c'è, e questo è il primo investimento che farei.

**D. Cosa succede se uno step fallisce a metà? E se l'utente rilancia?**

R. Il fallimento di uno step interrompe la pipeline e solleva l'eccezione fino al gestore della run. Quel gestore fa una cosa precisa: porta lo stato della run a `ERROR`, registra il tempo trascorso e salva il messaggio dell'eccezione nel record della simulazione su DynamoDB. Non c'è un retry automatico: se l'utente vuole riprovare, lo decide lui. La domanda da farsi è cosa resta scritto: gli snapshot intermedi già salvati su S3 durante la run. Qui la progettazione aiuta: ogni run ha un identificativo proprio, composto da data e nome, quindi un nuovo tentativo scrive in una cartella nuova e non sovrascrive né duplica nulla.

Anche i flussi successivi alla run — nuovo binning, modifica manuale, merge dei cluster, annullamento dell'ultimo step — non lavorano sulla run originale: clonano la cartella in un nuovo identificativo e ripartono da lì. È un modello vicino all'immutabilità: costa storage, ma il rerun non è una scommessa. Il punto debole sta altrove: in produzione il lavoro gira in un thread del processo del frontend, e se il processo muore la run resta nello stato `RUNNING` senza che nessuno la chiuda. Servirebbe un controllo sulle run ferme da troppo tempo.

Sul lato utente, il rerun deve essere una scelta consapevole. Nel codice non ho trovato un controllo che impedisca di lanciare due run con la stessa configurazione, perché il sistema le considera indipendenti. A questa scala va bene; se il calcolo avesse un costo rilevante metterei un controllo prima del lancio.

**D. Come scalerebbe con dieci volte i dati o più wave in parallelo?**

R. Prima misurerei dove va il tempo: lettura, calcolo, memoria, scrittura. Non direi "Spark" per riflesso. Se il collo di bottiglia è il singolo passo CPU-bound, un processo o un job esterno può bastare; se è la memoria, il problema è il modo in cui carico i dati; se è l'I/O, il formato e il layout contano più della potenza di calcolo. La pipeline attuale è pensata per girare in un processo, e questo è un vincolo da conoscere prima di promettere scalabilità.

Con più wave in parallelo, il problema diventa la concorrenza: ogni run ha il suo spazio di output? Condivide cache o file? Il runner è per-run o condiviso? Se le run sono indipendenti, si possono eseguire su task separati; se condividono stato, il parallelismo introduce race condition. A questa scala, la prima cosa è separare lo stato per run e rendere l'output indirizzabile per identificativo. Poi si misura se il collo di bottiglia è la CPU, la memoria o l'accesso allo storage.

Se i dati superassero la macchina, valuterei un motore distribuito, ma con un criterio: Spark conviene quando il volume e la fault tolerance distribuita valgono l'overhead. Per una pipeline di regole su uno snapshot, spesso è più efficace partizionare il lavoro per wave e usare processi. Non escludo Spark, ma non lo scelgo senza aver visto il profilo. La risposta onesta è: dipende dal collo di bottiglia, e oggi non l'ho misurato.

**D. Cosa hai fatto tu, e cosa non hai fatto?**

R. Ho esteso la pipeline: aggiunto e modificato step, lavorato su condizioni e strategie, letto il codice abbastanza da trovare due difetti di contratto. Ho lavorato dentro un'architettura progettata da un collega senior, e so spiegare perché è fatta così. Non ho progettato l'architettura iniziale, non ho scritto test automatici e non ho gestito l'infrastruttura. Non ho deciso io i pattern: li ho riconosciuti, usati e in un caso violati dal codice che ho trovato.

Questa distinzione mi serve anche per rispondere alla domanda "cosa faresti diversamente?". Se non ho progettato, posso comunque avere un giudizio: il runner è troppo permissivo, le `run_*` duplicano setup, i due difetti sono violazioni di contratto, i test mancano. Sono giudizi fondati sul codice che ho letto, non sulla mia autorità di architetto. È un ruolo diverso, e in un colloquio è importante non confonderli.

### Prova di padronanza

- So spiegare il problema di business prima dei pattern: ricerca sotto vincoli, target di marginalità, flusso con condizioni e cicli, non "un modello che indovina il prezzo".
- So distinguere Composite da una semplice lista di step: la pipeline è uno step, quindi il chiamante non sa se sta eseguendo un passo o una composizione; il costo è il debug ricorsivo.
- So spiegare il Template Method con la sua imperfezione: `process` logga, delega a `_process` e fa il cleanup opzionale, senza gestire errori; una sotto-pipeline lo scavalca di proposito per misurare prima e dopo, e il modo pulito sarebbe un hook.
- So spiegare il secondo difetto per quello che è, latente: un passo ricorrente registra una preparazione `Optional` anche quando è `None`, ma oggi tutte le composizioni la forniscono.
- So dire perché non è Observer con un esempio: la flag "ottimizzazione terminata" viene letta, non notificata; non c'è registrazione, non c'è handler, non c'è ordine di notifica.
- So distinguere Strategy da factory: Strategy è il comportamento intercambiabile di caricamento e salvataggio, le factory costruiscono le sotto-pipeline, e la scelta della strategia è un `if` sul `RunMode` ripetuto in cinque funzioni.
- So dire perché non una macchina a stati, un workflow engine o una clean architecture, e in quali condizioni cambierei idea: attese, retry, più servizi, più adapter.
- So descrivere il runner come punto di accoppiamento, non come "file lungo": molti parametri, contratto implicito, step che leggono tutto; la soluzione è separare configurazione, stato e dipendenze.
- So spiegare cosa succede a un rerun: stato persistito con `ERROR` ed eccezione, identificativo nuovo per ogni run, redrive che clonano la cartella; e dove resta il rischio, cioè una run ferma in `RUNNING` se il processo muore.
- So dire cosa non ho fatto — architettura iniziale, test automatici, infrastruttura, modello di churn — senza sminuire ciò che ho fatto: estensione, lettura critica, identificazione dei difetti.

### Cosa non dire

- "È Observer" perché il runner contiene stato: è l'errore più facile e l'intervistatore lo usa per capire se hai capito il pattern o solo il nome.
- "L'architettura l'ho fatta io": non è vero, e basta una domanda sul perché di una scelta per smontarti.
- Elencare Composite, Template Method, Strategy e Factory senza dire quale problema risolve ciascuno e quale costo porta.
- Dire "clean architecture" o "architettura esagonale" per un progetto che non le usa: sembra un'etichetta imparata, non una scelta.
- Proporre un big rewrite o un refactor enorme senza test di caratterizzazione e senza dire il costo.
- Dire "ci sono i test" quando non ci sono: nei progetti di lavoro non c'era una cultura del testing automatico, e va detto.
- Confondere "parametro" e "step": il primo configura, il secondo estende; se ogni eccezione è un parametro, il modello si dissolve.
- Parlare dei flussi post-run come se fossero il core: snapshot, clustering, modifica manuale, merge e tabella finale sono prodotto, non ottimizzazione.

### Se non lo so

Non ho progettato l'architettura iniziale: l'ha fatta un collega senior, e se mi chiedi perché una scelta è stata presa prima che arrivassi, posso ricostruire il contesto ma non attribuirmela. Non ho scritto test automatici su questa pipeline, quindi non posso raccontarti una strategia di testing che è stata realmente applicata: posso dirti cosa farei e perché. Se mi chiedi un dettaglio di implementazione che non ricordo, preferisco dirlo e ragionare sul contratto, invece di inventare una classe o un metodo.

---

## 1.3 — Cash flow: esecuzione e job asincroni

### Risposta principale (3–5 minuti)

Il problema di partenza è un vincolo fisico prima che architetturale: un calcolo di simulazione può durare minuti e usare molta CPU o memoria, mentre una richiesta HTTP deve tornare in tempi brevi. Se il calcolo gira dentro la richiesta, l'utente aspetta, la connessione può scadere, il worker resta occupato e il timeout diventa il comportamento normale. La soluzione non è "ottimizzare il calcolo" — a volte non si può — è separare l'accettazione del comando dall'esecuzione del lavoro. La richiesta avvia il job e restituisce subito un identificativo; il job gira altrove; l'utente controlla lo stato.

Nel cash flow il flusso è questo: l'utente avvia un job dal frontend, che invoca in modo asincrono una Lambda; la Lambda, che fa da dispatcher, costruisce un payload tipizzato in base al tipo di job, lo salva su DynamoDB come record dell'esecuzione e lancia un task ECS Fargate — ECS è l'orchestratore, Fargate il compute — passandogli solo il tipo di job e l'identificativo; il task rilegge il resto dal record, esegue il calcolo, aggiorna lo stato e scrive i risultati su S3 in Parquet, interrogabili con Athena. La Lambda non esegue il calcolo: è un punto di ingresso leggero. I tipi di job sono sei (forecast, consuntivazione, estrazione dell'output, estrazione pivot, estrazione Athena e una test suite) e passano tutti dallo stesso dispatcher. Il task è dimensionato nel codice del dispatcher, 8 vCPU e 40 GB, uguale per tutti: probabilmente sovradimensionato per le estrazioni più brevi, ma non l'ho misurato. La Lambda ha un timeout di 15 minuti, che non usa per il calcolo.

Per essere esplicito sui termini: **ECS è l'orchestratore, Fargate è il compute**. Fargate, in due parole, è il compute gestito: non noleggi l'istanza, dichiari CPU e memoria del task e AWS crea la capacità. La **task definition** è la ricetta — immagine, CPU e memoria, ruoli, rete, log — e il **task** è una sua singola esecuzione. Quello che parte nel cash flow è un task, non un **service**: un service terrebbe un numero desiderato di task sempre vivi. Il cluster e la task definition vivono nell'infrastruttura del team di piattaforma; io ho lavorato sul codice del job e sul salvataggio dei risultati. Il task è one-off: nessun desired count, nessun restart automatico, nessun load balancer davanti.

Perché una Lambda come dispatcher, e non il frontend che chiama direttamente `RunTask`? Prima una precisazione, perché è l'equivoco più comune: un task lanciato con `RunTask` è effimero. Parte, esegue e termina; non resta acceso tra un job e l'altro. Anche `RunTask` è già asincrono: restituisce subito. La Lambda non serve a tenere acceso ECS né a renderlo asincrono. Un processo sempre acceso lo avrei solo se mettessi un worker come servizio ECS che aspetta i job, tipicamente dietro una coda.

La ragione che vedo davvero nel progetto è di permessi e di superficie d'attacco. Il ruolo del frontend non ha `ecs:RunTask` né `iam:PassRole` verso i ruoli di simulazione: li hanno il ruolo di simulazione e quello di supporto. `RunTask` accetta override del comando del container, quindi chi lo chiama può eseguire comandi arbitrari con il ruolo del task. Con la Lambda in mezzo, il frontend può passare soltanto tipo di job, identificativo e parametri, perché il comando e la dimensione del task li costruisce il dispatcher. Il controllo non sparisce, si sposta in un punto stretto e controllabile. Altre ragioni plausibili, che non posso confermare perché il disegno non è mio, sono la coerenza con lo schema della piattaforma e la possibilità di avere altri canali di avvio oltre al frontend.

Detto questo, il dispatcher com'è fatto oggi fa poco: costruisce il payload, scrive lo stato, lancia il task. Non valida, non autorizza, non ritenta (`retry_attempts=0`) e non controlla l'esito di `run_task`. Il ruolo, poi, è lo stesso per Lambda e task, e anche quello del frontend ha policy gestite ampie: il minimo privilegio è applicato solo in parte. Se partissi da zero valuterei Step Functions con l'integrazione `ecs:runTask.sync`, che dà stato, retry e notifica per costruzione, oppure una coda con un servizio worker se il carico fosse continuo.

Perché ECS Fargate e non solo Lambda? Perché Lambda ha limiti di durata e memoria e non è pensata per un carico CPU-heavy lungo. Fargate dà più risorse e un ambiente isolato, anche se non controlli l'host: il task ha una quantità di CPU e memoria che scegli, un'immagine sua e un ciclo di vita indipendente dalla richiesta. Il costo è più infrastruttura: task definition, immagine, ruoli, rete, log. A questa scala il compromesso ha senso perché il job è lungo e pesante; per un job breve, Lambda sarebbe più semplice.

Nel repricing il contesto è diverso, e la scelta è diversa. In produzione la funzione asincrona usa un thread per le operazioni lunghe: la richiesta avvia il lavoro e il thread lo esegue nel processo. Il frontend è su EC2 con autoscaling, non su ECS. A voce dico "container su AWS" per non impantanarmi nel dettaglio, ma se l'intervistatore chiede so dire che il frontend è su EC2 e che l'async è un thread. Il thread è la soluzione più semplice: non aggiungi un dispatcher, non aggiungi un task, non aggiungi un'immagine. Ma vivi nei limiti del processo.

I limiti del thread sono concreti. Il GIL fa sì che, per codice Python CPU-bound, più thread non siano paralleli: si alternano, non scalano. La memoria è quella del processo: un job che ne consuma tanta può affamare gli altri. Se il processo muore, il job muore con lui. La gestione degli errori è nelle mani del codice che ha lanciato il thread: se nessuno lo aspetta, un'eccezione può passare inosservata. E la concorrenza è limitata dal numero di thread e dalle risorse della macchina. Non è la scelta giusta per un job enorme o per molti job concorrenti.

"Async" non significa "più veloce". Significa che chi chiama non aspetta il risultato. Il lavoro totale è lo stesso, a volte un po' di più per il coordinamento. Quello che cambia è il modello di attesa e di responsabilità: l'utente riceve un acknowledgement e poi interroga lo stato, oppure riceve una notifica. Il sistema deve avere un posto dove lo stato vive e un modo per riconciliarlo. Se l'async è solo "lancio un thread e spero", non è async: è un fire-and-forget che prima o poi perde un pezzo.

Concorrenza e parallelismo sono due cose diverse e vanno tenute separate. Concorrenza è gestire più attività nello stesso periodo, anche alternandole su un solo core; parallelismo è eseguirle nello stesso istante su più core o macchine. Un thread su Python è concorrente ma, per il GIL, non è parallelo per il codice CPU-bound. Un processo o un task ECS possono essere paralleli. Questa distinzione spiega perché il thread va bene per I/O e per un job leggero, ma non per un calcolo pesante e concorrente.

Il failure mode è la parte che distingue una risposta matura, e nel cash flow ha una forma precisa. Il dispatcher scrive il record su DynamoDB e solo dopo lancia il task. Quindi: se `run_task` fallisce senza sollevare un'eccezione (può restituire una lista di `failures`, per esempio per capacità), il record resta nello stato iniziale e nessuno lo sa. Se il container viene ucciso per memoria, il blocco di gestione degli errori del job non gira e lo stato resta `RUNNING` per sempre, senza un heartbeat né un timeout. Se invece il job solleva un'eccezione Python, lo stato diventa `ERROR` con il messaggio, ma l'SNS non c'entra: nel progetto è la destinazione di fallimento della Lambda, quindi copre il dispatcher e non il task. I rimedi sono semplici da dire: controllare `failures`, un controllo periodico che marca come fallite le esecuzioni ferme oltre una soglia, e una notifica sugli eventi di stop dei task ECS. Senza queste tre cose, un job asincrono è una scatola nera.

Sulla scalabilità non ho numeri da vendere. So che il modello è: coda o dispatcher, limite di concorrenza, worker che scalano, idempotenza, dead letter, monitoraggio. Con Lambda + ECS il dispatcher disaccoppia l'ingresso dall'esecuzione e il task scala; con i thread il processo è il limite. Prima di scegliere misurerei: quanti job, quanto durano, quanta CPU e memoria, qual è il collo di bottiglia. Non direi "Fargate scala" senza sapere se il collo di bottiglia è il task, il database o l'accesso a S3. La soluzione dipende dal carico, non dalla moda.

### Versione da 60 secondi

Nel cash flow un calcolo di minuti non può stare dentro una richiesta HTTP. Il frontend invoca in modo asincrono una Lambda, che costruisce un payload, lo salva su DynamoDB e lancia con `RunTask` un task ECS Fargate da 8 vCPU e 40 GB; il task rilegge il record, calcola, aggiorna lo stato e scrive i risultati su S3. ECS è l'orchestratore, Fargate è il compute; il task è effimero, non un processo sempre acceso. La Lambda è un dispatcher stretto che evita di dare al frontend il permesso di lanciare task con comandi arbitrari. I punti deboli: nessun retry, esito di `run_task` non controllato, stato che può restare `RUNNING` se il container viene ucciso. Nel repricing il lavoro lungo gira invece in un thread: più semplice, ma limitato al processo.

### Follow-up

**D. Perché una Lambda che lancia ECS e non solo Lambda?**

R. Solo Lambda sarebbe più semplice, ma il carico non è adatto. Lambda ha un limite di 15 minuti e una memoria limitata, e non è pensata per un calcolo CPU-heavy che dura a lungo: paghi il tempo di esecuzione e hai meno controllo sull'ambiente. Se il job sfora il timeout, viene interrotto a metà: è il tipo di fallimento che non vuoi scoprire in produzione. Il task ECS dà più risorse, un'immagine controllata e un isolamento migliore. La Lambda resta il punto di ingresso: veloce, event-driven, con un ruolo limitato.

C'è anche una ragione di responsabilità: il dispatcher deve fare poche cose e farle bene — validare, registrare, lanciare. Il worker deve fare il calcolo. Se metti tutto in Lambda, mescoli le due responsabilità e hai un punto di ingresso che dura quanto il job. Se metti il worker in un servizio ECS sempre acceso, paghi un processo in attesa anche quando gli eventi non arrivano. La separazione non è un dogma: è la conseguenza del fatto che i due carichi hanno profili diversi.

Il costo è la complessità: due runtime, due ruoli IAM, due immagini, un protocollo tra i due. Il beneficio è che puoi scalare il dispatcher e il worker in modo indipendente e che un fallimento del worker non blocca l'ingresso. A questa scala il beneficio vale il costo; per un job di trenta secondi, non lo varrebbe.

**D. Perché Fargate e non EC2?**

R. Perché il job è sporadico e pesante, e con EC2 si sarebbe dovuta tenere capacità pronta e gestire le istanze anche quando non servivano. Con Fargate la CPU e la memoria si dichiarano per task, AWS crea la capacità e la spegne alla fine: nessun host idle, nessun patching, nessun ASG da dimensionare. Il contro è che non si vede l'host: non si possono usare GPU, custom AMI o container privileged, ci sono meno margini di tuning e si paga il task per quello che dura. Se il carico fosse continuo e alto, EC2 o Managed Instances potrebbero costare meno e dare più controllo; se servisse una GPU, Fargate non sarebbe un'opzione. La scelta nel progetto non è mia: l'infrastruttura è del team di piattaforma. Io posso spiegare il criterio, non attribuirmelo.

**D. Perché non solo ECS, senza Lambda? Il frontend potrebbe lanciare direttamente il task.**

R. La domanda confonde due cose, e conviene separarle subito: un servizio ECS e un task ECS. Un servizio tiene sempre acceso un certo numero di task, quindi è un processo sempre acceso. Un task lanciato con `RunTask` è effimero: parte, esegue e termina. Nel flusso attuale il task di simulazione è effimero, e lo sarebbe anche se togliessi la Lambda e il frontend chiamasse `RunTask`. Il confronto corretto non è "Lambda contro ECS" ma tra tre architetture: frontend che lancia il task, dispatcher effimero che lancia il task, worker sempre acceso che consuma una coda.

Il frontend che lancia direttamente il task avrebbe senso per uno strumento interno, con pochi utenti fidati e un'orchestrazione banale: risparmi un pezzo e paghi in permessi, perché il frontend riceve `ecs:RunTask` e `iam:PassRole` e potrebbe passare override del comando. Il dispatcher effimero restringe quella superficie: il comando e la dimensione del task li costruisce lui. Il worker sempre acceso su coda ha senso quando il carico è alto e continuo, perché elimina l'avvio del task e dà controllo sulla concorrenza, ma costa idle, scaling, gestione della coda e dead letter. Per job sporadici e lunghi, un `RunTask` per job è più semplice.

Nel nostro caso il task one-off vince perché il job è sporadico: nessun idle, nessun worker in attesa, nessuna coda da gestire. Il contro è che non hai restart automatico né controllo della concorrenza: se il task muore, lo stato va riconciliato; se arrivano cento job, cento `run_task` sono una strategia fragile. Un service worker sarebbe più robusto per un carico continuo, ma a questa frequenza aggiungerebbe infrastruttura senza un beneficio misurato.

La lezione è che la Lambda non ha un valore magico: è un adattatore di control plane. Vale se fa qualcosa di utile in quel punto, cioè validare, autorizzare, registrare, costruire il comando. Nel progetto ne fa la parte minima. Quindi la risposta onesta all'intervistatore è: i motivi che vedo nel codice sono i permessi e il comando controllato; il disegno è del team di piattaforma e non l'ho scelto io; oggi lo migliorerei con retry, controllo dell'esito di `run_task` e notifiche sui task.

**D. Perché nel repricing un thread e non ECS?**

R. Perché il contesto non giustificava l'infrastruttura. Il repricing aveva già un'applicazione con un frontend su EC2 con autoscaling e una funzione asincrona che lanciava il lavoro in un thread. Qui EC2 è la macchina che ospita il frontend, non un compute ECS: non c'è un cluster ECS, non c'è una task definition, non c'è un service. L'async è un thread nel processo. Aggiungere ECS avrebbe significato un dispatcher, un'immagine, una task definition, ruoli, rete e una pipeline in più. Per il carico e la frequenza di quel progetto, il thread era sufficiente. La scelta più semplice che risolve il problema è spesso quella giusta, soprattutto quando il team è piccolo.

Il limite è che il thread vive nel processo: se il processo muore, il job muore; il GIL limita il CPU-bound; la memoria è condivisa; la concorrenza è quella della macchina. Non è una soluzione per un job enorme o per molti job simultanei. Se il carico crescesse, la prima cosa che farei è misurare, poi valuterei una coda e un worker esterno. Non direi "migriamo a ECS" senza sapere se il collo di bottiglia è la CPU, la memoria o il numero di richieste.

C'è anche un tema di osservabilità: un thread è più difficile da seguire di un task, perché non ha un'identità separata e i log si mescolano. Se il job fallisce, devi accorgertene dal codice che lo ha lanciato o da un log. Nel cash flow il task ha un'identità, uno stato su DynamoDB e un log suo; nel repricing anche la run ha uno stato persistito, ma il thread non ha un'identità separata e, se il processo muore, nessuno chiude quello stato. È un debito che riconosco: la soluzione era adatta al contesto, ma non è quella che sceglierei per un sistema che deve durare e crescere.

**D. Cosa succede se qualcosa va storto tra il dispatcher e il task?**

R. Dipende da dove si interrompe, ed è utile ragionare sull'ordine reale delle operazioni. Il dispatcher scrive prima il record su DynamoDB e poi chiama `run_task`. Se muore tra le due, il record esiste ma il task non è mai partito: l'esecuzione resta nello stato iniziale. Se `run_task` restituisce un fallimento senza eccezione, accade la stessa cosa, e il codice non lo controlla. Se invece il task parte e il dispatcher muore dopo, non cambia nulla: il task è indipendente dalla Lambda e porta a termine il suo lavoro. In tutti i casi che contano serve una riconciliazione tra lo stato registrato e i task che esistono davvero.

Il progetto ha già un ingrediente giusto: l'identificativo dell'esecuzione lo genera il frontend prima del dispatch e viene usato sia come chiave del record sia come parametro del task, quindi è una chiave di correlazione. Quello che manca è la parte operativa: un controllo periodico che confronta i record in stato iniziale o `RUNNING` con i task ECS realmente attivi, oppure un evento sugli stop dei task che aggiorna lo stato con il motivo (per esempio memoria esaurita, exit code 137). Con la Lambda configurata senza retry, un secondo task con lo stesso identificativo non nasce per ritentativo automatico; resta il caso dell'utente che rilancia, che richiede un controllo esplicito.

Nel progetto c'è lo stato su DynamoDB e c'è l'SNS come destinazione di fallimento della Lambda, ma non una riconciliazione automatica, e non te ne racconto una che non ho visto. La lezione è che il passaggio tra "scrivo lo stato" e "lancio il task" non è atomico: va progettato, non assunto.

**D. Come gestisci retry e idempotenza?**

R. Con un limite, un backoff e una chiave. Un retry cieco può duplicare l'output: se il job ha già scritto parte dei risultati e viene rieseguito, senza una chiave rischi due set di file o due righe. La chiave di esecuzione — lo stesso identificativo per lo stesso comando — permette di riconoscere il rerun e di sovrascrivere in modo controllato, oppure di non fare nulla se il risultato è già completo. Senza chiave, il retry è un moltiplicatore di errori.

Sul chi retrya: il dispatcher può ritentare la chiamata a `run_task` se questa fallisce, ma nel progetto la Lambda è configurata con zero ritentativi, quindi oggi non lo fa. Il task può ritentare un'operazione transitoria — una lettura S3, una chiamata di rete — ma non un errore di business. La distinzione è importante: ritentare un dato non valido non lo rende valido, e ritentare un calcolo deterministico che ha già fallito per un bug lo fa fallire di nuovo. Il retry va progettato per categoria di errore, non per "tutto".

Sul lato dati, l'idempotenza si ottiene con un output indirizzabile e una scrittura che non duplica: un prefisso per run, un overwrite controllato, o una scrittura in una posizione temporanea e un commit finale. Nel cash flow i risultati sono partizionati per `run_id`, il che aiuta: la stessa run scrive nella stessa partizione. Ma la partizione da sola non basta se il job scrive due volte lo stesso file con contenuti diversi: serve una regola su cosa è la verità.

**D. Come osservi un job asincrono? Come capisci dove si è bloccato?**

R. Lo stato persistito è la prima cosa: l'utente deve vedere "in coda, in esecuzione, completato, fallito", e tu devi vedere da quanto è in uno stato. Poi i log con un identificativo di correlazione: la stessa run deve comparire nel dispatcher, nel task e nelle chiamate a S3. Poi le metriche: durata, numero di fallimenti, retry, e se possibile il consumo di risorse. Senza queste tre cose, un job lento e un job bloccato sembrano uguali.

Per capire dove si è bloccato, guarderei l'ultimo evento registrato e il timestamp: se è "in esecuzione" da molto più della durata tipica, è bloccato o è lento. Poi guarderei i log del task, il suo stato ECS e le metriche di CPU e memoria. Se il task è vivo ma non fa progressi, il problema può essere un'attesa di rete, un lock, una query lenta o un ciclo. Se il task è morto, il problema è un crash o un OOM. La differenza si vede dai segnali, non dal "è fallito".

L'SNS del progetto è la destinazione di fallimento della Lambda: avvisa che il dispatcher è fallito, non che il task ha avuto un problema. Per il task la fonte di verità è lo stato su DynamoDB e il log del container, che non notificano nessuno. Un alert senza contesto allunga l'incidente: la cosa che vorrei è un evento di stato con il motivo e un link al log, e una dashboard che mostri la coda e la durata. È un'area in cui il progetto ha le basi, ma non la chiamerei osservabilità completa.

**D. Come scaleresti a cento job insieme?**

R. Prima misurerei: quanti job arrivano, quanto durano, quanta CPU e memoria chiedono, qual è il collo di bottiglia. Poi deciderei se il problema è il dispatcher, il numero di task concorrenti, la capacità di Fargate, il database di stato o l'accesso a S3. "Scalare" non è una cosa sola: è rimuovere il collo di bottiglia giusto. Se il collo è il dispatcher, Lambda scala; se è il task, aumenta la concorrenza o la dimensione; se è il database, il modello di accesso va rivisto.

Con Lambda + ECS, una coda tra dispatcher e worker disaccoppia e permette di controllare la concorrenza. Senza coda, cento richieste diventano cento tentativi di `run_task` e potresti sbattere contro le quote o saturare la capacità. La coda dà anche un posto dove mettere i fallimenti — una dead letter — e un modo per ritentare senza perdere richieste. È la prima cosa che aggiungerei se il volume crescesse.

Con i thread, cento job insieme sono cento thread nello stesso processo: il GIL li serializza per il CPU-bound, la memoria si somma e un job pesante può uccidere gli altri. Non è una questione di "più thread": è un limite del modello. Se il repricing dovesse gestire cento job concorrenti, la soluzione sarebbe un worker esterno o una coda, non un pool di thread più grande. Il costo di questa migrazione va confrontato con il beneficio atteso, non deciso a priori.

**D. Come ragioni sul costo di queste scelte?**

R. Il costo ha tre componenti: il costo variabile del servizio, il costo fisso dell'infrastruttura e il costo umano di gestione. Lambda si paga per invocazione e durata; Fargate per vCPU e memoria per ora; un thread "non costa" in fattura ma costa in affidabilità e in tempo di debug. Un job che gira su un thread può sembrare gratis, ma se blocca il processo o perde un errore, il costo emerge altrove. Non è una scelta economica solo sulla fattura cloud.

Per il cash flow, il task da 8 vCPU e 40 GB è una scelta di capacità: paghi per avere risorse sufficienti a un job pesante. Se il job non le usa, stai pagando troppo; se le usa, è il prezzo per non avere un timeout. La domanda è: il job è dimensionato bene? Senza misure non lo so. La cosa che farei è guardare l'utilizzo medio e massimo e provare a ridurre, ma non ho fatto benchmark formali e non invento un risparmio.

Il costo umano è quello che si dimentica: due runtime, due pipeline, due set di log, due modi di fare debug. La semplicità del thread ha un valore reale, finché il carico lo permette. La scelta matura non è "la soluzione più scalabile", è "la soluzione più semplice che regge il carico previsto, con un piano per quando non reggerà più". Questo è il criterio che userei per giustificare l'una o l'altra.

**D. Cosa hai fatto tu e cosa no?**

R. Nel cash flow ho lavorato sul codice dei job e sul salvataggio dei risultati; il meccanismo che li lancia — dispatcher, payload, stato su DynamoDB — l'ho letto e usato, ma fa parte della libreria del cliente. Non ho progettato l'infrastruttura ECS, la rete, IAM o la pipeline CDK: quelle erano del team di piattaforma. Nel repricing ho usato la modalità async esistente, che lancia il lavoro in un thread. So spiegare i concetti e i trade-off, ma non rivendico l'infrastruttura.

Questa distinzione è importante perché il tema "async" è pieno di gente che ha letto tre articoli e nessuno che ha gestito un job bloccato. Io ho lavorato su un flusso reale, con uno stato e un fallimento, ma non ho progettato l'autoscaling né fatto benchmark. Se mi chiedi "quanti job al secondo", non invento: dico che dipende dalla configurazione e che andrebbe misurato.

### Prova di padronanza

- So partire dal vincolo — richiesta HTTP breve, calcolo di minuti — e spiegare perché la separazione tra comando ed esecuzione è la soluzione, non un'ottimizzazione.
- So descrivere il flusso del cash flow nell'ordine reale: invocazione asincrona della Lambda, payload tipizzato, scrittura del record su DynamoDB, `run_task` Fargate, task da 8 vCPU e 40 GB che rilegge il record, stato aggiornato, risultati su S3 in Parquet interrogabili con Athena.
- So spiegare la gerarchia senza confonderla: ECS è l'orchestratore; EC2 e Fargate sono il compute; la task definition è il template; il task è l'esecuzione; il service è il desired state; il cluster è il contesto logico.
- So dire perché nel cash flow ho visto Fargate e non EC2: job sporadico e pesante, nessun host idle, nessun ASG da gestire; il contro è che non controllo l'host, non ho GPU e ho meno margini di tuning.
- So distinguere servizio e task ECS: il task lanciato con `RunTask` è effimero e già asincrono, il servizio è sempre acceso; so dire perché la Lambda non serve a tenere acceso ECS.
- So spiegare perché il dispatcher ha senso nel progetto — il frontend non ha `RunTask` e il comando è costruito nel dispatcher, quindi niente override arbitrari — e dove è debole: nessuna validazione, zero retry, esito di `run_task` non controllato, ruolo condiviso con il task.
- So dire cosa copre l'SNS: i fallimenti della Lambda, non quelli del task; e come chiuderei il buco con eventi di stop dei task o Step Functions.
- So enumerare i failure mode reali: `run_task` che fallisce senza eccezione, task ucciso per memoria con stato fermo su `RUNNING`, utente che rilancia, stato non riconciliato.
- So distinguere il contesto del repricing — thread, frontend su EC2 con autoscaling, non ECS — e spiegare perché la soluzione più semplice era adatta a quel carico.
- So spiegare "async non è più veloce": cambia il modello di attesa e la responsabilità, non il lavoro totale; e distinguere concorrenza e parallelismo e l'effetto del GIL sul codice Python CPU-bound.
- So parlare di idempotenza in modo operativo: chiave di esecuzione, output indirizzabile, overwrite controllato, scrittura temporanea e commit.
- So scalare la risposta in base al collo di bottiglia — dispatcher, task, database, storage — e non in base alla tecnologia preferita.

### Cosa non dire

- "Async è più veloce": è più veloce per l'utente, non per il calcolo; il lavoro totale è lo stesso o maggiore.
- "Lambda ha un limite di 15 minuti" senza dire che nel cash flow la Lambda non esegue il calcolo, lo lancia.
- "Thread e processo sono la stessa cosa" o "i thread scalano": il GIL e la memoria condivisa sono parte della risposta.
- "ECS è meglio di Lambda" o viceversa senza contesto: durata, risorse, frequenza, costo e isolamento sono i criteri.
- Dire che Fargate è un tipo di ECS o che EC2 è un orchestratore: Fargate ed EC2 sono compute, ECS è l'orchestratore.
- Usare task definition, task e service come sinonimi: la prima è il template, il secondo è l'esecuzione, il terzo è il desired state.
- Dimenticare il failure path: cosa succede se il dispatcher muore, se il task fallisce, se l'utente rilancia.
- Dire "c'è il retry" senza dire se è idempotente: un retry non idempotente è un duplicatore di dati.
- Attribuirti l'infrastruttura ECS, l'autoscaling o la pipeline: erano del team di piattaforma.
- Dire "il frontend del repricing è su ECS": è su EC2 con autoscaling; a voce meglio "container su AWS", ma il dettaglio va saputo.

### Se non lo so

Non ho progettato l'infrastruttura ECS, la rete o IAM, e non ho fatto benchmark formali sul cash flow: se mi chiedi quanti job al secondo regge, non ho un numero e non lo invento. So spiegare il flusso, i limiti e i failure mode perché ci ho lavorato, ma non rivendico le scelte di capacità. Se mi chiedi un dettaglio di configurazione che non ricordo, dico che non lo so e ragiono su come lo verificherei.

---

## 1.4 — Cash flow dati: S3, Parquet, Athena

### Risposta principale (3–5 minuti)

La prima distinzione è tra un database transazionale e un data lake. Un database è ottimizzato per scrivere e leggere righe, con vincoli e transazioni; un data lake su S3 è ottimizzato per conservare grandi quantità di dati e leggerli in modo analitico. Nel cash flow i risultati delle simulazioni sono un dato analitico: vengono prodotti una volta, letti per colonne, aggregati e confrontati. Non hanno bisogno di update frequenti o di transazioni multi-riga. Per questo S3 + Parquet è una scelta sensata, non un ripiego.

Parquet è un formato colonnare. In un formato a righe i valori di una riga sono contigui; in Parquet i valori della stessa colonna sono raggruppati. Se devo calcolare la media di una colonna, leggo solo quella colonna invece di tutte. In più, Parquet conserva lo schema, comprime bene e organizza i dati in row group con statistiche: un motore può saltare i blocchi che non contengono i valori cercati. Non è "un CSV più strano": è un formato progettato per query colonnari. Il rovescio è che recuperare una singola riga completa è meno efficiente, perché devi leggere più colonne e più blocchi.

Il partizionamento è la seconda idea chiave. Se i file sono organizzati per `run_id`, una query su una singola esecuzione legge solo quella cartella. Athena usa il partizionamento per fare pruning: non scansiona tutto il data lake, scansiona solo le partizioni che servono. Questo riduce il tempo e, soprattutto, i byte letti — e Athena si paga sui byte scansionati. Il rovescio è che tante esecuzioni producono tante partizioni e tanti file piccoli: ogni file ha un costo di metadati e apertura, e se sono minuscoli il tempo di listing e apertura può dominare.

Athena è un motore SQL serverless su S3. Non devi caricare i dati in un warehouse: definisci tabelle e partizioni, e Athena legge i file. È ottima per esplorazione, report e query batch; non è transazionale, non ha indici tradizionali e non è pensata per update in-place. La sua economicità è reale ma condizionata: una query senza filtro su tutta la tabella può costare molto, una query filtrata per `run_id` e con poche colonne costa poco. Il costo è una scelta di design, non solo di query.

La consistenza è un tema da dire con onestà. S3 è object storage e, dal 2020, offre consistenza forte in lettura dopo la scrittura sul singolo oggetto: se scrivo un oggetto e lo rileggo, vedo l'ultima versione. Quello che non esiste è una transazione tra più file. Se una query legge più file e uno viene sostituito a metà, può vedere una versione mista; non c'è uno snapshot coerente. Per questo i risultati sono trattati come immutabili e si evita di modificare un file in uso. Se servisse consistenza transazionale tra più file, servirebbero un table format come Iceberg o Delta, o un database, non un trucco sul naming.

Le alternative vanno confrontate sul carico. Un DBMS relazionale sarebbe meglio per update frequenti, join complessi e transazioni. Redshift sarebbe meglio per un warehouse gestito con query complesse e performance prevedibili, ma costa e richiede caricamento. DuckDB è ottimo per analisi single-node su Parquet, e infatti lo usiamo nello store orchestrator; ma non è un servizio multiutente con accesso condiviso. Athena è il compromesso serverless per il cash flow: nessun cluster, costo per query, integrazione con S3. Non è la scelta migliore in assoluto: è la scelta giusta per questo carico.

Il layout è parte del design, non un dettaglio operativo. Oltre alla partizione per `run_id`, contano la dimensione dei file, il numero di colonne, il modo in cui si scrive e la retention. Tanti file piccoli peggiorano le query; file troppo grandi riducono il parallelismo e rendono più costoso riscrivere. La compaction periodica è la soluzione tipica, ma va progettata per non riscrivere mentre altri leggono: si scrive in una nuova posizione e si aggiorna il puntatore, oppure si accetta una finestra di manutenzione. Nessuna di queste cose l'ho progettata io.

I failure mode che vedo sono: file corrotto o parziale, schema che cambia senza compatibilità, partizione duplicata o mancante, query che legge tutto perché il filtro non è scritto bene. Ognuno ha un rimedio diverso: validazione a monte, contratto di schema, convenzione di naming, query con filtro esplicito. Il filo comune è che un data lake senza qualità e senza convenzioni diventa una discarica: il formato non ti salva dal disordine.

Il mio ruolo: ho usato il salvataggio in Parquet e le query Athena nel flusso applicativo. Concretamente, i risultati si scrivono con awswrangler in dataset Parquet partizionati per `run_id`, con le tabelle Athena definite come modelli SQLAlchemy, e si interrogano con SQLAlchemy su PyAthena nel workgroup del progetto. Non ho progettato il data lake, le politiche di partizionamento o la governance, e non ho fatto benchmark su byte scansionati o tempi di query. So spiegare perché quella scelta ha senso per un carico analitico e quali sono i suoi limiti. Se l'intervistatore chiede "quanto costa una query", la risposta onesta è: dipende dai byte letti, e senza misure non do un numero.

### Versione da 60 secondi

Nel cash flow i risultati di una simulazione sono dati analitici immutabili: li scrivo in Parquet su S3, partizionati per run, e li interrogo con Athena. Parquet perché è colonnare, tipizzato e compresso, quindi una query su poche colonne legge poco; la partizione per `run_id` fa sì che una query su una simulazione legga solo quella cartella, e Athena si paga sui byte scansionati. Non è un database: niente transazioni o indici, e tante run producono tante partizioni e file piccoli. Con carico transazionale o aggiornamenti frequenti userei PostgreSQL o un table format. Il data lake non l'ho progettato io; so spiegare perché regge per questo carico.

### Follow-up

**D. Perché Parquet e non CSV?**

R. CSV è testuale, non tipizzato, non compresso bene e, per leggere una colonna, devi leggere tutte le righe. Parquet è binario, colonnare, compresso, con schema e statistiche. Per un'analisi che tocca poche colonne su molte righe, la differenza è di ordine di grandezza, non del dieci per cento. In più, con Parquet il tipo è nel file: non devi indovinare se una colonna è una data o una stringa, e non hai il problema delle virgole nei campi.

CSV resta comodo per scambio, debug e import in strumenti che non leggono Parquet. Non lo demonizzo: è un formato di interfaccia, non un formato di analisi. La scelta dipende dal consumatore. Se il file deve essere aperto da un utente su Excel, CSV è più diretto; se deve essere aggregato da Athena o DuckDB, Parquet è meglio. Nel cash flow il consumatore è analitico, quindi Parquet.

C'è anche un costo di scrittura: Parquet è più complesso da generare e non puoi appenderci una riga come faresti con un CSV. Scrivi un file nuovo. Questo è un vincolo che si sposa con l'immutabilità dei risultati: se il risultato è un'istantanea, il file nuovo è la scelta naturale. Se avessi bisogno di append frequenti, il formato e il layout andrebbero ripensati.

**D. Perché non un DBMS relazionale?**

R. Perché il carico è analitico e batch, e i risultati sono immutabili. Un DBMS relazionale brilla quando hai transazioni, update concorrenti, vincoli e query puntuali; qui scrivo un risultato una volta e lo leggo per colonne. Usare un DBMS significherebbe caricare i dati, gestire indici, capacity e backup, e pagare per un carico che non è il suo. Non è che non si possa fare: è che stai comprando caratteristiche che non usi.

C'è anche una questione di costo e di scala. S3 è storage a oggetti economico e quasi illimitato; Athena non ha un cluster da tenere acceso. Un database gestito ha un costo fisso e una capacità da dimensionare. Per un progetto con esecuzioni periodiche e volumi non enormi, il data lake è più semplice da gestire e da far crescere. Il trade-off è che perdi transazioni, vincoli e query puntuali veloci.

Quando cambierei: se i risultati dovessero essere aggiornati frequentemente, se servissero join complessi con dati transazionali, se la latenza di una query puntuale fosse un requisito. In quel caso un PostgreSQL o un warehouse sarebbero più adatti. La scelta non è "S3 è meglio": è "per questo access pattern, S3 è sufficiente e più economico".

**D. Cos'è un formato colonnare, e quando è peggio di un formato a righe?**

R. In un formato colonnare i valori della stessa colonna sono memorizzati insieme, in blocchi. Questo permette di leggere solo le colonne necessarie, comprimere meglio — valori simili stanno vicini — e usare statistiche per saltare blocchi. Per un'analisi che fa `sum`, `avg`, `group by` su poche colonne, è molto efficiente. Per un'applicazione che legge una riga completa per identificativo, invece, devi leggere molte colonne e ricomporre la riga: è meno efficiente di un formato a righe.

Parquet organizza i dati in row group, e dentro ogni row group in column chunk. Le statistiche — min, max, null count — stanno nei metadati e permettono il predicate pushdown: se cerco `run_id = X` e il row group non può contenerlo, lo salto. Questo è il motivo per cui il layout e l'ordine dei dati contano. Un file con una sola colonna e tante righe è diverso da un file con cento colonne e poche righe.

La scelta dipende dal pattern di lettura. Se leggi tutto di una riga, il formato a righe è meglio; se leggi poche colonne di molte righe, il colonnare è meglio. Non è una superiorità assoluta. Nel cash flow il pattern è analitico, quindi colonnare. In un'API transazionale, un formato a righe o un database è più adatto.

**D. Perché partizionare per `run_id`? Cosa succede quando le run diventano tante?**

R. Perché il caso d'uso principale è interrogare una singola simulazione: con la partizione per `run_id`, Athena legge solo quella partizione e riduce i byte scansionati. È un partizionamento che segue l'access pattern, non la data. Se il caso d'uso fosse "tutte le run di un mese", la data sarebbe una scelta più naturale; se fosse "tutte le run di un cliente", il cliente. La partizione giusta è quella che ti fa leggere meno dati per le query che fai davvero.

Il rovescio è che tante run producono tante partizioni e tanti file piccoli. Ogni partizione è una cartella, ogni cartella può contenere file; se ogni run scrive file minuscoli, il listing e l'apertura diventano il costo dominante. In più, il catalogo delle partizioni cresce e va gestito. Non è un problema di Parquet: è un problema di layout. La soluzione è compattare, partizionare in modo più grossolano o cambiare il formato di tabella.

Quando lo cambierei: se le query più frequenti fossero su intervalli di date o su aggregati tra run, aggiungerei una partizione per data o passerei a un layout diverso. Non terrei `run_id` per sempre solo perché era comodo all'inizio. Il partizionamento è una decisione che va rivista quando cambia l'access pattern, e l'access pattern cambia quando cambia il prodotto.

**D. Come si paga Athena e come ridurresti il costo?**

R. Si paga per i byte scansionati dalle query, oltre al costo dello storage S3. Non paghi per il cluster, perché non c'è un cluster tuo. La conseguenza è che una query che legge tutto il data lake costa molto, anche se restituisce poche righe; una query che filtra per partizione e seleziona poche colonne costa poco. Il costo è quindi una proprietà della query e del layout, non solo del volume.

Per ridurlo: filtrare sempre per partizione, selezionare solo le colonne necessarie, evitare `SELECT *`, usare formati colonnari e compressione, compattare i file piccoli, e — se la query è ripetuta — materializzare il risultato con una CTAS o una tabella. La CTAS ha un costo di scrittura ma evita di ripagare la stessa scansione. La cache applicativa è un'altra strada, ma va progettata con una chiave e una scadenza.

Il limite è che non puoi eliminare il costo di una query esplorativa su tutto. Se il prodotto ha bisogno di quel tipo di query, il costo va messo nel conto e magari limitato con quote o con un warehouse dedicato. Non ho fatto misure di costo nel progetto: so spiegare il meccanismo, non darti una cifra.

**D. Cosa succede se aggiorni un file Parquet mentre Athena lo legge?**

R. S3 è object storage: un oggetto è immutabile. Un "aggiornamento" è una scrittura di un nuovo oggetto, spesso con lo stesso nome, che sostituisce il riferimento. Una query che sta leggendo può vedere la versione vecchia o la nuova, a seconda del momento in cui ha aperto il file; non c'è una transazione che dia uno snapshot coerente tra più file. Se la query legge più file e uno viene sostituito a metà, puoi ottenere un risultato misto.

Per questo i risultati sono trattati come immutabili e si evita di modificare un file in uso. La strategia corretta è scrivere in una nuova posizione e aggiornare il puntatore — la tabella o il catalogo — solo quando la scrittura è completa. Un table format come Delta o Iceberg gestisce questo con un log transazionale; senza, devi progettare tu la convenzione. Non è un dettaglio: è la differenza tra un data lake affidabile e uno che ogni tanto restituisce numeri strani.

Nel cash flow i risultati sono per run e non vengono modificati mentre sono letti, quindi il rischio è limitato. Ma la domanda è giusta e la risposta non è "S3 è transazionale": non lo è.

**D. Perché non Redshift? Perché non DuckDB per tutto?**

R. Redshift è un warehouse gestito, ottimo per query complesse e performance prevedibili, ma costa, richiede caricamento e gestione, e ha senso quando il volume e la concorrenza lo giustificano. Qui i dati sono già su S3, il volume non è enorme e il team è piccolo: Athena è più semplice. Se il carico crescesse o servissero performance stabili su query concorrenti, Redshift sarebbe da valutare. Non è una scelta ideologica.

DuckDB è ottimo per analisi single-node su Parquet, ed è quello che usiamo nello store orchestrator. Per il cash flow, Athena dà un punto di accesso condiviso e serverless senza gestire un processo. DuckDB è più semplice e veloce in locale, ma non è un servizio multiutente: se più persone interrogassero gli stessi dati, avresti un processo per utente o un collo di bottiglia. La scelta dipende da chi deve interrogare e dalla concorrenza.

La differenza pratica è il modello di deployment: Athena è un servizio, DuckDB è una libreria. Per una dashboard embedded, la libreria è comoda; per un accesso condiviso e sporadico, il servizio è più adatto. Non c'è un vincitore: c'è un contesto.

**D. Come testeresti il layout e le query?**

R. Il layout si testa con un test che verifica la struttura: la partizione esiste, il nome è quello atteso, il file è leggibile, lo schema è compatibile. Non è un test di business, è un test di contratto. Poi un test di query: un dataset piccolo e noto, una query con filtro per `run_id`, e un'asserzione sul risultato e sui byte letti, se hai modo di misurarli. Senza un test sul layout, un cambio di naming può rompere tutte le query senza che nessuno se ne accorga fino alla dashboard.

Per la qualità: conteggi, nulli, duplicati, checksum, confronto con un output atteso. Per la compatibilità: un test che legge un file vecchio con lo schema nuovo e viceversa. Per le performance: un benchmark su un campione, non su tutto. Non testerei l'intero data lake: testerei il contratto e i casi critici.

Oggi nel progetto non c'è questa suite, e lo dico. Se dovessi costruirla, partirei dal layout e da una query di riferimento, perché sono i due punti che si rompono più facilmente e che nessuno guarda finché il risultato non è sbagliato.

**D. Cosa hai fatto tu in questo tema?**

R. Ho usato il salvataggio in Parquet e le query Athena nel flusso applicativo: ho lavorato sul codice che scrive i risultati e su quello che li legge. Non ho progettato il data lake, le partizioni, la governance o la retention, e non ho fatto benchmark. So spiegare perché la scelta ha senso e quali sono i limiti, ma non rivendico il design.

La distinzione è utile anche per capire cosa imparare: il prossimo passo sarebbe capire come si dimensiona un layout, come si misura il costo di una query e come si gestisce l'evoluzione dello schema. Sono cose che ho studiato, ma non ho ancora fatto in produzione. Dirlo è meglio che fingersi esperto e farsi smontare alla prima domanda sui byte scansionati.

### Prova di padronanza

- So partire dalla differenza tra carico transazionale e analitico, e spiegare perché i risultati delle simulazioni sono un dato analitico immutabile.
- So spiegare Parquet con i concetti giusti: column chunk, row group, statistiche, compressione, schema, predicate pushdown; e so dire quando un formato a righe è meglio.
- So spiegare il partizionamento per `run_id` come scelta legata all'access pattern, non come dettaglio: meno byte letti per una singola run, più partizioni e file piccoli quando le run crescono.
- So spiegare il problema dei file piccoli senza dire "è colpa di Parquet": è un problema di layout, si mitiga con compaction o con un partizionamento più grossolano.
- So spiegare come si paga Athena — byte scansionati — e collegarlo a filtri, selezione delle colonne, formati e materializzazione.
- So dire cosa succede se un file viene sostituito durante una query: S3 è immutabile, non c'è ACID tra file, servono convenzioni o un table format.
- So confrontare Redshift, DuckDB e Athena sul modello di deployment e sul carico, non su "quale è più moderno".
- So descrivere i failure mode del layout: file corrotto, schema drift, partizione duplicata, query senza filtro; e il rimedio per ciascuno.
- So dire cosa ho fatto — salvataggio e query — e cosa non ho fatto — data lake, partizioni, governance, benchmark.
- So ammettere che non ho misure di costo o di performance e spiegare come le otterrei.

### Cosa non dire

- "Parquet è un database": è un formato di file, non un motore con transazioni e indici.
- "Athena è veloce" senza parlare di partizioni e byte scansionati: la velocità dipende da quanto leggi.
- Ignorare il problema dei file piccoli o liquidarlo con "basta compattare" senza dire quando e come.
- Confondere partizione fisica e filtro di query: la prima è il layout, il secondo è come interroghi.
- Dire "S3 è transazionale" o "l'aggiornamento è atomico tra file": non lo è.
- Rivendicare il design del data lake o delle partizioni: non l'ho fatto.
- Dire "Redshift è meglio" o "DuckDB è meglio" senza contesto di concorrenza, volume e deployment.
- Inventare un risparmio o una riduzione di costo che non ho misurato.

### Se non lo so

Non ho progettato il data lake né le politiche di partizionamento, e non ho fatto benchmark su byte scansionati o tempi di query: se mi chiedi quanto costa una run o quanto impiega una query, non ho un numero e non lo invento. So spiegare il meccanismo e i criteri con cui lo misurerei. Se mi chiedi un dettaglio di configurazione di Athena o del catalogo, preferisco dirlo e ragionare sul contratto.

---

## 1.5 — DynamoDB e SQL vs NoSQL

### Risposta principale (3–5 minuti)

In DynamoDB non si parte dal modello dei dati, si parte dagli access pattern. È la differenza mentale più importante rispetto a un database relazionale. Nel nostro caso l'access pattern principale è semplice: leggere, scrivere e aggiornare un'esecuzione per identificativo, e mostrare lo storico delle esecuzioni. Non ci sono join tra entità, non ci sono transazioni multi-entità, non ci sono report ad hoc sul database. Ogni esecuzione è un'unità autonoma: la cerco per id, la aggiorno per id, la cancello per id.

Il modello che ne segue è una tabella con partition key `id`, senza sort key. Ci sono tabelle separate per i diversi tipi di esecuzione, e solo quella dei job ha un TTL (`expires_at`), perché sono dati temporanei. Ogni esecuzione è un item che contiene stato, tempi, parametri e i riferimenti ai risultati. La partition key `id` ha alta cardinalità e distribuisce bene le scritture: non c'è una hot partition strutturale, perché ogni run è diversa. Non abbiamo bisogno di una sort key perché non abbiamo un access pattern "tutte le esecuzioni di X ordinate per data" dentro la stessa partizione. Se lo avessimo, il modello sarebbe diverso.

La scelta di DynamoDB ha senso perché il carico è key-value, a bassa latenza, con access pattern noti e scala orizzontale. Non devi gestire un server, la capacità si adatta, e la latenza è prevedibile. Il prezzo è che il modello è rigido: le query sono quelle che hai progettato, non quelle che ti inventi dopo. Se domani il prodotto chiede "mostrami tutte le esecuzioni fallite dell'ultimo mese per cliente", con questa tabella non hai una query efficiente: hai uno scan o un indice nuovo.

Query e scan sono diversi. Query usa la partition key e legge solo quella partizione; è efficiente. Scan legge tutta la tabella e poi filtra; è costoso e peggiora con la crescita. Noi usiamo scan per lo storico, e qui c'è un limite concreto da conoscere: una chiamata di scan restituisce al massimo 1 MB di dati, e il codice non segue la paginazione. Oltre quella soglia l'elenco verrebbe troncato in silenzio. La risposta onesta è: scan va bene finché il volume è piccolo e lo sai, ma qui servirebbe come minimo la paginazione; quando il volume cresce serve un indice, un access pattern diverso o un altro database.

La consistenza è un'altra scelta. Di default DynamoDB è eventually consistent: una lettura può vedere un dato leggermente vecchio. Per le operazioni che devono vedere subito la scrittura, si può chiedere una read strongly consistent, a un costo maggiore. Per lo storico delle esecuzioni un ritardo di millisecondi è accettabile; per un saldo o per un'operazione che l'utente deve vedere immediatamente, no. La scelta dipende dal requisito, non dal default. Se l'utente lancia un job e ricarica la pagina, vuole vedere "in esecuzione": se la lettura è eventualmente consistente, può vedere lo stato precedente per un attimo. È un caso in cui valuterei una lettura forte o un aggiornamento ottimistico della UI.

I costi sono legati a letture, scritture, storage e indici. Una read strongly consistent costa più di una eventually consistent; uno scan costa più di una query; un item grande costa più di uno piccolo. Nel progetto le tabelle sono in modalità provisioned, con 5 unità di lettura e 5 di scrittura, e crittografia con chiave gestita dal cliente. A quella capacità lo scan pesa: consuma in proporzione ai dati letti, non ai risultati, quindi un elenco che cresce può esaurire la capacità e produrre throttling. Non ho scelto io la modalità, ma so leggerne le conseguenze: il modello e gli access pattern determinano il costo, non solo il volume.

I limiti vanno conosciuti. Un item può arrivare a 400 KB: oltre, bisogna frammentare o mettere il dato su S3 e tenere il riferimento. Non ci sono join: se ti servono, il modello è sbagliato o il database è sbagliato. Le transazioni esistono, ma non sono il punto di forza e hanno limiti e costi. Le GSI servono per access pattern alternativi, ma hanno un costo di scrittura e vanno aggiunte solo se il beneficio è chiaro. Il TTL è utile per dati temporanei, non è una politica di retention completa.

Quando userei PostgreSQL? Quando il dominio è relazionale, servono join, vincoli, transazioni, query ad hoc o report. Pynance è un esempio: dati finanziari, relazioni, vincoli, transazioni. In quel caso DynamoDB sarebbe una forzatura. La scelta non è "NoSQL è moderno": è "il modello degli accessi è semplice e noto, allora DynamoDB; il dominio è relazionale e le query cambiano, allora PostgreSQL". Sono due strumenti diversi per due problemi diversi.

Il mio ruolo: ho usato DynamoDB per leggere e scrivere lo stato delle esecuzioni, con boto3 e awswrangler. Non ho progettato la tabella, la partition key o le policy di capacità: il codice è del progetto e del cliente. So spiegare il modello, gli access pattern e i limiti rispetto a un database relazionale, ma non rivendico il design. Se mi chiedi "perché la partition key è `id` e non `stato`", so rispondere: perché `stato` ha pochi valori e creerebbe hot partition; ma la scelta non l'ho fatta io.

### Versione da 60 secondi

In DynamoDB si parte dagli access pattern. Nel cash flow servono due cose: leggere e scrivere un'esecuzione per `id` e mostrare lo storico. Quindi una tabella con partition key `id`, senza sort key, provisioned a 5 unità di lettura e scrittura, con TTL solo sui job. È una scelta coerente per un carico key-value; i limiti sono lo scan senza paginazione per lo storico, che oltre 1 MB tronca l'elenco, e gli aggiornamenti di stato fatti con get e put, quindi non atomici. Con query più ricche, report o join userei PostgreSQL. Non ho progettato io la tabella: l'ho usata e so spiegarne i trade-off.

### Follow-up

**D. Perché DynamoDB e non PostgreSQL?**

R. Perché l'access pattern è semplice e key-value: leggo e scrivo un'esecuzione per id, elenco lo storico. Non ho join, non ho aggregazioni complesse, non ho transazioni multi-entità. In questo caso DynamoDB è più semplice da gestire e scala orizzontalmente senza che io debba dimensionare un server. PostgreSQL sarebbe perfettamente in grado di fare la stessa cosa, ma mi darebbe caratteristiche che non uso e mi chiederebbe una gestione che non serve.

C'è anche un tema di latenza e di picchi: DynamoDB è pensato per accessi puntuali a bassa latenza e scala su molte partizioni. Un job asincrono che scrive lo stato ogni tanto non è un carico difficile per nessuno dei due, ma il modello key-value evita di occupare connessioni e transazioni. Se il volume di scritture fosse basso, PostgreSQL sarebbe più che sufficiente e più comodo per le query.

Quando cambierei: se il prodotto chiedesse report, filtri combinati, join con altre entità, o se il team volesse una sola tecnologia per tutto. In quel caso PostgreSQL sarebbe la scelta più pragmatica, anche a costo di perdere un po' di scala orizzontale. Non è una scelta definitiva: è una scelta che dipende dagli access pattern e dal costo di gestione.

**D. Cosa succede se scegli male la partition key?**

R. Se scegli una chiave con pochi valori — per esempio `stato`, con "in esecuzione", "completato", "fallito" — tutte le scritture finiscono in poche partizioni e crei una hot partition. DynamoDB distribuisce in base all'hash della chiave: se la chiave ha bassa cardinalità, la distribuzione è scarsa. Il sintomo è throttling sulle scritture e latenza che sale, anche se la tabella nel complesso non è satura. Non è un problema di capacità: è un problema di modello.

`id` è una buona scelta perché ogni run è diversa e la cardinalità è alta. Ma "buona" dipende dall'access pattern: se il caso principale fosse "tutte le run di un cliente", useresti il cliente come partition key e la data come sort key, accettando il rischio che un cliente molto attivo diventi una hot partition. In quel caso si può aggiungere un suffisso o cambiare la chiave. Non esiste una chiave giusta in astratto: esiste una chiave giusta per gli accessi.

Se dovessi cambiare modello, la strada è: capire quali query sono frequenti, scegliere la chiave che le serve, aggiungere una GSI per le altre, e misurare. Non si cambia partition key in produzione senza una migrazione: è una scelta che va presa prima, o pagata dopo.

**D. Query vs scan: perché lo scan è accettabile e quando smette di esserlo?**

R. Lo scan è accettabile quando il volume è piccolo, la lista serve tutta e la latenza non è critica. La UI dello storico ha bisogno di una lista, il numero di esecuzioni è contenuto, e filtrare lato applicazione è semplice. In quel contesto lo scan è una scelta pragmatica: non aggiungi un indice per una query che oggi costa poco. Il problema è che lo scan non scala: il costo e il tempo crescono con la tabella, non con il risultato.

Smette di essere accettabile quando la tabella cresce, quando la pagina diventa lenta, quando il costo delle letture si vede, o quando il filtro è quasi sempre lo stesso. A quel punto la soluzione è un access pattern dedicato: una GSI con una chiave che corrisponde al filtro, una tabella di indice, o una query per partizione. Non basta aggiungere `Limit`: la paginazione migliora l'esperienza ma non riduce il lavoro totale se poi l'utente scorre tutto.

La cosa matura è dichiararlo: "qui usiamo scan perché il volume è piccolo e lo sappiamo; se cresce, questo è il piano". Un intervistatore accetta una scelta con un limite dichiarato; non accetta "scan va bene sempre" o "NoSQL è veloce".

**D. Come aggiorni lo stato di un'esecuzione, e dove si rompe?**

R. Nel progetto l'aggiornamento dello stato è un metodo applicativo che fa due passi: legge l'intero record con `get_item` e lo riscrive con `put_item` dopo aver cambiato un campo. Non è una `UpdateItem` di DynamoDB. Il problema è duplice: la lettura è eventualmente consistente di default, quindi può leggere una versione vecchia, e la scrittura sovrascrive tutto il record, quindi se due scritture arrivano ravvicinate una cancella il lavoro dell'altra. È un classico lost update, e lo stato passa per quel metodo a ogni transizione: avvio, completamento, tempi, eccezione.

La correzione è usare `UpdateItem` con un'espressione che modifica solo l'attributo interessato e, dove serve, una condizione: per esempio impostare lo stato a `COMPLETED` solo se è `RUNNING`. In questo modo l'aggiornamento è atomico sul singolo item e le transizioni di stato invalide vengono rifiutate dal database invece che dal codice. Non l'ho corretto nel progetto: è una cosa che migliorerei, e mostra perché la scelta di DynamoDB va accompagnata da un uso coerente delle sue primitive.

**D. Consistenza eventuale vs forte: cosa cambia per l'utente?**

R. Con consistenza eventuale, una scrittura può non essere visibile immediatamente a una lettura successiva. Per lo stato di un job, questo significa che dopo aver lanciato una simulazione l'utente potrebbe vedere per un attimo lo stato precedente. Di solito è irrilevante, ma se la UI mostra "non avviato" e l'utente clicca di nuovo, può creare un doppio lancio. Quindi la consistenza non è solo un dettaglio del database: è un requisito di prodotto.

Con una read strongly consistent vedi l'ultima scrittura, ma paghi di più e hai una latenza leggermente maggiore. La scelta dipende da cosa l'utente deve vedere e quando. Per una lista storica, l'eventual consistency va bene; per la conferma di un comando, meglio forte o un aggiornamento ottimistico della UI che non dipende dalla lettura. In alcuni casi la soluzione non è nel database: è nel flusso.

Il costo della strong consistency è reale ma spesso piccolo rispetto al costo di un doppio lancio. La domanda da fare è: qual è il costo dell'incoerenza? Se è un refresh in più, accettabile; se è un'azione duplicata, no. Non si sceglie la consistenza per abitudine, si sceglie in base al rischio.

**D. On-demand o provisioned? E cosa succede quando c'è throttling?**

R. On-demand adatta la capacità al traffico e ti fa pagare per richiesta; provisioned ti fa prenotare capacità e costa meno se il carico è prevedibile, ma richiede di dimensionare e di gestire l'autoscaling. Per un carico irregolare come le esecuzioni di una simulazione, on-demand è più semplice; per un carico costante e alto, provisioned può essere più economico. Nel progetto le tabelle sono provisioned a 5 e 5, che per un carico piccolo e regolare è plausibile ma lascia poco margine se arrivano scan lunghi o scritture a raffica. Non ho deciso io quella scelta, ma so spiegarne il trade-off.

Il throttling arriva quando superi la capacità o quando una partizione è calda. Il sintomo è un errore di provisioned throughput exceeded, con retry che peggiorano la situazione se non hanno backoff. La strategia è: backoff esponenziale con jitter, limitare la concorrenza, distribuire meglio la chiave, e monitorare le metriche di throttling. Un retry immediato su una partizione calda è solo un modo per peggiorare.

La lezione è che "serverless" non significa "non devi pensare alla capacità": significa che non gestisci un server, ma devi comunque capire il modello di consumo. La capacità è una proprietà del design e del traffico, non un dettaglio del provider.

**D. Cosa fai se l'item è più grande di 400 KB?**

R. DynamoDB ha un limite di 400 KB per item: se il dato lo supera, devi frammentarlo o spostarlo. La soluzione tipica è mettere il payload grande su S3 e tenere nell'item i metadati e il riferimento. Per i risultati di una simulazione è naturale: i dati stanno su S3 in Parquet, l'item contiene lo stato e il puntatore. Non metterei un DataFrame in DynamoDB: non è il suo uso.

La frammentazione in più item è un'altra opzione, ma aggiunge complessità: devi ricomporre, gestire l'atomicità e le letture parziali. A quel punto conviene chiedersi se il database è quello giusto. Il limite di 400 KB non è un ostacolo da aggirare con un trucco: è un segnale che il dato appartiene a un altro storage.

Nel nostro caso il modello era già separato: stato nel database, risultati su S3. È una separazione sana, perché tiene piccolo l'item e lascia il dato analitico dove può essere interrogato. Non è una cosa che ho progettato io, ma è una ragione per cui il modello funziona.

**D. Transazioni e idempotenza: cosa useresti?**

R. DynamoDB supporta transazioni su più item, ma hanno limiti e costi e non sono il punto di forza del modello. Se devo aggiornare due entità insieme in modo atomico, le valuterei; se ogni esecuzione è indipendente, non servono. La transazione non sostituisce un modello sbagliato: se hai bisogno di atomicità multi-entità frequente, forse il dominio è relazionale.

Per l'idempotenza userei una scrittura condizionata: `put_item` con una condizione sullo stato o sull'identificativo, così un retry non sovrascrive un risultato già completo. È l'equivalente di un "insert if not exists" o di un update ottimistico. La chiave è l'identificativo della run: se lo stesso comando arriva due volte, la seconda scrittura non deve creare un duplicato. Questo è più efficace di un lock applicativo, perché la condizione è atomica nel database.

Non ho implementato io queste strategie nel progetto: parlo di ciò che farei. La differenza tra "so che esiste la scrittura condizionata" e "l'ho usata in produzione" è una differenza che dichiaro, perché in un colloquio conta.

**D. Come testeresti e osserveresti questo modello?**

R. Per testare, userei un database locale o un mock per la logica di accesso, e testerei i casi: item esistente, item mancante, aggiornamento concorrente, errore di throttling. La logica di business che dipende dallo stato va testata con stati finti, non con un DynamoDB vero. Poi un test di integrazione su un ambiente vero per verificare che la tabella, la chiave e i permessi siano corretti.

Per osservare: metriche di latenza, throttling, errori, capacità consumata; log con l'identificativo della run; alert su un tasso di errori anomalo. Un database key-value è facile da osservare finché qualcuno guarda; il problema è che spesso nessuno guarda finché la UI non è lenta. La dashboard delle metriche dovrebbe essere parte del progetto, non un'aggiunta dopo l'incidente.

Nel progetto non ho gestito questa parte: ho usato il database, non l'ho operato. So dire cosa guarderei, ma non ti racconto una monitorizzazione che non ho fatto.

**D. Cosa hai fatto tu?**

R. Ho usato DynamoDB per leggere e scrivere lo stato delle esecuzioni, con boto3 per letture, cancellazioni e scan e con awswrangler per le scritture; `update_item`, nel codice, è un metodo applicativo che fa get e put. Il codice è del progetto e del cliente. Non ho progettato la tabella, la partition key o la capacità, e non ho gestito le policy di accesso. So spiegare il modello e i trade-off, ma non rivendico il design.

Questa distinzione è importante perché il tema "NoSQL" è pieno di opinioni e povero di esperienza operativa. Io ho lavorato su un caso reale, ma non ho preso le decisioni di modellazione. Se l'intervistatore vuole sapere come progetterei una tabella da zero, posso ragionarci, ma lo dico come esercizio e non come esperienza.

### Prova di padronanza

- So partire dagli access pattern e non dal modello: "leggi/scrivi l'esecuzione X" e "mostra lo storico" sono i due casi che giustificano la tabella.
- So spiegare perché `id` è una buona partition key — alta cardinalità, point lookup — e perché `stato` sarebbe una cattiva scelta — pochi valori, hot partition.
- So distinguere query e scan e dire perché lo scan è accettabile a volume piccolo e quando smette di esserlo: crescita della tabella, latenza, costo, filtro ricorrente.
- So spiegare la consistenza eventuale con un esempio di prodotto: dopo il lancio, l'utente può vedere lo stato precedente e cliccare due volte; la soluzione può essere strong read o UI ottimistica.
- So collegare costi e modello: read strong più care, scan più costoso, item grande più costoso, GSI con costo di scrittura.
- So dire i limiti: 400 KB per item, niente join, transazioni limitate, query rigide; e cosa fare quando il dato è grande — S3 più riferimento.
- So confrontare DynamoDB e PostgreSQL su dominio, query e gestione, con Pynance come esempio relazionale.
- So parlare di idempotenza con una scrittura condizionata e di retry con backoff, distinguendo ciò che ho usato da ciò che farei.
- So descrivere l'osservabilità minima: latenza, throttling, errori, capacità, log correlati per run.
- So dire cosa ho fatto — uso via boto3 — e cosa non ho fatto — design della tabella e della capacità.

### Cosa non dire

- "DynamoDB è SQL ma più veloce": non è relazionale e non ha join; è un modello diverso.
- "NoSQL scala sempre meglio": dipende dalla partition key, dagli access pattern e dal carico.
- Non sapere la partition key o l'access pattern: è la prima cosa che un intervistatore chiede.
- Usare lo scan in produzione senza dichiararlo e senza un piano per quando cresce.
- Ignorare consistenza, item size e costi, o trattarli come dettagli del provider.
- Confondere una GSI con una foreign key o con un indice relazionale.
- Dire "basta mettere tutto in un item" senza considerare il limite di 400 KB.
- Rivendicare il design della tabella o delle policy di capacità.

### Se non lo so

Non ho progettato la tabella, la partition key o la capacità: il codice è del progetto e del cliente, e io l'ho usato. Non ho gestito throttling o incidenti su DynamoDB in produzione, quindi non ho storie operative da raccontare. So spiegare il modello, i trade-off e cosa farei, ma se mi chiedi un dettaglio di configurazione che non ricordo, preferisco dirlo invece di inventare.

---

## 1.6 — Store orchestrator: racconto e MVP

### Risposta principale (3–5 minuti)

Lo store orchestrator è una dashboard analytics per i punti vendita. Il vincolo che ne determina l'architettura non è il linguaggio né il database: è Streamlit. Streamlit riesegue lo script a ogni interazione: cambi un filtro, premi un bottone, ricarichi la pagina, e tutto il codice dall'inizio viene rieseguito. Se metti stato, filtri e query dentro la View, ogni rerun li ricrea; se la query è costosa, la rifai ogni volta; se lo stato è una variabile locale, lo perdi. Il problema non è estetico, è di correttezza e di prestazioni.

La separazione che ne è nata è in tre componenti: View, ViewManager e DataRepository. La View disegna la pagina: titolo, layout, componenti, input. Non dovrebbe contenere la logica di accesso ai dati. Il ViewManager tiene lo stato della pagina, i filtri, la cache dei risultati e le query: è il mediatore tra UI e dati. Il DataRepository possiede il motore DuckDB e registra le sorgenti Parquet; è condiviso tra sessioni perché è costoso e perché vogliamo una sola cache di processo.

Perché il ViewManager e non un controller? Perché in Streamlit non c'è un ciclo di richiesta-risposta come in un'app MVC: non c'è un controller che riceve l'input e coordina Model e View. L'input arriva dai widget, il rerun è gestito dal framework, e lo stato vive in `session_state` o in un oggetto legato alla sessione. Il ViewManager fa da presenter/viewmodel pragmatico: espone stato e metodi, la View lo chiama e rende. Non è un controller nel senso classico, ed è giusto dirlo.

Perché il DataRepository e non una connessione per pagina? Perché l'engine DuckDB e la registrazione delle sorgenti Parquet sono costosi e condivisi. Se ogni View creasse il suo engine, avremmo connessioni duplicate, cache incoerente e un consumo di memoria che cresce con le sessioni. Il repository è il punto unico per query e registrazione: un engine per processo, con una gestione della concorrenza esplicita. È una scelta di efficienza, non di purezza architetturale.

La classificazione è "ispirata a MVP", e ci tengo a spiegare perché non è MVP puro. In MVP la View è passiva: il Presenter contiene la logica di presentazione e aggiorna la View tramite un'interfaccia. Qui la View non è passiva: gestisce il rendering e parte dell'interazione, e in Streamlit è difficile separare completamente il layout dal comportamento. Non è MVC perché manca un controller separato. Non è MVVM puro perché non c'è un binding bidirezionale: il ViewManager espone stato e metodi, ma la View non si aggiorna automaticamente via binding; il rerun ricostruisce la pagina. È un ibrido pragmatico.

L'etichetta conta meno dei confini. La domanda utile è: chi decide cosa mostrare e quali dati caricare? Qui lo decide il ViewManager, con la View che rende. Chi possiede l'engine e le sorgenti? Il DataRepository. Chi gestisce lo stato che sopravvive ai rerun? Il manager, con l'aiuto di `session_state`. Se i confini sono chiari, l'etichetta è una comodità per comunicare; se non lo sono, l'etichetta nasconde il problema.

La cache è parte del design, non un'aggiunta, e ha quattro livelli. Il primo è il `DataRepository`, un singleton di processo con l'engine DuckDB (`cache_resource`). Il secondo è `cache_data`, con TTL di un'ora, per le opzioni dei filtri e i min/max. Il terzo è lo specchio locale dei Parquet: un job giornaliero scrive i file su S3, la dashboard li copia in locale all'avvio confrontando dimensione ed ETag, e un thread in background ripete la sincronizzazione ogni tre ore; DuckDB legge quei file come viste. Il quarto è la cache per sessione nel ViewManager, con chiave la fingerprint dello stato, che evita di rieseguire query quando i filtri non sono cambiati. Il rischio è la cache stantia: tra una sincronizzazione e l'altra l'utente vede dati vecchi di qualche ora. Non c'è un'invalidazione magica: va progettata.

La testabilità è il motivo pratico della separazione. La View è difficile da testare: dipende dal rendering. Il ViewManager è testabile: gli dai uno stato, verifichi filtri, fingerprint e query. Il DataRepository è testabile con un engine e dati di prova. Se la logica è nella View, devi avviare Streamlit per testarla; se è nel manager, no. Questo non significa che il progetto avesse una suite di test: c'è un solo file di test, sul report dei target, e lo dico. Significa che la separazione rende i test possibili, che è il primo passo.

Il mio ruolo: ho lavorato al presentation layer a metà con uno sviluppatore del cliente. Non ho progettato l'intera architettura, non ho deciso DuckDB e non ho scritto tutta la cache. So spiegare i componenti, il problema che risolvono e i limiti. I limiti principali sono: la View non è passiva, il controller non c'è, il confine tra stato di sessione e stato condiviso è sottile, e la cache può nascondere dati vecchi. Sono limiti accettabili a questa scala, ma vanno conosciuti.

### Versione da 60 secondi

Lo store orchestrator è una dashboard Streamlit su dati Parquet. Streamlit riesegue lo script a ogni interazione, quindi stato, filtri e query vanno separati dalla vista: `View` per il rendering, `ViewManager` per stato, filtri e query, `DataRepository` come singleton con l'engine DuckDB. La chiamo ispirata a MVP: il manager fa da presenter, ma la view non è passiva e non c'è un controller, perché il rerun fa le sue veci. Ho fatto il presentation layer a metà con lo sviluppatore del cliente, e so dire dove la cache può mostrare dati vecchi.

### Follow-up

**D. Perché non MVC? Dov'è il controller?**

R. MVC ha un controller che riceve l'input, aggiorna il model e coordina la view. In Streamlit non c'è un punto in cui l'input arriva a un controller: i widget aggiornano lo stato e il framework riesegue lo script. Potrei chiamare "controller" il ViewManager, ma sarebbe una forzatura: non riceve l'input in modo esplicito, non coordina un model separato, non gestisce un ciclo di richiesta. È più un presenter che un controller.

La conseguenza pratica è che la logica di presentazione è distribuita tra View e ViewManager, non concentrata in un controller. La View gestisce il rendering e parte dell'interazione; il manager gestisce stato e query. È un compromesso dettato dal framework. Se volessi un controller esplicito, potrei estrarlo, ma sarebbe un'astrazione in più senza un beneficio chiaro a questa scala.

Se l'intervistatore insiste "allora è MVC senza controller", la risposta è: no, è una separazione pragmatica. MVC senza controller non è MVC; è un nome che nasconde l'assenza di un pezzo. Meglio dire "ispirata a MVP" e spiegare i confini, che difendere un'etichetta sbagliata.

**D. Perché non MVVM puro?**

R. MVVM ha un ViewModel che espone stato e comandi, e la View fa binding: quando il ViewModel cambia, la View si aggiorna, e spesso il binding è bidirezionale. Qui non c'è binding: la View legge lo stato dal manager e renderizza, ma non c'è un meccanismo dichiarativo che osservi le proprietà e aggiorni i componenti. Il rerun di Streamlit ricostruisce la pagina, non fa binding.

Il ViewManager assomiglia a un ViewModel perché espone stato e metodi, ma gli manca il binding. È più vicino a un presenter che a un ViewModel. Se lo chiamassi MVVM, un esperto di frontend ti farebbe domande sul binding e sulla reattività, e la risposta sarebbe "non c'è". Meglio essere precisi: è un'architettura ispirata a MVP, adattata a un framework che non ha binding.

La differenza conta per il testing: in MVVM testi il ViewModel e il binding separatamente; qui testi il manager e il rendering separatamente. Il principio è simile, il meccanismo no. L'etichetta non cambia il codice, ma cambia la conversazione: se usi la parola giusta, l'intervistatore capisce cosa hai capito.

**D. Perché un ViewManager e non un service di dominio?**

R. Perché il ViewManager non contiene regole di business: contiene stato di presentazione, filtri, cache e orchestrazione delle query. Un service di dominio conterrebbe regole del dominio — cos'è un punto vendita, come si aggrega il fatturato, quali sono le invarianti — e sarebbe indipendente dalla UI. Qui il manager è legato alla pagina: sa quali filtri esistono, quali viste ci sono, come si compone lo stato. È un oggetto di presentazione, non di dominio.

Questa distinzione è importante perché evita di mettere regole di business nella UI. Se una regola è "il fatturato si calcola così", non deve stare nel ViewManager; deve stare in un modulo di dominio o nel repository, e il manager la chiama. Se una regola è "se l'utente seleziona due anni, mostra il confronto", quella è presentazione e sta nel manager. Il confine non è sempre netto, ma va dichiarato.

Nel progetto il presentation layer era fatto a metà con lo sviluppatore del cliente, e il confine tra presentazione e dominio è stato una delle cose discusse. Non ti dico che fosse perfetto: ti dico che la distinzione era chiara abbastanza da poter lavorare in due senza pestarci i piedi.

**D. Come gestisci lo stato e i rerun di Streamlit?**

R. Lo stato che deve sopravvivere ai rerun sta in `session_state` o in un manager legato alla sessione. Gli oggetti costosi e condivisi stanno in `cache_resource`, che li tiene per processo. Le variabili locali della funzione, invece, non sopravvivono: se metti un filtro in una variabile locale, al rerun successivo è perso. La regola è: ogni cosa che deve sopravvivere ha un posto esplicito, e ogni cosa che non deve sopravvivere non deve essere globale.

La View non tiene stato come attributo globale: usa un helper di stato legato alla sessione (`self.state`, con uno scope per pagina e uno generale) e legge dal manager ciò che deve mostrare. L'istanza della View vive per sessione e viene scartata quando si cambia pagina, quindi gli attributi dell'istanza sopravvivono ai rerun ma non ai cambi di pagina: per questo lo stato che deve restare sta nello `session_state` e nel manager. Il manager è legato alla sessione, il repository è condiviso: sono due scope diversi, e confonderli è il bug classico di Streamlit. Se metti l'engine in `session_state`, crei un engine per sessione; se metti lo stato utente in `cache_resource`, lo condividi tra tutti. Nessuno dei due è giusto in assoluto: dipende da cosa stai gestendo.

La fingerprint dello stato evita di rieseguire query inutili: se i filtri non sono cambiati, il risultato è lo stesso e si può riusare. È un'ottimizzazione semplice ma va progettata: la fingerprint deve includere tutto ciò che influenza la query, non solo i filtri visibili. Se dimentichi una dimensione, la cache restituisce un risultato sbagliato e sembra un bug di dati.

**D. Come eviti query duplicate e cosa costa la cache?**

R. Evito le query duplicate con tre strumenti: la fingerprint dello stato nel manager, `cache_data` per i risultati piccoli come le opzioni dei filtri, e i dati stessi, che stanno in locale come Parquet e vengono letti da DuckDB senza costo per query. La fingerprint dice "questa combinazione di filtri l'ho già vista"; la cache dice "questo risultato l'ho già calcolato". Sono due livelli: uno evita di chiamare, l'altro evita di ricalcolare.

Il costo è memoria e freschezza. `cache_data` occupa memoria nel processo; `cache_resource` tiene un oggetto condiviso; lo specchio locale dei Parquet occupa disco e introduce un ritardo, perché viene aggiornato ogni tre ore. La domanda è: quanto è grave un dato vecchio? Per una dashboard commerciale esplorativa, qualche ora è accettabile; per un report finanziario, no. La cache non è gratis: sposta il costo dalla CPU alla memoria e dalla freschezza alla latenza.

La cosa che migliorerei è la misura: hit rate, miss rate, tempo risparmiato, memoria usata. Senza misure, la cache è una scommessa. Nel progetto non avevamo queste metriche, e lo dico. Se dovessi rifarlo, metterei un contatore semplice per capire se la cache serve davvero o sta solo complicando il codice.

**D. Cosa succede con due sessioni che usano la dashboard insieme?**

R. Ogni sessione ha il suo stato e il suo ViewManager, ma condivide il DataRepository e l'engine DuckDB. La registrazione delle sorgenti è protetta da un lock perché modifica il catalogo; le query read-only possono girare in parallelo con un cursore per query. È un modello pensato per un singleton di processo, non per un cluster. Con due sessioni va bene; con molte, la concorrenza sull'engine e sulla memoria diventa un limite.

Il rischio è che una query pesante blocchi le altre o consumi memoria condivisa. DuckDB è veloce, ma non è un database multiutente con isolamento e priorità. Se il numero di utenti crescesse, valuterei un motore diverso o un'architettura con query asincrone e risultati materializzati. A questa scala, la semplicità vale più della scalabilità teorica.

La distinzione tra stato di sessione e stato condiviso è la chiave: se confondi i due, due utenti si vedono i filtri a vicenda. È un bug che sembra un problema di dati e invece è un problema di scope.

**D. Come testeresti questa UI?**

R. La logica del ViewManager e del DataRepository è testabile in isolamento: gli dai uno stato, verifichi i filtri, la fingerprint, la query e la cache. La View è difficile da testare end-to-end: dipende dal rendering e dal framework. Per la UI farei test di fumo — la pagina si carica, i componenti principali ci sono — e screenshot test se il progetto li giustifica. Non inseguirei ogni dettaglio visivo: il costo supera il beneficio.

Il test più utile è sul confine: dato uno stato, il manager chiama il repository con la query giusta? Dato un risultato, la View lo mostra? Se il confine è testato, il resto è rendering. Se la logica è nella View, invece, devi avviare Streamlit per testarla e il test diventa lento e fragile. La separazione serve anche a questo.

Nel progetto non c'è una suite di test vera: c'è un solo file, sul report dei target, e questo è un limite che dichiaro. So dire come la costruirei: test del manager, test del repository con dati finti, test di fumo della pagina. Non ti racconto una suite che non esiste.

**D. Cosa miglioreresti, e quando cambieresti architettura?**

R. Migliorerei tre cose. Primo: renderei più esplicito il confine tra stato di sessione e stato condiviso, con tipi o oggetti dedicati. Secondo: aggiungerei test sul manager e sul repository, partendo dai casi che si rompono più spesso — filtri vuoti, nessun dato, cache scaduta. Terzo: ridurrei le responsabilità della View, spostando nel manager la logica che oggi è mescolata al rendering. E sceglierei un'etichetta coerente: se resta "ispirata a MVP", va spiegato perché, senza pretendere che sia MVP puro.

Cambierei architettura se gli utenti crescessero, se servisse un frontend più ricco o se la latenza delle query diventasse un problema. In quel caso passerei a un'API backend e a un frontend separato, con query asincrone e cache lato server. Non è una migrazione da fare oggi: è il piano per quando il contesto cambia. La scelta di Streamlit era adatta a una dashboard interna e a un team piccolo; non è una scelta universale.

Il criterio è sempre lo stesso: l'architettura deve essere la più semplice che regge il contesto, con un piano per quando non regge più. Non devo difendere Streamlit per sempre, né migrare a React perché è di moda. Devo sapere quali sono i limiti e quando si manifestano.

**D. Cosa hai fatto tu?**

R. Ho lavorato al presentation layer a metà con uno sviluppatore del cliente: View, ViewManager, filtri, cache, rerun. Non ho progettato l'intera architettura e non rivendico le scelte di DuckDB o della cache S3. So spiegare i componenti, il problema che risolvono e i limiti. Se l'intervistatore chiede "chi ha deciso questa separazione?", la risposta è: è stata una decisione condivisa, non mia soltanto.

Questa onestà è più utile di una rivendicazione: mi permette di parlare di ciò che ho capito davvero, cioè i confini e i problemi, senza dover difendere scelte che non ho fatto. E mi permette di dire cosa farei diversamente, che è la parte più interessante per un intervistatore senior.

### Prova di padronanza

- So partire dal vincolo di Streamlit — rerun a ogni interazione, stato che non sopravvive, query costose — e spiegare perché la separazione è una necessità, non un'estetica.
- So descrivere i tre componenti con le loro responsabilità: View per il rendering, ViewManager per stato, filtri, cache e query, DataRepository per engine DuckDB e sorgenti Parquet.
- So dire perché non è MVC — manca un controller che riceve l'input e coordina — e perché non è MVVM puro — manca il binding bidirezionale.
- So riconoscere che la View non è passiva e che l'etichetta "ispirata a MVP" è un compromesso, non una classificazione da manuale.
- So spiegare la differenza tra stato di sessione e stato condiviso: manager per sessione, repository per processo, e cosa succede se li confondi.
- So spiegare i quattro livelli di cache: singleton del repository, `cache_data` con TTL, specchio locale dei Parquet sincronizzato da S3 ogni tre ore, cache per sessione nel manager; e il rischio di dati stantii.
- So spiegare la fingerprint dello stato e perché deve includere tutto ciò che influenza la query, non solo i filtri visibili.
- So dire come testerei: manager e repository in isolamento, test di fumo per la UI; e che oggi esiste un solo file di test, sul report dei target.
- So indicare i limiti e il piano: single process, concorrenza limitata, passaggio a API più frontend se gli utenti o la latenza crescono.
- So dire cosa ho fatto — presentation layer a metà con lo sviluppatore del cliente — e cosa non ho fatto — architettura complessiva, DuckDB, cache.

### Cosa non dire

- "È MVC" o "è MVVM puro": non è né l'uno né l'altro, e l'intervistatore lo capisce dalla prima domanda sul controller o sul binding.
- "L'ho fatto tutto io": il presentation layer è stato fatto a metà con lo sviluppatore del cliente.
- Non sapere dov'è il controller e perché non c'è: è la domanda che smonta una classificazione approssimativa.
- Ignorare i rerun di Streamlit e la differenza tra `session_state` e `cache_resource`.
- Confondere il ViewManager con un service di dominio: il manager gestisce presentazione, non regole di business.
- Dire "la cache risolve" senza parlare di TTL, invalidazione e dati stantii.
- Dire "basta passare a React" senza spiegare il costo e il contesto in cui avrebbe senso.
- Rivendicare l'intera architettura o la scelta di DuckDB.

### Se non lo so

Il presentation layer l'ho fatto a metà con uno sviluppatore del cliente, quindi non rivendico l'intera architettura né le scelte di DuckDB e della cache. Una suite di test vera non c'era (esiste un solo file sul report dei target), quindi non posso raccontarti una strategia di testing che è stata applicata: posso dirti come la costruirei. Se mi chiedi un dettaglio di Streamlit che non ricordo, preferisco dirlo e ragionare sul comportamento, invece di inventare un'API.

---

## 1.7 — Migrazione SAS → AWS: racconto onesto, ETL, Glue

### Risposta principale (3–5 minuti)

Questo è l'argomento in cui è più facile scivolare nell'invenzione, quindi parto da ciò che è certo. Ho partecipato a una migrazione di un data model da SAS ad AWS per una casa automobilistica, da febbraio 2023 a maggio 2024. Ero junior. Ho tradotto decine di flussi SAS interdipendenti in job ETL su AWS Glue con PySpark, organizzati in livelli di elaborazione dal preprocessing al reporting. L'architettura — Step Functions, database di stato, scelte di piattaforma — era dell'architetto. Io ho fatto la traduzione e lo sviluppo dei job. Non ho il repository, e non ricostruisco dettagli che non ricordo.

Il contesto è importante perché spiega il tipo di lavoro. Una casa automobilistica aveva un data model in SAS, con flussi interdipendenti e livelli che producevano output per il management. La migrazione non era "riscrivere un programma": era portare un insieme di trasformazioni, con dipendenze e semantiche diverse, su una piattaforma diversa. Il rischio non è che il codice non compili: è che il numero sia sbagliato e nessuno se ne accorga. Per questo una migrazione è soprattutto un problema di equivalenza, non di sintassi.

Il mio lavoro quotidiano era: leggere un flusso SAS, capire input, trasformazioni e output, tradurlo in PySpark ed eseguirlo su Glue. I job erano organizzati a livelli, dal preprocessing al reporting: ogni livello produceva un output trasformato per il successivo. Non progettavo l'orchestrazione: quella era dell'architetto. Ero dentro il flusso di traduzione, con una comprensione parziale del sistema complessivo.

L'architettura era di qualcun altro, e questo ha una conseguenza pratica: non so spiegarti perché Step Functions fosse configurato in un certo modo, né come fosse modellato il database di stato. So cos'è Step Functions e so cosa fa in una pipeline, ma non l'ho progettato. Se l'intervistatore chiede "come gestivate i retry?", la risposta onesta è: c'era un'orchestrazione dell'architetto, io non ne ho curato la configurazione. Posso ragionare su come dovrebbe funzionare, ma non spacciarlo per esperienza.

Il flusso di lavoro aveva un limite che racconto senza nasconderlo: lo sviluppo avveniva nei notebook Glue nel browser, poi il codice veniva copiato manualmente nella repository. Non era un passaggio automatico. Questo crea un rischio di drift tra ciò che gira e ciò che è versionato: se modifichi nel notebook e dimentichi di copiare, la repository non è la verità. Non lo racconto come una best practice, lo racconto come un limite. Oggi metterei il codice in repository con un flusso automatico e test, e userei il notebook solo per esplorare.

Sul piano ETL/ELT: il lavoro era per job ETL a livelli. Ogni livello estraeva, trasformava e produceva un output per il successivo. Non era un ELT puro, perché non caricavamo tutto grezzo in un warehouse e trasformavamo lì con SQL. Glue e Spark possono fare entrambi; la differenza è dove avviene la trasformazione e quanto dato grezzo conservi. La scelta ha conseguenze su costi, flessibilità e governance, e nel progetto era una scelta dell'architettura, non mia.

Le differenze tra SAS e Spark sono la parte che posso difendere meglio. Il DATA step di SAS è procedurale, row-by-row, con variabili che portano lo stato (`RETAIN`, `FIRST.` e `LAST.` sui gruppi `BY`); Spark è set-based e distribuito, con trasformazioni lazy, e quello stato diventa una window function o un'aggregazione. I missing value sono il punto più insidioso: in SAS un numerico mancante è minore di qualunque numero nei confronti e negli ordinamenti, quindi `x < 5` è vero per un missing; in Spark un confronto con `null` non è né vero né falso, è `null`, e la riga viene scartata. Ogni filtro va riscritto con un `isNull` esplicito. Le date SAS contano i giorni dal 1960-01-01, e una conversione sbagliata sposta un report di anni. Un `MERGE` SAS ha regole proprie su chiavi, duplicati e ordine; una join Spark con chiavi non uniche moltiplica le righe, e l'ordine non è garantito. Le macro non hanno un equivalente diretto: diventano funzioni Python o parametri.

Su come si verifica l'equivalenza di una migrazione posso parlare solo di come lo farei oggi, perché sul progetto specifico non ho elementi da portare e non voglio costruirli. Userei: conteggi per livello, schema di input e output, nulli e duplicati sulle chiavi, somme di controllo sulle misure, confronto per chiave tra sorgente e target, un campione verificato a mano e un golden dataset per le regole di business. Non mi fiderei del "numero di righe uguale": due dataset possono avere lo stesso numero di righe e valori diversi.

Perché parlo soprattutto di Spark? Perché è la tecnologia che posso difendere meglio e perché le domande su Spark sono più frequenti e più generali. Sul progetto specifico ho poche informazioni: non ho il repository, non ho progettato l'architettura, non ricordo i dettagli di sorgenti e destinazioni. Se l'intervistatore vuole approfondire il progetto, dico cosa so e cosa non so; se vuole approfondire Spark, posso dare il meglio. Non è una strategia per evitare le domande: è un modo per non inventare.

### Versione da 60 secondi

Da febbraio 2023 a maggio 2024, da junior, ho tradotto decine di flussi SAS interdipendenti in job ETL su AWS Glue con PySpark, organizzati in livelli dal preprocessing al reporting. L'architettura, con Step Functions e database di stato, era dell'architetto. Quello che porto è Spark come tecnologia e le differenze con SAS: i missing value (in SAS minori di ogni numero, in Spark `null`), le date (giorni dal 1960), il `MERGE` contro la join, lo stato accumulato che diventa window function. Dei dettagli del progetto non ricordo sorgenti e destinazioni, e lo dico.

### Follow-up

**D. Cosa facevi esattamente ogni giorno?**

R. Leggevo un flusso SAS e cercavo di capire cosa facesse davvero: quali input leggeva, quali trasformazioni applicava, quali output produceva, quali casi limite gestiva. Poi lo traducevo in PySpark e lo eseguivo su Glue. Il lavoro era ripetitivo nella forma ma non nella sostanza: ogni flusso aveva le sue regole, e le differenze di semantica tra SAS e Spark emergevano nei dettagli.

La parte più difficile non era scrivere PySpark: era capire cosa il flusso SAS volesse fare quando il codice era procedurale, con passi impliciti e macro. Bisognava ricostruire l'intenzione, non solo la sintassi: un passo che in SAS filtra, calcola e riordina insieme diventa in Spark più trasformazioni, e se se ne perde una l'output cambia.

Non gestivo l'orchestrazione e il mio perimetro era la traduzione e lo sviluppo dei job. Questo è il fatto; il resto sarebbe una ricostruzione.

**D. Perché Glue e non EMR?**

R. Glue è Spark serverless: non gestisci il cluster, hai integrazione con Data Catalog, S3 e Step Functions. Per un team piccolo e un lavoro di traduzione, era la scelta più semplice: scrivi il job, lo esegui, non devi dimensionare nodi o gestire un cluster. EMR dà più controllo — configurazione, dimensioni, versioni, costi — ma richiede più gestione e più competenze operative. La scelta dipende dal contesto: qui il contesto era una migrazione con molti job e un team che doveva consegnare, non gestire infrastruttura.

Il trade-off è il controllo. Con Glue hai meno visibilità su come vengono allocate le risorse e meno flessibilità su configurazioni particolari. Per carichi molto grandi o tuning spinto, EMR può convenire. Per un job ETL standard, Glue è più diretto. Non è "Glue è meglio": è "Glue riduce il carico operativo, a costo di controllo".

La scelta non l'ho fatta io: era dell'architetto. So spiegare il trade-off, ma non rivendico la decisione. Se mi chiedi "quanto costava al mese", non lo so.

**D. ETL vs ELT: quale hai usato?**

R. Job ETL a livelli: ogni livello trasformava e produceva un output per il successivo. Non era un ELT puro, perché non caricavamo tutto grezzo in un warehouse e trasformavamo lì. Era un flusso a stadi, con output intermedi persistenti. Questo rende più facile capire dove si è rotto qualcosa e riprendere da un livello, ma aggiunge storage e passaggi.

La differenza non è solo dove avviene la trasformazione: è chi ha il controllo e quanto raw conservi. ELT è più flessibile quando hai un warehouse potente e vuoi iterare con SQL; ETL è più diretto quando le trasformazioni sono codice e i livelli sono contratti. Nel progetto il codice era PySpark, quindi ETL a livelli era coerente con gli strumenti.

Se dovessi rifarlo, valuterei se alcuni livelli potessero essere espressi in SQL nel warehouse, per ridurre codice e facilitare l'analisi. Ma senza conoscere i requisiti e i costi, non dico che sarebbe stato meglio. Dipende dal volume, dal warehouse e dal team.

**D. SAS vs Spark: dammi tre differenze concrete.**

R. La prima: il modello di esecuzione. Il DATA step di SAS è procedurale e row-by-row; Spark è set-based e distribuito, con trasformazioni lazy. In SAS pensi a "per ogni riga fai questo", con variabili che portano lo stato (`RETAIN`, `FIRST.` e `LAST.`); in Spark pensi a "trasforma l'insieme", e quello stato diventa una window function o un'aggregazione. Cambia il modo di scrivere, non solo la sintassi.

La seconda: i missing value. In SAS un numerico mancante è minore di qualunque numero: `if x < 5` include i missing. In Spark un confronto con `null` è `null`, non vero né falso, e il filtro scarta la riga. Se non traduci la regola con un `isNull` esplicito, i risultati cambiano nei casi limite e nessun errore lo segnala.

La terza: merge e date. Un `MERGE` SAS ha regole specifiche su chiavi, duplicati e ordine; una join Spark con chiavi non uniche moltiplica le righe, e l'ordine non è garantito. Le date SAS sono giorni dal 1960-01-01: una conversione sbagliata sposta un report di anni. Le macro sono un quarto punto: non hanno un equivalente diretto, diventano funzioni Python o parametri. Sono differenze che si vedono solo sui casi limite, ed è per questo che i controlli di equivalenza contano.

**D. Come valideresti l'equivalenza tra due sistemi dopo una migrazione?**

R. Prima di tutto, non mi fiderei del confronto a occhio su un campione felice. Userei controlli automatici: conteggi per livello, schema di input e output, nulli e duplicati sulle chiavi, somme di controllo sulle misure, distribuzioni delle colonne principali, confronto per chiave tra sorgente e target. Poi un campione di righe verificato a mano, e un golden dataset con le regole di business più importanti. Se il numero di righe è uguale ma i valori no, il controllo per chiave lo scopre.

Aggiungerei test di trasformazione: input piccolo e noto, output atteso, casi limite — null, data minima, duplicati, chiavi mancanti. E un test di idempotenza: rieseguire lo stesso job sullo stesso input non deve duplicare l'output. Per una migrazione, il test più utile è il confronto tra il sistema vecchio e il nuovo sullo stesso periodo, con una spiegazione per ogni differenza accettata. Non "abbiamo controllato", ma "ecco le differenze e perché sono ok".

**D. Cosa non sai, e come lo scopriresti?**

R. Non conosco più i dettagli di sorgenti e destinazioni, non ho il repository, non ho progettato Step Functions o il database di stato, e non conosco volumi, tempi di esecuzione e costi. Sono lacune reali, non modestia. Se dovessi ricostruire, chiederei all'architetto: quali flussi erano critici, come erano modellati i livelli, come venivano gestiti retry e fallimenti, quale era la fonte di verità.

Poi guarderei il repository, se esistesse: la struttura dei job, i parametri, i test, la configurazione. Poi il lineage: da dove viene ogni tabella, quali trasformazioni, quali output. Poi confronterei un periodo tra vecchio e nuovo, con un campione e un controllo per chiave. Sono passi che posso descrivere anche senza il progetto, perché sono il modo in cui si verifica una migrazione.

La cosa che non farei è improvvisare una risposta plausibile. In un colloquio, una lacuna dichiarata e un metodo di verifica valgono più di un dettaglio inventato che crolla alla seconda domanda.

**D. Cosa faresti diversamente?**

R. Metterei il codice in repository con un flusso automatico, non con la copia manuale dai notebook. Aggiungerei test di trasformazione e controlli di qualità nei punti critici. Documenterei le differenze SAS/Spark per ogni flusso, in modo che la conoscenza non resti nella testa di chi ha tradotto. E userei il progetto per imparare Spark in modo più sistematico, non solo per tradurre: capire perché un job è lento, come sono partizionati i dati, dove va il tempo.

Sul piano del processo, metterei un confronto automatico tra vecchio e nuovo per ogni livello, con soglie e un registro delle differenze accettate. Non aspetterei la fine per scoprire che un numero non torna. È il tipo di investimento che sembra rallentare e invece evita settimane di debug.

Queste sono cose che oggi so; allora ero junior e il contesto era quello che era. Non mi do la colpa di non averle fatte, ma non le racconto come se fossero state fatte.

**D. Perché parli soprattutto di Spark?**

R. Perché è la tecnologia che posso difendere meglio: ho studiato PySpark, ho lavorato su job reali e le domande su Spark sono più frequenti e più generali. Sul progetto SAS specifico ho poche informazioni: non ho il repository, non ho progettato l'architettura, non ricordo i dettagli. Se l'intervistatore vuole parlare del progetto, dico cosa so e cosa non so; se vuole parlare di Spark, posso dare il meglio.

Non è una strategia per evitare le domande: è un modo per non inventare. Un candidato che dice "non ho il repository, non ricordo" è più credibile di uno che ricostruisce una pipeline perfetta a memoria. E se l'intervistatore vuole testare la profondità, Spark è un terreno dove posso essere testato davvero.

**D. Cosa hai imparato da questo progetto?**

R. Che una migrazione non è una traduzione. Cambiano semantiche, dati, ambiente e modo di lavorare. Ho imparato Spark sul campo, ho capito l'importanza di repository e test, e ho visto quanto conta la differenza tra "il job gira" e "il numero è giusto". È stata la mia porta d'ingresso al data engineering, anche se non era il mio ruolo principale.

Ho anche imparato a riconoscere i miei limiti: ero junior e non avevo il quadro completo. Non è una cosa negativa: è il modo in cui si impara. Oggi, in un colloquio, posso raccontare quel periodo senza gonfiarlo e senza sminuirlo. È un'esperienza reale, con confini reali.

### Prova di padronanza

- So dare il contesto esatto: casa automobilistica, febbraio 2023–maggio 2024, junior, decine di flussi SAS interdipendenti, Glue/PySpark, livelli preprocessing→reporting.
- So separare i fatti dalle inferenze: il notebook Glue e la copia manuale in repository sono un fatto dichiarato; il drift tra notebook e repository è un rischio che descrivo come tale, non come una cosa osservata.
- So dire cosa non ho fatto: architettura, Step Functions, database di stato, scelte di piattaforma, modello dati.
- So spiegare ETL vs ELT senza slogan: dove avviene la trasformazione, quanto raw conservi, quali sono i costi e la governance.
- So spiegare perché Glue e non EMR: serverless e integrazione contro controllo e gestione del cluster.
- So portare le differenze SAS/Spark che mordono: missing value minori di ogni numero contro `null`, date dal 1960, `MERGE` contro join, stato accumulato (`RETAIN`, `FIRST.`/`LAST.`) che diventa window function, macro che diventano funzioni.
- So descrivere come valuterei l'equivalenza di una migrazione, in generale: conteggi, schema, nulli, duplicati, checksum, confronto per chiave, campione, golden dataset, differenze spiegate.
- So spiegare cosa farei diversamente: repository con flusso automatico, test, documentazione delle differenze, confronto automatico tra vecchio e nuovo.
- So usare il progetto per parlare di Spark senza spacciarlo per un progetto di architettura dati.

### Cosa non dire

- "Ho progettato la pipeline" o "ho scelto Step Functions": non è vero, l'architettura era dell'architetto.
- Inventare sorgenti, destinazioni, volumi, tempi o risparmi: non ho il repository e non ho misure.
- Presentare il drift notebook/repository come un fatto osservato: è un rischio che descrivo come ipotesi.
- Dire "SAS e Spark sono uguali" o "ho fatto solo pandas": sono due modelli diversi e il lavoro era PySpark su Glue.
- Rispondere su dettagli che non conosco invece di riportare il discorso su Spark e sui concetti.
- Minimizzare il ruolo junior: non è un difetto, è il contesto, e spiega perché non ho progettato l'architettura.
- Raccontare il progetto come un successo personale senza dire cosa hanno fatto l'architetto e il team.

### Se non lo so

Non ho il repository e non conosco più i dettagli di sorgenti, destinazioni e volumi: se me li chiedi, dico che non lo so e non invento. Non ho progettato Step Functions, il database di stato o il modello dati: l'architettura era dell'architetto. So spiegare il contesto, il mio ruolo e i concetti di Spark e di migrazione; il resto lo tratto come una lacuna dichiarata, non come una storia da riempire.

---

## 1.8 — Apache Spark: architettura, partizioni, join, skew

### Risposta principale (3–5 minuti)

Spark è un motore distribuito per l'elaborazione di dati. La prima domanda da farsi non è "come funziona", è "quando conviene". Conviene quando i dati non stanno comodamente su una macchina, quando serve elaborarli in parallelo e quando vuoi una fault tolerance a livello di dato senza scrivere tu il coordinamento. Se i dati stanno in memoria su un laptop, pandas, Polars o DuckDB sono spesso più semplici e più veloci, perché non hanno l'overhead di un cluster. Spark non è "big data" in astratto: è una scelta per un certo ordine di grandezza e un certo tipo di carico.

L'architettura ha tre ruoli. Il driver è il processo che coordina: costruisce il piano, tiene lo stato, parla con il cluster manager. Gli executor sono i processi che eseguono i task e tengono i dati in memoria o su disco. Il cluster manager — YARN, Kubernetes, standalone o un servizio gestito — fornisce le risorse. In Glue il cluster è gestito dal servizio: scrivi PySpark e scegli tipo e numero di worker, ma non gestisci i nodi. Lo `SparkSession` è il punto di ingresso: da lì leggi, trasformi, esegui query e accedi alla configurazione.

Le trasformazioni sono lazy. Quando scrivi `select`, `filter`, `join`, Spark non esegue: costruisce un DAG, un grafo di trasformazioni. Solo un'azione — `count`, `collect`, `write`, `show` — scatena il calcolo. Questo permette all'ottimizzatore di vedere l'intero piano e applicare regole come predicate pushdown, column pruning e join reordering. Il rovescio è che un errore può comparire solo all'azione, non dove hai scritto la trasformazione: il traceback ti porta al punto in cui hai chiesto il risultato, non necessariamente alla trasformazione sbagliata.

L'esecuzione si divide in job, stage e task. Un'azione genera un job. Il job è diviso in stage ai confini degli shuffle: uno stage è un insieme di trasformazioni che possono essere eseguite senza scambiare dati tra executor. Ogni stage è diviso in task, uno per partizione. Gli executor eseguono i task. Se vedi molti stage e molti shuffle, probabilmente c'è un'operazione costosa. La Spark UI è lo strumento per capire dove va il tempo, non per guardare se "il job è verde".

Le partizioni sono l'unità di parallelismo. Troppe partizioni significano overhead di task e scheduling; troppo poche significano poco parallelismo e rischio di skew. `repartition` ridistribuisce i dati e può aumentare o diminuire il numero di partizioni, con shuffle; `coalesce` riduce senza shuffle, ma può creare partizioni sbilanciate. La scelta dipende dal volume, dal numero di core e dal tipo di operazione. Non c'è un numero magico: c'è un ordine di grandezza da verificare sulla UI.

Lo shuffle è lo scambio di dati tra executor, necessario per join, groupBy, distinct e repartition. È costoso perché usa rete, disco e serializzazione, e perché crea una barriera: gli stage successivi aspettano che lo shuffle sia completo. Ridurre lo shuffle o scegliere il join giusto è spesso la chiave delle prestazioni. Un join broadcast evita lo shuffle della tabella grande; un sort-merge join partiziona e ordina entrambe le tabelle. Non esiste un join gratis: esiste il join che sposta meno dati.

Il broadcast join funziona quando una tabella è abbastanza piccola da essere copiata su ogni executor. È veloce perché evita lo shuffle della tabella grande, ma se la tabella "piccola" non è piccola, esaurisce la memoria e il job fallisce o degrada. La soglia dipende dalla configurazione e dai dati; non è un numero universale. Il sort-merge join è il default per tabelle grandi: entrambe vengono partizionate sulla chiave e ordinate. La scelta dipende dalle dimensioni, dalla distribuzione e dalla memoria disponibile.

Lo skew è lo sbilanciamento delle partizioni: poche chiavi contengono molti dati, quindi pochi task durano molto e gli altri aspettano. Si riconosce dalla Spark UI: task con tempi, input, spill e output molto diversi. Aumentare le partizioni non basta se la chiave è sbilanciata: il problema è la distribuzione, non il numero. Da Spark 3, con Adaptive Query Execution attiva, il primo livello è automatico: a runtime Spark riconosce le partizioni sbilanciate di un join e le divide. Se non basta, si passa agli strumenti manuali: broadcast se una tabella è piccola, salting della chiave, pre-partitioning o dividere il job. Il salting aumenta lo shuffle: non è gratis, va misurato.

La cache/persist è utile quando riusi lo stesso DataFrame più volte e il costo di ricalcolarlo è maggiore del costo di tenerlo in memoria. Non è gratis: occupa memoria, può causare eviction e può peggiorare un job che non riusa il dato. Se lo usi una volta, spesso è inutile. La scelta va misurata, non decisa per abitudine. Un cache messo "per sicurezza" è un modo per consumare memoria senza sapere perché.

Catalyst è l'ottimizzatore: trasforma il piano logico in fisico applicando regole come predicate pushdown, column pruning, join reordering e, in alcuni casi, ottimizzazione basata sui costi. Tungsten riguarda l'esecuzione efficiente: gestione della memoria e generazione di codice. In colloquio bastano i concetti: il piano viene ottimizzato prima di eseguire, e il modo in cui scrivi le trasformazioni influenza ciò che l'ottimizzatore può fare. La fault tolerance si basa sul lineage: se una partizione è persa, Spark la ricalcola dalle trasformazioni. Non è una transazione: per durabilità e consistenza servono storage e formati adeguati.

Perché Spark e non pandas, Polars o DuckDB? Pandas gira su una macchina e tiene i dati in memoria; Polars è single-node ma colonnare e veloce; DuckDB è un OLAP embedded, ottimo su Parquet e query analitiche. Spark distribuisce e scala oltre la memoria di una macchina, con fault tolerance. La scelta dipende dal volume e dal tipo di lavoro. Per dati piccoli, Spark è più lento e più complesso; per dati che non stanno su una macchina, gli altri non bastano. Non userei Spark per 100 MB, e non userei pandas per un dataset che non entra in RAM.

### Versione da 60 secondi

Spark è un motore distribuito: conviene quando i dati non stanno su una macchina o serve fault tolerance; per dati piccoli pandas, Polars o DuckDB sono più semplici. Il driver costruisce un DAG di trasformazioni lazy; un'azione genera un job, diviso in stage ai confini di uno shuffle, e ogni stage in task, uno per partizione. Lo shuffle è il costo principale: lo si riduce filtrando prima, scegliendo il join giusto (broadcast se una tabella è piccola) e controllando le partizioni. Lo skew si legge sulla UI; da Spark 3 con AQE il primo livello è automatico, poi salting o broadcast. `cache()` è lazy e serve solo se riusi il DataFrame.

### Follow-up

**D. Cos'è la lazy evaluation e perché esiste?**

R. Le trasformazioni non eseguono subito: registrano un piano. Solo un'azione scatena il calcolo. Questo permette a Catalyst di ottimizzare il piano prima di eseguirlo, di evitare lavoro inutile e di combinare trasformazioni che altrimenti sarebbero passi separati. È il motivo per cui puoi scrivere `filter` dopo `join` e Spark può spostare il filtro prima del join, se è corretto farlo. Senza lazy evaluation, ogni operazione sarebbe un job e l'ottimizzazione globale sarebbe impossibile.

Il rovescio è che il momento dell'errore è diverso dal momento in cui scrivi. Se una colonna non esiste, te ne accorgi all'azione, non alla riga di codice. Questo rende il debug meno immediato, ma è un compromesso consapevole: l'ottimizzazione vale il costo. Un altro effetto è che il codice sembra non fare nulla finché non arrivi all'azione, il che confonde chi è abituato a pandas.

La cosa pratica è: non chiamare azioni intermedie senza motivo — `count`, `show`, `collect` — perché spezzano il piano e materializzano. Ogni azione è un job. Se fai `count` per loggare e poi `write`, stai eseguendo due volte lo stesso lavoro. È un errore comune e costoso.

**D. Job, stage, task: spiegami con un esempio.**

R. Immagina di leggere due tabelle, fare un join su una chiave e poi un groupBy. L'azione finale — una `write` — genera un job. Il join richiede che le righe con la stessa chiave finiscano sullo stesso executor: è uno shuffle, quindi il job si divide in stage. Lo stage prima dello shuffle legge e prepara i dati; lo stage dopo esegue il join e il groupBy. Ogni stage è diviso in task, uno per partizione. Se hai 200 partizioni, hai 200 task per stage, distribuiti sugli executor.

Il numero di stage ti dice quanti shuffle ci sono: ogni shuffle è un confine. Se vedi dieci stage, probabilmente ci sono nove shuffle, e ognuno è un punto in cui i dati vengono scritti su disco e riletti. Non è sempre un problema — a volte lo shuffle è necessario — ma è il primo posto da guardare quando un job è lento. La UI mostra per ogni stage il tempo, l'input, lo shuffle read/write, lo spill e la distribuzione dei task.

La differenza tra job e stage è importante: un job è un'azione, uno stage è una fase senza shuffle. Un job può avere molti stage; uno stage può avere molti task. Se l'intervistatore ti chiede "quanti task", la risposta è "uno per partizione per stage", non "uno per job". È il tipo di dettaglio che distingue chi ha letto la UI da chi ha memorizzato le parole.

**D. `repartition` vs `coalesce`: quando usi quale?**

R. `repartition` ridistribuisce i dati e può aumentare o diminuire il numero di partizioni, con shuffle. Serve quando vuoi più parallelismo, quando devi partizionare su una colonna per i join successivi o quando il numero di partizioni è chiaramente sbagliato. `coalesce` riduce le partizioni senza shuffle, ma può creare partizioni sbilanciate perché combina quelle esistenti. Serve soprattutto per ridurre il numero di file in output, non per aumentare il parallelismo.

La scelta dipende dall'obiettivo. Se devo scrivere 200 file piccoli e ne voglio 10, `coalesce(10)` è più economico di `repartition(10)`. Se devo aumentare il parallelismo o partizionare su una chiave, `repartition` è necessario. Se uso `coalesce` per ridurre troppo, perdo parallelismo e posso creare un collo di bottiglia. Non c'è una regola universale: c'è l'obiettivo e il costo.

Un errore comune è `repartition(1)` per scrivere un file unico: funziona, ma concentra tutto su un task e può essere lentissimo. Un altro è `coalesce` dopo un filtro che ha ridotto molto i dati: il numero di partizioni può essere troppo alto per il volume rimasto, e `coalesce` aiuta. La cosa giusta è guardare il volume e la distribuzione, non applicare una formula.

**D. Perché lo shuffle è costoso e come lo riduci?**

R. Lo shuffle è costoso perché i dati devono essere ripartiti tra gli executor: serializzazione, scrittura su disco, trasferimento in rete, lettura e deserializzazione. È un'operazione che coinvolge I/O e rete, non solo CPU, e crea una barriera: gli stage successivi aspettano che sia completo. In più, può generare spill se i dati non stanno in memoria. Per questo è spesso il collo di bottiglia, più del calcolo.

Per ridurlo: filtrare prima dello shuffle, selezionare solo le colonne necessarie, usare broadcast join quando una tabella è piccola, evitare `distinct` e `repartition` non necessari, aggregare prima di un join se possibile, e scegliere la chiave giusta. A volte il modo migliore è cambiare il modello dei dati: una tabella pre-aggregata o partizionata può eliminare uno shuffle ricorrente.

Non si elimina tutto: join e groupBy richiedono shuffle per natura. L'obiettivo è spostare meno dati e farlo una volta. Un job con zero shuffle è spesso un job che non fa join o che è già partizionato bene; un job con dieci shuffle è un candidato a ottimizzazione. La UI dice dove, il piano dice perché.

**D. Broadcast join vs sort-merge: quando quale, e cosa succede se sbagli?**

R. Broadcast join quando una delle due tabelle è abbastanza piccola da essere copiata su ogni executor: eviti lo shuffle della tabella grande e il join diventa una mappa in memoria. Sort-merge quando entrambe sono grandi: Spark partiziona entrambe sulla chiave, le ordina e le unisce. Il sort-merge è il default per tabelle grandi, ma richiede shuffle e ordinamento; il broadcast è più veloce ma ha un limite di memoria.

Cosa succede se sbagli: se fai broadcast di una tabella troppo grande, ogni executor prova a tenerla in memoria e puoi avere OOM o spill, con un job più lento di prima. Se eviti il broadcast quando la tabella era piccola, paghi uno shuffle inutile. La soglia dipende dalla configurazione — memoria per executor, numero di executor — e dai dati. Non è un numero fisso; è una misura.

In PySpark puoi dare un hint o affidarti all'ottimizzatore, che a volte decide da solo. L'importante è capire il meccanismo: il broadcast elimina lo shuffle di una tabella, ma moltiplica la memoria per il numero di executor. Il sort-merge distribuisce il costo su tutti, ma scambia dati. Non c'è un join gratis.

**D. Cos'è lo skew, come lo riconosci e come lo risolvi?**

R. Lo skew è uno sbilanciamento nella distribuzione delle chiavi: poche chiavi contengono molti dati, quindi poche partizioni sono enormi e il resto è piccolo. I task che gestiscono le partizioni grandi durano molto, gli altri finiscono e aspettano. Il job nel complesso è limitato dal task più lento. Si riconosce dalla Spark UI: la distribuzione dei task per durata, input, spill e output è molto disomogenea, e pochi task dominano il tempo.

Le cause sono nei dati: una chiave molto frequente — un cliente grande, un valore "unknown", un prodotto di punta — o un join su una chiave con bassa cardinalità. Aumentare le partizioni non risolve: se la chiave è la stessa, i dati con quella chiave finiscono nella stessa partizione. Il problema è la distribuzione, non il numero.

I rimedi, in ordine: prima verifico se l'Adaptive Query Execution è attiva e se basta (`spark.sql.adaptive.skewJoin.enabled`), perché è attiva di default dalla 3.2 e Glue 4.0 è su Spark 3.3. Poi broadcast se una tabella è piccola; salting della chiave, aggiungendo un suffisso casuale per distribuire i dati e poi aggregando in un secondo passo; pre-partitioning o bucketing se il pattern è noto; dividere il job o trattare separatamente le chiavi calde. Il salting aumenta lo shuffle e la complessità: va usato quando il beneficio è misurabile. Non esiste una ricetta universale: esiste la diagnosi sulla UI e la prova su un campione.

**D. Cos'è l'Adaptive Query Execution e cosa cambia nella pratica?**

R. È l'ottimizzazione che Spark fa a runtime, dopo aver visto le statistiche reali degli shuffle, e non solo in fase di pianificazione. Fa tre cose: fonde le partizioni piccole dopo uno shuffle, per cui il valore di default di 200 partizioni di shuffle pesa meno; converte un sort-merge join in broadcast quando scopre che un lato è più piccolo del previsto; e divide le partizioni sbilanciate di un join. La conseguenza pratica è che prima di scrivere salting o hint a mano conviene guardare il piano e la UI per capire se AQE ha già risolto. I due default da conoscere sono 200 partizioni di shuffle e 10 MB come soglia del broadcast automatico: sono punti di partenza, non verità.

**D. Quando cache/persist ha senso?**

R. Quando riusi lo stesso DataFrame più volte e il costo di ricalcolarlo è maggiore del costo di tenerlo in memoria. Per esempio, se calcoli una base pulita e la usi per tre aggregazioni diverse, il cache evita di rileggere e ritrasformare tre volte. Se lo usi una volta, il cache è spesso inutile e occupa memoria che potrebbe servire ad altro. La domanda è: quante volte lo riuso e quanto costa ricalcolarlo?

Il cache non è gratis: occupa memoria, può causare eviction, e se il dataset è più grande della memoria disponibile finisce su disco, con un beneficio minore. Ricorda anche che `cache()` e `persist()` sono lazy: non fanno nulla finché non arriva un'azione, e il dato viene materializzato alla prima azione successiva. Per questo si usa spesso un `count()` per forzarlo, e per questo un cache senza una seconda lettura non porta niente. Va usato con consapevolezza, non "per sicurezza".

La scelta va misurata: prova senza cache, poi con cache, e guarda il tempo e la memoria. Se il beneficio non è chiaro, togli il cache. Un job con troppi cache è un job che ha paura di ricalcolare, e spesso paga più di quanto risparmia.

**D. Come funziona la fault tolerance?**

R. Spark ricostruisce le partizioni perse usando il lineage delle trasformazioni: sa da quali dati e da quali operazioni è stata prodotta ogni partizione, e se un executor muore, il driver può ricalcolare i task persi su un altro executor. È una fault tolerance a livello di dato, non una transazione: se un task viene rieseguito, può essere rieseguito più volte, e questo è importante per gli effetti collaterali.

La conseguenza pratica è che le trasformazioni dovrebbero essere deterministiche e senza effetti collaterali. Se scrivi su un database dentro una `foreachPartition`, un retry può duplicare le scritture. Se scrivi su file, un output parziale può restare. Spark ti dà la tolleranza ai guasti del calcolo, non l'atomicità dell'output: quella devi progettarla tu, con overwrite controllato, scrittura temporanea o un formato transazionale.

La fault tolerance non è "non perdo dati": è "posso ricalcolare". La differenza è importante quando il costo del ricalcolo è alto o quando l'output ha effetti collaterali. In un job ETL, la prima cosa è rendere l'output idempotente; la seconda è il lineage.

**D. Spark vs pandas, Polars, DuckDB: come scegli?**

R. Dipende dal volume, dal tipo di lavoro e dall'ambiente. Pandas è semplice e ha un ecosistema enorme, ma è single-node e in memoria: va bene finché il dataset ci sta. Polars è single-node ma colonnare e veloce, con una API che spinge verso operazioni lazy; è una buona scelta per dati che stanno su una macchina e per chi vuole più performance di pandas. DuckDB è un OLAP embedded, ottimo per query SQL su Parquet e per analisi single-node; è quello che usiamo nello store orchestrator.

Spark conviene quando i dati non stanno su una macchina, quando serve distribuire il calcolo, quando vuoi fault tolerance su un cluster o quando l'ecosistema della piattaforma è Spark. Non conviene per 100 MB: l'overhead di scheduling, serializzazione e shuffle è maggiore del lavoro. La scelta non è "Spark è più potente": è "Spark è più adatto a un certo ordine di grandezza".

In un colloquio, la risposta matura è: "dipende dal volume e dal tipo di query; per dati piccoli userei pandas o Polars, per query analitiche su Parquet in locale DuckDB, per dati distribuiti Spark". E poi: "come misurerei? Con un benchmark sul caso reale, non con un'opinione".

**D. Come testeresti una pipeline Spark e come debugghi un job lento?**

R. Per testare: dati piccoli e deterministici, trasformazioni testate in isolamento, schema di input e output, casi limite, idempotenza. Poi un'integrazione su un campione e controlli di qualità. Non testerei l'intero cluster: testerei la logica e il contratto. Un test di regressione su un output atteso è molto utile, perché cattura i cambiamenti di semantica che i test unitari non vedono.

Per il debug: guardo la Spark UI, stage per stage. Cerco il collo di bottiglia: uno stage con shuffle enorme, task con spill, skew nella distribuzione, un join che non è broadcast quando potrebbe, un cache inutile. Poi provo una modifica alla volta e misuro. Non ottimizzo a caso: ogni modifica deve avere un'ipotesi e una misura.

La cosa che non farei è aggiungere `repartition` o `cache` per riflesso. Ogni modifica cambia il piano e può peggiorare. La UI e il piano fisico sono la fonte di verità, non l'intuizione.

**D. Cosa hai fatto tu con Spark?**

R. Ho scritto job PySpark su Glue, tradotto trasformazioni SAS e lavorato sui dati. Non ho progettato il cluster, non ho configurato Glue e non ho fatto tuning formale. So spiegare il modello di esecuzione, le partizioni, lo shuffle, i join e lo skew, e so dire cosa guarderei per ottimizzare. La differenza tra "so come funziona" e "l'ho ottimizzato in produzione" è una differenza che dichiaro.

Questa onestà è importante perché Spark è un tema dove è facile raccontare storie. Io ho lavorato su job reali, ma in un contesto junior e senza il repository. So difendere i concetti e il mio ruolo; non rivendico un'esperienza di performance engineering che non ho.

### Prova di padronanza

- So dire quando Spark conviene e quando no: dati oltre una macchina, distribuzione e fault tolerance contro overhead di cluster; e so citare pandas, Polars e DuckDB come alternative single-node.
- So descrivere driver, executor, cluster manager e `SparkSession` senza confonderli, e so collocare Glue come Spark serverless in cui scelgo tipo e numero di worker ma non gestisco i nodi.
- So spiegare la lazy evaluation e il suo costo: ottimizzazione globale del piano, ma errori che emergono all'azione e azioni intermedie che spezzano il piano; e che `cache()` è lazy.
- So spiegare job, stage e task: azione → job, shuffle → confine di stage, partizione → task, con un esempio di join più groupBy.
- So distinguere `repartition` e `coalesce`: shuffle e aumento o diminuzione del parallelismo contro riduzione senza shuffle e rischio di sbilanciamento.
- So spiegare lo shuffle come costo di rete, disco e serializzazione, e collegarlo a filtri anticipati, selezione colonne, broadcast e pre-aggregazione.
- So scegliere tra broadcast e sort-merge in base alle dimensioni e alla memoria, e so dire cosa succede se il broadcast è troppo grande: OOM o spill.
- So diagnosticare lo skew dalla UI e proporre rimedi con il loro costo, partendo da AQE (`skewJoin`) e poi broadcast, salting, pre-partitioning; e so dire che aumentare le partizioni non basta.
- So citare i default di riferimento — 200 partizioni di shuffle, 10 MB per il broadcast automatico — e dire che sono punti di partenza.
- So spiegare Catalyst, Tungsten e la fault tolerance basata sul lineage, e so distinguere "ricalcolo" da "transazione".

### Cosa non dire

- "Spark è sempre più veloce": su dati piccoli è più lento e più complesso di pandas o DuckDB.
- "Cache sempre": occupa memoria, può causare eviction e peggiorare il job.
- Confondere partizione Spark con partizione Hive o con un file: sono concetti diversi, anche se collegati.
- Ignorare shuffle e skew e parlare solo di "big data": sono i due punti che determinano le prestazioni.
- Confondere driver ed executor, o dire "Spark = Hadoop": Spark è un motore, Hadoop è un ecosistema.
- Non sapere che Glue è Spark serverless e che il cluster non l'hai gestito tu.
- Dire "ho ottimizzato" senza poter dire cosa hai misurato e quale ipotesi hai verificato.
- Raccontare il progetto SAS come se avessi progettato la pipeline o il cluster.

### Se non lo so

Non ho progettato il cluster né configurato Glue, e non ho fatto tuning formale su un job di produzione: so spiegare il modello e cosa guarderei, ma non ti racconto una ottimizzazione che non ho misurato. Se mi chiedi un valore di soglia per il broadcast o un numero di partizioni, dico che dipende dalla configurazione e dai dati, e che andrebbe misurato. Se mi chiedi un dettaglio del progetto SAS che non ricordo, lo dico e riporto il discorso sui concetti che so difendere.

---

## 1.9 — Repricing: estendibilità, limiti, flussi post-run

### Risposta principale (3–5 minuti)

L'estendibilità del repricing si gioca su due livelli, e vanno tenuti separati. Il primo è la composizione: se aggiungo una trasformazione, aggiungo uno step e lo inserisco nella pipeline, senza toccare gli altri. Questo livello è sano, perché il contratto è chiaro — snapshot in ingresso, snapshot in uscita — e il codice che esegue non deve conoscere il dettaglio. Il secondo è la configurazione: il runner contiene molti parametri e gli step leggono da lì. Questo livello è fragile, perché è comodo oggi e accoppiato domani. La maggior parte dei problemi di estendibilità non è nella composizione, è nella configurazione.

Per aggiungere uno step, la domanda giusta non è "dove lo metto", è "qual è la sua responsabilità e qual è il suo contratto". Uno step dovrebbe dichiarare cosa legge dallo snapshot, cosa scrive, quali invarianti rispetta, quali errori può lanciare e se è idempotente. Poi si inserisce nella composizione giusta, con una factory o con la pipeline che lo costruisce. Se lo step ha bisogno di un parametro, il parametro va dichiarato e documentato, non letto dal runner in modo implicito. Il criterio è che la pipeline resti leggibile a chi arriva dopo.

Per aggiungere un parametro, oggi lo aggiungo al runner e lo leggo dove serve. È veloce, ma aumenta la superficie del runner e rende il contratto implicito. La direzione giusta è un oggetto di configurazione per area — prezzo, clustering, output — passato solo agli step che ne hanno bisogno, con un default esplicito e una validazione. La differenza tra un parametro e uno step è questa: il parametro configura la stessa responsabilità, lo step aggiunge una responsabilità nuova. Se ogni eccezione diventa un parametro, il modello si dissolve.

Il confine non è sempre netto. Una nuova soglia è un parametro; una nuova regola che cambia il flusso è uno step; una nuova politica di caricamento è una strategia. La domanda da farsi è: sto cambiando un valore o sto cambiando il comportamento? Se cambio il comportamento in modo strutturale, e lo faccio con un parametro, sto nascondendo una decisione di design dentro un `if`. Il parametro è comodo, ma non è gratis: ogni parametro è una configurazione che qualcuno deve capire e che può essere sbagliata.

Il runner è il punto di accoppiamento. Non è un God Object nel senso che fa tutto — non calcola, non salva, non decide il flusso — ma è un contenitore con una superficie ampia: molti parametri, flag e stato di esecuzione. Ogni step può leggere qualsiasi cosa, e il contratto è implicito. La soluzione non è "togliere tutto", è separare tre cose: configurazione, stato di esecuzione e dipendenze. Un oggetto di configurazione per area, un contesto di esecuzione più piccolo, e dipendenze passate esplicitamente dove servono.

I flussi post-run sono un'altra fonte di complessità. Dopo il calcolo principale ci sono salvataggio degli snapshot intermedi, clustering dei prezzi, modifica manuale dei cluster, merge dei cluster e costruzione della tabella finale. Sono parte del prodotto, ma non del core dell'ottimizzazione. Il rischio è mescolare calcolo e presentazione dei risultati nella stessa pipeline: il confine si perde, i test diventano difficili e un cambiamento nell'output può rompere il calcolo. Separerei il core dai flussi di output, con contratti e test dedicati.

Le duplicazioni nelle funzioni che preparano le pipeline sono debito tecnico. Le cinque funzioni di avvio — ottimizzazione, binning, modifica manuale, merge dei cluster e annullamento dell'ultimo step — ripetono la scelta delle strategie con un `if` sul `RunMode`, il caricamento della configurazione e il setup del runner. All'inizio sembra innocuo; dopo un po', due pipeline divergono senza che nessuno lo voglia: una usa una strategia diversa, l'altra dimentica un parametro. Il primo punto da estrarre è la costruzione delle strategie in una sola funzione. Non è un refactor enorme, ma va fatto con test di regressione, perché tocca il punto in cui le pipeline vengono costruite.

I due difetti sono esempi di contratto fragile, e vanno descritti per quello che sono. Il primo: una sotto-pipeline ridefinisce il metodo template per misurare marginalità e colonne prima e dopo, e chiama direttamente l'implementazione interna, saltando il log del nome dello step e il cleanup opzionale. È una scelta deliberata ma fragile. Il secondo, latente: un passo ricorrente registra un passo di preparazione opzionale anche quando è `None`; oggi tutte le composizioni la forniscono, quindi non si manifesta. Sono piccoli, ma mostrano due cose: che il template va protetto con un hook, e che un tipo opzionale va gestito come tale. La correzione è banale; il test che manca no.

Il refactoring che proporrei è incrementale. Primo: test di caratterizzazione su un caso reale, per avere un output di riferimento. Secondo: estrarre un gruppo di parametri coerente in un oggetto, mantenendo il runner come facciata. Terzo: estrarre il setup comune delle pipeline. Quarto: correggere i due contratti e aggiungere i test per il caso `None` e per il logging/cleanup. Quinto: separare il core dai flussi post-run. Non mescolerei tutto in un commit: ogni passo deve essere verificabile e reversibile.

Il costo del refactoring è il tempo e il rischio di toccare codice che funziona. Il beneficio è che gli step diventano testabili con un oggetto piccolo, che le pipeline non divergono e che i difetti di contratto non si ripresentano. Il rischio opposto è il big rewrite: riscrivere la pipeline perché "è più pulita" è il modo per perdere comportamenti non documentati. La pipeline funziona; va migliorata, non rifatta. Questa è la distinzione che porterei in un colloquio.

Il mio ruolo: ho esteso la pipeline, ho lavorato su step, condizioni e strategie, ho letto il codice abbastanza da trovare i due difetti. Non ho progettato l'architettura iniziale e non ho fatto il refactoring che descrivo. Il debito tecnico che lascio è quello che ho trovato: runner troppo permissivo, `run_*` duplicate, due contratti violati, nessun test. Non è un giudizio sul lavoro degli altri: è la lista di ciò che migliorerei se avessi il tempo e la rete di sicurezza.

### Versione da 60 secondi

L'estendibilità del repricing ha due livelli: la composizione degli step, che è sana, e la configurazione nel runner, che è fragile. Aggiungo uno step se cambia una responsabilità, un parametro se cambia un valore. I limiti che conosco: un runner con circa venti parametri e contratto implicito, la scelta delle strategie ripetuta in cinque funzioni di avvio, un template scavalcato da una sotto-pipeline, un `Optional` non gestito, nessun test. Il refactoring che farei è incrementale: caratterizzazione su un caso reale, poi estrazione della costruzione delle strategie, poi della configurazione, mai un rewrite.

### Follow-up

**D. Come aggiungi uno step, in concreto?**

R. Prima definisco la responsabilità: cosa deve fare, su quale parte dello snapshot, con quali input e output. Poi scrivo il contratto: cosa legge, cosa scrive, quali invarianti rispetta, quali errori può lanciare, se è idempotente. Poi implemento il passo e lo inserisco nella composizione, usando la factory o la pipeline che già costruisce gli step. Infine lo testo con un input piccolo e un output atteso, prima di provarlo sulla pipeline completa.

Il punto delicato è l'ordine. Uno step non è indipendente dal contesto: dipende da ciò che gli step precedenti hanno prodotto e da ciò che i successivi si aspettano. Se lo inserisco nel punto sbagliato, il contratto può reggere ma il risultato cambiare. Per questo, prima di aggiungerlo, mi chiedo: quali invarianti devono essere vere prima e dopo? Se non lo so, non ho capito lo step.

Un esempio di criterio: se lo step arricchisce lo snapshot con una metrica, deve dichiarare se la metrica è opzionale o obbligatoria per gli step successivi. Se è obbligatoria e manca, il fallimento deve essere esplicito, non un `None` che esplode tre step dopo. È lo stesso problema del passo ricorrente che registra una preparazione `None`: il contratto dice una cosa, il codice ne assume un'altra.

**D. Come aggiungi un parametro, e come eviti una regressione silenziosa?**

R. Oggi lo aggiungo al runner, lo documento e lo leggo dove serve. È veloce, ma aumenta la superficie e rende il contratto implicito. La direzione che preferisco è un oggetto di configurazione per area, con un default esplicito e una validazione. Se il parametro è nuovo, lo tratto come una modifica al contratto: default, validazione, un test che copra il caso in cui manca, e una nota su cosa cambia nell'output.

La regressione silenziosa è il rischio principale. Un default sbagliato cambia il comportamento di tutte le run senza che nessuno se ne accorga, e il risultato sembra plausibile. Per evitarlo: il default deve essere esplicito e discusso, non "zero" o "None" per comodità; il parametro deve essere validato; e il test di regressione deve coprire il caso in cui il parametro è assente. Se il parametro cambia un numero, un test sul valore atteso lo scopre; se cambia una politica, serve un test sul comportamento.

Un'altra cosa che farei è separare i parametri per area. Un unico oggetto con cento campi è solo il runner con un altro nome. La granularità giusta è quella che permette a uno step di ricevere solo ciò che usa, e a un test di costruire solo ciò che serve. Non è una questione di eleganza: è una questione di superficie di errore.

**D. Quando un parametro è meglio di uno step, e quando è il contrario?**

R. Un parametro è meglio quando cambia un valore o una politica senza cambiare il flusso: una soglia, un fattore, un limite, un numero di iterazioni. Uno step è meglio quando aggiunge una responsabilità o cambia il comportamento in modo strutturale: una nuova trasformazione, una nuova regola, un nuovo calcolo. La domanda è: sto configurando la stessa cosa o sto aggiungendo una cosa nuova? Se la risposta è "sto aggiungendo", un parametro nasconde la decisione.

Il rischio di usare un parametro per tutto è che il codice diventa una serie di `if` configurabili, e nessuno sa più quale combinazione è valida. Il rischio di usare uno step per tutto è che la pipeline esplode in decine di passi, e l'ordine diventa difficile da capire. Il confine è il comportamento: se il flusso cambia in modo che un passo possa non essere eseguito o essere eseguito diversamente, è uno step o una condizione; se il passo è sempre lo stesso e cambia solo un numero, è un parametro.

C'è anche un criterio di testabilità: un parametro si testa con due valori, uno step con un input e un output. Se per testare la variazione devo costruire mezzo runner, forse è uno step. Se per testarla mi basta cambiare un numero, è un parametro. La forma giusta è quella che rende il test più semplice e il codice più leggibile.

**D. Il runner è un God Object? Come lo refattorizzeresti senza rompere tutto?**

R. È un contenitore con una superficie ampia: molti parametri, flag e stato. Non è un God Object che fa tutto, ma è un punto di accoppiamento: ogni step può leggere qualsiasi cosa e il contratto è implicito. La diagnosi è questa, non "il file è lungo". Il problema non è la quantità in sé, è che gli step dipendono da lui per cose che non sempre riguardano il loro compito.

Il refactoring che farei è incrementale. Primo: test di caratterizzazione su un caso reale, per avere un output di riferimento. Secondo: estrarre un gruppo di parametri coerente — per esempio la configurazione del prezzo — in un oggetto esplicito, mantenendo il runner come facciata per compatibilità. Terzo: migrare gli step che usano quel gruppo, uno alla volta. Quarto: rimuovere i parametri vecchi dal runner solo quando nessuno li legge. Non mescolerei l'estrazione della configurazione con la correzione del template: sono due cambiamenti diversi e vanno verificati separatamente.

Il costo è il tempo e il rischio di toccare codice che funziona. Il beneficio è che gli step diventano testabili con un oggetto piccolo e che il contratto diventa leggibile. Se non avessi test, il primo passo sarebbe salvare un output di riferimento e confrontarlo, non "fare il refactor e sperare". Il refactoring senza rete è solo movimento.

**D. Cosa sono i flussi post-run e perché li separeresti?**

R. Sono le cose che succedono dopo il calcolo principale: salvataggio degli snapshot intermedi, clustering dei prezzi, modifica manuale dei cluster, merge dei cluster e costruzione della tabella finale. Sono parte del prodotto, ma non del core dell'ottimizzazione. Il core produce i prezzi; il post-run li organizza, li rende modificabili e li presenta. Sono due responsabilità diverse, con cicli di vita diversi: il core è un calcolo, il post-run è un flusso di output e di interazione.

Il rischio di mescolarli è che un cambiamento nell'output rompe il calcolo, o che un test del core deve costruire anche il clustering. In più, la modifica manuale introduce uno stato che il core non dovrebbe conoscere: se l'utente cambia un cluster, il risultato non è più solo il frutto dell'algoritmo. Separare aiuta a capire cosa è calcolo e cosa è presentazione, e a testare l'uno senza l'altro.

Nel progetto ho toccato i flussi post-run, ma non li ho progettati. So dire perché andrebbero separati e quali confini metterei: il core produce un output con un contratto, il post-run lo consuma e produce la tabella finale. Non ti racconto una separazione che non è stata fatta: ti racconto cosa farei.

**D. Dove sono le duplicazioni e come le estrarresti senza rompere?**

R. Sono nelle cinque funzioni di avvio: ripetono la scelta delle strategie con un `if` sul `RunMode`, il caricamento della configurazione e il setup del runner. Il rischio non è la lunghezza: è la divergenza. Due pipeline che dovrebbero essere uguali possono differire per una strategia o un parametro, e nessuno se ne accorge finché un risultato non torna. Estrarre una funzione che costruisce le strategie e il runner riduce la duplicazione e rende esplicite le differenze.

L'estrazione va fatta con un test di caratterizzazione: prima salvo l'output attuale, poi estraggo, poi confronto. Se le pipeline hanno differenze legittime, la funzione comune deve avere parametri per quelle differenze, non un `if` che le nasconde. Il criterio è: la funzione comune deve rendere evidente cosa cambia tra le pipeline, non nasconderlo.

Non farei un'estrazione enorme in un colpo. Comincerei dalla parte più ripetuta — per esempio la scelta delle strategie — e lascerei il resto. Il refactoring incrementale è meno elegante e più sicuro. L'eleganza che rompe il comportamento non è un miglioramento.

**D. I due difetti: qual è la causa e come li correggeresti?**

R. Il primo è una forzatura del Template Method: una sotto-pipeline ridefinisce il metodo pubblico per misurare marginalità e colonne prima e dopo, e chiama direttamente l'implementazione interna, saltando il log del nome dello step e il cleanup opzionale. La causa è che il metodo interno era accessibile e conveniente, e il template non prevede un punto di estensione per "prima e dopo". La correzione non è vietare l'override: è dare al template un hook, cioè un metodo che le sottoclassi possono implementare, oppure usare un decorator, così lo scheletro resta intero. Poi un test che verifichi che logging e cleanup avvengono anche per le sotto-pipeline.

Il secondo è un `Optional` trattato come obbligatorio: un passo ricorrente registra una preparazione anche quando è `None`, quindi il contratto ammette l'assenza ma il codice la assume. È latente: oggi tutte le composizioni forniscono la preparazione. La correzione è registrare solo se esiste, oppure rendere obbligatorio il contratto e validarlo all'ingresso. Poi un test sul caso "preparazione assente", che è esattamente quello che nessuno scrive finché non serve.

La lezione generale è che i contratti vanno protetti e testati. Un tipo opzionale non è un dettaglio: è una promessa. Un metodo template non è una convenzione: è un punto di estensione. Se il codice può violarli, prima o poi lo farà. Il test non serve a dimostrare che oggi funziona: serve a impedire che domani qualcuno lo rompa.

**D. Come testeresti l'estendibilità?**

R. Con test di composizione: una pipeline con due step, una condizione vera e una falsa, un ciclo che termina e uno che non parte, una sotto-pipeline annidata, una preparazione assente. Verifico l'ordine, lo stato finale, le invarianti e che il logging e il cleanup siano stati eseguiti. Non testo l'implementazione interna: testo il contratto e il comportamento osservabile.

Poi un test di regressione su un caso reale: stesso input, stesso output atteso. Serve a catturare i cambiamenti di semantica che i test unitari non vedono. E un test di idempotenza: rieseguire la pipeline sullo stesso input non deve duplicare l'output. Se il passo salva file, il test deve verificare che il rerun non crei duplicati.

L'estendibilità non si testa con un test solo: si testa con una suite che rende sicuro aggiungere uno step. Se aggiungere uno step richiede di modificare dieci test, l'architettura non è estendibile. Se richiede di aggiungere un test e poco altro, lo è. Questo è il criterio che userei.

**D. Cosa non faresti, e quale debito lasceresti?**

R. Non farei un big rewrite. Non cambierei l'ordine degli step senza un test. Non introdurrei un framework di orchestrazione per un problema che è ancora una pipeline. Non aggiungerei un parametro per ogni eccezione: a un certo punto serve un modello. E non separerei il core dal post-run in un colpo solo, senza una rete di sicurezza.

Il debito che lascerei è quello che ho trovato: runner troppo permissivo, `run_*` duplicate, due contratti violati, nessun test. Non è un debito che ho creato io, ma è un debito che riconoscerei se fossi responsabile del progetto. La cosa onesta è dire: "questi sono i punti, questo è l'ordine in cui li affronterei, questo è il costo". Un colloquio non richiede che tu abbia già risolto tutto; richiede che tu sappia cosa è rotto e perché.

### Prova di padronanza

- So separare i due livelli di estendibilità: composizione degli step, che è sana, e configurazione nel runner, che è fragile.
- So definire il contratto di uno step: cosa legge, cosa scrive, invarianti, errori, idempotenza; e so dire perché l'ordine conta.
- So distinguere parametro e step con un criterio: configura un valore contro aggiunge una responsabilità; e so dire cosa succede se sbaglio la scelta.
- So descrivere il runner come punto di accoppiamento, non come "file lungo", e so separare configurazione, stato di esecuzione e dipendenze.
- So elencare i flussi post-run — snapshot, clustering, modifica manuale, merge, tabella finale — e spiegare perché separarli dal core.
- So individuare la duplicazione nelle cinque funzioni di avvio e proporre di estrarre per prima la costruzione delle strategie, con un test di caratterizzazione, senza nascondere le differenze legittime.
- So spiegare i due difetti per quello che sono: un template scavalcato di proposito (serve un hook) e un `Optional` trattato come obbligatorio, latente, con i test che mancano.
- So proporre un refactoring in cinque passi, con un ordine che non mescola cambiamenti diversi e con un criterio di stop.
- So dire il costo del refactoring e il rischio del big rewrite, senza vendere il refactoring come una cosa gratis.
- So dichiarare il debito che lascerei e ciò che non ho fatto: architettura iniziale, refactoring, test.

### Cosa non dire

- "Basta aggiungere un parametro" per ogni cambiamento: è il modo per far diventare il runner un contenitore di tutto.
- Non riconoscere il runner come punto di accoppiamento: è la diagnosi che distingue chi ha letto il codice da chi ha letto un elenco di pattern.
- Ignorare i flussi post-run o trattarli come parte del core: sono prodotto, non ottimizzazione.
- Dire "ci sono i test" quando non ci sono: nei progetti di lavoro non c'era una cultura del testing automatico.
- Proporre un refactor enorme senza test di caratterizzazione e senza dire il costo.
- Fare un big rewrite perché "è più pulito": il rischio di perdere comportamenti non documentati è reale.
- Confondere "estendibile" con "configurabile": un parametro in più non è un'estensione, è una superficie in più.
- Dire "il template method è solo una convenzione": è un contratto, e il difetto della sotto-pipeline lo dimostra.

### Se non lo so

L'architettura iniziale è del senior: io l'ho estesa, e se mi chiedi perché una scelta è stata presa prima che arrivassi, posso ricostruire il contesto ma non attribuirmela. Non ho fatto il refactoring che descrivo e non ho scritto i test che mancano, quindi te lo racconto come piano, non come realizzazione. Se mi chiedi un dettaglio di una `run_*` che non ricordo, preferisco dirlo e ragionare sul contratto, invece di inventare una funzione.

---

## 1.10 — Store orchestrator: DuckDB, cache da S3, Streamlit

### Risposta principale (3–5 minuti)

DuckDB è un motore OLAP embedded: non è un server, è una libreria che gira nel processo. Questo cambia il modello di deployment. Non devi avviare un database, non devi gestire una connessione di rete, non hai un servizio separato. Legge Parquet, CSV e DataFrame, e usa un motore colonnare e vettoriale ottimizzato per analytics. Per una dashboard su dati Parquet, è una scelta molto pragmatica: hai la potenza di un OLAP senza l'infrastruttura di un OLAP.

La distinzione tra view e tabella è importante. Una view su Parquet non copia i dati: registra una query che legge il file. Se il file cambia, la view vede la nuova versione alla prossima query. Una tabella materializzata copia i dati in DuckDB: è più veloce da leggere, ma resta ferma finché non la aggiorni. Uso la view per sorgenti che cambiano, la tabella per risultati stabili e costosi da ricalcolare. La scelta è un trade-off tra freschezza e prestazioni, non una preferenza.

La concorrenza è gestita con una connessione condivisa e un cursore per query. La registrazione delle sorgenti modifica il catalogo, quindi è protetta da un lock; le query read-only possono girare in parallelo. È pensato per un singleton di processo, non per un cluster. Se la concorrenza crescesse, valuterei un motore diverso o un pool. La semplicità ha un prezzo: non è un database multiutente con isolamento e priorità.

La cache ha quattro livelli, e conviene saperli nell'ordine. Il primo è il repository, un singleton di processo con l'engine DuckDB (`cache_resource`). Il secondo è `cache_data`, con TTL di un'ora, per le opzioni dei filtri e i min/max. Il terzo è lo specchio locale dei Parquet: un job giornaliero li scrive su S3, la dashboard li copia in locale all'avvio confrontando dimensione ed ETag, scrive prima su un file temporaneo e poi lo rinomina, e un thread in background ripete la sincronizzazione ogni tre ore. Il quarto è la cache per sessione nel ViewManager, con chiave la fingerprint dello stato. Nella libreria esiste anche un motore Athena con cache dei risultati su S3, basata su un hash della query e su un TTL, ma non l'ho trovato usato nella dashboard: la dashboard legge Parquet locale.

Streamlit riesegue lo script a ogni interazione. Questo è il vincolo che giustifica la separazione: lo stato che deve sopravvivere sta in `session_state` o in un manager; gli oggetti costosi stanno in `cache_resource`; le query sono cacheate. Se la View ricrea tutto a ogni rerun, perdi stato e cache. La separazione non è un'astrazione accademica: è una necessità del modello di esecuzione.

I limiti sono chiari. DuckDB è single-node: la CPU, la memoria e il disco sono quelli della macchina. Non è transazionale nel senso di un database multiutente, e non è pensato per scritture concorrenti. Va bene per il volume e la concorrenza attuali, non per un cluster. Se il dato crescesse o gli utenti aumentassero, servirebbe un motore diverso. La scelta è corretta per il contesto, non universale.

I failure mode sono quelli del disallineamento tra file, cache e view. Se un file Parquet viene sostituito mentre una query lo legge, la sostituzione è un rename atomico su filesystem locale: la query in corso resta sul file già aperto, la successiva vede il nuovo. Se una tabella materializzata non viene rinfrescata, vedi dati vecchi e sembra un bug di dati: qui il rinfresco è agganciato alla sincronizzazione, che ricostruisce le tabelle CTAS dopo ogni giro. Se il TTL di `cache_data` è troppo lungo, l'utente vede un'opzione di filtro vecchia; se due sessioni registrano una sorgente insieme, il lock serializza, ma una query pesante può bloccare le altre.

Il confronto con Athena è sul modello di deployment. Athena è serverless e ottima per query su S3, ma ha latenza di avvio e costi per dati scansionati. DuckDB è in-process, legge Parquet locale e risponde subito: per la dashboard è più adatto. Athena resta utile per dati più grandi o condivisi. La scelta dipende dal volume, dalla latenza e da chi deve interrogare. Non è "DuckDB è meglio": è "DuckDB è più adatto a una dashboard embedded".

Il mio ruolo: conosco il codice e so spiegare le scelte, ma non ho progettato l'intera architettura della cache e del repository. Il presentation layer l'ho fatto a metà con lo sviluppatore del cliente. So dire cosa migliorerei — metriche di hit e miss, un modo per forzare la sincronizzazione senza aspettare le tre ore, test sul comportamento di refresh dei Parquet — ma non ti racconto che l'ho fatto. La distinzione conta, perché questo è un tema dove è facile sembrare più esperto di quanto si sia.

### Versione da 60 secondi

DuckDB è un motore OLAP embedded: gira nel processo e legge Parquet locale come viste. I dati li scrive un job giornaliero su S3; la dashboard li sincronizza in locale per ETag all'avvio e ogni tre ore, e ricostruisce le tabelle materializzate dopo ogni sincronizzazione. Poi ci sono la cache per sessione nel manager, con fingerprint, e `cache_data` con TTL per i filtri. Limiti: single-node, nessuna transazionalità multiutente, dati vecchi fino a tre ore. Con più utenti o dati più grandi servirebbe un altro motore. Non ho progettato io la cache; so spiegarla e dire cosa misurerei.

### Follow-up

**D. Perché DuckDB e non Athena per tutto?**

R. Athena è serverless e ottima per query su S3, ma ha latenza di avvio e costi per dati scansionati. Per una dashboard che risponde a ogni interazione, la latenza di avvio è un problema: l'utente cambia un filtro e aspetta. DuckDB è in-process, legge Parquet locale e risponde subito, senza costi per query. Per il pattern della dashboard, è più adatto.

Athena resta utile per dati più grandi, per query condivise o per esplorazione su volumi che non stanno sulla macchina. La scelta non è "uno dei due": è "DuckDB per la dashboard, Athena per l'accesso condiviso e i dati che non stanno in locale". Se il volume crescesse, DuckDB potrebbe non bastare, e allora Athena o un warehouse tornerebbero in gioco.

Il costo di DuckDB è la gestione del processo e della cache: non è un servizio, quindi devi occuparti tu di refresh, concorrenza e memoria. Athena ti dà un servizio gestito, ma con latenza e costo per query. Il trade-off è tra controllo e comodità. A questa scala, il controllo vale la pena.

**D. View vs tabella: quando usi quale?**

R. Uso la view per sorgenti che cambiano: il file Parquet è la verità, e la view lo rilegge a ogni query. È più fresca, ma paga il costo di lettura ogni volta. Uso la tabella materializzata per risultati stabili e costosi da ricalcolare: copio i dati in DuckDB, la lettura è veloce, ma resta ferma finché non la aggiorno. La scelta è tra freschezza e prestazioni.

Il rischio della tabella è la staleness: se i dati cambiano e nessuno la aggiorna, l'utente vede un risultato vecchio e pensa che sia un bug. Per questo una tabella materializzata ha bisogno di una politica di refresh esplicita, e nel progetto ce l'ha: dopo ogni sincronizzazione le tabelle CTAS vengono ricostruite. La view non ha questo problema, ma può essere lenta su file grandi o query ripetute.

Un criterio pratico: se la sorgente cambia spesso e la query è leggera, view; se la sorgente è stabile e la query è pesante, tabella; se è entrambe, view più cache. Non c'è una regola universale, ma il criterio è chiaro. La cosa da non fare è materializzare "per sicurezza" senza sapere quando aggiornare.

**D. Cosa succede se il file Parquet cambia mentre una query lo legge?**

R. Nello store orchestrator il file letto da DuckDB è locale, non su S3: la sincronizzazione lo scarica in un file temporaneo e lo rinomina, quindi la sostituzione è atomica. Una query già in corso continua sul file che aveva aperto; la query successiva legge la nuova versione. Non vedo quindi un risultato a metà su un singolo file. Il rischio vero è un altro: una query che legge più file, per esempio più anni, mentre la sincronizzazione ne sostituisce alcuni può vedere versioni diverse dello stesso dataset. S3 è fortemente consistente sul singolo oggetto ma non dà snapshot transazionali tra più file.

La strategia corretta è sostituire in modo atomico, come qui, e dove serve uno snapshot coerente tra più file usare una convenzione di versione o un table format con log transazionale, come Iceberg o Delta. Non è un dettaglio: è la differenza tra un risultato affidabile e uno che ogni tanto è strano.

Con una view la query rilegge il file a ogni esecuzione, quindi vede subito la nuova versione; con una tabella materializzata il rischio è la staleness, non la lettura parziale: vede i dati dell'ultima ricostruzione, che qui avviene dopo ogni sincronizzazione. Sono due problemi diversi e vanno gestiti diversamente.

**D. Come gestisci la concorrenza tra sessioni Streamlit?**

R. Ogni sessione ha il suo stato e il suo ViewManager, ma condivide il DataRepository e l'engine DuckDB. La registrazione delle sorgenti è protetta da un lock perché modifica il catalogo; le query read-only possono girare in parallelo con un cursore per query. È un modello pensato per un singleton di processo, non per un cluster. Con poche sessioni va bene; con molte, la concorrenza sull'engine e sulla memoria diventa un limite.

Il rischio è che una query pesante blocchi le altre o consumi memoria condivisa. DuckDB è veloce, ma non è un database multiutente con isolamento e priorità. Se il numero di utenti crescesse, valuterei un motore diverso o un'architettura con query asincrone e risultati materializzati. A questa scala, la semplicità vale più della scalabilità teorica.

La distinzione tra stato di sessione e stato condiviso è la chiave: se confondi i due, due utenti si vedono i filtri a vicenda. È un bug che sembra un problema di dati e invece è un problema di scope. Il lock sulla registrazione è la parte facile; il vero confine è cosa è per sessione e cosa è per processo.

**D. `cache_data` vs `cache_resource`: qual è la differenza?**

R. `cache_data` cachea il risultato di una funzione in base agli argomenti: se chiami la stessa funzione con gli stessi argomenti, Streamlit restituisce il risultato senza rieseguirla. È utile per query piccole, distinct, min/max, trasformazioni deterministiche. `cache_resource` cachea un oggetto condiviso, come l'engine o il repository: una sola istanza per processo, non una per chiamata. Non sono intercambiabili: uno è dato, l'altro è risorsa.

La differenza pratica si vede nella memoria e nella condivisione. `cache_data` può crescere con il numero di combinazioni di argomenti; `cache_resource` tiene un oggetto globale. Se metti l'engine in `cache_data`, rischi di crearne uno per combinazione; se metti un risultato in `cache_resource`, lo condividi tra tutti e perdi l'isolamento. La scelta dipende da cosa stai cacheando.

La cosa da non fare è usare la cache senza capire la chiave. Se gli argomenti non includono tutto ciò che influenza il risultato, la cache restituisce un valore sbagliato. La fingerprint dello stato serve proprio a questo: rendere esplicita la chiave. Senza, la cache è una scommessa.

**D. Come si aggiornano i dati e cosa vede l'utente tra una sincronizzazione e l'altra?**

R. I dati li produce un job giornaliero che scrive i Parquet su S3; la dashboard li copia in locale all'avvio e un thread in background li risincronizza ogni tre ore, scaricando solo i file con dimensione o ETag diversi. Dopo ogni sincronizzazione ricostruisce le tabelle materializzate. Tra una sincronizzazione e l'altra l'utente vede dati vecchi di qualche ora: è la freschezza che il sistema accetta, e non c'è un'invalidazione magica.

Il rischio, qui, è la chiave della cache per sessione: la fingerprint deve includere tutto ciò che influenza la query, non solo i filtri visibili. Se manca una dimensione, la cache restituisce un risultato sbagliato e sembra un bug di dati.

La sincronizzazione va anche gestita: spazio su disco, errori di rete, permessi sul bucket, lock per evitare due sincronizzazioni insieme. Non ho gestito questa parte del progetto; so dire cosa guarderei: età media dei file, durata della sincronizzazione, errori e dimensione. Senza metriche, la cache è una scatola nera.

**D. Perché non un database, o un DataFrame in memoria?**

R. Perché i dati sono Parquet e il carico è analitico; DuckDB evita di caricare un database e di gestire un server. Un database sarebbe meglio per transazioni, update concorrenti e vincoli: qui non servono. Un DataFrame in memoria sarebbe più semplice, ma i dati possono essere più grandi della memoria e le query colonnari sono più efficienti in DuckDB.

La differenza tra DuckDB e pandas è il modello di esecuzione: DuckDB spinge il calcolo nel motore, legge solo le colonne necessarie e può spillare su disco; pandas carica tutto in memoria e opera in Python. Per una dashboard su Parquet, DuckDB è più adatto. Per un'analisi esplorativa su un file piccolo, pandas è più immediato.

La scelta dipende dal volume e dal tipo di query. Non direi "DuckDB è sempre meglio": direi "per questo carico, è il compromesso giusto". Se i dati stessero in memoria e le query fossero semplici, pandas sarebbe più semplice. Se servissero transazioni, un database. Se servisse multiuser, un servizio.

**D. Cosa hai fatto tu e cosa miglioreresti?**

R. Conosco il codice e so spiegare le scelte, ma non ho progettato l'intera architettura della cache e del repository. Il presentation layer l'ho fatto a metà con lo sviluppatore del cliente. So dire cosa migliorerei: metriche di hit e miss, un modo per forzare la sincronizzazione, test sul refresh dei Parquet, un confine più chiaro tra stato di sessione e stato condiviso. Non ti racconto che l'ho fatto.

La cosa che migliorerei per prima è l'osservabilità: senza hit rate e senza età dei dati, non sai se la cache serve o se sta solo nascondendo dati vecchi. La seconda è la freschezza: oggi dipende da un giro di sincronizzazione ogni tre ore, e un'azione manuale di refresh sarebbe utile. La terza è un test che simuli il cambio di un file e verifichi che la view veda la nuova versione e la tabella la veda solo dopo la ricostruzione.

Queste sono cose che farei, non cose che ho fatto. In un colloquio è importante dirlo, perché il tema è pieno di persone che sanno cos'è una cache e poche che hanno gestito l'invalidazione in produzione.

### Prova di padronanza

- So spiegare DuckDB come motore embedded, non come server: gira nel processo, legge Parquet, usa un motore colonnare; il deployment è diverso da un database.
- So distinguere view e tabella materializzata: freschezza contro prestazioni, e il fatto che nel progetto le tabelle vengono ricostruite dopo ogni sincronizzazione.
- So descrivere la concorrenza: una connessione condivisa, un cursore per query, lock sulla registrazione, query read-only in parallelo, singleton di processo.
- So spiegare i quattro livelli di cache: singleton del repository, `cache_data` con TTL, specchio locale dei Parquet sincronizzato da S3 per ETag ogni tre ore, cache per sessione con fingerprint; e dire che il motore Athena con cache su S3 esiste ma non l'ho trovato usato nella dashboard.
- So dire cosa succede se il file cambia durante una query: rename atomico in locale, query in corso sul file aperto, rischio di versioni miste solo tra più file; e quando servirebbe un table format.
- So confrontare DuckDB e Athena su latenza, costo, concorrenza e modello di deployment, senza dire "uno è meglio".
- So indicare i limiti: single-node, niente transazioni multiutente, memoria e CPU della macchina, concorrenza limitata, dati vecchi fino a tre ore.
- So descrivere i failure mode: tabella stantia, TTL sbagliato, chiave incompleta, lock contention, query pesante che blocca le altre.
- So dire cosa migliorerei — metriche di cache, refresh manuale, test sul refresh — e cosa non ho fatto.
- So spiegare perché non un database o un DataFrame in memoria, collegando la scelta al volume e al tipo di query.

### Cosa non dire

- "DuckDB è un database server": è embedded, gira nel processo.
- Confondere view e tabella materializzata, o dire che la tabella è sempre meglio perché è più veloce.
- Cache senza TTL, senza chiave o senza invalidazione: è una scommessa, non una strategia.
- Ignorare la concorrenza e i rerun di Streamlit: sono il vincolo che giustifica l'architettura.
- Dire "basta aumentare la cache" senza misurare hit rate, memoria e freschezza.
- Dire "S3 è transazionale" o "la query vede sempre un risultato coerente": non lo è.
- Rivendicare l'intera architettura della cache e del repository: il presentation layer è stato fatto a metà.
- Confondere il limite di DuckDB con un difetto del progetto: è una scelta per un contesto single-node.

### Se non lo so

Il codice lo conosco e so spiegare le scelte, ma non ho progettato l'intera architettura della cache e del repository: il presentation layer l'ho fatto a metà con lo sviluppatore del cliente. Non ho gestito l'invalidazione della cache in produzione né misurato hit rate e latenze, quindi non ti do numeri. Se mi chiedi un dettaglio di DuckDB o di Streamlit che non ricordo, preferisco dirlo e ragionare sul comportamento.

---

## 1.11 — Step Functions e qualità/test delle pipeline dati

### Risposta principale (3–5 minuti)

Step Functions è un orchestratore, non un motore di calcolo. La distinzione è fondamentale: descrive una macchina a stati e coordina chiamate a servizi — Glue, Lambda, ECS — ma il lavoro lo fanno i servizi chiamati. In una pipeline dati è utile quando il flusso ha più passi, dipendenze, retry e gestione degli errori. Se hai un solo job, probabilmente è overkill: aggiungi un servizio, una definizione e un costo di gestione per un flusso che potrebbe essere una singola esecuzione.

Gli stati principali sono: task, choice, parallel, map, wait. Un task esegue un'azione; choice decide il ramo in base a un input; parallel esegue rami in parallelo; map itera su una collezione; wait aspetta. L'input e l'output sono JSON, e la macchina può trasformare i dati tra uno stato e l'altro. Due dettagli contano nella pratica. Il primo è l'integrazione `.sync` con i servizi: con Glue, per esempio, lo stato di tipo task attende la fine del job prima di passare al successivo, ed è il meccanismo con cui si modellano le dipendenze tra flussi, cioè un livello parte quando il precedente è finito. Il secondo è la scelta tra workflow Standard, adatto a flussi lunghi e con esecuzione esattamente una volta, ed Express, per flussi brevi e ad alto volume, con durata massima di cinque minuti e semantica at-least-once: per una pipeline batch con job Glue serve Standard. Questo rende il flusso visibile e testabile: puoi vedere dove si è fermato, quale ramo ha preso, quale output ha prodotto.

Retry e catch gestiscono gli errori. Retry riprova un task con un intervallo e un numero massimo di tentativi; catch intercetta un errore e porta a uno stato di fallimento o a un percorso di compensazione. La cosa importante è che il retry sia selettivo: un errore transitorio — un timeout di rete, un throttling — ha senso ritentarlo; un errore di dati o di logica no, perché ritentarlo produce lo stesso errore. Il backoff evita di martellare il servizio, e il limite evita un ciclo infinito.

L'idempotenza è il concetto che distingue una pipeline robusta da una fragile. Un retry può rieseguire un task che aveva già scritto parte dell'output. Se il task non è idempotente, il retry duplica i dati. Le strategie: chiave di esecuzione unica, overwrite controllato, scrittura in una posizione temporanea e commit finale, oppure una scrittura condizionata. "Exactly-once" non è una proprietà gratuita: esiste solo se la progetti, e spesso la soluzione pratica è "at-least-once con output idempotente".

Il fallimento a metà è il caso che i diagrammi dimenticano. Se un task scrive tre file e fallisce sul quarto, cosa resta? Se il retry riparte dall'inizio, riscrive i primi tre: va bene se l'output è idempotente, male se non lo è. Se riparte dal quarto, deve sapere cosa è già stato scritto. Le strategie sono: scrittura atomica, pulizia prima del retry, scrittura in una posizione temporanea e commit, oppure una compensazione esplicita. Non basta "riprovare": bisogna sapere cosa è già stato fatto.

La qualità dei dati non è "il job è verde". Un job può finire senza errori e produrre un output sbagliato. I controlli: schema, nulli, duplicati, conteggi, checksum, distribuzioni, riconciliazione sorgente/target, golden dataset. Per una migrazione, aggiungerei un confronto per chiave e un campione verificato a mano. Il controllo di qualità deve essere parte della pipeline, non un'attività manuale dopo: se lo fai dopo, il dato sbagliato è già a valle.

Dove mettere i controlli? Ai confini: input grezzo, output di ogni trasformazione critica, prima della scrittura. Con una politica chiara: fail fast per i dati non validi, quarantena per i correggibili, metriche per monitorare i fallimenti. Validare tutto ovunque costa; validare i punti critici è il compromesso. La domanda è: se questo dato è sbagliato, quanto costa scoprirlo a valle? Se costa molto, il controllo vale; se costa poco, forse basta un test.

I test di una pipeline sono a livelli. Unit test sulle trasformazioni, con input piccoli e deterministici. Test di contratto sullo schema e sui quality check. Integrazione su un campione, con dati reali ma ridotti. Test di idempotenza: rieseguire non deve duplicare. Failure injection: far fallire un task e verificare che il flusso si comporti come previsto. Non serve testare l'intero cluster: si testa la logica, il contratto e il comportamento in caso di errore.

Il monitoraggio è parte del design, non un'aggiunta. Log strutturati con un identificativo di esecuzione, metriche per job — durata, errori, retry, dati scansionati — allarmi sui fallimenti e sui tempi, lineage per capire da dove viene un dato. La pipeline deve dire "dove si è rotto", non solo "è fallita". Un fallimento senza contesto è un incidente che dura il doppio, perché qualcuno deve ricostruire cosa è successo.

Nella migrazione SAS non ho progettato Step Functions né il database di stato: l'architettura era dell'architetto, e io ho tradotto i job. So spiegare i concetti e cosa mancherebbe per una pipeline più solida. Nel cash flow ho usato Pandera per validare gli input, che è un controllo di qualità a un confine: è l'unico punto in cui posso dire di aver toccato il tema in modo concreto.

### Versione da 60 secondi

Step Functions è un orchestratore, non un motore di calcolo: coordina Glue, Lambda o ECS con stati, retry e catch. Con l'integrazione `.sync` aspetta la fine di un job Glue, ed è come si modellano le dipendenze tra flussi. Il retry va dato per categoria di errore e richiede output idempotenti: at-least-once con scrittura idempotente, non exactly-once. La qualità dei dati è schema, conteggi, unicità, riconciliazione e un golden dataset ai confini, non solo "il job è verde". Nella migrazione SAS non ho progettato né Step Functions né i controlli; nel cash flow ho usato Pandera in un punto.

### Follow-up

**D. Cos'è Step Functions e quando è overkill?**

R. È un servizio di orchestrazione: definisci stati e transizioni, e il servizio esegue i task chiamando Lambda, Glue, ECS o altri servizi. Gestisce input/output, retry, catch, parallel, map e wait. Non è un motore di calcolo: è il direttore d'orchestra. Il valore è la visibilità e la gestione degli errori, non la potenza di calcolo.

È overkill quando il flusso è un singolo job o una sequenza lineare senza retry, attese o dipendenze complesse. In quel caso aggiungi una definizione, un servizio e un costo di gestione per un flusso che potresti eseguire direttamente. È overkill anche quando il team non ha esperienza di orchestratori: la macchina a stati diventa un'altra cosa da capire e da mantenere.

Il criterio è: la complessità del flusso giustifica un orchestratore? Se hai più passi, dipendenze, retry e gestione degli errori, sì. Se hai un job, no. Non è "più architetturale" usare Step Functions: è più adatto a un certo tipo di flusso.

**D. Retry e catch: come li configureresti e cosa non ritenteresti?**

R. Configurerei il retry per categoria di errore, non per "tutto". Un timeout di rete, un throttling o un errore transitorio del servizio hanno senso; un errore di schema, un dato mancante o un bug di logica no, perché ritentarli produce lo stesso errore. Il retry dovrebbe avere un backoff esponenziale con jitter, un numero massimo di tentativi e un limite di tempo. Senza limite, un errore permanente diventa un ciclo che consuma risorse.

Catch serve a gestire il fallimento in modo esplicito: portare a uno stato di errore, a una notifica, a una compensazione o a una dead letter. La differenza tra "il flusso fallisce e nessuno lo sa" e "il flusso fallisce e c'è un percorso" è tutta qui. Il catch dovrebbe registrare il contesto: quale task, quale input, quale errore. Un catch che dice solo "failed" non è utile.

Cosa non ritenterei: un errore di validazione, un errore di business, un errore di permessi. Un retry su un permesso mancante fallisce uguale e maschera il problema. La regola è: ritenta solo ciò che può cambiare tra un tentativo e l'altro. Se la causa è deterministica, il retry è solo rumore.

**D. Come rendi idempotente un job?**

R. Con una chiave di esecuzione e un output indirizzabile. La chiave identifica la run — stesso input, stessa configurazione, stesso identificativo — e l'output viene scritto in una posizione che dipende dalla chiave. Se il job viene rieseguito, sovrascrive lo stesso output invece di crearne un secondo. Se l'output è già completo, il job può anche non fare nulla. La chiave è il modo per riconoscere "questo l'ho già fatto".

Le strategie concrete: overwrite controllato, scrittura in una posizione temporanea e commit finale, scrittura condizionata su un database, oppure una tabella con una chiave unica. Ognuna ha un costo: l'overwrite può lasciare uno stato intermedio se fallisce a metà; la scrittura temporanea richiede una fase di commit; la scrittura condizionata richiede un database che la supporti. Non c'è una soluzione universale: c'è la soluzione per il tipo di output.

"Exactly-once" è possibile solo se il sistema lo supporta end-to-end. Nella pratica si lavora con "at-least-once" e output idempotente: il job può essere eseguito più volte, ma il risultato è lo stesso. È una scelta più robusta che cercare una garanzia che il sistema non ti dà. La differenza va spiegata, non nascosta dietro una parola.

**D. Cosa succede se un task fallisce a metà e hai già scritto parte dell'output?**

R. È il caso in cui l'idempotenza si vede. Se il task ha scritto tre file su quattro, il retry deve sapere che i primi tre esistono e che il quarto manca, oppure deve poter riscrivere tutto senza duplicare. La strategia più semplice è scrivere in una posizione temporanea e committare solo alla fine: se fallisce, la posizione temporanea viene scartata e il retry riparte pulito. La seconda è un overwrite controllato: il retry riscrive gli stessi file, e la chiave garantisce che non siano duplicati.

La compensazione è un'altra opzione: se il task ha effetti collaterali — una notifica, una scrittura su un sistema esterno — il catch può annullarli. È più complessa e va progettata per ogni effetto. In una pipeline dati, spesso la compensazione non serve se l'output è idempotente e la scrittura è atomica per partizione. In un flusso con servizi esterni, può servire.

La cosa da non fare è "riprovo e vediamo". Un retry senza sapere cosa è già stato scritto è un modo per duplicare. La domanda giusta è: qual è lo stato del sistema dopo un fallimento, e come lo riconosco? Se non lo so, non ho un piano di recovery.

**D. Quali controlli di qualità metteresti in una pipeline?**

R. Schema: le colonne ci sono, i tipi sono quelli attesi, i nulli sono ammessi dove previsto. Conteggi: il numero di righe è plausibile rispetto alla sorgente e al periodo. Unicità: le chiavi non sono duplicate. Completezza: le colonne critiche non hanno nulli inattesi. Dominio: i valori sono nei range o nei set ammessi. Riconciliazione: la somma delle misure nel target corrisponde alla sorgente, con una tolleranza spiegata.

Poi i controlli di business: un tasso, un totale, una distribuzione che dovrebbe essere stabile. Un golden dataset con casi noti, confrontato a ogni esecuzione. E un controllo di freshiness: il dato è del periodo giusto? Un job può essere verde e aver processato il mese sbagliato. La qualità non è solo "i tipi sono giusti": è "il numero ha senso".

Dove metterli: ai confini e dopo le trasformazioni critiche. Con una politica: fail fast per i dati non validi, quarantena per i correggibili, metriche per monitorare. La soglia va scelta con il business, non decisa dal tecnico da solo. Un controllo che blocca troppo diventa un collo di bottiglia; uno che non blocca mai è decorativo.

**D. Come testeresti una pipeline dati?**

R. Unit test sulle trasformazioni: input piccolo, output atteso, casi limite. Test di contratto: schema, nulli, duplicati, tipi. Integrazione su un campione: dati reali ma ridotti, per verificare che i pezzi si parlino. Test di idempotenza: rieseguire non duplica. Failure injection: far fallire un task e verificare che il flusso si comporti come previsto. Poi un test di regressione su un output atteso, per catturare i cambiamenti di semantica.

Non testerei l'intero cluster: testerei la logica e il contratto. Un test che richiede un cluster è lento, fragile e costoso; un test che richiede una funzione e un DataFrame piccolo è veloce e utile. L'integrazione serve per i confini, non per ogni caso. La piramide è la solita: molti test piccoli, pochi test grandi.

Per una migrazione, aggiungerei il confronto tra vecchio e nuovo: stesso periodo, stesso input, output confrontati per chiave, con un registro delle differenze accettate. Non mi fiderei del "numero di righe uguale". La migrazione è il caso in cui il test di equivalenza vale più di mille unit test.

**D. Come monitoreresti una pipeline in produzione?**

R. Log strutturati con un identificativo di esecuzione, così puoi seguire una run attraverso i task. Metriche per job: durata, successi, fallimenti, retry, dati scansionati, righe scritte. Allarmi sui fallimenti e sui tempi anomali, non su tutto. Lineage: da dove viene un dato e dove va, per capire l'impatto di un errore. Una dashboard che mostri lo stato delle run e le anomalie.

La pipeline deve dire "dove si è rotto", non solo "è fallita". Un alert che dice "job failed" senza contesto costringe chi è di turno a ricostruire. Un alert che dice "il task 3 è fallito per schema mismatch sulla colonna X, input del periodo Y" è azionabile. La differenza è il tempo di ripristino.

La dead letter queue è parte della strategia: i messaggi che non possono essere elaborati dopo i retry finiscono lì, per essere analizzati o riprocessati. Non è un dettaglio: è il posto dove non perdi il dato. Senza, un fallimento è una perdita silenziosa.

**D. Cosa hai fatto tu e cosa no?**

R. Nella migrazione SAS ho tradotto job e lavorato sui flussi. Non ho progettato Step Functions, il database di stato o i controlli di qualità: l'architettura era dell'architetto. Nel cash flow ho usato Pandera per validare gli input, che è un controllo a un confine, ma non ho progettato la strategia di qualità complessiva.

So spiegare i concetti e cosa servirebbe per una pipeline più solida, ma non rivendico un'esperienza di orchestrazione che non ho. Se l'intervistatore chiede "come hai configurato il retry?", la risposta è: non l'ho configurato io. Preferisco dirlo che inventare una configurazione plausibile.

**D. Come valideresti una migrazione tra due sistemi, in generale?**

R. Con un confronto a più livelli. Primo: schema e conteggi, per capire se il perimetro è lo stesso. Secondo: chiavi e duplicati, per capire se l'identità dei record è preservata. Terzo: misure e distribuzioni, per capire se i valori sono gli stessi. Quarto: un campione verificato a mano, per i casi che i controlli automatici non colgono. Quinto: un registro delle differenze, con una spiegazione per ognuna.

Poi il confronto per periodo: stesso intervallo, stesso input, output del vecchio e del nuovo sistema. Le differenze vanno classificate: errore, cambio voluto, caso limite noto. Non basta "i numeri tornano": bisogna sapere perché tornano e dove non tornano. Una migrazione senza un registro delle differenze è una scommessa.

Infine, il test di idempotenza e il rerun: rieseguire la migrazione non deve duplicare. E il test di rollback: se qualcosa va storto, si può tornare indietro? La migrazione non è finita quando il job gira: è finita quando sai che i numeri sono giusti e puoi tornare indietro.

### Prova di padronanza

- So separare orchestrazione e computazione: Step Functions coordina, Glue/Lambda/ECS eseguono; non è un motore di calcolo.
- So elencare gli stati principali — task, choice, parallel, map, wait — e spiegare input/output JSON e il valore della visibilità.
- So configurare un retry per categoria di errore, con backoff e limite, e so dire cosa non ritentare: errori deterministici, di validazione, di permessi.
- So spiegare l'idempotenza con strategie concrete: chiave di esecuzione, overwrite controllato, scrittura temporanea e commit, scrittura condizionata.
- So dire che "exactly-once" esiste solo se progettato, e che nella pratica si usa "at-least-once con output idempotente".
- So gestire il fallimento a metà: cosa è già stato scritto, come si riparte, quando serve una compensazione.
- So definire controlli di qualità concreti: schema, conteggi, unicità, completezza, dominio, riconciliazione, golden dataset, freshness.
- So collocare i controlli ai confini e distinguere fail fast, quarantena e metriche.
- So descrivere i test di una pipeline: unit, contratto, integrazione su campione, idempotenza, failure injection, regressione.
- So dire cosa ho fatto — traduzione job, Pandera nel cash flow — e cosa non ho fatto — Step Functions, DB di stato, strategia di qualità.

### Cosa non dire

- "Step Functions esegue Spark": orchestrazione non è computazione; il calcolo lo fa Glue o un altro servizio.
- Retry senza idempotenza: è il modo per duplicare i dati e sembrare che il problema sia risolto.
- Dire "ho progettato la pipeline" o "ho configurato Step Functions": non è vero, l'architettura era dell'architetto.
- Non avere controlli di qualità oltre al "job verde": un job può finire e produrre un numero sbagliato.
- Ignorare il fallimento a metà e il recupero: è il caso in cui si vede se la pipeline è robusta.
- Dire "i test li faccio dopo": la pipeline dati senza test è una scommessa che si paga in produzione.
- Confondere una dead letter queue con un log: la DLQ conserva il messaggio non elaborato, non solo l'errore.
- Parlare di "exactly-once" senza dire come lo ottieni: è una parola, non una garanzia.

### Se non lo so

Non ho progettato Step Functions né il database di stato: l'architettura era dell'architetto, e io ho tradotto i job. So spiegare i concetti, i failure mode e cosa farei per una pipeline più solida, ma se mi chiedi una configurazione specifica che non ho gestito, preferisco dirlo.

---

## 1.12 — Data modeling e livelli di elaborazione

### Risposta principale (3–5 minuti)

Il data modeling parte dalle domande, non dalle tabelle. Prima di scrivere uno schema devo sapere quali domande il sistema deve rispondere, quali misure contano, con quale granularità, e chi userà i dati. Solo dopo scelgo se normalizzare o denormalizzare, se usare un modello relazionale o dimensionale. Un modello senza requisiti è un disegno decorativo: bello, coerente, e inutile quando arriva la prima query che non era prevista.

I tre livelli del modello sono concettuale, logico e fisico. Il concettuale descrive entità, relazioni e regole del dominio, senza preoccuparsi delle tabelle: "un contratto appartiene a un cliente, ha un prodotto, una durata, un prezzo". Il logico traduce in tabelle, chiavi e attributi, con le relazioni e i vincoli. Il fisico decide tipi, indici, partizioni, distribuzione. Saltare il concettuale è il modo per costruire un modello che riflette il database invece del dominio.

Nel mondo transazionale normalizzo. La prima forma normale elimina i valori non atomici; la seconda elimina le dipendenze parziali dalla chiave; la terza elimina le dipendenze transitive. L'obiettivo è ridurre duplicazioni e anomalie: se un dato è in un posto solo, non può divergere. Il costo è che le query richiedono join. In OLTP, con molte scritture piccole e transazioni, la normalizzazione è la scelta giusta. Troppo denormalizzato significa duplicare fatti e rischiare incoerenza.

Nel mondo analitico uso il modello dimensionale. Il fatto contiene le misure e le chiavi verso le dimensioni; la dimensione contiene gli attributi descrittivi. Il grain è il livello di dettaglio del fatto: una riga per contratto, per giorno, per prodotto. Se il grain non è chiaro, il modello è ambiguo e le aggregazioni sono sbagliate. Il grain va dichiarato prima di scrivere la tabella, non dedotto dopo.

Un esempio: le vendite. Il fatto ha una riga per prodotto, negozio e giorno, con le misure quantità e importo. Le dimensioni sono prodotto, negozio, data. Il grain è "una riga per prodotto/negozio/giorno". Se aggiungo una riga per cliente, il grain cambia e le somme non sono più confrontabili. Il modello dimensionale non è più semplice del relazionale: è più semplice per le query analitiche, a costo di ridondanza controllata.

Lo star schema ha una tabella fatto al centro e dimensioni denormalizzate intorno: i join sono semplici e le query leggibili. Lo snowflake normalizza le dimensioni in più tabelle: meno duplicazione, più join. Per analytics, star è spesso preferibile per semplicità e prestazioni; snowflake può avere senso quando le dimensioni sono grandi e condivise. Non è una scelta religiosa: è una scelta di leggibilità e costo.

Le SCD gestiscono i cambi nel tempo. Tipo 1: sovrascrivi, perdi la storia. Tipo 2: nuova riga con validità, mantieni la storia. Tipo 3: colonna aggiuntiva per il valore precedente. La scelta dipende da quanto la storia è importante per l'analisi. Una SCD2 non è "aggiungi una colonna": è un modello con chiavi, intervalli di validità e query che tengono conto del tempo. Se non pensi alle query, la SCD2 diventa un incubo.

OLTP e OLAP sono carichi diversi. OLTP: molte transazioni piccole, scritture, vincoli, normalizzazione. OLAP: poche query grandi, letture, aggregazioni, dimensionale. Non ottimizzi lo stesso schema per entrambi. Un data warehouse può avere un'area staging normalizzata e un modello dimensionale per il reporting: non è contraddittorio, è la separazione dei carichi.

I livelli di pipeline — raw/staging, preprocessing, arricchimento, reporting — separano le responsabilità. Raw conserva l'originale; staging pulisce; preprocessing normalizza; reporting aggrega. Ogni livello è verificabile e rieseguibile. Se mescoli tutto, un errore è difficile da localizzare e il rerun duplica o corrompe. La separazione non è burocrazia: è ciò che rende una pipeline manutenibile.

Kimball e Inmon sono due approcci. Kimball: dimensionale, bottom-up, data mart e star schema. Inmon: warehouse normalizzato, top-down, poi data mart. Non è una religione: dipende dall'organizzazione, dai requisiti e dalla governance. Per un'analisi dipartimentale Kimball è spesso più diretto; per un warehouse aziendale ampio Inmon può avere senso. La scelta va spiegata, non imposta.

Nel progetto SAS ho lavorato sui flussi, non ho progettato il modello. Conosco la teoria e il contesto, ma non rivendico il design. Se mi chiedi "perché quella scelta", dico che non l'ho presa io. Questa onestà è parte della risposta: un colloquio non premia chi sa tutto, premia chi sa cosa ha fatto e cosa no.

### Versione da 60 secondi

Il data modeling parte dalle domande. Per OLTP si normalizza (3NF) per ridurre anomalie; per l'analisi si usa il modello dimensionale: un fatto con un grain dichiarato e le dimensioni intorno, star schema per semplicità, snowflake se le dimensioni sono grandi e condivise. Le SCD gestiscono i cambi nel tempo: tipo 1 sovrascrive, tipo 2 conserva la storia con chiave surrogata e intervalli di validità. I livelli di pipeline (raw, staging, preprocessing, reporting) rendono verificabile e rieseguibile ogni passo. Nel progetto SAS ho lavorato sui flussi, non ho progettato il modello.

### Follow-up

**D. Normalizzato vs dimensionale: quando usi quale?**

R. Normalizzato per OLTP: molte scritture piccole, transazioni, vincoli, necessità di evitare duplicazioni e anomalie. Dimensionale per OLAP: poche query grandi, aggregazioni, lettura per colonne, necessità di query leggibili. Non è "uno giusto e uno sbagliato": è il carico che decide. Un sistema può avere entrambi, in aree diverse.

Il costo del normalizzato è il join; il costo del dimensionale è la ridondanza. In OLTP, un join in più è spesso accettabile; in OLAP, un join in meno è spesso decisivo. La scelta dipende da chi scrive e chi legge, e con quale frequenza. Se il sistema è misto, si separa: staging normalizzato, reporting dimensionale.

Un errore comune è applicare il dimensionale a un sistema transazionale perché "è più moderno". Un altro è normalizzare un warehouse fino a rendere ogni query un puzzle. Il modello è uno strumento, non un'identità.

**D. Cosa sono fatto, dimensione e grain? Fammi un esempio.**

R. Il fatto è l'evento che voglio analizzare, con le misure e le chiavi verso le dimensioni. La dimensione è il contesto descrittivo: prodotto, negozio, tempo, cliente. Il grain è il livello di dettaglio di una riga del fatto: una riga per prodotto/negozio/giorno, per esempio. Se il grain non è dichiarato, non sai cosa stai sommando.

Esempio: vendite. Fatto: una riga per prodotto, negozio e giorno, con quantità e importo. Dimensioni: prodotto (nome, categoria, marca), negozio (città, regione), data (anno, mese, giorno). Il grain è "una riga per prodotto/negozio/giorno". Se aggiungo il cliente, il grain cambia: una riga per prodotto/negozio/giorno/cliente. Le somme per prodotto non sono più le stesse, perché ora sono dettagliate per cliente.

Il grain è la prima cosa da dichiarare e l'ultima da cambiare. Un fatto con grain ambiguo produce aggregazioni sbagliate e nessuno se ne accorge finché un numero non torna. La domanda "cosa rappresenta una riga?" deve avere una risposta sola.

**D. Star schema vs snowflake?**

R. Star: dimensioni denormalizzate, join semplici, query leggibili. Snowflake: dimensioni normalizzate in più tabelle, meno duplicazione, più join. Per analytics, star è spesso preferibile per semplicità e prestazioni; snowflake può avere senso quando le dimensioni sono grandi e condivise, o quando la duplicazione è un problema.

Il costo dello star è la ridondanza: un attributo di prodotto è ripetuto in ogni riga della dimensione, e se cambia va aggiornato in un posto solo (la dimensione), non nel fatto. Il costo dello snowflake è la complessità delle query e il numero di join. In un warehouse con molte query ad hoc, la leggibilità dello star vale spesso il costo della ridondanza.

Non c'è un vincitore assoluto. La scelta dipende dagli strumenti, dal volume, dal team e dalle query. Un modello star mal progettato è peggio di uno snowflake semplice; uno snowflake ben progettato è meglio di uno star caotico. Il criterio è la comprensione e la manutenzione, non l'etichetta.

**D. SCD: cosa sono e come implementeresti una SCD2?**

R. Slowly Changing Dimension: gestiscono i cambi nel tempo. Tipo 1: sovrascrivi, perdi la storia. Tipo 2: nuova riga con validità, mantieni la storia. Tipo 3: colonna aggiuntiva per il valore precedente. La scelta dipende da quanto la storia è importante per l'analisi. Se il business chiede "com'era il prezzo l'anno scorso", serve SCD2; se chiede solo il valore attuale, SCD1 basta.

Una SCD2 si implementa con una chiave surrogata, una chiave di business, un intervallo di validità — `valid_from`, `valid_to` — e un flag di corrente. Quando un attributo cambia, chiudi la riga corrente con `valid_to` e inserisci una nuova riga con `valid_from`. Le query "point-in-time" filtrano per la data di interesse: `valid_from <= data < valid_to`. La chiave surrogata evita di confondere la stessa entità in versioni diverse.

Il costo è la complessità: ogni query deve scegliere la versione giusta, e il caricamento deve gestire chiusura e apertura in modo atomico. Una SCD2 fatta male produce righe sovrapposte o buchi di validità, e i report diventano incoerenti. Non è "aggiungi una colonna": è un modello con regole.

**D. OLTP vs OLAP: qual è la differenza pratica?**

R. OLTP: molte transazioni piccole, scritture frequenti, vincoli, normalizzazione, latenza bassa per singola operazione. OLAP: poche query grandi, letture, aggregazioni, dimensionale, latenza accettabile ma throughput importante. Sono carichi diversi: ottimizzi lo stesso schema per entrambi e ottieni un compromesso che non eccelle in nessuno.

In pratica, un sistema OLTP ha bisogno di integrità e concorrenza; un OLAP ha bisogno di scansioni e aggregazioni. Per questo si separano: il dato viene estratto dall'OLTP, trasformato, caricato in un warehouse o in un data lake. La separazione non è un lusso: è il modo per non far competere le query analitiche con le transazioni.

Un esempio: un DB PostgreSQL per le prenotazioni è OLTP; un modello star su un warehouse per analizzare le prenotazioni per mese e città è OLAP. Lo stesso dato, due modelli. Non è duplicazione: è ottimizzazione per usi diversi.

**D. Perché separare i livelli di pipeline?**

R. Per isolare le responsabilità: raw conserva l'originale, staging pulisce, preprocessing normalizza, reporting aggrega. Ogni livello è verificabile e può essere rieseguito. Se mescoli tutto, un errore è difficile da localizzare e il rerun duplica o corrompe. La separazione è ciò che rende una pipeline manutenibile.

Il costo è lo storage e il numero di passaggi. Il beneficio è la diagnosi: se il report è sbagliato, puoi controllare il livello di reporting, poi il preprocessing, poi lo staging, e capire dove si è rotto. Senza livelli, devi rifare tutto e sperare. In una migrazione, i livelli sono anche il modo per confrontare vecchio e nuovo a ogni stadio, non solo alla fine.

L'idempotenza è collegata: ogni livello dovrebbe poter essere rieseguito senza duplicare. Se un livello scrive in append, il rerun duplica; se scrive in overwrite controllato o in una partizione, è più sicuro. La separazione senza idempotenza è solo un disegno.

**D. Kimball vs Inmon?**

R. Kimball: dimensionale, bottom-up, data mart e star schema. Inmon: warehouse normalizzato, top-down, poi data mart. Sono approcci diversi con compromessi diversi. Kimball è più diretto per un'analisi dipartimentale e per un team che vuole vedere risultati; Inmon è più adatto a un warehouse aziendale ampio, con governance e integrazione tra fonti.

Il costo di Kimball è la possibile incoerenza tra data mart se non c'è un bus di dimensioni condivise. Il costo di Inmon è il tempo e la complessità: il warehouse normalizzato richiede più lavoro prima di vedere valore. Non è una religione: dipende dall'organizzazione, dai requisiti e dalla maturità.

In un colloquio, direi: "dipende dal contesto; per un'analisi dipartimentale partirei da Kimball, per un'integrazione aziendale con più fonti valuterei Inmon". E aggiungerei che la scelta non è solo tecnica: è organizzativa.

**D. Come modelleresti un dominio nuovo?**

R. Prima le domande: chi usa i dati, quali decisioni deve prendere, quali misure contano, con quale granularità, con quale storia. Poi il modello concettuale: entità, relazioni, regole. Poi il logico: tabelle, chiavi, attributi. Poi il fisico: tipi, indici, partizioni. Poi test con query reali: il modello regge le domande che il business fa davvero?

Non partirei dalle tabelle. Un modello che parte dalle tabelle riflette il database, non il dominio. E non partirei dal "modello perfetto": partirei da un modello che risponde alle domande principali, con un margine per estendersi. Il modello evolve, non nasce finito.

La cosa che verificherei: il grain è chiaro? Le misure sono additive? Le dimensioni sono condivise? Le query principali sono semplici? Se una query richiede cinque join e tre sottoquery, forse il modello non è allineato alle domande. Il test è la query, non il diagramma.

**D. Cosa hai fatto nel data model SAS?**

R. Ho lavorato sui flussi e sulle trasformazioni, non ho progettato il modello. Conosco la teoria e il contesto, ma non rivendico il design. Se mi chiedi "perché quella scelta", dico che non l'ho presa io. Ho visto un modello esistente e ho contribuito a migrarlo, che è diverso da progettarlo.

Questa distinzione è importante perché il data modeling è un tema dove la teoria si impara in fretta e la pratica richiede anni. So spiegare normalizzazione, dimensionale, grain e SCD, ma non ho progettato un warehouse da zero. Se l'intervistatore vuole testare la teoria, posso rispondere; se vuole testare l'esperienza di design, dico cosa ho fatto e cosa no.

### Prova di padronanza

- So partire dalle domande e dai requisiti, non dalle tabelle, e so spiegare perché un modello senza requisiti è un disegno decorativo.
- So distinguere concettuale, logico e fisico, e so dire cosa succede se salto il concettuale.
- So spiegare 1NF, 2NF e 3NF con l'obiettivo — ridurre duplicazioni e anomalie — e il costo — più join.
- So definire fatto, dimensione e grain, con un esempio di vendite, e so dire perché il grain va dichiarato prima.
- So confrontare star e snowflake su leggibilità, ridondanza e join, senza dire che uno è sempre meglio.
- So spiegare SCD1, SCD2 e SCD3, e so descrivere l'implementazione di SCD2 con chiave surrogata, validità e flag di corrente.
- So distinguere OLTP e OLAP e spiegare perché si separano i carichi e i modelli.
- So spiegare i livelli raw/staging/preprocessing/reporting e collegarli a verificabilità e idempotenza.
- So confrontare Kimball e Inmon sul contesto organizzativo, non come tifoserie.
- So dire cosa ho fatto — flussi SAS, non design del modello — e cosa non rivendico.

### Cosa non dire

- "Dimensionale sempre meglio" o "normalizzato sempre meglio": dipende dal carico e dall'uso.
- Confondere fatto e dimensione, o non sapere qual è il grain: è la base del modello dimensionale.
- Descrivere SCD2 come "aggiungi una colonna": è un modello con validità, chiavi e query.
- Dire "ho progettato il data model SAS": non è vero, ho lavorato sui flussi.
- Parlare di Kimball e Inmon come squadre di calcio: sono approcci con compromessi.
- Ignorare il costo delle query e della manutenzione: un modello bello ma illeggibile è un debito.
- Dire "normalizzo tutto" o "denormalizzo tutto" senza contesto: sono strumenti, non dogmi.
- Saltare i livelli di pipeline e chiamarlo "semplificazione": spesso è solo perdita di diagnosi.

### Se non lo so

Non ho progettato il data model SAS: ho lavorato sui flussi e sulle trasformazioni, e il design era di qualcun altro. Non ho esperienza profonda di governance o master data management: so spiegare i concetti e applicarli a un caso semplice, ma non li rivendico come pratica. Se mi chiedi una scelta specifica del modello originale, dico che non l'ho presa io e ragiono sul contesto.

---

## 1.13 — IaC e CI/CD del cliente

### Risposta principale (3–5 minuti)

L'infrastruttura come codice serve a rendere l'ambiente riproducibile e revisionabile. Invece di creare risorse a mano nella console, le descrivi in file versionati. Questo riduce il drift — la differenza tra ciò che credi di avere e ciò che hai davvero — permette code review e rende possibile ricreare un ambiente. Il prezzo è la curva di apprendimento e la necessità di gestire stato, segreti e permessi. Non è "infrastruttura più moderna": è una scelta di controllo e ripetibilità.

AWS CDK è un framework che permette di definire infrastruttura in un linguaggio di programmazione — Python, TypeScript, Java. L'unità di base è la app; dentro ci sono gli stack; dentro gli stack ci sono i construct, che rappresentano risorse o gruppi di risorse. `synth` genera un template CloudFormation; `deploy` applica il template. Non è "CloudFormation scritto a mano": è un'astrazione che produce CloudFormation. Il template generato è ciò che AWS esegue, e questo è importante capirlo: CDK non è un servizio di runtime, è un generatore.

La pipeline CI/CD è una catena di gate. Source: il codice arriva da un repository. Build/test: si compila, si eseguono lint e test. Synth: si genera il template. Deploy: si applica sull'ambiente. In mezzo ci possono essere scan di sicurezza, approvazioni e smoke test. I buildspec descrivono i comandi; gli artifact passano tra gli stage. Un deploy senza test è un rischio, non una velocità. Un deploy senza approvazione in produzione è una scommessa.

IAM è la parte che si dimentica spesso. Least privilege significa che ogni ruolo ha solo i permessi necessari. Un job Glue non deve poter cancellare un bucket se non gli serve; una Lambda non deve avere `*`. Si usano policy specifiche, ruoli separati per funzione e revisione. È facile dare `*` per far funzionare tutto, ma è un debito di sicurezza: se una funzione viene compromessa, il danno è proporzionale ai suoi permessi.

ECR/ECS/Lambda hanno ruoli diversi. ECR conserva le immagini; ECS esegue task e servizi; Lambda esegue funzioni event-driven. La pipeline costruisce l'immagine, la pubblica su ECR, aggiorna il task definition o la funzione. Il ruolo di esecuzione di un task ECS è diverso dal ruolo del task: uno serve a ECS per pullare l'immagine e scrivere i log, l'altro è ciò che il codice usa per accedere alle risorse. Confonderli è un errore comune e una fonte di permessi troppo ampi.

Il rollback va progettato prima. Per il codice, un redeploy della versione precedente o un rollback dello stack; per i dati, backup e restore; per le migrazioni, compatibilità con la versione precedente o un piano esplicito. Il rollback di un template non è sempre sufficiente: se una risorsa stateful è stata modificata, tornare indietro può essere impossibile o costoso. La domanda "si può tornare indietro?" va fatta prima del deploy, non durante l'incidente.

I segreti non stanno nel repository e non stanno nell'immagine: stanno in un secret manager o in variabili d'ambiente iniettate. I log non devono contenerli. La pipeline deve avere un ruolo con permessi limitati anche sui segreti. La sicurezza non è un passaggio finale: è una proprietà del disegno. Un segreto committato è un segreto compromesso, anche se lo cancelli dopo.

Il deploy concorrente va ragionato al livello giusto. CloudFormation serializza le operazioni sullo stesso stack: se ne parte una mentre un'altra è in corso, la seconda viene rifiutata con un errore (lo stack è in `UPDATE_IN_PROGRESS`), quindi non si ottiene una risorsa incoerente. Il problema è nella pipeline: due esecuzioni in coda che si pestano i piedi, o una che sovrascrive l'effetto dell'altra. La soluzione è serializzare le esecuzioni per ambiente. Il drift è l'altro rischio: qualcuno modifica una risorsa dalla console, e il template non riflette più la realtà. Il controllo periodico e la disciplina di non toccare la console sono parte del gioco.

Nel progetto l'infrastruttura era del team di piattaforma del cliente. Gli stack stanno sopra una libreria interna del cliente (`advanalib`), che incapsula costrutti come l'hosting di un'app Streamlit su Fargate dietro un load balancer, ed è distribuita da un registro privato di pacchetti. I buildspec sono due: uno che produce il `synth` con un file di differenze rispetto allo stato esistente, uno che fa il `deploy`; entrambi assumono un ruolo nell'account di destinazione e leggono da Parameter Store le versioni di Python, Node e CDK. Io ho integrato l'applicazione e so leggere questi file, ma non ho progettato CDK, IAM o la pipeline. Questa distinzione è importante: non voglio attribuirmi un lavoro che non è mio, e non voglio nemmeno sembrare che l'infrastruttura sia un mistero. La usavo, la capivo a livello di flusso, ma non l'ho progettata.

### Versione da 60 secondi

IaC rende l'ambiente riproducibile e revisionabile. CDK genera template CloudFormation: `synth` li produce, `deploy` li applica. Nel cash flow gli stack stanno sopra una libreria interna del cliente, e i buildspec fanno `synth` con differenze e poi `deploy` assumendo un ruolo nell'account di destinazione. Il minimo privilegio si applica per ruolo e risorsa, e i segreti non vanno né nel repository né nell'immagine. Ho integrato l'applicazione e so leggere questi file; CDK, IAM e pipeline erano del team di piattaforma.

### Follow-up

**D. Cos'è IaC e perché non usare la console?**

R. Infrastructure as Code: descrivi l'infrastruttura in file versionati, invece di crearla a mano. I vantaggi sono riproducibilità, revisione, drift detection e rollback. Puoi ricreare un ambiente da zero, vedere chi ha cambiato cosa e confrontare due versioni. La console è comoda per esplorare, ma non lascia una traccia revisionabile e non è ripetibile: due persone che cliccano le stesse cose possono ottenere risultati diversi.

Il costo è la curva di apprendimento e la gestione dello stato. Con CDK, lo stato è in CloudFormation; con Terraform, in un file di state che va protetto e condiviso. I segreti e i permessi vanno gestiti con attenzione. Non è "gratis": è un investimento che si ripaga quando l'ambiente cresce e le persone cambiano.

La console resta utile per diagnosticare e per vedere lo stato reale. Non è il nemico: è uno strumento di esplorazione. Il problema è usarla per cambiare la produzione senza passare dal codice. La regola è: se una modifica è permanente, deve stare nel codice.

**D. CDK vs CloudFormation vs Terraform?**

R. CloudFormation è il servizio di AWS che applica i template; CDK è un'astrazione che genera template CloudFormation a partire da codice in un linguaggio di programmazione. Non sono alternativi: CDK produce CloudFormation. Il vantaggio di CDK è la tipizzazione, i costrutti e la possibilità di usare il linguaggio per comporre; il costo è un livello in più e la necessità di capire cosa genera.

Terraform è multi-cloud e usa un linguaggio dichiarativo proprio, con uno state file. È ottimo quando hai più provider o un team che lo conosce. CDK è più integrato con AWS e con i suoi costrutti. La scelta dipende dal contesto: se il cliente ha già CDK e il team di piattaforma lo usa, cambiare tool ha un costo che raramente si ripaga.

Non direi "Terraform è meglio" o "CDK è meglio": direi "dipende dal contesto, dal team e dal provider". In un colloquio, la risposta matura è spiegare il trade-off, non schierarsi.

**D. Cosa fa `synth` e cosa fa `deploy`? Cosa succede se il deploy fallisce?**

R. `synth` legge la app CDK, risolve i construct e genera il template CloudFormation, oltre agli asset — per esempio le immagini o i file — che servono al deploy. `deploy` applica il template: CloudFormation crea, aggiorna o elimina le risorse per raggiungere lo stato desiderato. `synth` non tocca l'ambiente; `deploy` sì. Confondere i due è un errore concettuale: puoi fare synth quante volte vuoi senza effetti, mentre un deploy ha conseguenze.

Se il deploy fallisce, CloudFormation può fare un rollback allo stato precedente, a seconda del tipo di risorsa e dell'errore. Ma il rollback non è magico: una risorsa stateful modificata può non tornare indietro, e una migrazione dati può essere irreversibile. Per questo il deploy va progettato con un piano di rollback e, se possibile, con compatibilità tra versioni. "Facciamo rollback" è una frase che va verificata, non assunta.

Un deploy fallito a metà può lasciare lo stack in uno stato inconsistente. La cosa da fare è leggere l'errore, capire quale risorsa ha fallito, e decidere se correggere e riapplicare o tornare indietro. Non è un processo che si improvvisa durante l'incidente: è una procedura che dovrebbe essere scritta.

**D. Come è strutturata una pipeline e cosa sono i buildspec?**

R. Una pipeline tipica: source, build/test, synth, deploy su un ambiente, smoke test, approvazione, deploy su produzione. I buildspec sono file che descrivono i comandi per ogni fase: installazione, test, synth, deploy. Gli artifact sono i file prodotti da uno stage e consumati dal successivo — per esempio il template CloudFormation o l'immagine. Gli ambienti possono avere approvazioni manuali e regole diverse.

Il valore della pipeline non è l'automazione in sé: è che ogni stage ha un output verificabile e un gate. Se il test fallisce, non si deploya; se il synth fallisce, non si applica; se lo smoke test fallisce, si torna indietro. La pipeline è il posto in cui la qualità diventa un vincolo, non un'opinione.

Nel progetto io ho usato i buildspec e integrato l'applicazione, ma non ho progettato la pipeline. Quello che vedo nei file: un buildspec produce il `synth` e un confronto con lo stato esistente, uno applica il `deploy`; entrambi assumono un ruolo cross-account, fanno login al registro dei pacchetti e a ECR, e leggono le versioni dei tool da Parameter Store. So spiegare gli stage e il flusso, non ti racconto una configurazione che non ho scritto. Se mi chiedi com'era fatta la pipeline nel suo insieme, non invento.

**D. IAM least privilege: come costruiresti una policy?**

R. Partirei da ciò che il servizio deve fare: quali azioni, su quali risorse, in quali condizioni. Per una Lambda che legge un bucket, `s3:GetObject` sul prefisso necessario, non `s3:*` su tutti i bucket. Per un task ECS, un ruolo di esecuzione per pullare l'immagine e scrivere i log, e un ruolo del task per le risorse che il codice usa. Poi verificherei con i log e con un test che i permessi mancanti emergano subito, non in produzione.

Il rischio di `*` è che un errore o una compromissione hanno un raggio d'azione enorme. È comodo per far funzionare tutto, ma è un debito che si paga quando meno te lo aspetti. La policy minima richiede tempo e conoscenza del servizio; è un investimento, non un lusso.

La revisione è parte del processo: chi scrive la policy dovrebbe spiegare perché ogni azione è necessaria. Una policy senza una spiegazione è difficile da mantenere e facile da allargare. Il least privilege non è un file, è una pratica.

**D. Come gestisci i segreti?**

R. Non nel repository, non nell'immagine, non nei log. Stanno in un secret manager o in variabili d'ambiente iniettate al runtime. La pipeline deve avere un ruolo con permessi limitati sui segreti, e gli artifact non devono contenerli. La rotazione va pianificata: un segreto che non ruota è un rischio che cresce nel tempo.

Il rischio pratico è che un segreto finisca in un log o in un messaggio di errore. Per questo i log vanno progettati per non stampare credenziali, e i test devono coprire i casi in cui un errore potrebbe esporle. Un segreto committato è compromesso anche se lo cancelli: la storia del repository lo conserva.

Non ho gestito i segreti del progetto: erano del team di piattaforma. So spiegare i principi e cosa farei, ma non rivendico un'operazione che non ho fatto.

**D. Cosa succede se due deploy partono insieme o se qualcuno modifica la console?**

R. Due aggiornamenti dello stesso stack non si eseguono insieme: CloudFormation serializza le operazioni e rifiuta la seconda mentre la prima è in corso, quindi non ottieni uno stato incoerente. Il rischio è a livello di pipeline: due esecuzioni che arrivano insieme, una delle quali fallisce o sovrascrive l'effetto dell'altra. La risposta è serializzare per ambiente, con una coda di esecuzione o un lock dello strumento di orchestrazione. Il lock dello stato è una caratteristica di Terraform, non di CDK. In un team piccolo la disciplina può bastare; in un team grande serve un meccanismo.

Il drift è l'altro problema: qualcuno modifica una risorsa dalla console, e il template non riflette più la realtà. Al prossimo deploy, CloudFormation può sovrascrivere la modifica o fallire. La prevenzione è la disciplina e il controllo periodico; la diagnosi è il confronto tra stato reale e template. Non è un problema che si risolve una volta: è una pratica continua.

Non ho gestito questi aspetti nel progetto. So dire cosa guarderei e perché, ma non ti racconto un incidente che non ho vissuto.

**D. Cosa hai fatto tu e cosa faresti da zero?**

R. Ho integrato l'applicazione e usato i buildspec: ho lavorato con le risorse create dal team, ho adattato il codice ai ruoli e alle configurazioni, ho usato la pipeline per deployare. Non ho progettato CDK, IAM o la pipeline. So spiegare i concetti e il flusso, e so dire cosa non è mio.

Se dovessi fare una pipeline da zero: source con branch protection; lint e test; build dell'immagine; scan di sicurezza; deploy su dev; smoke test; promozione con approvazione; deploy su prod; monitoraggio e rollback. I segreti fuori dal repository; IAM minimo; ambienti separati; ogni stage con un output verificabile. È un piano, non un'esperienza: lo dico come tale.

### Prova di padronanza

- So spiegare IaC come controllo e ripetibilità, non come moda: drift, revisione, ricreazione, rollback; e so dire il costo in apprendimento e gestione.
- So descrivere CDK con app, stack e construct, e distinguere `synth` — genera il template — da `deploy` — applica il template.
- So dire che CDK non è CloudFormation scritto a mano: è un'astrazione che produce CloudFormation, e il template è ciò che AWS esegue.
- So raccontare gli ancoraggi reali del progetto: stack sopra la libreria interna del cliente (`advanalib`), buildspec di synth con diff e di deploy, ruolo cross-account, versioni da Parameter Store.
- So descrivere una pipeline a stage con buildspec, artifact, ambienti, approvazioni e smoke test, e so dire perché ogni stage ha un output verificabile.
- So spiegare least privilege con esempi: `s3:GetObject` su un prefisso invece di `s3:*`; ruoli separati per funzione; ruolo di esecuzione ECS distinto dal ruolo del task.
- So spiegare il rollback e i suoi limiti: stack rollback, redeploy, risorse stateful, migrazioni dati, compatibilità tra versioni.
- So dire dove stanno i segreti e perché un segreto committato è compromesso anche se cancellato.
- So riconoscere i rischi operativi: CloudFormation rifiuta un secondo aggiornamento sullo stesso stack, quindi il problema è serializzare le esecuzioni della pipeline; il drift dalla console; il lock di stato è di Terraform.
- So confrontare CDK, CloudFormation e Terraform sul contesto, senza dire che uno è sempre meglio.
- So dire cosa ho fatto — integrazione applicazione, lettura dei buildspec — e cosa non ho fatto — CDK, IAM, pipeline.

### Cosa non dire

- "Ho fatto io CDK, IAM e la pipeline": non è vero, erano del team di piattaforma del cliente.
- "CDK è CloudFormation" senza distinguere synth e deploy: sono due passaggi diversi.
- Ignorare IAM e least privilege: è la parte che distingue un deploy che funziona da un deploy sicuro.
- Pipeline senza test, approvazioni o rollback: è automazione, non qualità.
- Dire "Terraform è meglio" o "CDK è meglio" senza contesto: dipende dal team, dal provider e dal cliente.
- Trattare i segreti come un dettaglio o metterli nell'immagine.
- Dire "il rollback è sempre possibile": con risorse stateful e migrazioni dati, non lo è.
- Raccontare un deploy o un incidente che non hai gestito.

### Se non lo so

Non ho progettato l'infrastruttura del cliente: CDK, IAM e pipeline erano del team di piattaforma, e io ho integrato l'applicazione e usato i buildspec. Non ho gestito un rollback o un incidente di deploy in prima persona, quindi non ti racconto una storia operativa. So spiegare i concetti e il flusso, e se mi chiedi un dettaglio di configurazione che non ricordo, preferisco dirlo.

---

## 1.14 — Validazione input con Pandera

### Risposta principale (3–5 minuti)

Pandera è una libreria per validare DataFrame. Il problema che risolve è semplice: un DataFrame è un contenitore senza contratto. Puoi avere una colonna che dovrebbe essere un intero e contiene stringhe, un importo che dovrebbe essere positivo e contiene zero, una chiave che dovrebbe essere unica e ha duplicati. Se il calcolo va avanti, il risultato è sbagliato e te ne accorgi a valle, quando è più costoso. Pandera ti permette di dichiarare il contratto e verificarlo all'ingresso.

Uno schema Pandera descrive colonne, tipi, nullabilità, vincoli e coercizioni. Puoi dire che una colonna è `int64`, che non ammette nulli, che i valori sono in un range, che una colonna è unica, che una relazione tra due colonne deve valere. `coerce` converte i tipi quando possibile; è comodo, ma va usato con cautela, perché una coercizione silenziosa può nascondere un dato sbagliato. La differenza tra "il dato è valido" e "il dato è diventato valido per forza" è importante.

Pandera non è Pydantic. Pydantic valida oggetti e JSON all'ingresso o all'uscita di un'API: forma, tipi, campi obbligatori. Pandera valida DataFrame: colonne, tipi, vincoli statistici o di dominio. Sono livelli diversi. I vincoli del database sono l'ultima linea di difesa per la persistenza: proteggono ciò che è scritto, non ciò che è calcolato in memoria. Usare uno al posto dell'altro è un errore di livello, non di strumento.

La politica di fallimento va decisa prima. Fail fast: il job si ferma al primo errore. Raccolta: si validano tutti i dati, si raccolgono tutti gli errori e si decide dopo. Quarantena: il dato non valido viene separato, il resto prosegue. Metriche: si registra il tasso di fallimento senza bloccare. La scelta dipende dal costo dell'errore e dal tipo di dato. Nel cash flow la scelta è la raccolta: la validazione è `lazy`, quindi Pandera raccoglie tutti gli errori in un'unica passata invece di fermarsi al primo, così chi carica un file Excel lo corregge una volta sola.

Il costo è a runtime. Validare un DataFrame grande occupa tempo e memoria: Pandera controlla i valori, non solo lo schema. Si può validare su un campione, su colonne critiche o in un punto solo della pipeline. Non è gratis: va misurato. Per dati piccoli è trascurabile; per big data può essere significativo. La domanda è: quanto costa un errore a valle rispetto al costo della validazione? Se il costo dell'errore è alto, la validazione si ripaga.

Nel cash flow l'ho usato per validare gli input, e la forma del progetto è questa. Ogni sorgente esterna ha un parser che valida con uno schema Pandera in modalità `lazy`, accumula gli errori per sorgente e li classifica: alcuni sono fatali e bloccano il calcolo, altri no. Gli errori di Pandera vengono tradotti in messaggi in italiano comprensibili per l'utente tramite classi dedicate; la validazione è uno step separato dell'esecuzione (`INPUT_VALIDATION`) e il risultato finisce nel record della run, così l'utente vede cosa correggere prima di lanciare il calcolo costoso. Non l'ho usato ovunque e non è una competenza che rivendico su tutti i progetti: ho usato lo strumento in un punto preciso, e questo è il fatto.

I limiti sono chiari. Pandera non sostituisce i test: valida il dato a runtime, i test verificano il codice e il comportamento. Non sostituisce i vincoli del database: protegge il dato in memoria, non la persistenza. Non è pensato per streaming o per dataset enormi senza un piano. E lo schema stesso può essere sbagliato: se dichiari un vincolo troppo stretto, blocchi dati validi; se lo dichiari troppo largo, lasci passare dati sporchi. Lo schema è un contratto, e come ogni contratto va testato e versionato.

L'evoluzione dello schema è un tema da non sottovalutare. Se aggiungi una colonna, i lettori devono tollerarla; se cambi un tipo, i dati storici possono rompersi. Lo schema va versionato e trattato come un'interfaccia. In una pipeline, un cambio di schema può rompere più a valle di un bug di calcolo, perché è silenzioso: il dato passa, ma con un'altra forma.

Il mio approccio, se dovessi progettarlo oggi: validare ai confini, con uno schema per input e uno per output critico; fail fast per i dati non validi; quarantena per i correggibili; metriche per monitorare; test dello schema con casi validi e invalidi. Non validerei tutto ovunque: sceglierei i punti in cui un errore costa di più. La validazione è un investimento, non un rituale.

### Versione da 60 secondi

Pandera valida i DataFrame con uno schema dichiarativo: colonne, tipi, nullabilità, vincoli. Non è Pydantic, che valida oggetti e API, né un vincolo di database, che protegge la persistenza: sono tre livelli diversi. Nel cash flow lo uso sugli input in modalità `lazy`: raccolgo tutti gli errori in un'unica passata, li classifico in fatali e non, li traduco in messaggi in italiano e li mostro prima del calcolo. Il costo è a runtime e va misurato; lo schema è un contratto da testare e versionare. L'ho usato solo nel cash flow.

### Follow-up

**D. Cos'è Pandera e perché usarlo?**

R. È una libreria di schema validation per DataFrame. Definisci il contratto dei dati — colonne, tipi, nullabilità, range, set di valori, unicità — e lo verifichi a runtime. Se il dato non rispetta lo schema, ottieni un errore con il dettaglio, invece di un risultato sbagliato a valle. Il valore è fermarsi presto e con un messaggio utile.

Perché usarlo invece di controlli manuali? Perché i controlli manuali sono sparsi, duplicati e difficili da mantenere. Uno schema è un contratto dichiarativo, testabile e riusabile. Se cambia il requisito, cambi lo schema in un posto; se un controllo manca, si vede. La differenza è tra "qualcuno ha controllato" e "il contratto è dichiarato".

Non è l'unico strumento: puoi usare Great Expectations, controlli custom o vincoli nel database. Pandera è leggero e si integra bene con pandas. La scelta dipende dall'ecosistema e dal tipo di dato. Non è "Pandera è meglio": è "per un DataFrame pandas, è diretto".

**D. Pandera vs Pydantic vs vincoli del database?**

R. Pydantic valida la forma dei dati in ingresso o uscita di un'API: oggetti, JSON, tipi, campi obbligatori. Pandera valida DataFrame: colonne, tipi, vincoli di dominio, relazioni tra colonne. I vincoli del database sono l'ultima linea di difesa per la persistenza: proteggono ciò che è scritto, non ciò che è calcolato in memoria. Sono tre livelli diversi, non alternative.

Un esempio: l'API riceve un oggetto con un campo `importo`; Pydantic verifica che sia un numero. Il DataFrame che arriva al calcolo ha una colonna `importo`; Pandera verifica che sia positivo e non nullo. Il database ha un vincolo `importo > 0`; protegge il dato salvato. Se manca uno dei tre, hai un buco: l'API può accettare un numero, il calcolo può produrre un negativo, il database può accettarlo se non ha il vincolo.

La scelta dipende da dove vuoi fermare l'errore. Più a monte è meglio, ma il controllo a monte non conosce il contesto del calcolo. Per questo i tre livelli convivono: ognuno protegge un confine diverso.

**D. Cosa controlli in uno schema e cosa lasci fuori?**

R. Controllo ciò che, se sbagliato, rompe il calcolo: tipi, nullabilità delle colonne critiche, range dei valori, valori ammessi per le categorie, unicità delle chiavi, coerenza tra colonne. Per un dato finanziario, anche segno e precisione. Non controllo tutto: controllare ogni colonna con ogni regola costa e produce rumore. Il criterio è il rischio a valle.

Lascio fuori i controlli che richiedono una conoscenza che lo schema non ha: per esempio, se un importo è "plausibile" rispetto al mercato, non è un vincolo di schema, è una regola di business. Lascio fuori anche i controlli statistici pesanti su grandi dataset, a meno che non siano il cuore del problema. Lo schema valida la forma e i vincoli; le regole di business vanno in test o in controlli separati.

Un errore comune è validare solo i tipi. Un DataFrame con i tipi giusti può avere valori assurdi: importi negativi, date nel futuro, chiavi duplicate. Il tipo è il minimo, non il contratto. Il contratto include i vincoli di dominio.

**D. `coerce`: quando è utile e quando è pericoloso?**

R. `coerce` converte i tipi quando possibile: una stringa "123" diventa un intero 123, una data testuale diventa un timestamp. È utile quando la sorgente è eterogenea — CSV, JSON, Excel — e vuoi normalizzare. È comodo perché evita di scrivere conversioni a mano e rende lo schema più robusto a piccole variazioni.

È pericoloso quando nasconde un dato sbagliato. Se una colonna dovrebbe essere un importo e contiene "N/A", `coerce` può diventare `NaN` o fallire, a seconda della configurazione; se contiene "1.000" con il punto delle migliaia, può diventare 1.0 invece di 1000. La coercizione silenziosa è un modo per far passare un dato che non rispetta il contratto. La regola è: coercizione esplicita, con una regola chiara, e un test sui casi ambigui.

In un pipeline finanziaria, preferirei fallire su un valore ambiguo piuttosto che convertirlo. In una pipeline esplorativa, la coercizione può essere accettabile. La scelta dipende dal costo dell'errore. `coerce` non è un dettaglio: è una politica.

Nel progetto la coercizione che c'è è quella di pandas, non di Pandera: i loader convertono le date con `pd.to_datetime(..., errors='coerce')`, che trasforma i valori illeggibili in `NaT` senza segnalarlo. È proprio il rischio descritto qui, e un punto che migliorerei: renderei esplicito l'errore, o almeno conterei e segnalerei quanti valori sono stati scartati.

**D. Cosa succede se la validazione fallisce?**

R. Dipende dalla politica, e nel cash flow è questa: gli errori vengono raccolti per sorgente in un'unica passata, tradotti in messaggi in italiano che dicono quale colonna e quale regola hanno fallito, e mostrati all'utente. Se tra gli errori ce n'è uno fatale, il calcolo non parte. Meglio fermarsi che propagare un dato sbagliato: un errore a monte è più facile da diagnosticare di un numero sbagliato a valle. Gli errori non fatali vengono segnalati ma non bloccano.

Se il dato è recuperabile, si può mettere in quarantena: separare le righe non valide, far proseguire il resto, e avvisare. La quarantena è utile quando il dato è parzialmente buono e il costo di fermare tutto è alto. Ma richiede un processo: chi esamina, chi corregge, come si reintegra. Senza processo, la quarantena è solo un posto dove i dati vanno a morire.

La politica va decisa con il business: un errore su un dato finanziario può richiedere stop; un errore su una dashboard esplorativa può richiedere un avviso. Non c'è una risposta universale. La cosa importante è che la politica sia esplicita, non improvvisata caso per caso.

**D. Qual è il costo a runtime?**

R. Validare un DataFrame grande costa tempo e memoria: Pandera controlla i valori, non solo lo schema. Per dati piccoli è trascurabile; per big data può essere significativo. Si può validare su un campione, su colonne critiche o in un punto solo della pipeline. La scelta va misurata, non decisa a priori.

Il costo dipende anche dal tipo di controllo: un controllo di tipo è economico, un controllo di unicità richiede ordinamento o hashing, un controllo tra colonne richiede di materializzare. Più il controllo è ricco, più costa. La domanda è: quanto costa un errore a valle rispetto al costo della validazione? Se il costo dell'errore è alto, la validazione si ripaga.

Non ho misurato il costo nel progetto, e non invento un numero. So dire che va misurato e come: un benchmark su un campione rappresentativo, con e senza validazione, e una stima del costo dell'errore. Senza, la decisione è un'opinione.

**D. Dove lo hai usato e dove no?**

R. Nel cash flow, per gli input: schemi Pandera in modalità lazy sui file delle sorgenti esterne, con errori raccolti, classificati e tradotti per l'utente. Non l'ho usato ovunque e non è una competenza che rivendico su tutti i progetti. Se l'intervistatore chiede "dove altro", la risposta onesta è "solo nel cash flow".

Non lo considero un punto debole grave, ma non lo considero nemmeno un punto di forza. È uno strumento che ho usato in un contesto, con una comprensione sufficiente a spiegarlo. Se dovessi progettare una strategia di validazione per un sistema più grande, studierei di più e farei un piano, non improvviserei.

**D. Pandera sostituisce i test?**

R. No. Pandera valida il dato a runtime; i test verificano il codice e il comportamento. Sono complementari: Pandera ti dice che il dato non rispetta il contratto, i test ti dicono che la trasformazione è corretta. Un test può usare uno schema Pandera per costruire input validi, ma non sostituisce l'asserzione sul risultato.

Un errore comune è pensare che "c'è lo schema, quindi sono coperto". Lo schema controlla la forma, non la logica: un importo può essere positivo e comunque sbagliato. I test coprono la logica, i casi limite e le regressioni. La validazione copre l'ingresso. Servono entrambi.

Lo schema stesso va testato: casi validi, casi invalidi, colonne mancanti, tipi sbagliati. Un schema troppo stretto blocca dati validi; uno troppo largo lascia passare dati sporchi. Il test dello schema è parte della qualità, non un extra.

**D. Come gestisci l'evoluzione dello schema?**

R. Con versionamento, compatibilità e test. Se aggiungi una colonna, i lettori devono tollerarla; se cambi un tipo, i dati storici possono rompersi. Lo schema è un contratto: va trattato come tale. In una pipeline, un cambio di schema può rompere più a valle di un bug di calcolo, perché è silenzioso.

La strategia: versionare lo schema, documentare le modifiche, mantenere la compatibilità all'indietro per un periodo, e testare i dati vecchi con lo schema nuovo. Se il cambio è breaking, serve una migrazione e un piano di rollback. Non si cambia lo schema "al volo" in produzione.

Non ho gestito l'evoluzione dello schema nel progetto: ho usato Pandera per gli input, non ho progettato un sistema di versionamento. So dire cosa farei, ma lo dico come piano, non come esperienza.

### Prova di padronanza

- So spiegare il problema: un DataFrame senza contratto può avere tipi, nulli e valori sbagliati, e l'errore emerge a valle quando costa di più.
- So descrivere uno schema Pandera: colonne, tipi, nullabilità, vincoli, relazioni, `coerce`; e so dire che lo schema è un contratto.
- So distinguere Pandera, Pydantic e vincoli del database: dato tabellare in memoria, API, persistenza; tre livelli, non alternative.
- So spiegare la differenza tra fail fast, quarantena e metriche, e so dire che la politica dipende dal costo dell'errore.
- So avvertire sul rischio di `coerce`: una coercizione silenziosa può nascondere un dato sbagliato.
- So dire che il costo è a runtime e va misurato, e che si può validare su un campione o su colonne critiche.
- So dire dove l'ho usato — input del cash flow — e dove no, senza rivendicare un'esperienza che non ho.
- So spiegare che Pandera non sostituisce i test né i vincoli del database, e che lo schema stesso va testato.
- So descrivere l'evoluzione dello schema come un problema di compatibilità e versionamento, non come un dettaglio.
- So progettare una politica di validazione ai confini, con criteri e compromessi, senza dire "validiamo tutto".

### Cosa non dire

- Confonderlo con Pydantic: Pandera valida DataFrame, Pydantic valida oggetti e API.
- Dire che sostituisce i vincoli del database: protegge il dato in memoria, non la persistenza.
- Validare solo i tipi e non i vincoli di dominio: un importo positivo è un contratto, non un tipo.
- Ignorare il costo a runtime e il rischio di `coerce`: sono le due cose che distinguono un uso consapevole da un uso ingenuo.
- Dire "c'è lo schema, quindi sono coperto": lo schema non copre la logica.
- Rivendicare utilizzi che non hai fatto o un'esperienza profonda che non hai.
- Dire "validiamo tutto" senza parlare di punti critici e costo.
- Trattare l'evoluzione dello schema come un dettaglio: è un cambio di contratto.

### Se non lo so

Non l'ho usato in produzione su tutti i progetti: nel cash flow, per gli input, e non è una competenza che rivendico come profonda. Non ho misurato il costo a runtime né gestito l'evoluzione di uno schema in un sistema grande, quindi non ti do numeri o pratiche che non ho applicato. So spiegare a cosa serve, i limiti e come lo integrerei, senza inventare altri utilizzi.

---

# Come usare questa versione

Questa versione non è pensata per essere letta dall'inizio alla fine come un manuale. È pensata per essere usata in tre modi: per orientarti, per raccontare e per simulare. La differenza rispetto alla versione precedente non è la lunghezza: è che ogni risposta dichiara il problema, il vincolo, la scelta, il costo e il failure mode. Se leggi solo le risposte principali, hai la struttura; se leggi i follow-up, hai il ragionamento; se leggi le prove di padronanza, hai i dettagli che distinguono un senior da un junior.

## Cosa è cambiato rispetto alla versione precedente

La versione precedente era una buona base, ma molte risposte erano da junior: una definizione, un esempio, una chiusura. Mancavano il contesto, il trade-off, il costo, il failure mode e la distinzione tra ciò che hai fatto tu e ciò che hanno fatto gli altri. In questa versione ogni argomento parte dal problema e dal vincolo, non dalla definizione. Ogni scelta è confrontata con un'alternativa. Ogni argomento ha una sezione "cosa non dire" e una sezione "se non lo so", perché la credibilità è parte della competenza.

La struttura è la stessa per tutti i 14 argomenti: risposta principale, follow-up, prova di padronanza, cosa non dire, se non lo so. Non è una gabbia: è un modo per assicurarti che non stai saltando una parte. Se un argomento ti viene naturale, puoi leggere solo i follow-up e la prova di padronanza. Se un argomento ti è ostico, leggi la risposta principale ad alta voce e poi prova a rispondere ai follow-up senza guardare.

## I tre marcatori

Mentre leggi, usa tre marcatori. Il primo è "so": la risposta ti viene naturale, con un esempio e un trade-off. Il secondo è "da rivedere": conosci il concetto ma non sai raccontarlo o ti manca un dettaglio. Il terzo è "non mio": è un tema che non hai fatto, e devi dirlo con onestà. La preparazione non consiste nel trasformare tutto in "so"; consiste nel trasformare "da rivedere" in "so" e nel rendere "non mio" una risposta dignitosa.

Non cancellare i "non mio". Sono la parte della preparazione che ti protegge quando l'intervistatore scava. Un candidato che sa dire "questo non l'ho fatto, ma so come lo verificherei" è più credibile di uno che ha una risposta per tutto. La sezione "se non lo so" di ogni argomento è lì per questo: non è una resa, è una formula.

## Piano di lettura in due settimane

**Giorni 1–2 — Orientamento.** Leggi le risposte principali di 1.1–1.7. Non prendere appunti dettagliati: segna solo i punti in cui ti blocchi. Alla fine dei due giorni, racconta a voce, senza guardare, i tre progetti: legacy, repricing, dashboard. Se non riesci a distinguere cosa hai fatto tu, torna su 1.1 e 1.2.

**Giorni 3–4 — Dati e pipeline.** Leggi 1.4, 1.5, 1.7 e 1.8. Sono gli argomenti in cui è più facile inventare: S3, Parquet, Athena, DynamoDB, SAS, Spark. Per ognuno, scrivi tre fatti che sai e tre cose che non sai. Se un "non so" è troppo grande, non provare a colmarlo in un giorno: impara la formula onesta e passa avanti.

**Giorni 5–6 — Architettura e UI.** Leggi 1.2, 1.6, 1.9 e 1.10. Sono gli argomenti in cui devi mostrare giudizio, non erudizione: perché una pipeline, perché MVP, perché DuckDB, perché una cache. Esercitati a dire "dipende da" seguito da un criterio, non da un'opinione.

**Giorni 7–8 — Orchestrazione, modeling, infrastruttura.** Leggi 1.11, 1.12, 1.13 e 1.14. Sono gli argomenti in cui il rischio è raccontare ciò che non hai fatto. Per ognuno, prepara una frase di confine: "l'architettura era dell'architetto", "l'infrastruttura era del team di piattaforma", "il modello non l'ho progettato". Poi prepara una frase su cosa sai fare comunque.

**Giorni 9–10 — Simulazione.** Fatti fare le domande da qualcuno, o registrati. Un argomento per sessione, cinque minuti di risposta principale, dieci minuti di follow-up. Riascolta e segna: dove hai divagato, dove hai inventato, dove hai detto "noi" invece di "io". Non correggere tutto insieme: scegli le tre cose che ti costano di più.

**Giorni 11–12 — Ripasso mirato.** Torna sui "da rivedere". Per ogni argomento, racconta la risposta principale in tre minuti, poi rispondi a tre follow-up scelti a caso. Se non riesci, non hai capito: rileggi la sezione, non memorizzare le frasi.

**Giorni 13–14 — Prova completa.** Una simulazione da un'ora: presentazione, due argomenti tecnici, un behavioral, domande finali. Registra. Riascolta. Poi smetti di studiare per un giorno: il riposo fa parte della preparazione, soprattutto per la voce.

## Come raccontare la risposta principale

La risposta principale non va recitata. Va costruita su una struttura che tieni in testa: problema, vincolo, scelta, meccanismo, trade-off, applicazione, failure mode, testing, cosa faresti diversamente. Non tutte le risposte hanno bisogno di tutti i passaggi, ma se ne mancano troppi, la risposta sembra un compendio.

La prima frase deve dire il problema o il vincolo, non "allora, il progetto...". Esempio: "Il problema era che un calcolo di minuti non poteva stare dentro una richiesta HTTP". Poi il meccanismo: "Abbiamo separato il comando dall'esecuzione: una Lambda lancia un task ECS". Poi il trade-off: "Il costo è più infrastruttura; il beneficio è che l'utente non aspetta e il job ha risorse dedicate". Poi il confine: "Io ho lavorato sul flusso applicativo, non sull'infrastruttura". Poi il failure mode: "Se la Lambda muore dopo il lancio, lo stato va riconciliato". Poi la chiusura: "È la soluzione che sceglierei a questa scala; con cento job concorrenti cambierei".

Se ti accorgi di aver parlato per due minuti senza un trade-off, fermati e aggiungilo. Se ti accorgi di aver detto "noi" per trenta secondi, correggi: "noi" va bene per il contesto, ma l'intervistatore vuole sapere cosa hai fatto tu. Se ti accorgi di non sapere un dettaglio, dillo: "questo non lo ricordo, ma so come lo verificherei".

## Come usare i follow-up

I follow-up sono la parte più utile, perché sono le domande che l'intervistatore farà davvero. Non leggerli come un elenco: scegli una domanda, chiudi il documento e rispondi a voce. Poi riapri e confronta. Se la tua risposta è più corta di due paragrafi, probabilmente stai dando una definizione, non un ragionamento.

Le domande ricorrenti sono: "perché non X?", "cosa succede se...?", "come lo testeresti?", "come scalerebbe?", "quale costo?", "cosa hai fatto tu?". Per ognuna, prepara una struttura. "Perché non X" richiede un'alternativa e un criterio, non un rifiuto. "Cosa succede se" richiede un failure mode e una mitigazione. "Come lo testeresti" richiede un caso, un'asserzione e un limite. "Come scalerebbe" richiede un collo di bottiglia e una misura. "Quale costo" richiede variabile, fisso e umano. "Cosa hai fatto tu" richiede un confine e un esempio.

## Griglia di autovalutazione

Per ogni argomento, datti un voto da 1 a 3 su cinque criteri. Non serve a giudicarti: serve a decidere dove investire il tempo.

1. **Problema e vincolo.** Sai partire dal problema, non dalla definizione? Voto 1 se parti dalla tecnologia; 2 se
parti dal problema ma non dal vincolo; 3 se il vincolo è chiaro e spiega la scelta.
2. **Meccanismo.** Sai spiegare come funziona, senza elencare nomi? Voto 1 se usi parole senza meccanismo; 2 se
spieghi il flusso ma non i dettagli; 3 se puoi rispondere a "cosa succede se" senza perderti.
3. **Trade-off.** Sai dire il costo della scelta e l'alternativa? Voto 1 se dici "è meglio"; 2 se citi un'alternativa
senza costo; 3 se spieghi quando cambieresti.
4. **Confine.** Sai dire cosa hai fatto tu e cosa no? Voto 1 se dici "noi" o "ho fatto tutto"; 2 se distingui ma ti
giustifichi; 3 se il confine è netto e non ti sminuisce.
5. **Failure mode.** Sai dire cosa si rompe e come te ne accorgi? Voto 1 se dici "funziona"; 2 se citi un errore
generico; 3 se hai un caso concreto e una mitigazione.

Se un argomento ha un 1, non è pronto: rileggi la risposta principale e i follow-up. Se ha un 2, è pronto per un colloquio, ma non per una domanda cattiva: esercitati sul "cosa succede se". Se ha un 3, puoi usarlo come punto di forza e come ponte verso gli altri argomenti.

## Ripasso a intervalli

Non ripassare tutto ogni giorno. Usa tre intervalli: dopo un giorno, dopo tre giorni, dopo una settimana. Al primo ripasso, racconta la risposta principale a voce. Al secondo, rispondi a tre follow-up. Al terzo, fai una simulazione con un argomento scelto a caso. Se un argomento torna "da rivedere" due volte, cambia metodo: non rileggere, insegna a qualcuno o scrivi un riassunto di dieci righe.

Il ripasso serve a rendere la risposta disponibile, non a memorizzarla. La differenza si sente: una risposta memorizzata è rigida e crolla alla prima deviazione; una risposta capita si adatta alla domanda. L'intervistatore bravo lo capisce subito.

## Simulazione: come farla bene

Una simulazione utile ha tre caratteristiche: è a voce, è cronometrata, è seguita da una riascolto. Se scrivi le risposte, stai allenando la scrittura, non il colloquio. Se non cronometri, parli troppo. Se non riascolti, ripeti gli stessi errori.

La struttura: cinque minuti di risposta principale, dieci minuti di follow-up, cinque minuti di behavioral, cinque minuti di domande tue. Per i follow-up, chiedi a chi ti ascolta di scegliere le domande più scomode: "perché non X?", "cosa succede se il job fallisce?", "cosa hai fatto tu?". Se non hai nessuno, registrati e fai le domande da solo, ma cambia tono: non addolcire.

Dopo la simulazione, segna tre cose: un punto in cui hai divagato, un punto in cui hai inventato o esagerato, un punto in cui hai detto "non lo so" bene. Il terzo è importante quanto gli altri: se non ne hai nessuno, forse stai raccontando una storia troppo liscia.

## Cosa aggiornare quando cambia il contesto

Questo documento è una fotografia del Ramo 1. Se cambi lavoro, se un progetto evolve, se impari una tecnologia nuova, alcune risposte cambiano. Le parti che restano sono i criteri: problema, vincolo, trade-off, failure mode, confine. Le parti che cambiano sono i fatti: i progetti, le tecnologie, i ruoli.

Quando aggiorni, non riscrivere tutto: cambia il fatto e mantieni il ragionamento. Se un progetto finisce, non cancellarlo: trasformalo in "cosa ho imparato" e "cosa farei diversamente". Se impari una tecnologia, non aggiungerla all'elenco: aggiungi un problema che sai risolvere con quella tecnologia e un caso in cui non la useresti.

## Gli errori da evitare nello studio

Il primo errore è leggere senza parlare. La lettura dà l'illusione di sapere; il colloquio richiede di produrre. Il secondo è memorizzare le frasi: suonano bene nella tua testa e male a voce. Il terzo è studiare solo i punti di forza: l'intervistatore trova il punto debole, e se non sei preparato crolli. Il quarto è gonfiare il ruolo: una storia gonfiata regge tre domande, non dieci. Il quinto è non riposare: la voce e la lucidità contano quanto il contenuto.

Il sesto errore è credere che una risposta senior sia una risposta lunga. Non lo è. È una risposta che dichiara il problema, la scelta, il costo e il confine, e si ferma. Se una frase non aggiunge nessuno di questi elementi, tagliala. Il silenzio dopo una risposta secca è meglio del rumore dopo una risposta lunga.

## Regola finale

Non recitare. Se una risposta non ti convince, non dirla. Meglio una frase onesta e precisa che un paragrafo inventato. L'intervistatore non cerca la perfezione: cerca qualcuno che sappia ragionare, che conosca i propri limiti e che sappia imparare. Questo documento è costruito per aiutarti a fare proprio questo: portare fatti reali, con confini reali, e ragionare su ciò che non sai senza paura di dirlo.
