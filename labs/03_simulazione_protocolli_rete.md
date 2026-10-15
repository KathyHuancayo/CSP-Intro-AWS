# Lab 03 - Simulazione e protocolli di rete

## Obiettivi

Al termine del laboratorio saprai rappresentare uno scambio client/server, distinguere IP, porte e risorse, leggere richieste e risposte HTTP, ricostruire una conversazione TCP e diagnosticare un guasto con una controprova di ripristino.

Il percorso sviluppa i temi delle slide: modello, incapsulamento, URL, TCP/UDP, DNS, analisi del traffico e metodo «predire, misurare, spiegare».

## Durata e organizzazione

**130 minuti di laboratorio**, inclusi consegna e cleanup. Nelle 3 ore della sessione: 40 minuti per spiegazioni e mini-esercizi delle slide, 10 minuti di pausa e 130 minuti per questo percorso.

| Attività                           |  Minuti | Risultato concreto                              |
| ---------------------------------- | ------: | ----------------------------------------------- |
| 1. Preparazione                    |      10 | Cartelle e strumenti pronti                     |
| 2. Modello e previsioni            |      10 | Schema, URL e ipotesi                           |
| 3. GET, 404 e contenuto modificato |      25 | Tre prove commentate                            |
| 4. GET contro HEAD                 |      15 | Confronto di richieste e risposte               |
| 5. Conversazione TCP               |      20 | Analisi della cattura o del modello alternativo |
| 6. Nomi, indirizzi e DNS           |      10 | Analisi di una query o del caso offline         |
| 7. Guasti e ripristino             |      25 | Arresto, recupero e ticket individuale          |
| 8. Consegna e cleanup              |      15 | Relazione verificabile e server fermato         |
| **Totale**                         | **130** |                                                 |

Il laboratorio è individuale. Ogni studente esegue le prove, scrive le previsioni e raccoglie le proprie evidenze. Nel ticket registra la causa introdotta e verifica la diagnosi con una controprova.

## Prerequisiti e scenario

Python 3.7 o successivo e curl già disponibili. Su Windows usare PowerShell e **`curl.exe`**, evitando l'alias di Windows PowerShell. Wireshark, con supporto alla cattura loopback, è facoltativo e deve essere già predisposto; senza cattura si svolge il percorso alternativo dell'attività 5.

Un piccolo servizio deve pubblicare un messaggio. Il client segnala, in momenti diversi, contenuto inatteso, risorsa mancante o nessuna risposta. Il compito è distinguere queste situazioni attraverso evidenze, modificando una sola variabile alla volta.

Client e server saranno sullo stesso computer, su `127.0.0.1`. È traffico reale su loopback, non una simulazione di Internet: non attraversa AP, switch o router. Nessuna risorsa AWS, modifica della rete o credenziale è necessaria. Solo la query DNS può richiedere la rete esterna e dispone di un'alternativa offline.

## 1. Preparazione - 10 minuti

1. Nel file manager creare `Documenti/lab03-protocolli`, indicata qui come `LAB03`. Se esiste già un'esecuzione precedente, usare una nuova cartella.
2. Creare al suo interno le cartelle `sito`, `evidenze` e `sito-vuoto`. Lasciare vuota quest'ultima.
3. Nell'editor creare `sito/messaggio.txt`, testo semplice UTF-8, contenuto `LAB03-OK`. Mostrare le estensioni per evitare `messaggio.txt.txt`. Un a capo finale è ammesso e conterà nella lunghezza del contenuto.
4. Creare `consegna03.md` e `evidenze/log-server.txt`. In quest'ultimo incollare durante le prove le righe pertinenti del server, precedute dall'identificativo della prova.
5. Aprire **due terminali in `LAB03`, non in `sito`**. Usare “Apri nel Terminale” dal file manager; su macOS digitare `cd ` e trascinare la cartella nel Terminale. Verificare con `pwd` su Linux/macOS o `Get-Location` in PowerShell.

```text
lab03-protocolli/
├── consegna03.md
├── sito/
│   └── messaggio.txt
├── sito-vuoto/
└── evidenze/
    └── log-server.txt
```

Pubblicheremo soltanto `sito`. Salvare le risposte in `evidenze` evita che una prova crei accidentalmente la risorsa che dovrebbe mancare.

Verificare gli strumenti nel terminale 2, eseguendo solo il blocco del proprio sistema:

```bash
# Linux/macOS
python3 --version
curl --version
```

```powershell
# Windows PowerShell
py -3 --version
curl.exe --version
```

Se uno strumento manca, usare una postazione predisposta dal docente.

Copiare in `consegna03.md` questo modello e completarlo durante il lavoro:

```markdown
# Consegna Lab 03
Studente:
Sistema operativo e versioni Python/curl:
Porta usata:
Attività 5: cattura reale / modello sintetico
Attività 6: query reale / caso offline

## Schema e URL

## Registro delle prove
| ID  | Unica variabile modificata | Previsione PRIMA del test | Stato HTTP o errore | Uscita curl | Corpo | File di evidenza e log server | Conclusione |
| --- | -------------------------- | ------------------------- | ------------------- | ----------- | ----- | ----------------------------- | ----------- |

## Confronto GET/HEAD

## Analisi TCP

## Nomi e DNS

## Ticket di diagnosi
Sintomo:
Ipotesi iniziale e livello coinvolto:
Prova scelta e motivo:
Risultato osservato:
Correzione:
Controprova di ripristino:

## Risposte finali e cleanup
```

**Checkpoint:** entrambi i terminali sono in `LAB03`; `sito` contiene solo `messaggio.txt`; `sito/assente.txt` non esiste.

## 2. Modello e previsioni - 10 minuti

Prima di avviare il server:

1. Disegnare nella consegna `client curl -> TCP/IP su loopback -> server Python -> sito/messaggio.txt`. Racchiudere tutto in un riquadro che rappresenta **un solo computer**.
2. Scomporre `http://127.0.0.1:8080/messaggio.txt` in schema, host, porta e percorso. Indicare quale elemento cambiare per richiedere un altro file.
3. Associare HTTP a metodo/percorso/stato, TCP alle porte e alla connessione, IP agli indirizzi. Spiegare perché due processi possono comunicare usando lo stesso IP locale.
4. Scrivere le previsioni per file presente, file assente e server fermato: ci sarà una risposta HTTP? Una riga di richiesta nel log server?

Discutere: la porta del client deve essere 8080? Serve DNS per questo URL? Dove si trova il router dell'aula nel disegno?

**Checkpoint:** il disegno non introduce dispositivi intermedi inesistenti e distingue la porta del servizio dalla porta temporanea del client.

## 3. Tre prove HTTP controllate - 25 minuti

### Avvio del server e cattura facoltativa

Nel **terminale 1**, da `LAB03`, avviare il server. `--directory sito` sceglie la cartella pubblicata:

```bash
# Linux/macOS
python3 -m http.server 8080 --bind 127.0.0.1 --directory sito
```

```powershell
# Windows PowerShell
py -3 -m http.server 8080 --bind 127.0.0.1 --directory sito
```

Attendere il messaggio di ascolto e lasciare il terminale occupato dal server. Se la porta è occupata, scegliere con il docente `18080` e sostituire **8080 in tutti i comandi, URL e filtri**. Non terminare processi sconosciuti. Il binding mantiene il servizio sul computer locale.

Se Wireshark è disponibile, avviare ora una cattura sull'interfaccia `lo` (Linux), `lo0` (macOS) oppure sull'adattatore di cattura loopback di Npcap (Windows). Impostare il **capture filter** `tcp port 8080`. Non selezionare Wi-Fi o Ethernet; usare solo il testo sintetico del lab.

### Regola comune a tutte le prove

Scrivere la previsione, eseguire nel **terminale 2**, leggere il file prodotto e copiare la riga corrispondente del log dal terminale 1. Se non compare una nuova richiesta, annotarlo.

**Subito dopo ogni comando curl**, prima di altri comandi, leggere il codice di uscita e trascriverlo nel registro:

```bash
# Linux/macOS
echo $?
```

```powershell
# Windows PowerShell
$LASTEXITCODE
```

Le opzioni usate sono: `-sS` elimina la barra di avanzamento mantenendo gli errori; `-i` include le intestazioni; `--max-time 5` limita l'attesa; `--noproxy '*'` esclude il proxy per queste richieste locali. La redirezione salva risposta ed errori.

### P1 - File presente

```bash
# Linux/macOS
curl --noproxy '*' --max-time 5 -sS -i http://127.0.0.1:8080/messaggio.txt > evidenze/p1-presente.txt 2>&1
```

```powershell
# Windows PowerShell
curl.exe --noproxy "*" --max-time 5 -sS -i http://127.0.0.1:8080/messaggio.txt *> evidenze/p1-presente.txt
```

Identificare riga di stato, almeno due intestazioni, riga vuota di separazione e corpo `LAB03-OK`. Nel log cercare metodo, percorso e stato della stessa richiesta.

### P2 - Cambia soltanto il percorso

Verificare nel file manager che `sito/assente.txt` non esista, poi eseguire:

```bash
# Linux/macOS
curl --noproxy '*' --max-time 5 -sS -i http://127.0.0.1:8080/assente.txt > evidenze/p2-assente.txt 2>&1
```

```powershell
# Windows PowerShell
curl.exe --noproxy "*" --max-time 5 -sS -i http://127.0.0.1:8080/assente.txt *> evidenze/p2-assente.txt
```

Confrontare stato, tipo di contenuto e corpo con P1. Nel caso atteso il corpo è una pagina di errore e lo stato è **404**: il server ha risposto.

Questi comandi non usano `--fail`: una risposta HTTP 404 completa normalmente lascia curl con uscita **0**. Il trasferimento è riuscito, ma la risorsa manca. **Codice di uscita e stato HTTP sono informazioni diverse.**

### P3 - Stesso URL, contenuto diverso

Nell'editor sostituire il contenuto di `sito/messaggio.txt` con `LAB03-VERSIONE-2` e salvare, lasciando il server avviato. Prevedere che cosa cambierà:

```bash
# Linux/macOS
curl --noproxy '*' --max-time 5 -sS -i http://127.0.0.1:8080/messaggio.txt > evidenze/p3-modificato.txt 2>&1
```

```powershell
# Windows PowerShell
curl.exe --noproxy "*" --max-time 5 -sS -i http://127.0.0.1:8080/messaggio.txt *> evidenze/p3-modificato.txt
```

Confrontare P1 e P3: che cosa è invariato? Che cosa è cambiato? Annotare il `Content-Length` osservato: conta byte, inclusi eventuali a capo o marcatori di codifica, non soltanto lettere visibili. Ripristinare poi `LAB03-OK` nel file e salvare.

**Checkpoint:** tre righe del registro, tre risposte e log corrispondenti. Spiegare perché 200, da solo, non garantisce il contenuto desiderato.

## 4. GET contro HEAD - 15 minuti

Prima di proseguire, verificare che il contenuto sia tornato a `LAB03-OK`.

### P4 - Leggere richiesta e risposta

L'opzione `-v` mostra dettagli di connessione e richiesta. Salvare il corpo separatamente dalla traccia:

```bash
# Linux/macOS
curl --noproxy '*' --max-time 5 -sS -v http://127.0.0.1:8080/messaggio.txt -o evidenze/p4-corpo.txt 2> evidenze/p4-traccia.txt
```

```powershell
# Windows PowerShell
curl.exe --noproxy "*" --max-time 5 -sS -v http://127.0.0.1:8080/messaggio.txt -o evidenze/p4-corpo.txt 2> evidenze/p4-traccia.txt
```

Nella traccia cercare le righe di richiesta precedute da `>` e quelle di risposta precedute da `<`: metodo, percorso, `Host` e stato. Le righe con `*` sono diagnostica di curl. Registrare separatamente le versioni HTTP di richiesta e risposta: non devono necessariamente coincidere.

### P5 - Richiedere soltanto le intestazioni

Prevedere se HEAD restituirà il corpo, poi usare `-I` (i maiuscola):

```bash
# Linux/macOS
curl --noproxy '*' --max-time 5 -sS -I http://127.0.0.1:8080/messaggio.txt > evidenze/p5-head.txt 2>&1
```

```powershell
# Windows PowerShell
curl.exe --noproxy "*" --max-time 5 -sS -I http://127.0.0.1:8080/messaggio.txt *> evidenze/p5-head.txt
```

Compilare nella consegna:

| Elemento              | GET di P4 | HEAD di P5 |
| --------------------- | --------- | ---------- |
| Metodo nel log server |           |            |
| Stato HTTP            |           |            |
| Content-Type          |           |            |
| Content-Length        |           |            |
| Corpo ricevuto        |           |            |

Spiegare perché HEAD può riportare un `Content-Length` maggiore di zero senza trasferire il corpo: descrive la dimensione che avrebbe la risposta GET alla stessa risorsa. Non sostituire `-I` con `-X HEAD`.

**Checkpoint:** cambia il metodo; host, porta, percorso e contenuto del file rimangono uguali.

## 5. Ricostruire la conversazione TCP - 20 minuti

### Percorso A - Wireshark disponibile

1. Fermare e salvare la cattura in `evidenze/cattura-http.pcapng`. Se era stata avviata dopo le richieste, avviarne una nuova e ripetere P4 salvando gli output con suffisso `-ripetizione`.
2. Applicare il **display filter** `tcp.port == 8080`: la sintassi differisce dal capture filter.
3. Selezionare un pacchetto della richiesta al file, leggere `tcp.stream` e isolare quel flusso con `tcp.stream == N`, sostituendo N con il numero osservato.
4. Identificare SYN, SYN/ACK e ACK, se catturati. Annotare numeri dei frame, IP e porte sorgente/destinazione, flag e ruolo degli estremi. Gli IP possono coincidere: le porte distinguono client e server.
5. Individuare GET e risposta. Se necessario usare **Analyze -> Decode As -> HTTP** per la porta del lab. Con **Follow -> TCP Stream** confrontare i dati ricostruiti con P4 e il log server.
6. Rimuovere il filtro sul singolo flusso e cercare il 404 con `http.response.code == 404`, quando il traffico è decodificato come HTTP. Confrontarlo con il flusso riuscito.
7. Osservare la chiusura: annotare FIN/ACK o RST soltanto se presenti. Non aspettarsi un numero fisso di frame o un messaggio HTTP completo in ogni riga.

Aggiungere alla consegna la tabella `Evento | Frame osservato | Direzione e porte | Significato` per apertura, richiesta, risposta e chiusura. Scrivere `non osservato` per eventi mancanti e spiegarne il limite.

### Percorso B - Senza Wireshark o permessi di cattura

Scrivere `Cattura non eseguita`. Usare la traccia curl P4 come **evidenza reale applicativa** e questo **modello sintetico TCP**, senza inventare numeri di frame:

| Evento sintetico | Sorgente        | Destinazione    | Informazione                  |
| ---------------- | --------------- | --------------- | ----------------------------- |
| A                | 127.0.0.1:53000 | 127.0.0.1:8080  | SYN                           |
| B                | 127.0.0.1:8080  | 127.0.0.1:53000 | SYN, ACK                      |
| C                | 127.0.0.1:53000 | 127.0.0.1:8080  | ACK                           |
| D                | Client          | Server          | GET /messaggio.txt            |
| E                | Server          | Client          | Risposta 200 e corpo LAB03-OK |

D ed E non rappresentano necessariamente singoli segmenti. La porta 53000 è un esempio, non una misura del PC.

Per ogni evento spiegare funzione e livello; disegnare le frecce di ritorno; associare D ed E alle righe reali di P4. Elencare ciò che la traccia curl **non** dimostra, per esempio singoli flag TCP e numeri dei frame. Prevedere quali eventi applicativi mancherebbero con server fermato.

### Discussione comune

Rispondere in 2–3 frasi per punto:

- Perché un 404 non implica un handshake TCP fallito?
- Qual è la differenza tra segmento TCP e messaggio HTTP ricostruito?
- Perché su loopback non bisogna inventare MAC di switch o router?
- Lo scambio osservato usa TCP. Quali garanzie mancherebbero al trasporto usando UDP? Questo basta per affermare che UDP sia sempre più veloce? Considerare ordine, consegna e responsabilità dell'applicazione.

**Checkpoint:** osservazioni e modello sono distinti; non si deducono prestazioni di Internet da una prova locale.

## 6. Nomi, indirizzi e DNS - 10 minuti

### Query esplicita, se disponibile

Nel terminale 2 interrogare il resolver già configurato, senza cambiarlo:

```bash
# Linux/macOS
nslookup example.com > evidenze/dns.txt 2>&1
```

```powershell
# Windows PowerShell
nslookup example.com *> evidenze/dns.txt
```

Se il comando manca, la rete non è disponibile o la query fallisce, annotarlo e usare il caso offline. Se rimane in attesa, interrompere con Ctrl+C dopo circa 10 secondi.

Distinguere il **resolver interrogato** dagli **indirizzi del nome richiesto**. Annotare almeno un indirizzo restituito e la sua famiglia, IPv4 o IPv6. Gli indirizzi possono cambiare: non confrontarli con un valore prefissato. La risposta DNS non prova la disponibilità di un servizio HTTP.

### Caso offline alternativo

Analizzare questi dati **sintetici**, senza fare richieste agli indirizzi indicati:

```text
Resolver interrogato: 192.0.2.53
Nome richiesto: portale.example
Risposta A: 192.0.2.20
Risposta AAAA: 2001:db8::20
```

Identificare resolver, nome, IPv4 e IPv6. Descrivere il prossimo controllo necessario per sapere se HTTP sia disponibile. Riportare caso e analisi nella consegna.

### Confronto per tutti

| URL o comando                         | Interpretazione                                                                              |
| ------------------------------------- | -------------------------------------------------------------------------------------------- |
| `http://127.0.0.1:8080/messaggio.txt` | Host numerico: non serve risolverlo con DNS.                                                 |
| `http://localhost:8080/messaggio.txt` | Il nome può essere risolto localmente, anche in IPv6: non è garantita una query DNS in rete. |
| `nslookup example.com`                | Query esplicita a un resolver: non richiede un file al server del lab.                       |

Spiegare perché sostituire l'IP con `localhost` non garantisce una destinazione identica: il server è associato a `127.0.0.1`, mentre il nome può includere `::1`.

**Checkpoint:** non cercare questa query DNS nella cattura filtrata con `tcp port 8080`: quel filtro seleziona il traffico TCP del servizio locale.

## 7. Guasti e ripristino - 25 minuti

### P6 - Server fermato, URL invariato

Se si usa Wireshark, avviare una nuova cattura loopback con lo stesso capture filter. Nel terminale 1 fermare il proprio server con Ctrl+C, poi nel terminale 2 eseguire:

```bash
# Linux/macOS
curl --noproxy '*' --max-time 5 -sS -i http://127.0.0.1:8080/messaggio.txt > evidenze/p6-fermato.txt 2>&1
```

```powershell
# Windows PowerShell
curl.exe --noproxy "*" --max-time 5 -sS -i http://127.0.0.1:8080/messaggio.txt *> evidenze/p6-fermato.txt
```

Leggere subito il codice di uscita. Normalmente si osservano rifiuto di connessione, uscita curl 7 e nessuno stato HTTP; registrare il risultato effettivo. Un timeout è un esito diverso, da indagare, non una prova automatica di server spento.

Se effettuata, fermare e salvare la cattura in `evidenze/cattura-fermato.pcapng`. Cercare SYN ed eventuale RST e confrontare con la cattura funzionante. Annotare l'assenza di nuove richieste GET nel log.

### P7 - Ripristino controllato

Riavviare nel terminale 1 **lo stesso comando dell'attività 3**, con `--directory sito`. Ripetere il comando curl di P6, cambiando soltanto il file di output in `evidenze/p7-ripristino.txt`. Registrare uscita, stato e corpo: il recupero è dimostrato quando tornano risposta 200 e `LAB03-OK`.

### Ticket individuale - Guasto controllato e diagnosi verificabile

Dopo P7, scegliere **una sola** carta, annotare la causa introdotta e prevedere il sintomo. Eseguire la modifica, poi raccogliere le evidenze che permettono di distinguere quella causa dalle altre. Il problema da descrivere nel ticket è: “l'URL originale non restituisce più LAB03-OK”.

| Carta                          | Unica modifica da introdurre                                                                                             | Ripristino dopo la diagnosi                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------- |
| A - File spostato              | Rinominare `sito/messaggio.txt` in `messaggio-backup.txt` nella stessa cartella, lasciando il server attivo.             | Ripristinare il nome `messaggio.txt`.                   |
| B - Cartella pubblicata errata | Fermare il server; riavviarlo con il comando originale sostituendo solo `--directory sito` con `--directory sito-vuoto`. | Fermarlo e riavviare il comando con `--directory sito`. |
| C - Servizio assente           | Fermare il server con Ctrl+C.                                                                                            | Riavviare il comando originale.                         |

Completare individualmente i seguenti passi:

1. Prima della richiesta, scrivere il risultato previsto, il livello coinvolto e la prova che distingue il guasto introdotto da un'altra possibile causa.
2. Richiedere l'URL originale con il comando di P6, salvando in `evidenze/ticket-prima.txt`; annotare il codice di uscita.
3. Leggere risposta e log, poi controllare gli elementi pertinenti: processo, comando di avvio, cartella pubblicata, nome del file.
4. Proporre una correzione, confrontarla con la carta e applicarla.
5. Ripetere la richiesta salvando in `evidenze/ticket-dopo.txt`, verificare 200 e corpo atteso e completare il ticket nella consegna.

Se rimane tempo, ripetere individualmente con una seconda carta e usare `ticket2-prima.txt` e `ticket2-dopo.txt`.

**Checkpoint:** A e B possono produrre lo stesso 404 per cause diverse. Lo stato da solo non basta a scegliere la correzione: servono cartella, comando e file. Distinguere sintomo, evidenza e causa.

## 8. Consegna e cleanup - 15 minuti

### Domande finali

Rispondere citando le prove svolte:

1. Quale evidenza dimostra che in P2 il server HTTP era raggiungibile?
2. Perché curl può terminare con uscita 0 e mostrare HTTP 404?
3. Come distingui HEAD corretto da GET con contenuto inatteso?
4. Quali informazioni distinguono gli estremi TCP quando gli IP sono uguali?
5. Che cosa dimostra DNS e che cosa rimane da verificare?
6. Perché connessione rifiutata e timeout non sono diagnosi equivalenti?
7. Quale controprova dimostra il ripristino del ticket?
8. Quali conclusioni su Wi-Fi, router, Internet e HTTPS non puoi trarre da questo lab?

### Output da consegnare

- `consegna03.md`: schema, URL, registro P1–P7, confronto GET/HEAD, analisi TCP, DNS, ticket, risposte e cleanup.
- `sito/messaggio.txt`, ripristinato a `LAB03-OK`.
- In `evidenze`: `p1-presente.txt`, `p2-assente.txt`, `p3-modificato.txt`, `p4-corpo.txt`, `p4-traccia.txt`, `p5-head.txt`, `p6-fermato.txt`, `p7-ripristino.txt`, `log-server.txt`, `ticket-prima.txt`, `ticket-dopo.txt` e `cleanup.txt`.
- Se effettuate: `cattura-http.pcapng`, `cattura-fermato.pcapng` e `dns.txt`. Altrimenti dichiarare nella consegna il percorso alternativo svolto e i limiti delle osservazioni.

Verificare autonomamente che almeno una prova sia ricostruibile dai file consegnati. Non correggere il verbale per far coincidere risultati reali e attesi: spiegare divergenze e verifiche eseguite.

### Criteri di riuscita

| Criterio                | Evidenza minima                                               |
| ----------------------- | ------------------------------------------------------------- |
| Modello corretto        | Un host, due processi, IP/porte/percorso distinti             |
| Esperimenti controllati | Previsioni e risultati P1–P7; una variabile per prova         |
| Lettura HTTP            | Stato, intestazioni, corpo e log collegati; GET/HEAD distinti |
| Lettura TCP e DNS       | Misure separate da modelli e dati non disponibili             |
| Diagnosi verificabile   | Ipotesi, prova, correzione e controprova                      |
| Chiusura completa       | File conservati, server fermato e catture arrestate           |

### Cleanup obbligatorio

1. Dopo il recupero del ticket, fermare il server del lab nel terminale 1 con Ctrl+C.
2. Ripetere il comando di P6 salvando in `evidenze/cleanup.txt`; leggere subito l'uscita e confermare nella consegna l'assenza di risposta HTTP. Se risponde ancora, verificare terminale e porta prima di intervenire.
3. Fermare eventuali catture ancora attive, salvarle e chiudere Wireshark. Non lasciare il server avviato.
4. Conservare la cartella didattica. Non ci sono risorse cloud da eliminare.

## Troubleshooting rapido

| Sintomo                              | Controllo utile                                        | Interpretazione o azione                                           |
| ------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------ |
| Python o curl non trovato            | Comandi di versione                                    | Usare una postazione predisposta.                                  |
| Porta occupata                       | Messaggio all'avvio                                    | Scegliere la porta alternativa e aggiornare URL e filtri.          |
| P1 restituisce 404                   | Posizione dei terminali, directory, nome ed estensione | Il server risponde ma non trova il file pubblicato.                |
| P2 restituisce 200                   | Esistenza di `sito/assente.txt`                        | Usare un nome assente; salvare sempre le evidenze fuori da `sito`. |
| P3 mostra il vecchio testo           | File modificato e salvataggio                          | Controllare di avere modificato il file effettivamente servito.    |
| curl non mostra niente               | File indicato dalla redirezione                        | L'output è stato salvato: aprire quel file.                        |
| PowerShell mostra un errore in rosso | File di errore e `$LASTEXITCODE` immediato             | Lo stderr visualizzato non è uno stato HTTP; leggere il messaggio. |
| Wireshark non mostra traffico        | Loopback, porta, filtro e momento di avvio             | Correggere la selezione o usare il percorso alternativo.           |
| Wireshark mostra TCP ma non HTTP     | Decode As sulla porta del lab                          | L'etichetta del decoder da sola non prova assenza di HTTP.         |
| Cattura non consentita               | Permessi predisposti                                   | Usare il percorso B senza cambiare privilegi.                      |
| DNS fallisce ma P1 funziona          | URL numerico locale                                    | Le prove verificano funzioni diverse; usare il caso DNS offline.   |
| P6 restituisce ancora HTTP           | Terminale fermato, URL e porta                         | Identificare il listener senza terminare processi sconosciuti.     |

## Riferimenti tecnici

- [Python - http.server e cartella pubblicata](https://docs.python.org/3/library/http.server.html)
- [curl - opzioni, diagnostica e codici di uscita](https://curl.se/docs/manpage.html)
- [Wireshark - analisi dei pacchetti catturati](https://www.wireshark.org/docs/wsug_html_chunked/ChapterWork.html)
