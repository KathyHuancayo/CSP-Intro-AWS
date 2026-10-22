# Lab 05 - Classificazione dei servizi cloud

## Obiettivi e collegamento alle slide

Scegliere un modello di servizio a partire dai requisiti, documentare il confine operativo e rivedere la decisione quando cambiano i vincoli. Il laboratorio riprende i tre casi delle slide - gestionale legacy, nuova API e videoconferenza - insieme a EC2, Elastic Beanstalk, RDS e S3.

Non basta associare un prodotto a IaaS, PaaS o SaaS: occorre spiegare chi interviene su sistema, runtime, codice, dati e accessi, e quale prova rende verificabile la scelta.

## Durata, modalità e prerequisiti

**Laboratorio individuale di 150 minuti.** Ogni studente produce la propria analisi e la propria consegna. Le 4 ore comprendono 60 minuti di spiegazioni e mini-esercizi, 15 di pausa, 150 di laboratorio e 15 di discussione finale.

| Attività                     |  Minuti | Prodotto                                |
| ---------------------------- | ------: | --------------------------------------- |
| 1. Requisiti e vincoli       |      10 | Schede dei tre casi                     |
| 2. Osservazione dei servizi  |      20 | Cinque schede con fonti                 |
| 3. Responsabilità operative  |      25 | Matrice e confronti                     |
| 4. Decisioni motivate        |      25 | Tre decisioni con prove di accettazione |
| 5. Architettura e dipendenze |      25 | Schema e percorso di un errore          |
| 6. Incidenti                 |      20 | Tre ticket di analisi                   |
| 7. Requisiti modificati      |      15 | Revisione di una decisione              |
| 8. Verifica e consegna       |      10 | Elaborato completo e inventario finale  |
| **Totale**                   | **150** |                                         |

La Console AWS Academy è facoltativa: se disponibile, usare la sessione assegnata e `us-east-1`. Il percorso completo si può svolgere anche con le schede documentali incluse. **Nessun provisioning:** non creare EC2, RDS, ambienti Beanstalk, bucket, identità o account SaaS. Non modificare risorse esistenti per ottenere una schermata da consegnare.

Nel file manager creare `Documenti/lab05-servizi`, con `consegna05.md`, `architettura05.md` ed eventualmente una cartella `evidenze`. Usare un editor di testo.

## 1. Estrarre i requisiti - 10 minuti

Leggere i casi senza scegliere subito un prodotto:

| Caso                  | Requisiti del committente                                                                                                                             |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| A - Gestionale legacy | Applicazione esistente; modulo specifico del sistema operativo guest; aggiornamenti in una finestra concordata; dati da recuperare dopo un incidente. |
| B - API manutenzione  | Codice sviluppato dall'azienda; runtime standard; rilasci frequenti; carico variabile; nessun modulo personalizzato del sistema operativo richiesto.  |
| C - Videoconferenza   | Applicazione pronta all'uso; utenti e ruoli gestiti dall'azienda; registrazioni riservate; esportazione dei dati alla fine del contratto.             |

Per ciascun caso scrivere: due requisiti obbligatori, una preferenza e una domanda ancora aperta. Non trasformare una preferenza in un obbligo senza motivazione.

Esempio di domanda utile per B: “Quali versioni del runtime e quali dipendenze deve supportare la piattaforma?”. “Usare cloud” da solo non è un criterio di accettazione.

**Checkpoint:** ogni requisito descrive un bisogno verificabile, non soltanto un nome commerciale.

## 2. Osservare funzioni e parametri - 20 minuti

### Percorso Console in sola lettura

Aprire gli elenchi seguenti. Non premere i pulsanti finali di creazione e non modificare configurazioni:

| Servizio          | Percorso indicativo                                         | Che cosa cercare                                       |
| ----------------- | ----------------------------------------------------------- | ------------------------------------------------------ |
| EC2               | `EC2 -> Instances`; `Network & Security -> Security Groups` | Immagine del sistema, capacità, dischi, regole di rete |
| Elastic Beanstalk | `Elastic Beanstalk -> Environments`                         | Ambiente, piattaforma/runtime, versione applicativa    |
| RDS               | `RDS -> Databases`, `Parameter groups`, `Subnet groups`     | Motore, configurazione, rete, backup disponibili       |
| S3                | `S3 -> Buckets`                                             | Regione, oggetti, accessi, ciclo di vita               |

Un elenco vuoto dimostra soltanto che non mostra risorse: non prova come siano gestite. Se una proprietà non è visibile o l'accesso è negato, usare la relativa scheda documentale e registrare il limite. Non aprire dati o configurazioni di altri utenti.

### Schede documentali utilizzabili senza Console

Queste schede descrivono il confine operativo; non sono output misurati nel proprio account:

- **EC2:** il cliente sceglie immagine, tipo di istanza, dischi e regole di rete. Gestisce il sistema guest e l'applicazione; AWS gestisce infrastruttura fisica e virtualizzazione. Il requisito di un modulo specifico va verificato sull'OS/AMI scelto, non dato per soddisfatto dal solo nome EC2.
- **Elastic Beanstalk:** il cliente fornisce codice e configurazione e sceglie una piattaforma supportata. Il servizio orchestra risorse e deployment. Restano decisioni su aggiornamenti, dipendenze, accessi e osservabilità; le politiche di aggiornamento vanno verificate.
- **RDS:** database relazionale gestito. Il cliente sceglie configurazioni disponibili e gestisce schema, query, account DB e accessi. Backup e aggiornamenti seguono le capacità e le impostazioni del servizio: la recuperabilità va provata.
- **S3:** object storage gestito. Il cliente governa oggetti, permessi e politiche di conservazione; non amministra il sistema operativo dello storage. Non è una piattaforma che esegue automaticamente il codice dell'API.
- **Videoconferenza SaaS:** caso sintetico di applicazione pronta all'uso. Il fornitore gestisce software e infrastruttura; l'azienda configura utenti, ruoli, registrazioni e condivisione. Esportazione e conservazione dipendono dal contratto: non assumere che ogni prodotto abbia le stesse funzioni.

Creare cinque schede in `consegna05.md`:

```markdown
### Servizio o caso
Funzione offerta:
Parametro controllato dal cliente:
Attività delegata al provider:
Categoria proposta e limite della classificazione:
Fonte: Console / documentazione ufficiale / caso sintetico
Evidenza osservata oppure informazione ancora da verificare:
```

**Checkpoint:** distinguere ciò che è stato visto da ciò che è descritto nella documentazione. La presenza di una console web non trasforma un servizio in SaaS.

## 3. Costruire la matrice delle responsabilità - 25 minuti

Compilare la seguente matrice con `cliente`, `provider` oppure `ripartita`, aggiungendo una motivazione operativa per ciascuna riga. La classificazione orienta l'analisi, non sostituisce il contratto del servizio.

| Attività                                       | IaaS / EC2 | PaaS / Beanstalk | SaaS videoconferenza |
| ---------------------------------------------- | ---------- | ---------------- | -------------------- |
| Sostituire un disco fisico guasto              |            |                  |                      |
| Applicare patch al sistema operativo guest     |            |                  |                      |
| Selezionare e aggiornare runtime/dipendenze    |            |                  |                      |
| Correggere il codice dell'applicazione fornita |            |                  |                      |
| Creare utenti e assegnare ruoli aziendali      |            |                  |                      |
| Decidere quali dati inserire                   |            |                  |                      |
| Configurare condivisione e conservazione       |            |                  |                      |
| Definire e provare il recupero dei dati        |            |                  |                      |

Per PaaS evitare “il provider fa tutto”: indicare che cosa il cliente deve selezionare, configurare o verificare. Per SaaS distinguere il codice del prodotto da eventuali integrazioni sviluppate dal cliente.

Aggiungere tre confronti di 3–4 frasi:

1. **IaaS/PaaS:** che cosa cambia nella gestione di sistema e runtime?
2. **PaaS/SaaS:** chi distribuisce il codice dell'applicazione principale?
3. **Tutti i modelli:** perché dati, identità e condivisione richiedono ancora decisioni del cliente?

Estendere poi la matrice con una riga per RDS e una per S3: funzione, configurazione del cliente e parte delegata. Non forzare questi servizi nella casella “applicazione SaaS completa”.

**Checkpoint:** ogni responsabilità è un'azione concreta, per esempio “verificare il ripristino” anziché il generico “sicurezza”.

## 4. Scrivere tre decisioni motivate - 25 minuti

Per A, B e C confrontare almeno due opzioni. Prima verificare i requisiti obbligatori: un candidato incompatibile non diventa idoneo perché ottiene un buon punteggio sulle preferenze.

Compilare per ciascun caso:

```markdown
### Decisione A / B / C
Requisiti obbligatori:
Candidato scelto e modello:
Alternativa considerata:
Motivo della scelta (80–120 parole):
Tre attività che restano al cliente:
Un limite o compromesso accettato:
Informazione non ancora verificata:
Prova di accettazione e risultato necessario:
Piano di uscita o ripristino:
```

La prova deve avere un esito verificabile. Esempi da adattare:

- A: installare il modulo richiesto in un ambiente di prova compatibile e verificare il gestionale; in questo lab si **progetta** il test, senza creare la VM.
- B: distribuire una versione campione e verificare un rollback; precisare quali risposte confermerebbero il recupero.
- C: esportare registrazioni e metadati, verificare formato e leggibilità; qui si descrive la procedura, senza aprire account.

### Mini-esperimento sui criteri di scelta

Per B usare questa **scheda sintetica di valutazione**, non un confronto commerciale di prodotti reali. Entrambi i candidati sono dichiarati compatibili con il runtime:

| Criterio                                     | Peso | Candidato X: VM amministrata dal cliente | Candidato Y: piattaforma gestita |
| -------------------------------------------- | ---: | ---------------------------------------: | -------------------------------: |
| Minore lavoro operativo                      |    4 |                                        1 |                                3 |
| Controllo del sistema                        |    3 |                                        3 |                                1 |
| Facilità di rollback nel caso ipotizzato     |    2 |                                        2 |                                3 |
| Facilità di esportazione nel caso ipotizzato |    1 |                                        3 |                                2 |

I punteggi vanno da 1 a 3 e sono dati dell'esercizio, non garanzie AWS. Calcolare la somma `peso × punteggio`: X = 20, Y = 23. Poi assegnare peso 1 al lavoro operativo e 6 al controllo del sistema, lasciando invariati gli altri pesi: che cosa cambia? Spiegare perché i pesi devono derivare dai requisiti, non dalla soluzione che si vuole far vincere.

**Checkpoint:** le tre decisioni contengono una scelta, un'alternativa e una prova; non promettono automaticamente costo minimo o assenza di manutenzione.

## 5. Disegnare un'architettura e seguirne le dipendenze - 25 minuti

In `architettura05.md` disegnare, anche con uno schema testuale:

```text
Utente -> API su piattaforma applicativa -> database relazionale gestito
                                      -> oggetti in S3
Operatore -> videoconferenza SaaS
```

1. Assegnare a ogni componente funzione, confine di gestione e dati trattati.
2. Numerare il percorso “invia richiesta di manutenzione con allegato”: ingresso API, validazione, scrittura del record, caricamento dell'allegato, risposta all'utente. Dichiarare l'ordine scelto.
3. Per ogni freccia indicare identità usata, autorizzazione necessaria e tipo di errore da gestire, senza scrivere credenziali reali.
4. Analizzare il caso in cui il database confermi il record ma S3 rifiuti il caricamento. Quale stato incompleto rimane? Come lo segnaleresti? Quale verifica precede un nuovo tentativo per evitare duplicati?
5. Identificare almeno due dipendenze che non vengono risolte scegliendo semplicemente l'etichetta PaaS: per esempio permessi sui dati e compatibilità del runtime.
6. Descrivere una procedura di esportazione che includa sia record DB sia allegati e una verifica che i riferimenti rimangano coerenti.

Non è richiesto implementare codice, transazioni distribuite o nuove risorse. L'output è una sequenza ragionata con punti di controllo.

**Checkpoint:** lo schema distingue applicazione e servizi di supporto; le frecce hanno un significato operativo, non rappresentano soltanto “connessione al cloud”.

## 6. Analizzare tre incidenti - 20 minuti

Per ciascuno scrivere `Sintomo | Evidenza disponibile | Ipotesi | Responsabile dell'azione | Verifica | Correzione proposta | Controprova`. Le azioni sono progettate, non eseguite su servizi reali.

### Incidente 1 - Registrazioni SaaS condivise pubblicamente

Il prodotto è disponibile e aggiornato; una registrazione sintetica è visibile senza login dopo una modifica alla condivisione. Distinguere aggiornamento del provider e configurazione del cliente. Proporre revoca, verifica del perimetro e nuova prova di accesso autorizzato/non autorizzato. Quali evidenze servono prima di affermare chi abbia letto il file?

### Incidente 2 - API non avviabile dopo un rilascio

Il nuovo codice richiede una versione di runtime non disponibile nell'ambiente scelto; il precedente rilascio funzionava. Separare stato della piattaforma e compatibilità del codice. Indicare verifica di runtime/dipendenze, scelta tra rollback e piattaforma compatibile, e richiesta di controllo dopo l'intervento.

### Incidente 3 - Database gestito, dati cancellati dall'applicazione

Il database è raggiungibile ma mancano record dopo un'operazione applicativa errata. La console indica che i backup sono configurati, ma nessuno ha provato il ripristino. Quali informazioni servono su istante recuperabile, destinazione del ripristino e verifica dei dati? Perché “servizio disponibile” non significa “dati corretti”?

**Checkpoint:** non attribuire al provider ogni incidente di un servizio gestito. Separare il componente operativo dalla responsabilità di dati e configurazione.

## 7. Cambiare un requisito e rivedere la scelta - 15 minuti

Applicare al caso B questa nuova richiesta: l'API necessita ora di un modulo del sistema operativo non supportato dalla piattaforma candidata.

1. Individuare il requisito obbligatorio che cambia.
2. Scrivere quale evidenza rende incompatibile il candidato iniziale; non basta dire “IaaS è meglio”.
3. Confrontare due strade: modificare l'applicazione eliminando la dipendenza oppure scegliere un ambiente che supporti il modulo e assumersi la relativa gestione.
4. Aggiornare decisione, attività operative, rischio e prova di accettazione. Conservare anche la versione iniziale della decisione.
5. Verificare se il cambio del componente API modifica davvero la classificazione di S3, RDS o SaaS. Motivare separatamente per ciascuno.

**Checkpoint:** una nuova scelta deve discendere dal nuovo requisito. I punteggi dell'attività 4 non possono compensare un vincolo obbligatorio non soddisfatto.

## 8. Verificare, consegnare e chiudere - 10 minuti

In `consegna05.md` includere:

- requisiti dei tre casi e cinque schede dei servizi con fonte;
- matrice di otto attività, tre confronti e responsabilità di RDS/S3;
- tre decisioni complete e calcolo con i due insiemi di pesi;
- riferimento ad `architettura05.md` e analisi dell'errore tra database e oggetti;
- tre ticket e decisione B prima/dopo il cambio di requisito;
- dichiarazione finale sulle risorse.

### Autoverifica individuale

| Criterio              | Controllo                                                           |
| --------------------- | ------------------------------------------------------------------- |
| Tracciabilità         | Ogni scelta cita almeno un requisito obbligatorio.                  |
| Confine operativo     | Ogni servizio ha almeno un'attività del cliente e una del provider. |
| Qualità delle fonti   | Console, documentazione e casi sintetici sono distinti.             |
| Verificabilità        | Ogni decisione contiene una prova con un esito atteso.              |
| Gestione degli errori | Ogni ticket distingue sintomo, causa ipotizzata e controprova.      |
| Revisione             | Il nuovo requisito produce una scelta riesaminata e motivata.       |

Se è stata usata la Console, confrontare gli elenchi iniziali e finali di EC2, RDS, Beanstalk e S3 senza cancellare risorse preesistenti. Non chiudere ambienti o account di altri progetti.

Scrivere `Risorse create dal lab: nessuna`. Nel percorso offline aggiungere `Console non utilizzata; analisi documentale`. Conservare i file di consegna e chiudere le pagine di lavoro; non eliminare l'elaborato.

## Troubleshooting e riferimenti

- Pagina vuota o AccessDenied: usare la scheda documentale e dichiarare il limite, senza creare una risorsa per popolare l'elenco.
- Categoria ambigua: descrivere funzione e responsabilità prima di proporre l'etichetta.
- Informazione commerciale mancante: registrarla come verifica aperta, senza inventare prezzo, SLA, runtime supportato o condizioni di esportazione.
- Scelta cambiata: conservare requisiti e motivazioni delle due versioni, non riscrivere la decisione iniziale come se fosse sempre stata quella finale.

Riferimenti: [AWS - responsabilità condivisa](https://aws.amazon.com/compliance/shared-responsibility-model/), [Elastic Beanstalk - piattaforme e runtime](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/platforms-glossary.html), [NIST - definizione dei modelli cloud](https://www.nist.gov/publications/nist-definition-cloud-computing).
