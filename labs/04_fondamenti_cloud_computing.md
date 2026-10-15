# Lab 04 - Fondamenti di Cloud Computing

## Obiettivi e collegamento alle slide

Realizzare il ciclo completo di una risorsa S3: requisito, provisioning, configurazione, utilizzo, verifica e rimozione. Distinguere servizio, risorsa, dato e identità; dimostrare che cifratura, autorizzazioni e recuperabilità risolvono problemi diversi.

Le attività riprendono le slide sul rapporto da conservare, Console/API, responsabilità condivisa, regione/AZ, elasticità e consumo misurato. Non occorre avviare una macchina virtuale per archiviare un oggetto.

## Durata e organizzazione

**150 minuti di laboratorio.** Nella lezione di 4 ore: 60 minuti di spiegazioni e mini-esercizi, 15 di pausa, 150 di laboratorio e 15 di discussione finale.

| Attività                         |  Minuti | Prodotto                              |
| -------------------------------- | ------: | ------------------------------------- |
| 1. Requisiti e modello           |      10 | Schema servizio/risorsa/dato/identità |
| 2. Preparazione e inventario     |      15 | File sintetici e registro             |
| 3. Provisioning e configurazione |      25 | Bucket documentato                    |
| 4. Upload, download e integrità  |      25 | Confronto delle copie                 |
| 5. Sovrascrittura e recupero     |      20 | Prove prima/dopo e ripristino         |
| 6. Accessi e diagnosi            |      20 | Matrice dei controlli e ticket        |
| 7. Consumo e disponibilità       |      15 | Calcoli e decisione motivata          |
| 8. Cleanup e consegna            |      20 | Inventario finale verificato          |
| **Totale**                       | **150** |                                       |

Il laboratorio è individuale. Ogni studente esegue le prove e prepara la propria consegna. Per ogni prova annotare prima l'atteso, poi risultato, evidenza e conclusione.

## Prerequisiti e confini

Sessione AWS Academy Learner Lab, Console assegnata e regione `us-east-1`. La CLI è facoltativa e usa soltanto le credenziali temporanee complete della sessione, incluso il token. Non creare utenti, ruoli, policy IAM o access key permanenti.

Un solo bucket general purpose `cloud04-SUFFISSO`, con suffisso univoco minuscolo, per esempio `ab4821`. Se occupato, aggiungere cifre. Usare esclusivamente file sintetici di pochi byte. Non abilitare Object Lock, retention o legal hold: sono esclusi dalla policy Academy fornita. Non attivare versioning, hosting pubblico, replica o nuove chiavi KMS.

Se S3 non è disponibile, svolgere il **percorso documentale** descritto in fondo con gli stessi output analitici. Dichiarare ciò che non è stato eseguito; non inventare screenshot o download.

## 1. Tradurre il requisito in un modello - 10 minuti

Scenario: l'ufficio manutenzione consegna un rapporto. Deve conservarlo privatamente, recuperarlo senza alterazioni e sapere quale contenuto è attualmente pubblicato. Dopo la prova, la risorsa deve essere rimossa.

1. Scrivere tre criteri di accettazione osservabili: chi legge, quale contenuto deve tornare, quale evidenza chiude il ciclo.
2. Disegnare `Operatore -> Console o CLI -> API autenticate -> bucket -> oggetto`.
3. Compilare `Elemento | Nome nel caso | Funzione | Evidenza necessaria` per servizio S3, bucket, chiave dell'oggetto, sessione temporanea.
4. Spiegare perché una chiave S3 come `rapporto04.txt` non è una chiave di accesso AWS e perché passare alla CLI non aumenta i privilegi.

**Checkpoint:** non sono necessari EC2 o un sistema operativo guest; lo schema distingue il servizio dalla risorsa creata.

## 2. Preparare file, sessione e registro - 15 minuti

Nel file manager creare `Documenti/lab04-cloud` con questa struttura; usare un editor di testo e mostrare le estensioni:

```text
lab04-cloud/
  consegna04.md
  originale-v1/rapporto04.txt
  originale-v2/rapporto04.txt
  scaricato-v1/
  scaricato-v2/
  ripristinato/
  evidenze/
```

Nel file di `originale-v1` scrivere `LAB04-V1 - rapporto sintetico`; nel file di `originale-v2` scrivere `LAB04-V2 - rapporto sintetico aggiornato`. Salvare entrambi in UTF-8 e conservare V1: sarà la copia per il ripristino.

In `consegna04.md` predisporre:

```markdown
# Consegna Lab 04
Studente:
Modalità: esecuzione Academy / analisi documentale
Regione e nome bucket:
Risorse preesistenti da non modificare:

## Requisiti e schema

## Registro
| Prova | Previsione | Operazione | Risultato osservato | Evidenza | Limite della conclusione |
| ----- | ---------- | ---------- | ------------------- | -------- | ------------------------ |

## Configurazione e responsabilità

## Consumo e disponibilità

## Ticket
Sintomo / ipotesi / verifica / correzione / controprova:

## Cleanup
Risorse create / rimosse / eventuali residue:
```

Avviare Learner Lab, attendere l'indicatore verde e aprire la Console. Selezionare `US East (N. Virginia) us-east-1`. Aprire S3 e annotare solo le risorse pertinenti all'esercizio; non ispezionare dati di altri utenti.

Se la CLI è già configurata, verificare localmente la sessione:

```bash
aws sts get-caller-identity --region us-east-1
```

Non allegare credenziali, token, URL firmati o schermate con dettagli dell'account non necessari. Se la CLI non è predisposta, proseguire interamente dalla Console.

## 3. Creare il bucket e documentarne le proprietà - 25 minuti

Aprire `S3 -> Buckets -> Create bucket`. Impostare e verificare prima della conferma:

| Campo               | Valore del laboratorio                | Motivo da spiegare                                         |
| ------------------- | ------------------------------------- | ---------------------------------------------------------- |
| Tipo                | General purpose                       | Archiviazione di oggetti nel caso delle slide              |
| Nome                | `cloud04-SUFFISSO`                    | Identifica la sola risorsa del singolo studente            |
| Regione             | `us-east-1`                           | Regione prevista dal lab                                   |
| Object Ownership    | ACLs disabled / Bucket owner enforced | Non gestire ACL degli oggetti                              |
| Block Public Access | Tutte e quattro le opzioni attive     | Bloccare l'esposizione pubblica                            |
| Versioning          | Disable                               | Osservare esplicitamente una sovrascrittura senza versioni |
| Default encryption  | SSE-S3                                | Cifratura gestita da S3, senza creare chiavi KMS           |
| Object Lock         | Disable                               | Nessun blocco della cancellazione                          |
| Tag                 | `Course=IntroAWS`, `Lab=04`           | Attribuire la risorsa all'esercizio                        |

Se la Console presenta una scelta del namespace del nome, usare quello globale condiviso compatibile con `cloud04-SUFFISSO`; non scegliere tipi di bucket diversi per risolvere un errore di nome.

Creare il bucket, poi riaprire **Properties** e **Permissions**. Registrare valore effettivo e provenienza di ciascun controllo. Non basta aver selezionato un'opzione nel modulo: verificarne la configurazione salvata.

Per ciascuna voce assegnare il livello corretto: bucket o oggetto. Annotare anche che un bucket general purpose ha una regione; nel caso non scegliamo una singola AZ.

**Checkpoint:** il bucket è quello del singolo studente, nella regione prevista, con blocco pubblico attivo. Se la creazione fallisce, conservare il messaggio e correggere solo il parametro coinvolto.

## 4. Caricare, recuperare e confrontare - 25 minuti

### P1 - Primo upload

1. Nel bucket scegliere `Objects -> Upload -> Add files` e selezionare **solo** `originale-v1/rapporto04.txt`, senza trascinare la cartella.
2. Mantenere la classe S3 Standard e completare l'upload.
3. Aprire l'oggetto: la chiave deve essere esattamente `rapporto04.txt`. Annotare dimensione, ultima modifica, classe e cifratura SSE-S3.
4. Selezionare `Download`, collocare il file in `scaricato-v1` e verificarne il nome. Se il browser salva in Download o aggiunge `(1)`, spostare/rinominare esclusivamente questa copia.
5. Aprire originale e copia con l'editor e confrontare il testo. Non sovrascrivere l'originale con il download.

### P2 - Verifica dei byte

Da un terminale in `lab04-cloud` calcolare gli hash, usando solo il proprio sistema:

```powershell
# Windows PowerShell
Get-FileHash originale-v1/rapporto04.txt -Algorithm SHA256
Get-FileHash scaricato-v1/rapporto04.txt -Algorithm SHA256
```

```bash
# Linux
sha256sum originale-v1/rapporto04.txt scaricato-v1/rapporto04.txt
```

```bash
# macOS
shasum -a 256 originale-v1/rapporto04.txt scaricato-v1/rapporto04.txt
```

Registrare i due valori. Se differiscono, verificare di aver scaricato la versione corretta e di non aver risalvato la copia con codifica o a capo differenti. Non dedurre l'integrità soltanto dal nome o dalla dimensione. Non assumere che ogni ETag S3 sia un MD5 del file: gli hash di questa prova sono calcolati localmente.

### Controllo CLI facoltativo

Sostituire `SUFFISSO` con il nome scelto. Il comando è valido anche in PowerShell:

```bash
aws s3api head-object --bucket cloud04-SUFFISSO --key rapporto04.txt --region us-east-1 --query '{Encryption:ServerSideEncryption,Size:ContentLength}' --output json
```

`AES256` indica SSE-S3. Confrontare la dimensione con la Console e il file, ma verificare l'accesso pubblico separatamente. Se si usa soltanto la Console, riportare le proprietà da lì.

**Checkpoint:** sono presenti un download effettivo e due hash uguali oppure una divergenza spiegata con una nuova verifica.

## 5. Sovrascrittura, errore operativo e recupero - 20 minuti

### P3 - Stessa chiave, nuovo contenuto

Prima di agire, prevedere quanti oggetti compariranno e quale testo verrà scaricato.

1. Caricare **solo** `originale-v2/rapporto04.txt` nella radice dello stesso bucket. Confermare la sostituzione se richiesta.
2. Aggiornare l'elenco: la chiave è sempre `rapporto04.txt`. Registrare dimensione e ultima modifica.
3. Scaricare in `scaricato-v2`, confrontare contenuto e hash con `originale-v2` usando i comandi precedenti con i nuovi percorsi.
4. Spiegare perché due upload non significano due versioni recuperabili quando il versioning non è abilitato. La copia V1 conservata localmente non è una versione S3.

### P4 - Recupero della versione desiderata

Il responsabile comunica che V2 era una modifica errata. Scrivere nel ticket la correzione proposta, poi caricare di nuovo il file da `originale-v1`. Scaricare in `ripristinato` e confrontare l'hash con V1.

**Checkpoint:** il recupero è dimostrato dal contenuto, non dal messaggio “upload completato”. Spiegare che cosa sarebbe mancato senza la copia V1; distinguere disponibilità del servizio e backup dei dati.

## 6. Accessi e diagnosi - 20 minuti

### P5 - Autenticato e anonimo

1. Verificare nuovamente Block Public Access, Object Ownership e l'assenza di una bucket policy aggiunta personalmente.
2. Copiare dalle proprietà l'**Object URL**, senza query string o firma, e aprirlo in una finestra privata del browser. Non usare il link del pulsante Download né creare URL firmati.
3. Prevedere il risultato prima di aprirlo. Atteso: nessun contenuto disponibile al lettore anonimo, normalmente `AccessDenied`/403.
4. Registrare il risultato senza conservare URL firmati. Se il testo diventa visibile, fermare la prova e confrontare URL e configurazione con il docente, senza disabilitare protezioni.

Il risultato vale per quella richiesta anonima: non censisce tutti gli accessi autorizzati. Un 403 isolato non identifica da solo la policy responsabile. Un errore di rete o TLS non è una prova di accesso negato da S3.

Compilare `Controllo | Che cosa verifica | Che cosa NON verifica` per hash locale, SSE-S3, Block Public Access e download autenticato.

### P6 - Ticket di diagnosi individuale

Scegliere un caso; scrivere prima la verifica minima e poi la correzione, senza modificare IAM:

| Caso               | Evidenze fornite                                                   | Domanda                                               |
| ------------------ | ------------------------------------------------------------------ | ----------------------------------------------------- |
| Chiave errata      | Elenco: `rapporto04.txt`; richiesta: `Rapporto04.txt`              | Quale parametro controlli prima di cambiare permessi? |
| Sessione scaduta   | Prima il download funzionava; ora la sessione temporanea è scaduta | Quale sessione rinnovi e come riprovi?                |
| Cifrato ma esposto | Un progetto esterno usa SSE-S3 ma consente lettura pubblica        | Perché la cifratura non risolve l'esposizione?        |

Il terzo caso è solo documentale: non riprodurre l'esposizione. Per il primo, chi dispone della CLI può richiedere `head-object` con la sola chiave errata e poi riprovare quella corretta. Su una chiave assente, HEAD può restituire 404 oppure 403 in base ai permessi di elenco: non diagnosticare l'esistenza dell'oggetto dal solo errore. Con la Console confrontare la chiave visibile e riscaricare l'oggetto corretto.

## 7. Consumo, elasticità e disponibilità - 15 minuti

Analizzare questo **carico sintetico**, senza caricare file aggiuntivi:

- 12.000 rapporti al giorno, ciascuno di 250 KB decimali;
- conservazione di 30 giorni, nessuna compressione;
- 3 download per rapporto;
- ogni upload è un singolo PUT e ogni download un singolo GET nel modello.

Calcolare nuovi GB/giorno, GB conservati a regime, PUT/giorno, GET/giorno e GB/giorno scaricati. Usare `1 GB = 1.000.000 KB`. Esplicitare che il traffico calcolato non equivale automaticamente a trasferimento fatturabile: dipende dalla destinazione e dalle condizioni del servizio.

Controllo del calcolo, da leggere dopo il tentativo: **3 GB/giorno, 90 GB a regime, 12.000 PUT e 36.000 GET/giorno, 9 GB/giorno scaricati**. Non includono richieste di controllo, metadati e altri oggetti.

Scrivere una formula con prezzi come variabili, senza inventare tariffe AWS: storage-tempo + richieste per categoria + trasferimenti applicabili. Quali dati mancano per un preventivo reale?

Confrontare infine due casi: archiviazione S3 del lab e API su un solo server con sessioni sul disco locale. Spiegare perché aggiungere un secondo server richiede gestione dello stato e perché scegliere una regione non rende automaticamente multi-AZ l'intera applicazione.

## 8. Cleanup, responsabilità e consegna - 20 minuti

Completare una matrice `Fase | Responsabilità AWS | Responsabilità cliente | Evidenza` per creazione, configurazione, uso, recupero e rimozione. Per ogni fase citare una prova svolta.

1. Prima di eliminare, verificare che nome e regione corrispondano al bucket creato personalmente e che contenga solo il file sintetico.
2. In `Objects` selezionare `rapporto04.txt`, scegliere `Delete` e confermare la frase richiesta dalla Console.
3. Aggiornare l'elenco e verificare che sia vuoto; quindi tornare a `Buckets`, selezionare il solo bucket del lab e scegliere `Delete`, confermando il nome esatto.
4. Aggiornare e cercare il nome nella lista. Annotare la verifica nella consegna. Un semplice `AccessDenied` della CLI non prova la cancellazione.
5. Se il versioning è stato attivato per errore, nel **solo bucket del lab** mostrare ed eliminare versioni e delete marker prima di cancellarlo; annotare la deviazione. Non operare su risorse preesistenti.
6. Chiudere la sessione Academy secondo la procedura del docente dopo la verifica: terminare la sessione non sostituisce il cleanup.

Consegnare `consegna04.md`, originali e copie sintetiche con gli hash, più evidenze essenziali delle proprietà e del cleanup. Non consegnare segreti o URL firmati. Se una cancellazione fallisce, indicare risorsa residua ed errore, senza dichiarare concluso il cleanup.

### Domande e criteri di riuscita

- Quali risultati dimostrano identità del contenuto e quali controllo dell'accesso?
- Perché P3 ha sostituito il contenuto e quale copia ha permesso P4?
- Quali parti dell'infrastruttura non hai amministrato direttamente?
- Quali componenti contribuiscono al consumo anche senza una VM?
- Quale evidenza distingue “sessione chiusa” da “bucket rimosso”?

Il lab è completo quando registro P1–P6, matrice dei controlli, calcoli, responsabilità e cleanup sono verificabili. Per i ticket documentali dichiarare la fonte.

## Percorso documentale in caso di S3 non disponibile

Conservare l'errore di accesso. Preparare realmente i file locali e confrontare gli hash di V1 e V2; non chiamare una copia locale “download S3”. Usare questo caso sintetico per le attività cloud:

| Momento           | Stato sintetico                                                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------------------------------- |
| Creazione         | General purpose, us-east-1, ACL disabilitate, blocco pubblico attivo, SSE-S3, versioning e Object Lock disabilitati |
| Primo upload      | Chiave `rapporto04.txt`, contenuto V1                                                                               |
| Secondo upload    | Stessa chiave, contenuto V2; V1 rimane solo nella copia locale                                                      |
| Recupero          | Nuovo upload dalla copia V1                                                                                         |
| Lettura anonima   | 403; lettura autenticata autorizzata riuscita                                                                       |
| Chiusura prevista | Eliminare oggetto, verificare elenco vuoto, eliminare bucket, verificarne l'assenza                                 |

Completare schema, previsioni, ticket, responsabilità e calcoli. Scrivere per ogni riga quale evidenza reale sarebbe necessaria e concludere: `Risorse non create; upload, download e cleanup AWS non eseguiti`. Non attribuire al caso dimensioni o hash misurati su AWS.

## Troubleshooting e riferimenti

- Nome occupato: cambiare solo il suffisso. AccessDenied: controllare sessione, azione, risorsa e regione; non creare nuove identità.
- Hash diverso: controllare file scelto, destinazione del download, codifica e a capo prima di accusare il servizio.
- Chiave con prefisso inatteso: controllare se è stata caricata una cartella anziché il solo file.
- Cleanup negato: conservare il nome della risorsa del lab e il messaggio per il docente.

Riferimenti: [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html), [metadati ed ETag](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html), [Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html), [HEAD e permessi](https://docs.aws.amazon.com/cli/latest/reference/s3api/head-object.html).
