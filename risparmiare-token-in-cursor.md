# Risparmiare token in Cursor

## Token, contesto e agenti: come spendere meno senza lavorare peggio

## Indice

1. [Token: che cosa sono](#1-token-che-cosa-sono)
2. [Contesto: definizione e composizione](#2-contesto-definizione-e-composizione)
3. [Ridurre il consumo nelle richieste](#3-ridurre-il-consumo-nelle-richieste)
4. [Come si forma il consumo](#4-come-si-forma-il-consumo)
5. [Gestire conversazioni lunghe e passaggi di fase](#5-gestire-conversazioni-lunghe-e-passaggi-di-fase)
6. [Prompt caching e cache hit](#6-prompt-caching-e-cache-hit)
7. [Modello, effort e Auto](#7-modello-effort-e-auto)
8. [Misurare il costo reale](#8-misurare-il-costo-reale)
9. [Leggere i benchmark](#9-leggere-i-benchmark-senza-farsi-ingannare)
10. [Procedura consigliata](#10-procedura-consigliata)
11. [Checklist rapida](#checklist-rapida)

## Introduzione

Quando chiedi a Cursor di scrivere o modificare codice, il modello AI che hai selezionato in Cursor elabora la richiesta e genera una risposta o una modifica. Oltre al testo che hai scritto, Cursor può inviargli la cronologia della chat, le istruzioni del progetto e i file coinvolti. Quando usi Agent, l'agente può fare altre chiamate al modello mentre esplora il codice o verifica il risultato. Il consumo di token dipende dal testo inviato, da quello generato e dal numero di chiamate. Prezzi e modalità di conteggio variano in base al piano; i dettagli aggiornati sono nella pagina [Models & Pricing](https://cursor.com/docs/models-and-pricing).

## In breve

- I token sono unità di testo definite dal tokenizer del modello; non corrispondono sempre a parole intere.
- Nel contesto possono entrare cronologia, file, regole, tool, risultati, MCP, skill e istruzioni di sistema.
- La *prompt specificity* indica quanto una richiesta sia concreta e attuabile.
- `Cache read` può sommare le letture di più chiamate interne e superare la finestra di una singola richiesta.
- Esplorazioni, chiamate ai tool, errori e retry aggiungono passaggi e possono aumentare il consumo.
- Disattiva MCP e skill che non servono al task. Per una modifica locale valuta Tab o l'editing inline; usa Agent se occorre esplorare o fare più passaggi.
- Una chat nuova separa task indipendenti, ma può richiedere di reinviare regole, file e stato.
- Skill mirate sono più semplici da mantenere di un grande file di regole sempre attivo. Il formato JSON, da solo, non riduce i token.
- Mantieni agente o modello durante una fase di lavoro. Se li cambi, passa decisioni e stato con un handoff.
- Confronta configurazioni che completano lo stesso task e considera costo, qualità, passaggi, tempo e rework.

## 1. Token: che cosa sono

I modelli linguistici dividono il testo in **token**, che possono corrispondere a parole brevi, parti di parole, spazi, segni di punteggiatura o simboli. Il risultato dipende dal modello, dal suo encoding e dalla lingua. Contare le parole dà quindi solo una stima approssimativa.

[OpenAI spiega](https://help.openai.com/en/articles/4936856-understanding-and-counting-tokens) che un token può rappresentare un carattere, una parte di parola, una parola o un segno di punteggiatura; inoltre il conteggio del testo semplice non coincide necessariamente con quello della richiesta completa, perché contano anche ruoli dei messaggi, tool, schemi, file e immagini.

Per l'italiano, il codice e i nomi tecnici le stime molto approssimative come "un token ogni qualche carattere" possono essere fuorvianti. Per dati reali usa il conteggio mostrato da Cursor o il tokenizer del modello interessato.

### Lingua e tokenizzazione

Alcuni tokenizer codificano l'inglese in modo più compatto, anche perché le sequenze inglesi frequenti possono essere meglio rappresentate nel loro vocabolario. La differenza cambia però con modello, tokenizer, testo e task. Per esempio, la stima dal 25% al 55% citata da Paul Simmering riguarda prompt giapponesi riscritti in inglese su alcuni modelli occidentali; non descrive prompt italiani in Cursor. Le tabelle che assegnano un rapporto fisso fra lingua e token sono indicative.

Tradurre ogni richiesta in inglese può far perdere sfumature e, se la traduzione la fa il modello, aggiunge un passaggio. Inoltre non riduce il contesto del repository, che può includere codice, commenti e documentazione. Se ripeti spesso lo stesso task, puoi confrontare due formulazioni equivalenti con il tokenizer del modello o con i dati di utilizzo di Cursor. Scegli la lingua in cui riesci a esprimere meglio requisiti e vincoli; non abbreviare identificatori o codice per ridurre il conteggio.

## 2. Contesto: definizione e composizione

### Finestra di contesto

La **finestra di contesto** è il numero massimo di token che il modello può considerare in una richiesta, compresi input e spazio riservato all'output. Una finestra più ampia permette di includere più materiale, ma non rende quel materiale automaticamente utile e non misura la qualità del risultato.

Cursor descrive ogni chat come una finestra che si riempie con file, conversazione e risultati dei tool. La vista del contesto può separare, tra gli altri elementi, prompt di sistema, tool, regole, skill, MCP, subagent, conversazione riassunta e conversazione originale ([Prompting agents](https://cursor.com/docs/agent/prompting)).

### Com'è composto il contesto

Il contesto può includere molto più dell'ultimo messaggio. A seconda del prodotto, della modalità e del task, comprende per esempio:

| Parte | Che cosa può contenere |
|---|---|
| Istruzioni | Prompt di sistema, regole del progetto, skill e preferenze attive. |
| Conversazione | Richieste e risposte precedenti, decisioni e sintesi della cronologia. |
| Repository | File e cartelle referenziati, codice recuperato con la ricerca, diff e stato del lavoro. |
| Tool e integrazioni | Definizioni e schemi degli strumenti, server MCP e risultati delle chiamate. |
| Lavoro dell'agente | Piani, output dei tool, log, errori e modifiche già proposte o applicate. |

Gli elementi presenti cambiano da una richiesta all'altra. In Cursor, la vista del contesto può mostrare prompt di sistema, tool, regole, skill, MCP, subagent e conversazione originale o riassunta ([Prompting agents](https://cursor.com/docs/agent/prompting)).

Il prompt engineering si occupa di come formulare le istruzioni. Il **context engineering** riguarda la scelta e l'aggiornamento di tutto ciò che il modello riceve: cronologia, file, tool, stato del repository, log e decisioni già prese.

[Anthropic definisce il contesto come l'insieme dei token disponibili durante l'inferenza](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) e suggerisce di curare il più piccolo insieme di informazioni ad alto segnale che permetta di ottenere il risultato. Il principio è generale e vale anche quando si lavora in Cursor.

### Intenzione e stato

È utile separare due tipi di informazione, distinti anche nella guida di Cursor [Working with Context](https://docs.cursor.com/en/guides/working-with-context):

- Intenzione: il risultato che vuoi ottenere, i vincoli da rispettare e il formato richiesto.
- Stato: situazione attuale, file coinvolti, errori, modifiche già fatte e test falliti.

Una richiesta utile descrive sia l'intenzione sia lo stato. "Sistema questo bug" lascia all'agente da ricostruire il problema; incollare il repository senza spiegare l'obiettivo aggiunge materiale senza chiarire che cosa fare.

### Context degradation: quando il contesto perde efficacia

Questo fenomeno è chiamato *context rot* o *context degradation*. Può comparire anche quando la finestra non è piena. Nel report tecnico di Chroma, esperimenti controllati su 18 modelli hanno rilevato prestazioni meno affidabili in diversi task al crescere dell'input, con un ulteriore calo quando venivano aggiunte informazioni distraenti. Non tutti i modelli partecipavano a ogni esperimento. Il report non dimostra un peggioramento lineare per ogni conversazione lunga e non fissa una soglia valida per Cursor. La guida di Morph applica il tema agli agenti di coding, ma include anche risultati e proposte legati ai prodotti dell'azienda: prima di generalizzare i suoi numeri, controlla metodo e ambito.

Nella pratica, conta soprattutto mantenere il contesto pertinente:

- Limita la ricerca ai file e ai risultati pertinenti.
- Conserva evidenze, decisioni e vincoli; togli tentativi superati e piste abbandonate.
- Fai un checkpoint quando cambia la fase del lavoro, anche se la finestra è ancora lontana dal limite.
- Dopo una sintesi, controlla che contenga i fatti e i riferimenti ai file necessari.
- Delega o separa la chat per esplorazioni indipendenti solo se puoi riunire i risultati in un elenco breve e verificabile. Più agenti possono aumentare il costo.

![Infografica sulla selezione del contesto: file, errori, vincoli e tool pertinenti passano nel prompt focalizzato; log completi, tool inutili e tentativi superati restano fuori.](assets/context-signal-not-volume.png)

*Figura. Ridurre il rumore aiuta a selezionare il contesto, ma non garantisce da solo un risultato più affidabile.*

La compattazione rende il contesto più maneggevole, ma può perdere dettagli e non corregge gli errori già introdotti. Conviene quindi ridurre il rumore e controllare le sintesi anche quando la finestra è ampia.

Per esempio, al posto di "Perché non funziona?", indica il simbolo e il comportamento: "Perché `validateLoginForm` in `src/auth/login.ts` accetta una password composta solo da spazi quando viene chiamata dal form di registrazione?" Un riferimento preciso può evitare ricerche e letture inutili. Se però alleghi file o spiegazioni più ampi del necessario, il risparmio svanisce.

## 3. Ridurre il consumo nelle richieste

### Scrivere richieste che riducono esplorazione e output

Una richiesta efficace chiarisce che cosa deve fare l'agente. Puoi usare questo schema e omettere i campi che non servono:

```text
Obiettivo:
  [una frase sul risultato da ottenere]

Ambito:
  [file, cartelle, componenti o API interessati]

Fuori ambito:
  [ciò che non deve essere modificato]

Contesto di stato:
  [errore, comportamento attuale, diff o vincoli tecnici]

Vincoli:
  [compatibilità, stile, dipendenze, sicurezza, performance]

Criteri di accettazione:
  - [...]
  - [...]

Verifica:
  [test, lint, comando o controllo manuale da eseguire]

Formato della risposta:
  [solo risultato, file modificati, test e blocchi]
```

### Prompt specificity: concretezza, non lunghezza

Una rubrica pratica divide la **Prompt Specificity** in tre livelli, a seconda di quanto la richiesta aiuta ad agire. È bassa se mancano riferimenti al codice, criteri di accettazione e vincoli. Sale quando la richiesta include uno di questi elementi; è alta quando combina i dettagli pertinenti o descrive chiaramente il risultato.

Per rendere la richiesta più precisa, aggiungi solo ciò che serve:

- Il risultato e il comportamento atteso.
- Il file, la funzione o il componente, se li conosci.
- Il comportamento attuale o i passaggi per riprodurre il problema.
- Vincoli e modifiche fuori ambito.
- Come verificare che il risultato sia corretto.

Per un micro-task basta anche "Rinomina `userID` in `userId` in questo file". Una richiesta lunga resta vaga se non dice che cosa ottenere. Usa il punteggio come indicazione e confrontalo con correttezza al primo tentativo, chiarimenti, chiamate ai tool, retry e costo per risultato. Aggiungere dettagli irrilevanti per alzarlo non aiuta.

![Schema di un flusso efficace: richiesta con obiettivo, file, vincoli e verifica; un agente attraversa una fase coerente; al checkpoint un handoff conciso può trasferire stato, decisioni, test e prossimo passo.](assets/specific-prompt-handoff.png)

*Figura. Una richiesta precisa riduce le ambiguità. Se cambi agente, il checkpoint chiarisce che cosa trasferire.*

### Ridurre l'output senza perdere informazioni utili

- Chiedi il diff e un riepilogo, senza ristampare i file modificati.
- Indica il formato della risposta.
- Richiedi spiegazioni estese quando ti servono per imparare o prendere una decisione.
- Se vuoi approvare il piano, chiedi all'agente di fermarsi dopo l'analisi.
- Definisci quando il task è concluso, così l'esplorazione ha un limite.
- Evita di far ripetere prompt e piano a ogni turno.
- Per una modifica locale, prova un modello rapido con contesto mirato.
- Per un task multi-file incerto, valuta un modello più capace se può ridurre retry e rework.

"Sii più breve" può accorciare la risposta visibile, ma non riduce per forza input, reasoning, esplorazione o chiamate ai tool. Per contenere il consumo, definisci bene l'ambito.

Per una correzione locale o un completamento breve, prova Cursor Tab o l'editing inline prima di avviare una sessione Agent con più passaggi. Tab suggerisce codice in base alle modifiche recenti, al contesto circostante e agli errori del linter ([documentazione Tab](https://prod.cursor.com/help/ai-features/tab)). Al 4 ottobre 2026, [Pro, Pro Plus e Ultra includono completamenti Tab illimitati](https://cursor.com/docs/models-and-pricing); Hobby ha dei limiti. Verifica le condizioni del tuo piano prima di scegliere questa opzione.

### Skill di brevità: poche regole, ambito chiaro

Una skill per risposte concise dovrebbe specificare che cosa tagliare e che cosa non sacrificare. Regole brevi e verificabili tendono a essere più facili da seguire di una lunga lista di divieti ed esempi. Per esempio:

```text
Rispondi in modo diretto. Elimina formule di cortesia e ripetizioni;
mantieni precisi termini tecnici, vincoli e incertezze importanti.
Per i task di codice riporta modifica, verifica ed eventuali blocchi.
Non alterare codice, citazioni o log da riportare fedelmente.
```

Limita la skill ai contesti in cui serve. Se resta sempre attiva, le istruzioni entrano anche nelle richieste che non richiedono uno stile telegrafico. In Cursor puoi circoscriverla ai file pertinenti con `paths` oppure richiamarla manualmente con `disable-model-invocation: true`. Tieni brevi le istruzioni iniziali e carica i dettagli solo quando servono. Evita regole come "non esprimere mai dubbi": accorciano la risposta, ma possono renderla meno precisa.

Una regola come "non eseguire comandi da terminale senza permesso" riguarda sicurezza e controllo. Può essere adatta al tuo flusso, ma le conferme aggiungono turni. Inseriscila nelle istruzioni persistenti se vuoi applicarla a tutti i task.

Un benchmark indipendente di Kuba Guzik confronta un prompt "caveman" da 552 token con una versione breve da 85 token. Nei test, condotti su due modelli Claude e due attività di sviluppo, la versione breve ha ridotto i token di output del 14% e del 21%; quella completa del 13% e del 9%. L'autore riporta risposte corrette in tutti i 72 tentativi. Il campione è ristretto e non mostra se i risparmi si ripetano con altri modelli o task ([metodo e risultati](https://dev.to/jakguzik/i-benchmarked-the-viral-caveman-prompt-to-save-llm-tokens-then-my-6-line-version-beat-it-2o81)). Le percentuali riguardano l'output, non il costo totale: nel confronto includi token della skill, input, tool call, reasoning e retry. Per capire se una versione breve conviene, confrontala con quella già in uso sugli stessi task e criteri di qualità.

### Selezionare il contesto con precisione

Quando sai dove intervenire, scegli il riferimento più utile:

- `@file` indica un file specifico.
- `@code` indica una funzione, classe o simbolo.
- `@folder` indica una cartella se gran parte dei suoi contenuti è pertinente.
- `@Git` o il diff corrente mostrano modifiche già esistenti.
- `@Terminals` porta nel contesto un errore o un log concreto.

La documentazione di Cursor su [@Files & Folders](https://docs.cursor.com/context/%40-symbols/%40-files-and-folders) distingue il riferimento a una cartella dal caricamento del suo contenuto completo. Quest'ultima opzione può aumentare molto i token di input, soprattutto con una cartella grande o con una modalità di contesto esteso.

Per restringere il contesto:

- Allega solo i file necessari, non l'intero workspace.
- Fai riferimento al file invece di incollarne una copia.
- Invia le righe dell'errore e il contesto vicino; evita i log completi se non servono.
- Parti dal test più mirato. Se l'output è lungo, condividi gli errori con le righe circostanti e conserva l'esito del comando originale.
- Escludi build, vendor, cache, documentazione generata e file binari non pertinenti.
- Usa `.cursorignore` o le impostazioni disponibili, poi verifica che i file necessari restino accessibili.
- Se non conosci ancora i file giusti, lascia esplorare Agent e indicagli obiettivo e area di ricerca.

### Ricerca del codice con grep in Cursor

La ricerca testuale trova stringhe o espressioni regolari e restituisce i file e le righe corrispondenti. Cursor Agent usa **Instant Grep**, un motore locale indicizzato, quando citi simboli precisi; supporta anche regex e confini di parola ([documentazione Cursor Search](https://cursor.com/docs/agent/tools/search)). Per esempio, `PaymentFailedError` trova le occorrenze esatte, mentre `import.*PaymentService` può individuare import che seguono uno schema.

La ricerca restringe il campo, ma non azzera i token. Quando Agent apre una corrispondenza, può inviare al modello il contenuto del file. Chiedi quindi percorsi e righe pertinenti, poi approfondisci solo i file necessari. Per esempio:

```text
Cerca `PaymentFailedError` nel progetto. Riporta i percorsi e le righe rilevanti;
apri solo i file necessari a capire dove viene generato e gestito l'errore.
```

La documentazione specifica che Instant Grep costruisce e interroga l'indice sulla macchina, senza caricare codice o percorsi per creare l'indice. Se Agent apre un file, il contenuto può comunque essere inviato al modello. Da terminale, l'equivalente con ripgrep è `rg -n "PaymentFailedError" src/`.

### Regole, skill, MCP e tool

Le istruzioni persistenti fanno risparmiare ripetizioni, ma regole, skill, schemi dei tool e cataloghi MCP possono aggiungere contesto a ogni richiesta. Tieni una regola se evita errori o rework che costerebbero più dei token che occupa.

Scrivile in modo che siano:

- brevi e coerenti tra loro;
- utili per decisioni che si ripetono;
- specifiche per il progetto o il team;
- prive di spiegazioni generiche già note all'agente;
- distinte dalle indicazioni valide per un singolo task.

Un catalogo MCP ampio può occupare contesto con descrizioni e schemi dei tool, anche prima che l'agente li invochi. Un utente Cursor ha riportato nel forum che, nella propria configurazione, prompt di sistema, definizioni dei tool, regole e skill occupavano circa 47.500 token di contesto iniziale senza file del repository: è un esempio personale, non un valore tipico garantito.

Per ridurre il contesto non necessario:

- Disattiva i server MCP che non servono al progetto.
- Nella chat puoi attivare o disattivare i singoli tool dalla lista degli strumenti.
- Se gestisci un server MCP, rendi i tool selezionabili e mantieni brevi descrizioni e schemi.
- Lascia attivi solo i tool utili al task. Così riduci il contesto e le chiamate accidentali.
- Riattiva gli altri tool quando il lavoro li richiede.

Anche le skill dovrebbero entrare nel contesto solo quando servono. Cursor permette di caricare le risorse in modo progressivo. Per limitarne l'impatto:

- Tieni `SKILL.md` focalizzato e sposta le spiegazioni lunghe in `references/`, da consultare quando servono.
- Usa `paths` per limitare la skill ai file a cui si applica.
- Imposta `disable-model-invocation: true` se vuoi caricarla solo quando la richiami con `/nome-skill`.
- Disattiva Custom Mode e istruzioni specialistiche quando la fase relativa è finita.

#### Skillset JSON annidati: conta la selezione, non il formato

In un [post del forum Cursor](https://forum.cursor.com/t/using-nested-json-skillsets-to-save-90-of-tokens-with-cursor/141399), un autore attribuisce al proprio sistema di skillset JSON annidati un risparmio dell'87% e miglioramenti di velocità e successo. È un'esperienza personale, non un benchmark indipendente. Il post descrive anche un selettore MCP proprietario che recupera le informazioni pertinenti: il possibile vantaggio viene dalla selezione, non dal formato JSON.

JSON può occupare più token di Markdown per via delle chiavi ripetute, delle parentesi e delle stringhe. I campi strutturati sono invece utili per schemi, contratti API e dati che un tool deve interpretare. Le sottocartelle, da sole, non fanno caricare i file in modo selettivo: serve un sistema che scelga o recuperi i contenuti. Cursor documenta skill native in `SKILL.md`, cartelle annidate, attivazione su richiesta e ambito tramite `paths`. Parti da queste funzioni. Aggiungi un indice JSON o un MCP se risolvono un problema concreto, poi misura anche il costo di descrizioni dei tool, ricerche e contenuti recuperati.

Nel suo articolo sulla *context pollution*, [Sam McLeod](https://smcleod.net/2025/08/stop-polluting-context-let-users-disable-individual-mcp-tools/) mostra quanto possano variare le descrizioni dei tool e sostiene il controllo granulare. I conteggi si riferiscono al suo ambiente, non sono misure ufficiali di Cursor. Le funzioni descritte qui sono documentate nelle guide Cursor [MCP integrations](https://prod.cursor.com/help/customization/mcp) e [Agent Skills](https://cursor.com/docs/skills).

## 4. Come si forma il consumo





### Come un agente fa crescere il consumo

Una richiesta a un agente può richiedere più prompt e risposte. In genere il ciclo è questo:

```text
obiettivo + istruzioni + contesto iniziale
                    ↓
                 modello
                    ↓
             chiamata a un tool
                    ↓
          risultato del tool nel contesto
                    ↓
           nuovo passaggio del modello
                    ↺
```

Durante il ciclo, l'agente può:

1. cercare file o simboli e leggere il codice pertinente;
2. proporre una modifica e applicarla;
3. eseguire test o comandi e leggerne l'output;
4. correggere gli errori e ripetere i passaggi necessari fino alla verifica.

I passaggi non hanno una categoria di token separata. Ognuno può aggiungere input, output, chiamate ai tool e materiale al contesto. Un modello economico per token può quindi costare di più sul task completo se richiede molte esplorazioni, retry o correzioni manuali.

Gli agenti in background o in parallelo possono aumentare il throughput, ma ciascuno avvia la propria esplorazione e usa contesto e tool. Usali per attività indipendenti quando il tempo risparmiato giustifica il costo aggiuntivo. Per una modifica piccola o ambigua, spesso basta un agente.

## 5. Gestire conversazioni lunghe e passaggi di fase

Quando la finestra si avvicina al limite, Cursor riassume o comprime parti della conversazione. Per file e cartelle usa strategie diverse ([Summarization](https://docs.cursor.com/en/agent/chat/summarization)). La sintesi permette di continuare, ma può perdere dettagli o rendere generica una decisione precisa.

Non c'è una soglia valida per tutti, come "al 50% apro una nuova chat". Osserva piuttosto se l'agente:

- ripete ricerche o confonde file con nomi simili;
- perde di vista vincoli e decisioni;
- ripropone soluzioni già scartate;
- dà risposte più vaghe o più lunghe e richiede altri retry;
- continua a portarsi dietro tentativi ormai superati.

La ricerca [Lost in the Middle](https://aclanthology.org/2024.tacl-1.9/) mostra che i modelli possono usare meno bene le informazioni collocate nel mezzo di contesti lunghi. Non descrive un comportamento specifico di Cursor, ma aiuta a capire perché aggiungere contesto non migliori sempre l'accuratezza.

### Quando conviene una nuova chat

Apri una nuova chat quando:

- inizi un task indipendente o il problema è cambiato;
- la cronologia contiene molte piste abbandonate;
- la sintesi non conserva più i dettagli necessari;
- vuoi confrontare due approcci in contesti separati.

Una chat separata tiene distinti obiettivi e decisioni. Il costo dipende però dai token e dai passaggi: una nuova chat può richiedere di reinviare regole, strumenti, file e stato, mentre un handoff lungo aggiunge altro input. Per proseguire lo stesso task, resta nella chat finché il contesto è utile. Riparti quando il rumore costa più della ricostruzione; la compressione automatica di Cursor rende comunque incerto il risparmio.

### Una chat per le fasi collegate della stessa feature

Se analisi, chiarimenti, soluzione e implementazione riguardano la stessa feature, puoi tenerli nello stesso thread. Chiedi prima un'analisi senza modifiche, poi chiarisci i punti emersi. Quando la soluzione è pronta, definisci criteri di accettazione e test, quindi implementa un ticket alla volta. Nei messaggi successivi aggiungi la domanda specifica senza ripetere informazioni già presenti.

### Mantenere continuità e fare handoff

La continuità riduce il rischio di rifare lavoro e aiuta a contenere i costi.

#### Che cosa significa "cambiare agente"

In Cursor, "cambiare agente" può indicare:

- cambiare modello dal model picker;
- passare da Ask ad Agent o a una Custom Mode, con tool e istruzioni diversi;
- aprire una nuova chat con una cronologia separata.

Cursor consente di cambiare modello durante una conversazione e applica la scelta ai turni successivi ([Prompting agents](https://cursor.com/docs/agent/prompting)). Il cambio quindi non è impossibile e non cancella necessariamente il contesto.

Evita di cambiare modello o agente a ogni turno o nel mezzo di una fase. Il passaggio può comportare alcuni costi:

1. Il nuovo modello deve ricostruire lo stato dalla cronologia o dalla sintesi; non vede il ragionamento interno del modello precedente.
2. Può avere capacità, regole, tool o un metodo di esplorazione diversi e rileggere file o ripetere verifiche.
3. Può interpretare diversamente decisioni provvisorie; anche contesto e cache disponibili possono cambiare.
4. Il lavoro duplicato può costare più del risparmio ottenuto scegliendo un modello meno caro.

Sono rischi, non regole assolute. Per esempio, puoi usare un modello per esplorare e passare a uno più capace per un'implementazione complessa o una revisione indipendente. Fai il cambio a un **confine di fase**, dopo aver salvato lo stato.

#### Strategia consigliata

Per ogni fase, mantieni lo stesso agente e segui questo ordine:

1. Esplora struttura, vincoli e causa.
2. Definisci approccio e criteri di accettazione.
3. Applica le modifiche.
4. Esegui i test e correggi gli errori.
5. Rivedi il diff e i rischi residui.

Puoi cambiare modello tra le fasi. Prima salva stato, decisioni e risultati delle verifiche; se cambi modalità, controlla anche tool e istruzioni attivi.

#### Handoff minimo

Prima di passare a un altro agente o a una nuova chat, chiedi un riepilogo in questo formato:

```text
Obiettivo:
Stato attuale:
File coinvolti:
Modifiche già fatte:
Decisioni prese e alternative scartate:
Vincoli da non rompere:
Test/comandi eseguiti e risultati:
Problemi aperti:
Prossimo passo esatto:
```

Chiedi al nuovo agente di leggere i file elencati e controllare diff e stato prima di intervenire. L'handoff deve restare più breve della cronologia e rimandare agli artefatti utili.

#### Rendere l'handoff una skill riutilizzabile

Puoi codificare il flusso in una skill richiamata manualmente, evitando di ripetere il prompt di handoff in ogni task. La skill di Matt Pocock salva un file temporaneo e rimanda agli artefatti esistenti. Serve a rendere il task **portabile**; non garantisce meno token. La sessione destinataria deve poter leggere il file o riceverne il contenuto tramite un canale accessibile. I documenti che devono restare nel repository vanno comunque salvati lì.

Esempio essenziale adattato alle skill di Cursor:

```markdown
---
name: handoff
description: Crea un handoff conciso per trasferire un task in corso a un'altra sessione o agente. Usalo solo su richiesta esplicita.
disable-model-invocation: true
---

# Handoff

Prepara un documento che permetta a una nuova sessione di continuare il task.
Se l'utente indica lo scopo della prossima sessione, focalizza il documento su quello.

- Controlla lo stato corrente del repository, il diff, i file rilevanti e gli esiti
  verificabili; non ricostruire lo stato solo dalla cronologia.
- Salva il documento nella directory temporanea del sistema operativo, fuori dal
  workspace, con un nome univoco. Non sovrascrivere file esistenti.
- Riporta obiettivo e prossimo focus, stato (completato/in corso), decisioni e
  motivazioni, file o artefatti da consultare, verifiche già eseguite e risultati,
  problemi aperti, rischi e prossimo passo concreto.
- Aggiungi una sezione "Skill suggerite" solo con skill pertinenti per il seguito.
- Non duplicare specifiche, piani, ADR, issue, commit o diff: cita i relativi
  percorsi o URL. Non copiare l'intera conversazione.
- Rimuovi segreti e dati personali non necessari. Non inventare fatti mancanti:
  segnala l'incertezza.
- Rileggi il file per verificarne completezza, brevità e assenza di dati sensibili;
  poi comunica il percorso salvato.
```

Con `disable-model-invocation: true`, la skill entra nel contesto solo quando la richiami. Cursor documenta i campi `name`, `description`, `paths`, `disable-model-invocation`, `icon`, `color` e `metadata`, ma non `argument-hint`, presente nell'esempio originale. Per trasferire il focus del task, specificalo nel testo di richiamo e controlla quali campi supporta il tuo ambiente.

#### Quando cambiare è sensato

Il cambio può servire quando:

- un modello rapido ha concluso la ricognizione e serve un modello più capace per design o refactoring;
- vuoi una review indipendente su un commit o diff stabile;
- il modello corrente è bloccato e vuoi provare un'altra strategia;
- vuoi separare implementazione e revisione.

Una risposta lenta o una tariffa per token più bassa non bastano a giustificare il cambio. Confronta il costo dell'intero task, comprese riletture, retry e correzioni manuali.

## 6. Prompt caching e cache hit

### Come funziona il prompt caching

Il **prompt caching** riutilizza il calcolo che il modello ha già fatto su una parte iniziale e stabile del prompt. In genere il provider conserva gli stati interni (*KV cache*) e li riusa quando una richiesta successiva ripresenta lo stesso prefisso. Il modello elabora comunque la richiesta e genera una nuova risposta.

La **cache semantica** funziona in modo diverso: cerca una domanda uguale o simile e può restituire una risposta già memorizzata, evitando una nuova chiamata al modello. Prompt caching e cache semantica operano quindi a livelli diversi.

Le metriche principali sono:

- **Cache write:** registra un prefisso idoneo per riutilizzarlo in seguito.
- **Cache read:** una richiesta successiva trova e riusa quel prefisso.
- **Input non cached:** la parte nuova o diversa dal prefisso viene elaborata normalmente.

Nelle cache di memoria, un *hit* indica che il dato richiesto è già disponibile; un *miss* significa che va recuperato o ricalcolato ([introduzione generale ai cache hit](https://www.geeksforgeeks.org/computer-organization-architecture/cache-hits-in-memory-organization/)). Nel prompt caching si riusa il calcolo del prefisso e il modello elabora i dati nuovi. Per ottenere un hit conta la corrispondenza del prefisso secondo le regole del provider: richieste solo simili nel significato non bastano.

![Schema del prompt caching: la prima richiesta scrive il prefisso stabile; una richiesta successiva riusa il prefisso compatibile, elabora i dati nuovi e genera una risposta nuova.](assets/prompt-caching-flow.png)

*Figura. Il prompt caching riusa il calcolo del prefisso; il modello genera una nuova risposta.*

Le tariffe di write e read dipendono dal provider e dal modello; in alcuni casi cambia anche il prezzo in base alla durata della cache. Nella tabella Cursor consultata il 4 ottobre 2026, GPT-5.6 Terra costa $2 per milione di token input, $2,50 per milione di cache write e $0,20 per milione di cache read. Le tariffe Anthropic per la scrittura cambiano con la durata. Sono valori datati: controlla [Models & Pricing](https://cursor.com/docs/models-and-pricing) prima di confrontarli.

#### Perché i cache read possono superare la finestra di contesto

Una risposta nel forum ufficiale di Cursor spiega che il dato mostrato per una richiesta può sommare le chiamate al modello fatte dall'agente durante quel turno. Per esempio, con 20.000 token iniziali e dieci chiamate, la prima può contabilizzare circa 20.000 token input. Se le successive riusano il prefisso, possono aggiungere circa 180.000 cache read. Il dashboard mostra così oltre 200.000 token, anche se nessuna chiamata supera da sola quella finestra.

Un valore alto di `cache read` non indica per forza un singolo prompt enorme o token fatturati al prezzo pieno. Può riflettere il riuso del contesto in molti passaggi; con una tariffa scontata, il costo marginale può essere basso. Conviene comunque capire perché il task abbia richiesto tante chiamate e se tool o contesto iniziale siano troppo ampi.

Per interpretare il dato, controlla:

- costo effettivo e modello usato;
- quantità di cache write e cache read;
- numero di chiamate, tool e passaggi;
- dimensione e categorie del contesto iniziale;
- risultato, retry e tempo impiegato.

Con le tariffe riportate sopra, un prefisso di 100.000 token scritto una volta e letto nove volte costerebbe circa $0,25 per il write e $0,18 per i read. Dieci input non cached della stessa dimensione costerebbero $2,00. Il calcolo non include output, nuovi token, limiti o condizioni del piano. Per questo il totale dei cache read non equivale al costo dello stesso numero di token input ordinari.

#### Quando la cache non si riutilizza

Il prompt caching richiede in genere un prefisso identico fino al punto memorizzato, una dimensione minima e una richiesta entro il periodo di conservazione. Cambiare modello, tool, istruzioni o contenuto precedente può ridurre la parte riutilizzabile. I requisiti dipendono dal provider e dal modello.

Per le integrazioni via API, le guide ufficiali OpenAI e Anthropic consigliano di tenere all'inizio del prompt il contenuto stabile e di aggiungere dopo i dati variabili. Raccomandano anche di misurare hit, miss e costi. In Cursor l'utente non controlla tutta l'orchestrazione: questi principi aiutano a capire la cache, ma non garantiscono che una modifica aumenti il cache hit rate.

Se controlli direttamente il sistema, puoi aumentare le possibilità di riuso così:

- Metti istruzioni e riferimenti condivisi prima dei dati variabili.
- Aggiungi le nuove informazioni in fondo invece di riscrivere la cronologia.
- Quando puoi, lascia invariati modello, tool e relativi schemi.
- Sposta in fondo timestamp, stato aggiornato e altri dati che cambiano spesso.
- Misura tariffe e numero di riusi: scrivere in cache un prefisso usato una sola volta può non convenire.

Le soglie minime, la durata e i punti di cache cambiano in base al modello. Anche un prompt apparentemente identico può non produrre un hit, per via delle regole di caching o del routing del provider. In Cursor queste indicazioni servono a capire il meccanismo, ma non sono impostazioni che controllano tutta l'orchestrazione. Per una panoramica introduttiva, leggi [Why Care About Prompt Caching in LLMs?](https://datacream.substack.com/p/why-care-about-prompt-caching-in) di Maria Mouschoutzi, PhD; per i dettagli tecnici consulta le guide dei provider.

La discussione sul [forum Cursor](https://forum.cursor.com/t/why-does-cursor-consume-an-absurd-amount-of-cache-read-tokens/151439) chiarisce come possono essere aggregati i contatori, ma non spiega ogni anomalia né esclude errori di visualizzazione. Se i costi non tornano, conserva gli ID delle richieste e chiedi a Cursor di verificare il caso.

### Favorire i cache hit in Cursor

Come descritto nella sezione 5, puoi mantenere un thread per le fasi collegate di una feature e aggiungere domande man mano. Questo può conservare un prefisso riutilizzabile, a seconda di ciò che viene inviato, del provider, del modello e della durata della cache. Il caching è automatico e nessuna frase garantisce un hit.

Quando puoi, mantieni stabili modello, tool e ordine delle istruzioni. I riferimenti @file e i percorsi selezionano il contesto, ma la cache dipende dal contenuto e dal prefisso inviati. Se cambia un file, può cambiare la parte del prompt successiva; non per questo si invalida automaticamente tutta la cache della conversazione.

Apri una nuova chat per un task indipendente o quando la cronologia diventa rumorosa. Non farlo a ogni follow-up per inseguire gli hit. Separare analisi e scrittura può aiutare a rivedere le modifiche, ma non aumenta in modo affidabile il cache hit rate. Le chiamate ripetute possono comunque sommarsi nel dashboard, come spiega lo [staff Cursor](https://forum.cursor.com/t/why-does-cursor-consume-an-absurd-amount-of-cache-read-tokens/151439).

### Leggere le metriche cache di Cursor

Per confrontare `Cache Read` e `Cache Write`, fissa il modello e verifica quale abbia gestito le richieste in Auto. Lo staff Cursor ha spiegato che queste colonne mostrano la cache in stile Anthropic. Un modello instradato verso un altro provider può usare un meccanismo diverso o non avere caching, e quindi non comparire nello stesso modo. Uno zero nelle colonne non esclude altre forme di riuso. Routing e rendicontazione dei token possono cambiare le categorie mostrate; i thread del forum descrivono casi e versioni specifici ([chiarimento su Auto e cache](https://forum.cursor.com/t/auto-mode-prompt-caching-not-working/154654), [variazione delle categorie riportate](https://forum.cursor.com/t/sudden-change-in-token-cache-usage-after-subscription-renewal/173529)).

## 7. Modello, effort e Auto

Valuta modello e impostazioni rispetto al lavoro da svolgere:

| Tipo di lavoro | Impostazione iniziale | Quando salire di livello |
|---|---|---|
| Domanda, ricerca locale o modifica piccola | Modello rapido, contesto mirato, effort basso o medio. | Se il modello interpreta male il codice o richiede retry. |
| Bug circoscritto | Modello rapido con errore riproducibile e test indicato. | Se la causa attraversa più componenti o il primo approccio fallisce. |
| Refactoring multi-file | Modello più capace e piano esplicito. | Se servono ragionamento architetturale o molte dipendenze. |
| Task ripetibile | Modello ed effort fissi per rendere il confronto riproducibile. | Dopo aver misurato qualità e costo su più esempi. |
| Esplorazione iniziale | Auto o modello economico. | Al checkpoint, se l'implementazione richiede ragionamento più profondo. |

Cursor Router offre modalità Auto orientate a **Cost**, **Balance** e **Intelligence**. Disponibilità e comportamento dipendono dalla versione e dal piano. Auto può cambiare modello tra richieste; è comodo per iniziare, ma rende meno rigorosi i confronti tra configurazioni.

Usa un effort alto quando il task richiede più ragionamento. Per capire se conviene, confronta task equivalenti e misura il costo per risultato riuscito.

Allarga la finestra di contesto quando il task richiede file o specifiche che non entrano in quella predefinita. Il costo dipende dai token usati e dal piano. Una finestra più ampia può includere materiale irrilevante e rendere meno visibile ciò che serve.

**Max Mode** è disponibile solo nei piani legacy con fatturazione a richieste, secondo la documentazione consultata. Estende la finestra e costa il prezzo API del modello più il 20%. Nei piani basati sull'utilizzo, la dimensione del contesto si sceglie dal model picker. Controlla il tuo piano, parti dalla finestra predefinita e allargala se il task lo richiede ([Max Mode on legacy plans](https://prod.cursor.com/help/ai-features/max-mode), [Models & Pricing](https://cursor.com/docs/models-and-pricing)).

## 8. Misurare il costo reale

### Le categorie principali di consumo

| Categoria | Che cosa comprende | Perché conta |
|---|---|---|
| **Input** | Prompt, cronologia, istruzioni, regole, file, immagini, tool e risultati precedenti reinseriti nel contesto. | È spesso la parte più grande nelle conversazioni lunghe e nei task multi-file. |
| **Cache read** | Token di un prefisso già elaborato e riutilizzato in una richiesta successiva. | Hanno una tariffa specifica, spesso più bassa dell'input normale. Il contatore può sommare letture ripetute su più chiamate. |
| **Cache write** | Token di input inseriti o aggiornati nella cache del provider. | La tariffa varia per modello e durata della cache; può essere più alta dell'input normale. |
| **Output** | Testo della risposta, codice, diff, argomenti e contenuto generato per i tool. | Le tariffe possono essere molto diverse da quelle dell'input. |
| **Reasoning** | Token usati internamente dai modelli che supportano il ragionamento, anche se non sono mostrati come testo della risposta. | Una risposta visibile breve può avere avuto un lavoro interno più lungo. La contabilizzazione dipende dal modello. |

Le categorie e le tariffe cambiano tra provider. Per OpenAI, ogni token viene conteggiato secondo la categoria a cui appartiene, come input, cache read o cache write. `Cache write` non è una sovrattassa da aggiungere al prezzo dell'input. Controlla la tabella del modello e del piano che usi.

Nomi e categorie possono variare tra prodotto, modello e piano. L'[SDK di Cursor](https://cursor.com/docs/sdk/typescript) e le pagine Usage possono mostrare più metriche; per la fatturazione, fai riferimento alla documentazione del tuo piano.

### Costo e limite tecnico

Un task può costare molto senza esaurire la finestra, per esempio se l'agente fa molti passaggi. Può anche raggiungere il limite con un costo contenuto, se il modello ha tariffe basse o parte dell'input è in cache.

Il costo effettivo dipende almeno da:

```text
costo del task ≈
somma di input nuovi + cache write + cache read + output/reasoning
su tutte le richieste del task,
con le tariffe del modello e del piano in uso
```

La formula serve a ragionare sul costo, non a ricostruire la fattura. Cursor applica pool e regole diversi a seconda del piano, del modello e dell'eventuale routing Auto. La pagina [Models & Pricing](https://cursor.com/docs/models-and-pricing) spiega che Auto addebita ogni richiesta in base al modello scelto e che le tariffe possono cambiare.

### Misurare il costo reale

Per confrontare gruppi di task, consulta Usage nell'editor o il dashboard Cursor. Le metriche disponibili e il percorso dell'interfaccia dipendono dal piano e possono cambiare ([Models & Pricing](https://cursor.com/docs/models-and-pricing)). Considera il costo effettivo insieme ai dati qui sotto, non il solo numero di messaggi.

| Metrica | Domanda a cui risponde |
|---|---|
| Input tokens | Quanto contesto nuovo sto inviando? |
| Cache read | Quanto contesto è stato riutilizzato e quante chiamate possono averlo riletto? |
| Cache write | Quanto contesto è stato scritto o aggiornato nella cache, e a quale tariffa? |
| Output e reasoning | Quanto lavoro genera il modello? |
| Tool call e steps | Quante iterazioni sono necessarie? |
| Retry | Quante volte devo correggere o ripetere? |
| Tempo | Quanto dura il percorso completo? |
| Rework manuale | Quanto lavoro aggiuntivo faccio io? |
| Costo | Quanto consumo effettivamente? |
| Esito | Il task è corretto e verificato? |

La metrica più utile è:

```text
costo per risultato riuscito =
costo totale di tutte le richieste e retry
÷ numero di risultati corretti e verificati
```

Confronta task simili con gli stessi criteri di successo e verifica. La lunghezza della risposta visibile non basta a stimare l'efficienza: contano anche file, struttura dei messaggi, cache, reasoning e chiamate ai tool. Il totale `cache read` nel dashboard può sommare le letture di più passaggi interni.

## 9. Leggere i benchmark senza farsi ingannare

Usa i benchmark per formulare ipotesi. Per capire se valgono nel tuo repository, confronta:

1. Qualità o pass rate.
2. Costo medio per task.
3. Token e numero di passaggi.
4. Tempo e affidabilità.
5. Tipo di task e metodo di valutazione.

[CursorBench](https://cursor.com/cursorbench) è il benchmark più vicino al prodotto e riporta score, costi, token e passaggi. [DeepSWE](https://deepswe.datacurve.ai/) misura task di software engineering a lungo raggio. I risultati non sono direttamente confrontabili perché dataset, harness e criteri possono differire.

### Frontiera tra costo e risultato

![Grafico benchmark con score percentuale sull'asse verticale e costo medio per task sull'asse orizzontale, con una configurazione evidenziata in verde.](assets/benchmark-frontiera-costo-score.png)

*Figura 1. La frontiera mostra il compromesso tra score e costo; valuta anche token, passaggi e variabilità.*

### Costo e qualità non crescono sempre insieme

![Grafico benchmark che confronta configurazioni di modelli con score vicino e costi medi per task differenti.](assets/benchmark-score-costo.png)

*Figura 2. Configurazioni con score simile possono avere costi medi molto diversi.*

![Grafico benchmark che mostra la relazione tra score, costo medio per task e livelli diversi di effort.](assets/benchmark-effort-costo.png)

*Figura 3. Scegli l'effort in base alla qualità e al costo totale.*

I grafici mostrano come leggere i dati presenti nel repository. Non sono una tabella prezzi: modelli, tariffe, pool e modalità di Cursor possono cambiare.

## 10. Procedura consigliata

1. Definisci obiettivo, ambito, vincoli e criteri di accettazione. Decidi anche come verificare il risultato.
2. Scegli modello e agente per la fase, poi mantienili fino al checkpoint.
3. Fornisci solo il contesto utile: riferimenti `@file`, `@code`, `@folder`, diff o terminale.
4. Per modifiche locali usa Tab o editing inline; scegli Agent quando servono esplorazione, tool o più passaggi. Chiedi prima un piano se il task è ambiguo o rischioso.
5. Limita l'esplorazione all'area necessaria. Controlla regole, skill, MCP e tool attivi e disattiva quelli che non servono.
6. Verifica il risultato con test o controlli concreti: una risposta plausibile non basta.
7. Se il contesto diventa rumoroso, riassumi le decisioni o crea un handoff. Usa una chat separata per un task indipendente; cambia agente o modello a un confine di fase e indica che cosa va ricontrollato.
8. Scegli la finestra adatta al piano e al task. Attiva opzioni estese quando il lavoro richiede davvero più contesto.
9. Misura token, cache read/write, passaggi, retry, tempo, rework e costo per risultato.
10. Aggiorna regole e documentazione con informazioni riutilizzabili, non con il riepilogo di una singola chat.

## Checklist rapida

Prima di inviare:

- [ ] Ho descritto il risultato che mi serve?
- [ ] Ho indicato i file coinvolti e ciò che è fuori ambito?
- [ ] Ho fornito l'errore o lo stato attuale del sistema?
- [ ] Ho spiegato come verificare il risultato?
- [ ] Ho dato riferimenti e criteri sufficienti per agire?
- [ ] Ho allegato solo il contesto pertinente?

Durante il lavoro:

- [ ] L'agente segue ancora l'obiettivo iniziale?
- [ ] Sta ripetendo ricerche o tentativi già fatti?
- [ ] Tool call e retry aumentano senza far avanzare il lavoro?
- [ ] MCP, skill e regole attivi servono al task?
- [ ] Cerco prima simboli ed errori specifici e apro solo i file necessari?
- [ ] Distinguo il volume dei `cache read` dal costo e dalle dimensioni di una singola finestra?
- [ ] Se è cambiato modello o modalità, ho preparato un handoff?
- [ ] Tool e istruzioni giustificano il contesto che occupano?

Alla fine:

- [ ] Ho eseguito i test o i controlli previsti?
- [ ] Il diff resta nell'ambito richiesto?
- [ ] Ho incluso retry e passaggi intermedi nel costo totale?
- [ ] Se il lavoro continua in un'altra chat, ho salvato decisioni e stato?

## Fonti e risorse

Fonti consultate dal 4 al 7 ottobre 2026. Interfacce, modelli, prezzi, modalità e limiti di Cursor possono cambiare; verifica la documentazione aggiornata.

- [Cursor Models & Pricing](https://cursor.com/docs/models-and-pricing): modelli, pool di utilizzo, Auto e tariffe.
- [Cursor Prompting agents](https://cursor.com/docs/agent/prompting): categorie del contesto, @ mentions, modalità e cambio modello durante la chat.
- [Cursor @Files & Folders](https://docs.cursor.com/context/%40-symbols/%40-files-and-folders): riferimenti mirati e contenuto delle cartelle.
- [Cursor Summarization](https://docs.cursor.com/en/agent/chat/summarization): sintesi delle conversazioni e condensazione di file e cartelle.
- [Cursor Working with Context](https://docs.cursor.com/en/guides/working-with-context): contesto di intenzione e di stato, oltre alla ricerca mirata.
- [Cursor TypeScript SDK](https://cursor.com/docs/sdk/typescript): metriche di utilizzo disponibili nell'SDK.
- [OpenAI - Understanding and counting tokens](https://help.openai.com/en/articles/4936856-understanding-and-counting-tokens): token, input, output, cache, reasoning e limiti di contesto.
- [Paul Simmering - Every Trick to Save Token Costs](https://simmering.dev/blog/save-token-costs/): strategie e compromessi descritti dall'autore. Le stime dipendono da modelli ed esempi e non prevedono direttamente l'uso di Cursor.
- [PromptCost.org - LLM Tokenization Explained](https://promptcost.org/en/blog/llm-tokenization-explained/): panoramica e stime illustrative sul rapporto fra lingue e token. Le percentuali non sono benchmark universali o specifici per Cursor.
- [Chroma - Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://www.trychroma.com/research/context-rot): report tecnico del 2025 con esperimenti controllati su 18 modelli; i risultati riguardano i task studiati.
- [Morph - Context Rot](https://www.morphllm.com/context-rot): sintesi applicata ai coding agent, da leggere distinguendo la ricerca dalle affermazioni e dai risultati sui prodotti dell'autore.
- [Matt Pocock - Handoff skill](https://github.com/mattpocock/skills/blob/main/skills/productivity/handoff/SKILL.md): esempio di skill che crea un handoff temporaneo e rimanda agli artefatti esistenti.
- [DeepWiki - Handoff](https://deepwiki.com/mattpocock/skills/7.1-handoff): panoramica del flusso e dei suoi casi d'uso.
- [OpenAI - Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching): riuso del prefisso, condizioni di cache hit, tariffe e diagnostica.
- [Anthropic - Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching): cache read/write, durata, tariffe e prefissi stabili.
- [Anthropic - Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): selezione del contesto, compaction e gestione dei task a lungo raggio.
- [Cursor Forum - Why does Cursor consume an absurd amount of cache read tokens?](https://forum.cursor.com/t/why-does-cursor-consume-an-absurd-amount-of-cache-read-tokens/151439): thread community sull'aggregazione delle chiamate per richiesta.
- [Cursor Forum - Auto mode: Prompt caching not working](https://forum.cursor.com/t/auto-mode-prompt-caching-not-working/154654): risposta dello staff sul routing Auto e sulle colonne Cache Read/Write. Il comportamento dipende da modello e versione.
- [Cursor Forum - Sudden change in token/cache usage after subscription renewal](https://forum.cursor.com/t/sudden-change-in-token-cache-usage-after-subscription-renewal/173529): caso in cui lo staff collega una variazione delle categorie al modello instradato da Auto.
- [Cursor Forum - Cursor high token usage](https://forum.cursor.com/t/cursor-high-token-usage/156924): risposta dello staff sul contesto reinviato tra chiamate e passaggi Agent; le esperienze degli utenti non sono benchmark generali.
- [Cursor MCP integrations](https://prod.cursor.com/help/customization/mcp): attivazione e disattivazione di server e singoli tool MCP.
- [Cursor Agent Skills](https://cursor.com/docs/skills): caricamento progressivo, attivazione esplicita e ambito delle skill.
- [Cursor Tab completion](https://prod.cursor.com/help/ai-features/tab): comportamento e impostazioni dei suggerimenti inline.
- [Cursor Max Mode on legacy plans](https://prod.cursor.com/help/ai-features/max-mode): disponibilità e fatturazione di Max Mode nei piani legacy.
- [Cursor - Saving Tokens in Cursor](https://note.com/travelnatsu/n/ne7d9c9306313?hl=en): raccomandazioni personali su ambito, regole, Max Mode e log. La pagina segnala che la versione inglese è tradotta automaticamente.
- [Cursor Forum - Using nested JSON Skillsets to save 90% of tokens](https://forum.cursor.com/t/using-nested-json-skillsets-to-save-90-of-tokens-with-cursor/141399): proposta community. Il risparmio dipende da un sistema e strumenti personalizzati, non è un risultato generalizzabile del formato JSON.
- [Sam McLeod - Stop Polluting Context](https://smcleod.net/2025/08/stop-polluting-context-let-users-disable-individual-mcp-tools/): prospettiva indipendente sui costi di contesto delle definizioni MCP.
- [MyEngineeringPath - LLM Caching](https://myengineeringpath.dev/genai-engineer/llm-caching/): panoramica secondaria su prompt cache, KV cache e cache semantica.
- [GeeksforGeeks - Cache Hits in Memory Organization](https://www.geeksforgeeks.org/computer-organization-architecture/cache-hits-in-memory-organization/): spiegazione generale degli hit e miss in una cache di memoria, usata qui come analogia.
- [Kuba Guzik - I Benchmarked the Viral “Caveman” Prompt](https://dev.to/jakguzik/i-benchmarked-the-viral-caveman-prompt-to-save-llm-tokens-then-my-6-line-version-beat-it-2o81): benchmark personale di prompt estesi e compatti, da leggere considerando campione e tipo di task.
- [Cursor Search / Instant Grep](https://cursor.com/docs/agent/tools/search): ricerca di simboli e pattern, indicizzazione locale e uso dell'Explore subagent per limitare il contesto principale.
- [Towards Data Science - Why Care About Prompt Caching in LLMs?](https://towardsdatascience.com/why-care-about-promp-caching-in-llms/): link non consultato durante questa verifica. Per l'introduzione è disponibile la [versione di Maria Mouschoutzi, PhD](https://datacream.substack.com/p/why-care-about-prompt-caching-in); per prezzi e requisiti tecnici consulta i provider.
- [Dre Dyson - articolo su cache read e Cursor](https://dredyson.com/how-i-solved-the-why-does-cursor-consume-an-absurd-amount-of-cache-read-tokens-problem-step-by-step-guide-a-complete-beginners-fix-for-reducing-millions-of-unnecessary-cache-tokens-in-cu/): lettura community non verificata; non viene usata per supportare numeri o affermazioni tecniche.
- [Liu et al., Lost in the Middle](https://aclanthology.org/2024.tacl-1.9/): ricerca pubblicata su *Transactions of the Association for Computational Linguistics* sull'uso di contesti lunghi.
- [CursorBench](https://cursor.com/cursorbench): benchmark del prodotto con score, costo, token e passaggi.
- [DeepSWE](https://deepswe.datacurve.ai/): benchmark indipendente per task di software engineering a lungo raggio.
