# Lab 06 - Dal container al sito su AWS: immagini, ECR e rilascio

## Il risultato da costruire

Pubblicherai un piccolo sito «Officina Cloud» su una vera istanza EC2. Creerai due immagini del sito, le distribuirai tramite un repository privato Amazon ECR e farai un aggiornamento seguito da rollback.

La consegna principale è **una dimostrazione funzionante**: pagina personalizzata, immagine recuperata dal registry, versione 2 online e ritorno alla versione 1. Non serve una relazione Markdown. Conserva quattro screenshot essenziali e una breve scheda con nomi delle risorse, digest e cleanup.

## Organizzazione - 150 minuti

Nella lezione di 4 ore: 60 minuti di spiegazioni, 15 di pausa, 150 di laboratorio e 15 di confronto finale.

| Attività                                      | Minuti | Prova pratica                         |
| --------------------------------------------- | -----: | ------------------------------------- |
| 1. Preparare EC2 e Docker                     |     25 | Terminale remoto e motore funzionanti |
| 2. Personalizzare e confrontare due container |     25 | Due pagine diverse dalla stessa base  |
| 3. Creare e pubblicare immagini su ECR        |     25 | Due tag nel repository privato        |
| 4. Distribuire, aggiornare e tornare indietro |     30 | Sito raggiungibile e rollback         |
| 5. Riparare due guasti                        |     25 | Servizio ripristinato dopo ogni prova |
| 6. Dimostrazione e cleanup                    |     20 | Risorse eliminate e verificabili      |

Lavoro individuale; se gli account o le postazioni sono condivisi, lavorare in coppia alternando operatore e verificatore a metà del lab. Entrambi devono eseguire un rilascio.

**Come usare i comandi:** ogni blocco terminale contiene una sola operazione. Copiare un blocco, attendere, leggere il risultato e svolgere la verifica indicata prima di continuare. Il comando di login contiene una pipeline: è un'unica operazione che passa il token direttamente a Docker.

**Dove eseguirli:** tutti i comandi di questo lab si eseguono nel terminale Linux della propria EC2 aperto nel browser tramite Session Manager, anche se il proprio computer è Windows. Il browser del PC serve a visitare il sito. Non usare PowerShell, CloudShell o il terminale locale per questi comandi.

## Prima della lezione - preparazione del docente

Verificare nel Learner Lab reale: avvio EC2, associazione di `LabInstanceProfile`, Session Manager, accesso a ECR e download delle immagini. La policy fornita ammette i servizi coinvolti in `us-east-1` e limita dimensioni e ruoli; non garantisce disponibilità, budget o assenza di ulteriori vincoli dell'account.

Servono una subnet pubblica già predisposta, una route verso Internet Gateway, assegnazione IPv4 pubblico e uscita HTTPS/DNS. Si può usare la VPC predefinita se presente e verificata. Non chiedere agli studenti di creare NAT Gateway o infrastruttura di rete aggiuntiva durante questo lab.

Predisporre una VM sostitutiva per eventuali problemi Academy: **la pratica non viene sostituita da una descrizione ipotetica**. L'eventuale recupero deve includere esecuzione e verifica reali. I nomi del lab devono essere liberi nell'account e sulla macchina scelta.

## 1. Creare il server e preparare Docker - 25 minuti

### 1.1 Provisioning reale in Console

Avviare Learner Lab e aprire la Console assegnata. Selezionare **us-east-1**.

In `EC2 → Instances → Launch instances` impostare:

| Campo                                   | Valore                                                                |
| --------------------------------------- | --------------------------------------------------------------------- |
| Name                                    | `lab06-SIGLA`, sigla personale minuscola                              |
| AMI                                     | Amazon Linux 2023 ufficiale, x86_64                                   |
| Tipo                                    | `t3.micro`, On-Demand, tenancy default                                |
| Key pair                                | Proceed without key pair: useremo Session Manager                     |
| Rete                                    | Subnet pubblica predisposta dal docente                               |
| Auto-assign public IP                   | Enable                                                                |
| Security group nuovo                    | `lab06-SIGLA-sg`                                                      |
| Ingresso                                | Custom TCP **8080**, sorgente **My IP /32**                           |
| SSH                                     | Nessuna regola 22: Session Manager non la richiede                    |
| Uscita                                  | Configurazione della rete didattica che consenta Internet/HTTPS e DNS |
| Disco                                   | 8 GiB gp3, Delete on termination attivo                               |
| Advanced details → IAM instance profile | **LabInstanceProfile** già esistente                                  |
| Metadata                                | IMDSv2 richiesto; mantenere hop limit 1                               |

Non creare ruoli IAM, chiavi permanenti, Elastic IP o load balancer. Il sito usa HTTP e dati sintetici, accessibili solo dall'IP autorizzato; non inserire nomi di clienti, password o dati personali.

Lanciare **una sola istanza**. Attendere i controlli di stato, quindi `Connect → Session Manager → Connect`. La registrazione SSM può richiedere alcuni minuti.

Se il pulsante non si abilita, controllare profilo associato, agente SSM e connettività con il docente. Non aprire SSH a tutto Internet per aggirare il problema.

### 1.2 Preparare la sessione

Entrare in Bash, così tutti usano la stessa shell:

```bash
bash
```

Creare una cartella solo per questo esercizio:

```bash
mkdir -p "$HOME/lab06"
```

Entrare nella cartella:

```bash
cd "$HOME/lab06"
```

Installare Docker e l'editor sulla propria istanza didattica:

```bash
sudo dnf install -y docker nano
```

Avviare il servizio:

```bash
sudo systemctl enable --now docker
```

Verificare che compaiano sia Client sia Server:

```bash
sudo docker version
```

Verificare la CLI presente nell'AMI:

```bash
aws --version
```

Verificare l'identità utilizzata dall'istanza:

```bash
aws sts get-caller-identity --region us-east-1
```

L'ARN deve riferirsi al ruolo predisposto `LabRole`. Non eseguire `aws configure`: le credenziali temporanee arrivano dal profilo dell'istanza. Se la CLI manca o l'identità non è quella prevista, far verificare l'AMI e il profilo.

**Checkpoint pratico:** mostrare al docente il terminale remoto e Docker Server funzionante.

## 2. Una base, due esecuzioni, una pagina propria - 25 minuti

Scaricare la base:

```bash
sudo docker pull nginx:alpine
```

Leggere il riferimento per contenuto:

```bash
sudo docker image inspect nginx:alpine --format '{{json .RepoDigests}}'
```

Conservare il digest nella scheda. `alpine` è un tag mobile: non ripetere il pull durante questo esercizio.

Avviare il container di lavorazione, raggiungibile soltanto dall'host EC2:

```bash
sudo docker run -d --name lab06-autore -p 127.0.0.1:8081:80 nginx:alpine
```

Avviare una seconda esecuzione indipendente:

```bash
sudo docker run -d --name lab06-controllo -p 127.0.0.1:8082:80 nginx:alpine
```

Aprire un file con l'editor:

```bash
nano index.html
```

**Contenuto del file, da incollare nell'editor, non nel terminale:**

```html
<!doctype html>
<html lang="it">
<meta charset="utf-8">
<title>Officina Cloud</title>
<style>
body { font: 20px system-ui; max-width: 760px; margin: 60px auto; padding: 20px; }
h1 { color: #146c94; }
article { padding: 24px; background: #edf6fa; border-radius: 12px; }
</style>
<h1>Officina Cloud - squadra SIGLA</h1>
<article>
  <h2>Versione 1</h2>
  <p>Servizio: assistenza alle postazioni del laboratorio.</p>
  <ul>
    <li>Controllo delle connessioni di rete</li>
    <li>Ripristino delle applicazioni</li>
    <li>Preparazione dei container</li>
  </ul>
</article>
</html>
```

Sostituire SIGLA, cambiare il colore e aggiungere un quarto servizio inventato. Salvare con **Ctrl+O**, Invio; uscire con **Ctrl+X**.

Copiare la pagina nel solo container autore:

```bash
sudo docker cp index.html lab06-autore:/usr/share/nginx/html/index.html
```

Richiedere la sua pagina:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8081/
```

Deve comparire il proprio HTML. Ora controllare l'altra esecuzione:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8082/
```

La pagina di controllo deve essere ancora quella originale Nginx. Non basta avere due container Running: verificare **due contenuti diversi**.

Individuare la modifica nel filesystem:

```bash
sudo docker diff lab06-autore
```

Creare una prima immagine dal container modificato:

```bash
sudo docker commit lab06-autore officina06:1.0
```

Questa fotografia è utile per capire cosa entra in un'immagine. Non registra una ricetta ripetibile delle operazioni; nel lab 07 la sostituiremo con un Dockerfile. Non inserire credenziali o dati nel container prima del commit.

Riaprire la pagina:

```bash
nano index.html
```

Cambiare «Versione 1» in «Versione 2» e aggiungere una riga «Nuovo servizio: gestione ticket». Salvare e uscire.

Aggiornare il container autore:

```bash
sudo docker cp index.html lab06-autore:/usr/share/nginx/html/index.html
```

Controllare che serva V2:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8081/
```

Creare il secondo artefatto:

```bash
sudo docker commit lab06-autore officina06:2.0
```

Confrontare gli identificatori:

```bash
sudo docker image inspect officina06:1.0 officina06:2.0 --format '{{.RepoTags}} {{.Id}}'
```

**Checkpoint pratico:** i due tag locali hanno ID diversi. Il container di controllo è rimasto invariato.

## 3. Pubblicare davvero in Amazon ECR - 25 minuti

In Console aprire `ECR → Private repositories → Create repository`. Creare `lab06-SIGLA` in `us-east-1`, con tag immutabili e cifratura predefinita AES-256, senza nuova chiave KMS. Non abilitare filtri di mutabilità, replica o funzionalità aggiuntive.

Copiare l'URI del proprio repository. Ha forma:

```text
123456789012.dkr.ecr.us-east-1.amazonaws.com/lab06-sigla
```

Nel terminale EC2 assegnare l'URI **reale**, sostituendo tutto il valore di esempio:

```bash
REPO_URI='123456789012.dkr.ecr.us-east-1.amazonaws.com/lab06-sigla'
```

Ricavare il nome del registry, cioè la parte prima della barra:

```bash
REGISTRY=${REPO_URI%%/*}
```

Verificare i valori prima di usarli:

```bash
printf 'Repository: %s\nRegistry: %s\n' "$REPO_URI" "$REGISTRY"
```

Autenticare Docker senza stampare il token:

```bash
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin "$REGISTRY"
```

Attendere `Login Succeeded`. Se fallisce, verificare sessione, ruolo e regione; non proseguire con il push e non copiare credenziali nei file.

Applicare il riferimento remoto alla V1:

```bash
sudo docker tag officina06:1.0 "$REPO_URI:1.0"
```

Pubblicare V1:

```bash
sudo docker push "$REPO_URI:1.0"
```

Annotare il digest del push. Il push trasferisce un'immagine: **non ha avviato un sito**.

Applicare il riferimento remoto alla V2:

```bash
sudo docker tag officina06:2.0 "$REPO_URI:2.0"
```

Pubblicare V2:

```bash
sudo docker push "$REPO_URI:2.0"
```

In ECR aggiornare la pagina del repository: devono comparire `1.0` e `2.0` con digest differenti. Conservare lo screenshot 1, ritagliando i dati dell'account non necessari.

**Piccola sfida:** prima di eseguirlo, spiegare a voce se un altro tag locale produrrebbe necessariamente un nuovo contenuto. Verificarlo con:

```bash
sudo docker tag officina06:1.0 officina06:stabile
```

Confrontare gli ID:

```bash
sudo docker image inspect officina06:1.0 officina06:stabile --format '{{.RepoTags}} {{.Id}}'
```

Gli ID coincidono: aggiungere un nome non ricostruisce il software. I tag immutabili di ECR impediscono invece di sovrascrivere un tag remoto già pubblicato; per un nuovo rilascio si sceglie un nuovo tag.

## 4. Recuperare dal registry, distribuire e fare rollback - 30 minuti

### 4.1 Togliere i riferimenti locali alla V1

Rimuovere i riferimenti alla V1, mantenendo i container di lavoro:

```bash
sudo docker image rm officina06:stabile officina06:1.0 "$REPO_URI:1.0"
```

Dimostrare che il riferimento remoto non è più risolvibile localmente:

```bash
sudo docker image inspect "$REPO_URI:1.0"
```

**Errore atteso:** No such image. Non cancellare altri oggetti per ottenere l'errore.

Recuperare l'immagine da ECR:

```bash
sudo docker pull "$REPO_URI:1.0"
```

Confrontare il digest mostrato con quello del push. Docker può riusare layer in cache: il pull non prova che ogni byte sia stato riscaricato.

### 4.2 Pubblicare il sito sulla porta autorizzata

Avviare il rilascio V1:

```bash
sudo docker run -d --name lab06-sito --restart unless-stopped -p 8080:80 "$REPO_URI:1.0"
```

Controllare prima dalla EC2:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/
```

Nel **browser del PC**, aprire `http://IP-PUBBLICO-EC2:8080` sostituendo l'indirizzo dalla Console. Deve apparire V1 con la propria sigla. Usare HTTP esplicito; non è configurato TLS.

Conservare screenshot 2. La pubblicazione Docker ascolta sulle interfacce del server; il security group limita chi può raggiungerla. `127.0.0.1` nel browser del PC indicherebbe il PC, non EC2.

### 4.3 Rilasciare V2

Scaricare il riferimento V2:

```bash
sudo docker pull "$REPO_URI:2.0"
```

Fermare il rilascio corrente:

```bash
sudo docker stop lab06-sito
```

Rimuovere il container fermato:

```bash
sudo docker rm lab06-sito
```

Avviare V2 sulla stessa porta:

```bash
sudo docker run -d --name lab06-sito --restart unless-stopped -p 8080:80 "$REPO_URI:2.0"
```

Ricaricare il browser, anche con Ctrl+F5. Verificare la nuova scritta e il nuovo servizio; conservare screenshot 3. Questo rilascio causa una breve interruzione: non è un aggiornamento senza downtime.

### 4.4 Rollback operativo

Il docente annuncia: «La versione approvata è ancora V1». Non modificare HTML dentro il container pubblicato: ridistribuire l'artefatto precedente.

Fermare V2:

```bash
sudo docker stop lab06-sito
```

Rimuoverla:

```bash
sudo docker rm lab06-sito
```

Riavviare V1:

```bash
sudo docker run -d --name lab06-sito --restart unless-stopped -p 8080:80 "$REPO_URI:1.0"
```

Verificare contenuto nel browser e riferimento del container:

```bash
sudo docker inspect lab06-sito --format '{{.Config.Image}} {{.Image}}'
```

**Checkpoint pratico:** stesso indirizzo web, ritorno a V1, immagine corretta. Conservare screenshot 4 della verifica.

## 5. Due guasti reali da riparare - 25 minuti

### Guasto A - Il processo si ferma

Fermare volontariamente il proprio sito:

```bash
sudo docker stop lab06-sito
```

Provare una richiesta dalla EC2:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/
```

Non deve arrivare una risposta HTTP. Un errore di connessione non è un 404.

Cercare il container anche tra quelli fermati:

```bash
sudo docker ps -a --filter name=lab06-sito
```

Leggere gli ultimi log:

```bash
sudo docker logs --tail 15 lab06-sito
```

Ripristinare:

```bash
sudo docker start lab06-sito
```

Ripetere la stessa richiesta:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/
```

Deve tornare V1. `unless-stopped` non annulla uno stop esplicito dell'operatore.

### Guasto B - Il processo funziona, la rete non lo lascia raggiungere

Nella Console aprire **il solo security group creato per questo lab**. Annotare la regola iniziale e rimuovere temporaneamente l'ingresso TCP 8080.

Aprire una nuova finestra privata sul PC e riprovare il sito. Le connessioni già stabilite possono sopravvivere per il connection tracking; non usare la pagina rimasta in cache come prova. Una richiesta nuova deve fallire dopo l'applicazione della regola.

Dal terminale EC2 verificare invece:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/
```

La risposta locale deve funzionare. Ripristinare la sola regola TCP 8080 da **My IP /32**, quindi riprovare dal browser.

**La prova è conclusa quando l'accesso esterno torna a funzionare.** Non modificare immagini o container per correggere una regola di rete.

### Sfida finale - 5 minuti, compresi nei 25

Il docente sceglie V1 o V2. Senza ricopiare HTML, pubblicare la versione richiesta usando il ciclo stop → rm → run già sperimentato. Mostrare pagina, immagine utilizzata e una riga di access log generata da una nuova richiesta.

Se resta tempo, creare una V3 con una modifica scelta dallo studente, pubblicarla con un nuovo tag e dimostrare il ritorno alla versione precedente.

## 6. Consegna pratica e cleanup - 20 minuti

Il docente verifica dal vivo quattro risultati: pagina personalizzata, due immagini in ECR, aggiornamento/rollback e recupero dei due guasti. La scheda finale può essere un foglio o un file di testo: **sigla, ID istanza, nome repository, due digest, risorse rimosse**. Non servono tabelle di previsioni o una relazione Markdown.

Prima di eliminare, salvare i quattro screenshot sul PC. Se si vuole conservare il sorgente, visualizzarlo e copiarlo in un file locale:

```bash
cat index.html
```

### Cleanup sul server

Rimuovere soltanto i tre container creati qui:

```bash
sudo docker rm -f lab06-sito lab06-autore lab06-controllo
```

Verificare che non compaiano più:

```bash
sudo docker ps -a --filter name=lab06-
```

Disconnettere Docker dal registry:

```bash
sudo docker logout "$REGISTRY"
```

Non usare prune e non cancellare oggetti di altri laboratori.

### Cleanup in Console

1. In ECR selezionare **il proprio repository lab06-SIGLA** e cancellarlo con le sue immagini. Rileggere il nome prima di confermare; verificare che non compaia più.
2. In EC2 selezionare **la propria istanza lab06-SIGLA → Instance state → Terminate**. Stop non chiude il ciclo: i volumi restano allocati.
3. Attendere `Terminated`. In Volumes verificare che il disco radice, di cui si era annotata l'associazione, sia stato eliminato. Non cancellare dischi sconosciuti.
4. Eliminare il solo security group del lab dopo il rilascio delle interfacce di rete. Se è ancora in uso, attendere e ricontrollare.
5. Annotare eventuali residui e l'errore; chiudere Learner Lab secondo la procedura dell'aula.

La terminazione elimina anche immagini e file locali sul disco radice. Non è necessario rimuoverli singolarmente prima di terminare l'istanza. L'uso di EC2, EBS, IPv4 pubblico ed ECR consuma budget Academy: niente risorse lasciate attive fra le lezioni senza un accordo esplicito del docente.

## Diagnosi rapida

| Sintomo                                        | Verifica utile                                       | Azione                                            |
| ---------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| Session Manager non disponibile                | Profilo, agente, IPv4 pubblico, route e uscita HTTPS | Correggere l'ambiente con il docente              |
| ECR AccessDenied                               | ARN LabRole, regione, URI e permessi effettivi       | Non creare nuovi ruoli; conservare l'errore       |
| Tag remoto già esistente                       | Repository con tag immutabili                        | Usare un nuovo tag per il nuovo contenuto         |
| Docker non risponde                            | Stato servizio e uso di sudo                         | Avviare il servizio sulla propria VM              |
| Sito locale corretto, browser esterno bloccato | IP pubblico, TCP 8080, sorgente My IP                | Correggere solo l'accesso di rete                 |
| Pagina Nginx standard                          | Container, porta e COPY effettuata con docker cp     | Controllare quale immagine è stata pubblicata     |
| Sessione terminale riaperta                    | Variabili REPO_URI e REGISTRY assenti                | Riassegnarle con i comandi della sezione 3        |
| Budget o provisioning bloccato                 | Messaggio Academy                                    | Usare la postazione reale predisposta dal docente |

## Riferimenti operativi

- [AWS: ciclo di vita delle immagini in ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html)
- [AWS: autenticazione Docker in ECR](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login-password.html)
- [AWS: prerequisiti Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)
- [Docker: commit di un container](https://docs.docker.com/reference/cli/docker/container/commit/)

