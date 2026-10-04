# Risparmiare token in Cursor

## Indicazioni pratiche per spendere meno e lavorare con il contesto giusto

Per capire dove si spende in Cursor bisogna guardare sia i token sia il modo in cui il task viene svolto. Il costo dipende soprattutto dai token in input e in output, dal prezzo del modello e dal numero di passaggi necessari per arrivare al risultato. Nel conto entrano anche `cache read`, `cache write` e, a seconda del piano, il pool di utilizzo a cui appartiene il modello. La pagina ufficiale [Models and Pricing](https://cursor.com/docs/models-and-pricing) documenta queste componenti e le relative tariffe.

Il numero di passaggi non è una categoria separata di token, ma può far crescere il consumo. Ogni passaggio aggiunge potenzialmente una richiesta, nuovo contesto, chiamate ai tool e altro output. Delimitare il lavoro prima di scegliere modello ed effort aiuta a contenere questo effetto. Un modello economico può costare di più se richiede molti passaggi o correzioni successive.

## La mappa dei costi

Per leggere il costo di un task è utile distinguere il volume di token, le modalità di contabilizzazione e i passaggi che possono farlo aumentare. L'[SDK Cursor](https://cursor.com/docs/sdk/typescript) espone infatti input, output, cache e total tokens come metriche distinte.

| Fattore | Che cosa comprende | Come intervenire |
|---|---|---|
| Input | Richiesta, conversazione, file e cartelle referenziati, regole, strumenti e risposte dei tool reinserite nel contesto. | Ridurre il perimetro e citare solo ciò che serve. |
| Cache | Parti già viste dal modello possono essere conteggiate separatamente come `cache read` o `cache write`. | Non assumere che ogni token ripetuto abbia lo stesso costo dell'input nuovo. |
| Output | Testo, codice, diff, argomenti delle chiamate ai tool e, quando previsto dal modello, token di reasoning contabilizzati. | Chiedere risposte concise e limitare i dettagli al necessario. |
| Modello e modalità | Ogni modello ha prezzi e capacità diverse; Auto può instradare le richieste verso modelli differenti. | Confrontare costo e risultato sullo stesso tipo di task. |
| Passaggi | Esplorazione, modifiche, test, errori e retry dell'agente. | Dare vincoli, criteri di completamento e verifiche mirate. |

## Ridurre i token in input

### Dare un perimetro al lavoro

Un agente lasciato completamente libero può esplorare più file, eseguire più comandi e produrre più tentativi del necessario. Un obiettivo circoscritto riduce questo rischio e chiarisce che cosa va fatto prima di iniziare l'implementazione.

Puoi separare analisi, piano e implementazione quando il problema è ancora poco definito. Non è una garanzia di risparmio: anche i vincoli aggiungono qualche token e una fase separata conviene solo se evita lavoro ripetuto.

- Indica i file o l'area interessata, il comportamento atteso e i criteri di accettazione.
- Chiedi di cercare prima nei percorsi rilevanti e di riferire i risultati prima di aprire altro contesto.
- Usa una sessione per specifica e piano quando il problema è ancora ambiguo, un'altra per l'esecuzione quando il piano è stabile.

### Gestire conversazioni lunghe

La cronologia cresce con i prompt, i file referenziati e le risposte. Cursor documenta la sintesi automatica delle conversazioni lunghe e la gestione di file e cartelle troppo grandi ([Summarization](https://docs.cursor.com/en/agent/chat/summarization)). Nel CLI, `/summarize` e il suo alias `/compress` servono a ridurre il contesto. L'interfaccia può cambiare tra editor, CLI e versioni: se il comando non compare, usa la funzione di summarization disponibile nel tuo ambiente.

Il 40-50% non è una soglia tecnica ufficiale: è un'euristica personale. Non aspettare sempre il limite massimo, perché la qualità dipende anche da come sono selezionate e organizzate le informazioni. La ricerca sul fenomeno *lost in the middle* mostra che i modelli possono usare peggio le informazioni rilevanti quando sono immerse in contesti lunghi, soprattutto se si trovano nella parte centrale del contesto ([Liu et al., 2023](https://arxiv.org/abs/2307.03172)). Anthropic descrive un rischio simile in termini di *context pollution* e rilevanza delle informazioni nei task lunghi, e propone compaction, note strutturate e architetture multi-agente come tecniche di gestione ([Anthropic, Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).

Questi studi non misurano Cursor e non indicano una percentuale oltre la quale il modello "dimentica" automaticamente tutto. Suggeriscono però alcuni segnali da osservare: risposte incoerenti, ripetizione di esplorazioni già fatte, mancato rispetto di decisioni precedenti, aumento dei retry o difficoltà a individuare il file giusto.

La compattazione può ridurre il contesto, ma può anche perdere dettagli e richiede lavoro del modello. Usala quando il beneficio supera il costo e verifica poi che il riepilogo conservi obiettivo, vincoli, file coinvolti, decisioni, test e problemi aperti.

### Cercare file e cartelle in modo mirato

Quando conosci già il file, riferiscilo con `@Files & Folders` o trascinalo nella chat. Quando conosci solo la cartella, considera che Cursor può includere un percorso e una panoramica, mentre il contenuto completo della cartella aumenta l'input e può diventare costoso. La documentazione di [@Files & Folders](https://docs.cursor.com/context/%40-symbols/%40-files-and-folders) descrive proprio questa differenza tra il riferimento alla cartella e `Full Folder Content`.

Evita di chiedere genericamente di leggere tutto un workspace molto grande: dai prima il percorso, il pattern o il sottoinsieme utile.

Il costo dipende da dimensione, contenuto incluso, condensazione e cache. Seleziona quindi il contesto in modo esplicito, invece di affidarti alla ricerca indiscriminata dell'agente.

### Cambiare chat con un handoff

Aprire una nuova chat riduce il rumore della cronologia, ma non conserva automaticamente le decisioni prese. Prima di cambiare sessione, prepara un handoff breve con stato corrente, file coinvolti, decisioni, test eseguiti, problemi aperti e prossimo passo.

La nuova chat deve ricevere solo quel riepilogo e i riferimenti necessari. Se il riepilogo diventa lungo quanto la cronologia, il vantaggio si riduce.

```text
Handoff: obiettivo | stato | file coinvolti | decisioni | test | problemi aperti | prossimo passo
```

## Ridurre i token in output

Quello che l'agente produce comprende la risposta testuale, il codice o il diff proposto e gli argomenti delle chiamate ai tool generate dal modello. Le risposte dei tool diventano nuovo contesto quando vengono reinserite nella conversazione. L'[SDK Cursor](https://cursor.com/docs/sdk/typescript) espone anche `reasoningTokens` quando il modello li rende disponibili.

Quando il codice è già visibile nel diff, è spesso sufficiente chiedere una risposta breve e orientata all'azione.

- Chiedi un formato preciso, per esempio: conclusione, file modificati, test eseguiti ed eventuali problemi.
- Chiedi una spiegazione estesa solo quando serve per imparare o prendere una decisione.
- `caveman` e `ponytail` puntano a ridurre verbosità o codice superfluo; `i-have-adhd` punta a rendere le risposte più dirette. Possono aiutare, ma le loro istruzioni aumentano anche il contesto: attivale solo quando il beneficio è reale.

```text
Rispondi in modo conciso. Mostra solo: risultato, file modificati, test eseguiti e blocchi. Niente recap se non richiesto.
```

Una skill di stile aiuta a ridurre la verbosità, ma il perimetro resta la leva principale. Se l'agente esplora troppo, indica meno file, limita gli obiettivi contemporanei e definisci criteri di completamento più chiari.

## Scegliere modello, effort e modalità Auto

Per confrontare due configurazioni, considera il costo del task finito: tariffa dei token, numero di passaggi, tempo, retry e qualità della prima soluzione.

Un modello economico con effort alto può usare molti passaggi. Un modello più capace può costare di più per richiesta, ma chiudere il lavoro prima.

| Situazione | Partenza consigliata | Ottimizzazione successiva |
|---|---|---|
| Task quotidiano e poco definito | Auto in modalità Balance, se disponibile. | Controlla quale modello viene scelto e confrontalo con una selezione fissa. |
| Task ripetibile | Modello ed effort fissi per rendere il confronto riproducibile. | Misura costo, score o test superati e passaggi su più esempi. |
| Task complesso o multi-file | Effort più alto solo se migliora il risultato atteso. | Confronta una configurazione forte con una economica su task equivalenti. |
| Task semplice o modifica locale | Modello rapido e contesto minimo. | Se il retry aumenta, prova un modello più capace invece di aggiungere istruzioni. |

Le modalità Auto di Cursor Router sono **Cost**, **Balance** e **Intelligence**. Poiché il router può cambiare modello tra richieste, Auto è comodo per iniziare ma rende meno rigoroso il confronto tra due prompt.

### Come funziona Cursor Router

Cursor Router è il componente che alimenta Auto. Per ogni richiesta dell'agente, valuta il tipo e la complessità del task e la instrada verso un modello della pool disponibile. Secondo la documentazione di Cursor, l'obiettivo è scegliere il modello meno costoso che mantenga una qualità comparabile, non quello più economico in assoluto ([Cursor Router](https://prod.cursor.com/help/models-and-usage/cursor-router)).

Per usarlo dal model picker:

1. selezionare **Auto**;
2. scegliere l'obiettivo di ottimizzazione;
3. lasciare che il router scelga il modello per quella richiesta.

Le tre modalità hanno intenti diversi:

- **Cost:** ottimizza la spesa per token;
- **Balance:** cerca un compromesso tra qualità e costo ed è la modalità predefinita per i nuovi utenti;
- **Intelligence:** privilegia modelli più capaci ed è pensata per task complessi o multi-step.

Cursor dichiara che Intelligence produce circa il 20-30% di qualità in più selezionando modelli first-party della pool. È una metrica riportata da Cursor, non una garanzia per ogni progetto ([Cursor Router](https://prod.cursor.com/help/models-and-usage/cursor-router)). Il modello sottostante può cambiare tra richieste. Le richieste Auto vengono fatturate al prezzo del modello scelto per quella richiesta; per alcuni piani e modelli di terze parti possono applicarsi anche costi aggiuntivi di Cursor ([Models and Pricing](https://cursor.com/docs/models-and-pricing)).

Auto è una buona impostazione iniziale. Per capire se Medium, High o Max convengono, fissa modello ed effort e ripeti lo stesso gruppo di task. Se un modello è bloccato dall'amministratore del team, il router può saltarlo o, in alcuni casi, non essere disponibile.

### Metriche da osservare

Per una singola richiesta o per una serie di task, le metriche più utili sono:

| Metrica | Che cosa dice |
|---|---|
| Input tokens | Quanto contesto nuovo è stato inviato. |
| Cache read e cache write | Quanto contesto è stato riutilizzato o scritto in cache; può avere tariffe diverse dall'input nuovo. |
| Output tokens | Quanto testo, codice e reasoning contabilizzato il modello ha prodotto. |
| Total tokens | Il volume complessivo registrato per la richiesta. |
| Steps | Quanti passaggi agente, tool call o iterazioni sono serviti. CursorBench li riporta separatamente dal costo, quindi sono soprattutto un indicatore operativo. |
| Cost | Il costo stimato o applicato alla richiesta o al task. |
| Success rate | Quante volte il task è stato completato correttamente. |

Il modello dati dell'SDK Cursor espone `inputTokens`, `outputTokens`, `cacheReadTokens`, `cacheWriteTokens`, `totalTokens` e, quando disponibile, `reasoningTokens` ([Cursor TypeScript SDK](https://cursor.com/docs/sdk/typescript)). CursorBench usa anche score, costo, token e steps. Cursor avverte che piccoli scarti di score possono non essere statisticamente significativi ([CursorBench](https://cursor.com/cursorbench)).

Per confrontare due approcci, guarda il **costo per risultato riuscito**, includendo retry e correzioni manuali. Un task economico ma fallito non è più efficiente di uno leggermente più costoso che arriva al risultato corretto al primo tentativo.

Se due livelli di effort hanno quasi lo stesso score nel grafico, non concludere subito che siano equivalenti: la differenza può rientrare nella variabilità del benchmark. La domanda utile è se il livello più alto migliora abbastanza la probabilità di successo da giustificare il costo aggiuntivo sui tuoi task.

## Leggere i benchmark senza farsi ingannare

I benchmark aiutano a formulare ipotesi sul rapporto tra qualità e costo. Non predicono da soli quanto spenderai nel tuo progetto. Leggi sempre insieme:

1. qualità o pass rate;
2. costo medio per task;
3. numero di token o passaggi.

Confronta configurazioni eseguite sullo stesso benchmark e considera la variabilità statistica prima di trarre conclusioni.

**[CursorBench](https://cursor.com/cursorbench)** è il riferimento più vicino al prodotto: usa task multi-file provenienti da sessioni reali di Cursor e riporta score, costo, token e passaggi.

**[DeepSWE](https://deepswe.datacurve.ai/)** è un benchmark indipendente per task di software engineering a lungo raggio; la leaderboard mostra anche costo medio, output token e passi dell'agente. Le metodologie e gli harness non sono identici, quindi i numeri non sono intercambiabili.

### Frontiera tra costo e risultato

![Grafico benchmark con score percentuale sull'asse verticale e costo medio per task sull'asse orizzontale, con una configurazione evidenziata in verde.](assets/benchmark-frontiera-costo-score.png)

*Figura 1. Un punto sulla frontiera può offrire un buon compromesso tra score e costo, ma il valore va letto insieme a token, passaggi e intervallo di incertezza.*

Nel grafico allegato, la linea evidenziata mostra questo compromesso. A parità di risultato apparente, una configurazione con più effort non è automaticamente preferibile se il costo cresce senza un miglioramento misurabile.

### Costo e qualità non crescono sempre insieme

Una curva benchmark può avere un tratto piatto: il costo aumenta mentre il punteggio cambia poco. In quel caso vale la pena testare effort diversi sul proprio lavoro, invece di lasciare sempre l'impostazione più alta.

![Grafico benchmark che confronta configurazioni di modelli con score vicino e costi medi per task differenti.](assets/benchmark-score-costo.png)

*Figura 2. Esempio di confronto tra configurazioni con score simile e costo medio per task molto diverso.*

Il confronto è significativo solo se le configurazioni usano lo stesso insieme di task e lo stesso metodo di valutazione. Un benchmark può favorire task lunghi, bugfix, refactoring o tool use in modo diverso dal tuo progetto.

### Il punto economico è il costo totale

Quando un modello più economico produce molti più passaggi, il prezzo basso per token può non tradursi in un prezzo basso per task. Allo stesso modo, un modello più costoso può essere conveniente se riduce retry e interventi manuali.

![Grafico benchmark che mostra la relazione tra score, costo medio per task e livelli diversi di effort.](assets/benchmark-effort-costo.png)

*Figura 3. Il confronto tra livelli di effort va fatto sul rapporto tra risultato e costo, non sul nome del modello.*

I tre screenshot conservano i valori originali visibili nelle immagini fornite. Non sono una tabella prezzi di Cursor: prezzi, modelli, pool di utilizzo e modalità possono cambiare.

## Procedura consigliata

1. Scrivi obiettivo, area del progetto, vincoli, criteri di accettazione e verifica richiesta.
2. Usa `@Files & Folders` per file e cartelle note; evita il workspace intero se non serve.
3. Per i task ordinari parti con Auto o con un modello economico; non massimizzare l'effort per abitudine.
4. Guarda test, retry, token, costo e passaggi; annota anche il tempo risparmiato o perso.
5. Usa `/summarize` quando il contesto è diventato rumoroso e prepara un handoff prima di cambiare chat.
6. Per task ripetibili, prova modello ed effort fissi su più esempi e scegli il punto migliore tra qualità e costo.

## Fonti e risorse

Fonti consultate il 2 ottobre 2026. Le pagine di prodotto e le tariffe sono soggette a cambiamento.

- [Cursor Models and Pricing](https://cursor.com/docs/models-and-pricing) - pool di utilizzo, tariffe per token, Auto e Max Mode.
- [Cursor Router](https://prod.cursor.com/help/models-and-usage/cursor-router) - modalità Cost, Balance e Intelligence e comportamento di Auto.
- [How Cursor Router chooses the right model](https://prod.cursor.com/blog/how-cursor-router-works) - spiegazione di Cursor sul routing, sul costo per richiesta e sulle configurazioni Auto.
- [Cursor Summarization](https://docs.cursor.com/en/agent/chat/summarization) - sintesi automatica e condensazione di file e cartelle.
- [Cursor slash commands](https://prod.cursor.com/docs/cli/reference/slash-commands) - comandi `/summarize` e `/compress` nel CLI.
- [Cursor @ Files and Folders](https://docs.cursor.com/context/%40-symbols/%40-files-and-folders) - riferimenti mirati e costo del full folder content.
- [Lost in the Middle](https://aclanthology.org/2024.tacl-1.9/) - articolo pubblicato su *Transactions of the Association for Computational Linguistics* sulla degradazione dell'uso del contesto lungo e sulla posizione delle informazioni rilevanti.
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) - compaction, context pollution e gestione dei task a lungo raggio.
- [Cursor TypeScript SDK](https://cursor.com/docs/sdk/typescript) - metriche token disponibili a livello di SDK.
- [Cursor Working with Context](https://docs.cursor.com/en/guides/working-with-context) - definizione di contesto, token input/output e uso mirato degli `@` symbols.
- [CursorBench](https://cursor.com/cursorbench) - benchmark del prodotto con score, costo, token e passaggi.
- [DeepSWE](https://deepswe.datacurve.ai/) - benchmark indipendente per task di software engineering a lungo raggio.
- [caveman](https://github.com/JuliusBrussee/caveman) - skill orientata a ridurre la verbosità dell'agente.
- [i-have-adhd](https://github.com/ayghri/i-have-adhd) - skill orientata a risposte più dirette e azionabili.
- [ponytail](https://github.com/DietrichGebert/ponytail) - skill orientata a semplificare e ridurre codice o lavoro superfluo.
