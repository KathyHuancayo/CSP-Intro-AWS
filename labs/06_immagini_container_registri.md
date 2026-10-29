# Lab 06 - Dal container al sito su AWS: immagini, ECR e rilascio

## Il risultato da costruire

Pubblicherai un piccolo sito «Officina Cloud» su una vera istanza EC2. Creerai due immagini del sito, le distribuirai tramite un repository privato Amazon ECR e farai un aggiornamento seguito da rollback.

La consegna principale è **una dimostrazione funzionante**: pagina personalizzata, immagine recuperata dal registry, versione 2 online e ritorno alla versione 1. Non serve una relazione Markdown. Conserva quattro screenshot essenziali e una breve scheda con nomi delle risorse, digest e cleanup.

## Organizzazione - 150 minuti

Nella lezione di 4 ore: 60 minuti di spiegazioni, 15 di pausa, 150 di laboratorio e 15 di confronto finale.

| Attività                                      | Minuti | Prova pratica                         |
| --------------------------------------------- | -----: | ------------------------------------- |
| 1. Preparare EC2 e Docker                     | 40 | Terminale remoto e motore funzionanti |
| 2. Personalizzare e confrontare due container | 20 | Due pagine diverse dalla stessa base  |
| 3. Creare e pubblicare immagini su ECR        | 20 | Due tag nel repository privato        |
| 4. Distribuire, aggiornare e tornare indietro | 25 | Sito raggiungibile e rollback         |
| 5. Riparare due guasti                        | 20 | Servizio ripristinato dopo ogni prova |
| 6. Dimostrazione e cleanup                    | 25 | Risorse eliminate e verificabili      |

Lavoro individuale; se gli account o le postazioni sono condivisi, lavorare in coppia alternando operatore e verificatore a metà del lab. Entrambi devono eseguire un rilascio.

**Come usare i comandi:** ogni blocco terminale contiene una sola operazione. Copiare un blocco, attendere, leggere il risultato e svolgere la verifica indicata prima di continuare. Il comando di login contiene una pipeline: è un'unica operazione che passa il token direttamente a Docker.

**Scegliere i comandi del proprio ambiente:** nelle sezioni 2, 3 e 4.1 ogni comando ha la variante **Linux — terminale EC2** e **Windows — PowerShell sul PC**. Eseguire una sola variante. Il contenuto HTML è comune.

- **Linux:** preparare e pubblicare le immagini dalla EC2, come nel percorso originale.
- **Windows:** preparare e provare le immagini con Docker Desktop sul PC, pubblicarle su ECR e poi distribuirle sulla EC2. La sezione Windows seguente prepara il PC.
- **Deploy AWS:** dalla sezione 4.2 tutti usano il terminale Linux della EC2 tramite Session Manager nel browser. I comandi del server restano Linux anche quando il PC è Windows. Le variabili PowerShell non vengono trasferite al terminale remoto.

La sezione 1 prepara il server AWS per entrambi i percorsi. Non eseguire due volte gli stessi esercizi sui due motori Docker.

## Windows - preparare PowerShell sul PC

Per i blocchi **Windows — PowerShell sul PC** usare Docker Desktop con container Linux e PowerShell 7.3 o successivo. Se mancano, completarli una sola volta prima della lezione:

1. **PowerShell:** aprire la [pagina Microsoft](https://learn.microsoft.com/en-us/powershell/scripting/install/install-powershell-on-windows), scaricare l'installer MSI della versione stabile per il proprio PC, aprirlo → **Next → Install → Finish**. Dal menu Start aprire **PowerShell 7**.
2. **WSL 2:** aprire PowerShell **come amministratore** e verificare:

```powershell
wsl --status
```

Solo se WSL manca, installarlo:

```powershell
wsl --install --no-distribution
```

Riavviare Windows se richiesto, poi aggiornare WSL:

```powershell
wsl --update
```

3. **Docker Desktop:** aprire la [pagina ufficiale Windows](https://docs.docker.com/desktop/setup/install/windows-install/), scaricare l'installer adatto al PC e avviarlo. Se proposta, scegliere l'opzione **WSL 2**. Completare l'installazione, riavviare se richiesto e aprire **Docker Desktop** dal menu Start. In **Settings → General**, attivare **Use the WSL 2 based engine**, se l'opzione è disponibile, e applicare. Attendere che il motore sia avviato; dal menu Docker scegliere **Switch to Linux containers** solo se si stanno usando container Windows.
4. Se l'installazione richiede permessi amministrativi o la virtualizzazione risulta disabilitata, far completare la configurazione al supporto della postazione. Non serve aprire una shell Ubuntu: i comandi Windows del lab si eseguono in PowerShell.

Aprire una nuova finestra PowerShell dedicata al lab, senza privilegi amministrativi. Verificare la versione:

```powershell
$PSVersionTable.PSVersion
```

Se appare 5.1, aprire PowerShell 7 dal menu Start. Verificare il motore:

```powershell
docker version
```

Devono comparire Client e Server. Controllare il tipo di container:

```powershell
docker info --format '{{.OSType}}'
```

Atteso `linux`. Se il daemon non risponde, avviare Docker Desktop; se compare `windows`, passare a Linux containers dal menu Docker Desktop. Non eseguire `sudo`, `apt-get`, `dnf` o `systemctl` in PowerShell.

Controllare che il contesto punti al proprio Docker Desktop, non a un server remoto:

```powershell
docker context ls
```

I nomi del lab devono essere liberi anche nel Docker Desktop locale. Se esistono risorse di altri esercizi con gli stessi nomi, far verificare la postazione prima di continuare.

Preparare la cartella in Documenti, anche se reindirizzata su OneDrive:

```powershell
$LabDir = Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'lab06-windows'
```

Creare la cartella:

```powershell
New-Item -ItemType Directory -Path $LabDir -Force
```

Entrarvi:

```powershell
Set-Location $LabDir
```

Verificare il percorso prima di aprire i file o lanciare una build:

```powershell
Get-Location
```

Nei blocchi Windows, `notepad.exe` sostituisce `nano`. Incollare il contenuto nell'editor, salvare in **UTF-8** e scegliere **Tutti i file**: `Dockerfile`, `.dockerignore` e `app.py` non devono diventare file `.txt`. Per salvare usare Ctrl+S; Ctrl+O/Ctrl+X sono invece le scorciatoie descritte per nano su Linux.

Controllare i nomi, comprese le estensioni:

```powershell
Get-ChildItem -Force
```

Per HTTP usare `curl.exe`, non `curl`. Per leggere il codice di uscita di un comando Docker/AWS/curl, eseguire immediatamente:

```powershell
$LASTEXITCODE
```

## Preparazione dell’ambiente

Verificare nel Learner Lab reale: avvio EC2, associazione di `LabInstanceProfile`, Session Manager, accesso a ECR e download delle immagini. La policy fornita ammette i servizi coinvolti in `us-east-1` e limita dimensioni e ruoli; non garantisce disponibilità, budget o assenza di ulteriori vincoli dell'account.

Gli studenti creano la propria rete seguendo la sezione 1.0: VPC, subnet pubblica, Internet Gateway, route table e security group. Non è richiesta una rete preesistente.

Predisporre una VM sostitutiva per eventuali problemi Academy: **la pratica non viene sostituita da una descrizione ipotetica**. L'eventuale recupero deve includere esecuzione e verifica reali. I nomi del lab devono essere liberi nell'account e sulla macchina scelta.

## 1. Creare il server e preparare Docker - 40 minuti

### 1.0 Creare la rete AWS - guida breve completa

Usare la **Console web**, su Windows o Linux. Sostituire `SIGLA` con una sigla minuscola, es. `ab12`. Annotare gli ID delle risorse create; lasciare invariate le reti esistenti.

1. **AWS:** Academy → **Start Lab** → indicatore verde → **AWS**. Regione: **US East (N. Virginia), us-east-1**. Aprire **VPC**.

2. **VPC:** **Your VPCs → Create VPC → VPC only**. Name tag: `lab06-SIGLA-vpc`; IPv4: **Manual input**, `10.60.0.0/16`; IPv6: **No IPv6 CIDR block**; Tenancy: **Default** → **Create VPC**.

3. **DNS:** selezionare la VPC → **Actions → Edit VPC settings**. Abilitare **DNS resolution** e **DNS hostnames** → **Save changes**. Mantenere il DHCP option set predefinito, con **AmazonProvidedDNS**.

4. **Subnet:** **Subnets → Create subnet**. VPC: quella appena creata; Name: `lab06-SIGLA-public`; AZ: **us-east-1a** (o altra AZ standard disponibile); VPC CIDR: `10.60.0.0/16`; Subnet CIDR: `10.60.1.0/24` → **Create subnet**.

5. **IPv4 pubblico:** selezionare la subnet → **Actions → Edit subnet settings** → abilitare **Enable auto-assign public IPv4 address** → **Save**.

6. **Gateway:** **Internet gateways → Create internet gateway**. Name: `lab06-SIGLA-igw` → **Create internet gateway**. Poi **Actions → Attach to VPC** → `lab06-SIGLA-vpc` → **Attach internet gateway**. Stato atteso: **Attached**.

7. **Route table:** **Route tables → Create route table**. Name: `lab06-SIGLA-rt-public`; VPC: `lab06-SIGLA-vpc` → **Create route table**. In **Routes → Edit routes → Add route**: Destination `0.0.0.0/0`; Target **Internet Gateway** → proprio `igw-...` → **Save changes**. Conservare `10.60.0.0/16 → local`.

8. **Associazione:** nella route table → **Subnet associations → Edit subnet associations** → selezionare solo `lab06-SIGLA-public` → **Save associations**. Lasciare **Main: No**.

9. **Security group:** **Security groups → Create security group**. Name: `lab06-SIGLA-sg`; Description: `Accesso web laboratorio 06`; VPC: `lab06-SIGLA-vpc`. **Inbound → Add rule:** tipo **Custom TCP**, TCP **8080**; Source **My IP**, cioè IP pubblico del PC con `/32`. **Outbound:** **All traffic → 0.0.0.0/0** → **Create security group**. Nessun ingresso SSH o All traffic.

10. **Controllo:** **Subnets → `lab06-SIGLA-public` → Route table**: verificare `lab06-SIGLA-rt-public` e `0.0.0.0/0 → igw-...`, **Active**. Mantenere la **Network ACL predefinita**, con traffico consentito in ingresso e uscita.

**Per la EC2:** scegliere questa VPC, questa subnet e questo security group; **Auto-assign public IP: Enable**. Non servono NAT Gateway, Elastic IP o VPC endpoint. Se cambia l'IP del PC, aggiornare **My IP /32** nel gruppo.

**IAM:** **Advanced details → IAM instance profile → LabInstanceProfile**, fornito da Academy con `LabRole`. Se manca, verificare account/sessione con il supporto Academy; la policy studente non consente di creare un nuovo ruolo IAM.

### 1.1 Provisioning reale in Console

Avviare Learner Lab e aprire la Console assegnata. Selezionare **us-east-1**.

Aprire **EC2 → Instances → Launch instances**. In **Network settings → Edit**, selezionare esplicitamente VPC e subnet della tabella; in **Firewall (security groups)** scegliere **Select existing security group**, rimuovere eventuali selezioni automatiche e scegliere solo il gruppo del lab. Impostare:

| Campo                                   | Valore                                                                |
| --------------------------------------- | --------------------------------------------------------------------- |
| Name                                    | `lab06-SIGLA`, sigla personale minuscola                              |
| AMI                                     | Amazon Linux 2023 ufficiale, x86_64                                   |
| Tipo                                    | `t3.micro`, On-Demand, tenancy default                                |
| Key pair                                | Proceed without key pair: useremo Session Manager                     |
| Rete | VPC `lab06-SIGLA-vpc`, subnet `lab06-SIGLA-public` create nella sezione 1.0 |
| Auto-assign public IP                   | Enable                                                                |
| Security group | **Select existing security group** → `lab06-SIGLA-sg` |
| Ingresso                                | Custom TCP **8080**, sorgente **My IP /32**                           |
| SSH                                     | Nessuna regola 22: Session Manager non la richiede                    |
| Uscita | All traffic → `0.0.0.0/0`, nel gruppo creato nella sezione 1.0 |
| Disco                                   | 8 GiB gp3, Delete on termination attivo                               |
| Advanced details → IAM instance profile | **LabInstanceProfile** già esistente                                  |
| Metadata                                | IMDSv2 richiesto; mantenere hop limit 1                               |

Non creare ruoli IAM, chiavi permanenti, Elastic IP o load balancer. Il sito usa HTTP e dati sintetici, accessibili solo dall'IP autorizzato; non inserire nomi di clienti, password o dati personali.

**Prima di premere Launch instance:** espandere **Configure storage → Advanced** e verificare **Delete on termination** per il disco radice; in **Advanced details** selezionare `LabInstanceProfile`, **Metadata version: V2 only (token required)** e **Metadata response hop limit: 1**. Dopo l'avvio, in **Instances → lab06-SIGLA → Networking** verificare VPC/subnet corrette e un **Public IPv4 address** non vuoto. In **Security** deve esserci solo `lab06-SIGLA-sg`. Annotare ID istanza e volume radice.

Lanciare **una sola istanza**. Attendere i controlli di stato, quindi `Connect → Session Manager → Connect`. La registrazione SSM può richiedere alcuni minuti.

Se il pulsante non si abilita, controllare profilo associato, agente SSM e connettività. Non aprire SSH a tutto Internet per aggirare il problema.

**Se Session Manager non si connette:** attendere qualche minuto; verificare i controlli di stato, il profilo in **Actions → Security → Modify IAM role**, la route della subnet e l'uscita del security group. Se il profilo era assente, scegliere `LabInstanceProfile` e salvare. Usare l'AMI ufficiale indicata, con agente SSM; non aprire la porta 22 per sostituire questi controlli.

### 1.2 Preparare la sessione Linux sulla EC2 - anche da un PC Windows

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

Nel terminale Session Manager del browser, installare Docker e l'editor sulla EC2. Questo comando non va eseguito in PowerShell:

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

**Checkpoint pratico:** verificare che il terminale remoto sia accessibile e Docker Server sia funzionante.

## 2. Una base, due esecuzioni, una pagina propria - 20 minuti

Scaricare la base:

**Linux — terminale EC2:**

```bash
sudo docker pull nginx:alpine
```

**Windows — PowerShell sul PC:**

```powershell
docker pull --platform linux/amd64 nginx:alpine
```

Leggere il riferimento per contenuto:

**Linux — terminale EC2:**

```bash
sudo docker image inspect nginx:alpine --format '{{json .RepoDigests}}'
```

**Windows — PowerShell sul PC:**

```powershell
docker image inspect nginx:alpine --format '{{json .RepoDigests}}'
```

Conservare il digest nella scheda. `alpine` è un tag mobile: non ripetere il pull durante questo esercizio.

Avviare il container di lavorazione, raggiungibile soltanto dall'host EC2:

**Linux — terminale EC2:**

```bash
sudo docker run -d --name lab06-autore -p 127.0.0.1:8081:80 nginx:alpine
```

**Windows — PowerShell sul PC:**

```powershell
docker run -d --name lab06-autore -p 127.0.0.1:8081:80 nginx:alpine
```

Avviare una seconda esecuzione indipendente:

**Linux — terminale EC2:**

```bash
sudo docker run -d --name lab06-controllo -p 127.0.0.1:8082:80 nginx:alpine
```

**Windows — PowerShell sul PC:**

```powershell
docker run -d --name lab06-controllo -p 127.0.0.1:8082:80 nginx:alpine
```

Aprire un file con l'editor:

**Linux — terminale EC2:**

```bash
nano index.html
```

**Windows — PowerShell sul PC:**

```powershell
notepad.exe .\index.html
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

**Linux — terminale EC2:**

```bash
sudo docker cp index.html lab06-autore:/usr/share/nginx/html/index.html
```

**Windows — PowerShell sul PC:**

```powershell
docker cp index.html lab06-autore:/usr/share/nginx/html/index.html
```

Richiedere la sua pagina:

**Linux — terminale EC2:**

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8081/
```

**Windows — PowerShell sul PC:**

```powershell
curl.exe --noproxy "*" --max-time 5 -i http://127.0.0.1:8081/
```

Deve comparire il proprio HTML. Ora controllare l'altra esecuzione:

**Linux — terminale EC2:**

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8082/
```

**Windows — PowerShell sul PC:**

```powershell
curl.exe --noproxy "*" --max-time 5 -i http://127.0.0.1:8082/
```

La pagina di controllo deve essere ancora quella originale Nginx. Non basta avere due container Running: verificare **due contenuti diversi**.

Individuare la modifica nel filesystem:

**Linux — terminale EC2:**

```bash
sudo docker diff lab06-autore
```

**Windows — PowerShell sul PC:**

```powershell
docker diff lab06-autore
```

Creare una prima immagine dal container modificato:

**Linux — terminale EC2:**

```bash
sudo docker commit lab06-autore officina06:1.0
```

**Windows — PowerShell sul PC:**

```powershell
docker commit lab06-autore officina06:1.0
```

Questa fotografia è utile per capire cosa entra in un'immagine. Non registra una ricetta ripetibile delle operazioni; nel lab 07 la sostituiremo con un Dockerfile. Non inserire credenziali o dati nel container prima del commit.

Riaprire la pagina:

**Linux — terminale EC2:**

```bash
nano index.html
```

**Windows — PowerShell sul PC:**

```powershell
notepad.exe .\index.html
```

Cambiare «Versione 1» in «Versione 2» e aggiungere una riga «Nuovo servizio: gestione ticket». Salvare e uscire.

Aggiornare il container autore:

**Linux — terminale EC2:**

```bash
sudo docker cp index.html lab06-autore:/usr/share/nginx/html/index.html
```

**Windows — PowerShell sul PC:**

```powershell
docker cp index.html lab06-autore:/usr/share/nginx/html/index.html
```

Controllare che serva V2:

**Linux — terminale EC2:**

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8081/
```

**Windows — PowerShell sul PC:**

```powershell
curl.exe --noproxy "*" --max-time 5 -i http://127.0.0.1:8081/
```

Creare il secondo artefatto:

**Linux — terminale EC2:**

```bash
sudo docker commit lab06-autore officina06:2.0
```

**Windows — PowerShell sul PC:**

```powershell
docker commit lab06-autore officina06:2.0
```

Confrontare gli identificatori:

**Linux — terminale EC2:**

```bash
sudo docker image inspect officina06:1.0 officina06:2.0 --format '{{.RepoTags}} {{.Id}}'
```

**Windows — PowerShell sul PC:**

```powershell
docker image inspect officina06:1.0 officina06:2.0 --format '{{.RepoTags}} {{.Id}}'
```

**Checkpoint pratico:** i due tag locali hanno ID diversi. Il container di controllo è rimasto invariato.

## 3. Pubblicare davvero in Amazon ECR - 20 minuti

### Solo Windows - credenziali temporanee Academy per ECR

Questi comandi si eseguono nella finestra **PowerShell dedicata**. Sulla EC2 si continua a usare `LabInstanceProfile`: non copiare credenziali sul server o dentro le immagini.

Se sul PC manca AWS CLI v2: scaricare il [pacchetto MSI ufficiale](https://awscli.amazonaws.com/AWSCLIV2.msi), aprirlo → **Next → accettare la licenza → Install → Finish**. Chiudere e riaprire PowerShell 7 per aggiornare il PATH, poi tornare nella cartella del lab. Verificare:

```powershell
aws --version
```

Nella pagina Learner Lab aprire **AWS Details → AWS CLI** e usare i tre valori della sessione corrente. Inserirli ai prompt qui sotto, senza incollarli direttamente nei comandi o salvarli nei sorgenti. Non creare access key permanenti. I campi segreti sono mascherati; le variabili valgono soltanto nella finestra corrente.

Inserire l'access key ID temporaneo:

```powershell
$env:AWS_ACCESS_KEY_ID = Read-Host "AWS_ACCESS_KEY_ID della sessione Academy"
```

Inserire la secret access key:

```powershell
$env:AWS_SECRET_ACCESS_KEY = [System.Net.NetworkCredential]::new("", (Read-Host "AWS_SECRET_ACCESS_KEY" -AsSecureString)).Password
```

Inserire anche il token di sessione, obbligatorio:

```powershell
$env:AWS_SESSION_TOKEN = [System.Net.NetworkCredential]::new("", (Read-Host "AWS_SESSION_TOKEN" -AsSecureString)).Password
```

Verificare account e identità della sessione Academy prima del login ECR:

```powershell
aws sts get-caller-identity --region us-east-1
```

Se le credenziali sono scadute, rinnovare la sessione Academy e reinserire **tutti e tre** i valori. L'identità del PC è quella temporanea Academy; non deve necessariamente avere lo stesso ARN LabRole dell'istanza EC2.

### Repository e pubblicazione - entrambi i percorsi

In Console aprire `ECR → Private repositories → Create repository`. Creare `lab06-SIGLA` in `us-east-1`, con tag immutabili e cifratura predefinita AES-256, senza nuova chiave KMS. Non abilitare filtri di mutabilità, replica o funzionalità aggiuntive.

Copiare l'URI del proprio repository. Ha forma:

```text
123456789012.dkr.ecr.us-east-1.amazonaws.com/lab06-sigla
```

Nel terminale del percorso scelto assegnare l'URI **reale**, sostituendo tutto il valore di esempio:

**Linux — terminale EC2:**

```bash
REPO_URI='123456789012.dkr.ecr.us-east-1.amazonaws.com/lab06-sigla'
```

**Windows — PowerShell sul PC:**

```powershell
$RepoUri = '123456789012.dkr.ecr.us-east-1.amazonaws.com/lab06-sigla'
```

Ricavare il nome del registry, cioè la parte prima della barra:

**Linux — terminale EC2:**

```bash
REGISTRY=${REPO_URI%%/*}
```

**Windows — PowerShell sul PC:**

```powershell
$Registry = $RepoUri.Split('/')[0]
```

Verificare i valori prima di usarli:

**Linux — terminale EC2:**

```bash
printf 'Repository: %s\nRegistry: %s\n' "$REPO_URI" "$REGISTRY"
```

**Windows — PowerShell sul PC:**

```powershell
Write-Output "Repository: $RepoUri; Registry: $Registry"
```

Autenticare Docker senza stampare il token:

**Linux — terminale EC2:**

```bash
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin "$REGISTRY"
```

**Windows — PowerShell sul PC:**

```powershell
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin "$Registry"
```

Attendere `Login Succeeded`. Se fallisce, verificare sessione, ruolo e regione; non proseguire con il push e non copiare credenziali nei file.

Applicare il riferimento remoto alla V1:

**Linux — terminale EC2:**

```bash
sudo docker tag officina06:1.0 "$REPO_URI:1.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker tag officina06:1.0 "${RepoUri}:1.0"
```

Pubblicare V1:

**Linux — terminale EC2:**

```bash
sudo docker push "$REPO_URI:1.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker push "${RepoUri}:1.0"
```

Annotare il digest del push. Il push trasferisce un'immagine: **non ha avviato un sito**.

Applicare il riferimento remoto alla V2:

**Linux — terminale EC2:**

```bash
sudo docker tag officina06:2.0 "$REPO_URI:2.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker tag officina06:2.0 "${RepoUri}:2.0"
```

Pubblicare V2:

**Linux — terminale EC2:**

```bash
sudo docker push "$REPO_URI:2.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker push "${RepoUri}:2.0"
```

In ECR aggiornare la pagina del repository: devono comparire `1.0` e `2.0` con digest differenti. Conservare lo screenshot 1, ritagliando i dati dell'account non necessari.

**Piccola sfida:** prima di eseguirlo, spiegare a voce se un altro tag locale produrrebbe necessariamente un nuovo contenuto. Verificarlo con:

**Linux — terminale EC2:**

```bash
sudo docker tag officina06:1.0 officina06:stabile
```

**Windows — PowerShell sul PC:**

```powershell
docker tag officina06:1.0 officina06:stabile
```

Confrontare gli ID:

**Linux — terminale EC2:**

```bash
sudo docker image inspect officina06:1.0 officina06:stabile --format '{{.RepoTags}} {{.Id}}'
```

**Windows — PowerShell sul PC:**

```powershell
docker image inspect officina06:1.0 officina06:stabile --format '{{.RepoTags}} {{.Id}}'
```

Gli ID coincidono: aggiungere un nome non ricostruisce il software. I tag immutabili di ECR impediscono invece di sovrascrivere un tag remoto già pubblicato; per un nuovo rilascio si sceglie un nuovo tag.

## 4. Recuperare dal registry, distribuire e fare rollback - 25 minuti

### 4.1 Togliere i riferimenti locali alla V1

Rimuovere i riferimenti alla V1, mantenendo i container di lavoro:

**Linux — terminale EC2:**

```bash
sudo docker image rm officina06:stabile officina06:1.0 "$REPO_URI:1.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker image rm officina06:stabile officina06:1.0 "${RepoUri}:1.0"
```

Dimostrare che il riferimento remoto non è più risolvibile localmente:

**Linux — terminale EC2:**

```bash
sudo docker image inspect "$REPO_URI:1.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker image inspect "${RepoUri}:1.0"
```

**Errore atteso:** No such image. Non cancellare altri oggetti per ottenere l'errore.

Recuperare l'immagine da ECR:

**Linux — terminale EC2:**

```bash
sudo docker pull "$REPO_URI:1.0"
```

**Windows — PowerShell sul PC:**

```powershell
docker pull "${RepoUri}:1.0"
```

Confrontare il digest mostrato con quello del push. Docker può riusare layer in cache: il pull non prova che ogni byte sia stato riscaricato.

### 4.2 Pubblicare il sito sulla porta autorizzata

**Da qui: Linux nella EC2 per tutti, anche per chi ha costruito le immagini su Windows.** Aprire `EC2 → Connect → Session Manager`. Per il percorso Windows, inizializzare qui le variabili: quelle del PC non esistono sul server.

Impostare l'URI reale del repository già popolato da PowerShell:

```bash
REPO_URI='123456789012.dkr.ecr.us-east-1.amazonaws.com/lab06-sigla'
```

Ricavare il registry:

```bash
REGISTRY=${REPO_URI%%/*}
```

Autenticare il Docker del server usando il suo profilo IAM, senza copiare le credenziali del PC:

```bash
aws ecr get-login-password --region us-east-1 | sudo docker login --username AWS --password-stdin "$REGISTRY"
```

Scaricare sul server la V1 pubblicata in ECR:

```bash
sudo docker pull "$REPO_URI:1.0"
```

Avviare il rilascio V1:

```bash
sudo docker run -d --name lab06-sito --restart unless-stopped -p 8080:80 "$REPO_URI:1.0"
```

Controllare prima dalla EC2:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/
```

Nel **browser del PC**, aprire `http://IP-PUBBLICO-EC2:8080` sostituendo l'indirizzo dalla Console. Deve apparire V1 con la propria sigla. Usare HTTP esplicito; non è configurato TLS.

**Windows — controllare il deploy dal PC, in PowerShell.** Inserire l'IPv4 pubblico reale della EC2, senza protocollo o porta:

```powershell
$Ec2Ip = Read-Host "IPv4 pubblico della EC2"
```

Aprire il sito remoto:

```powershell
Start-Process "http://${Ec2Ip}:8080"
```

Verificare anche la risposta HTTP reale:

```powershell
curl.exe --noproxy "*" --max-time 5 -i "http://${Ec2Ip}:8080/"
```

Usare questa stessa richiesta dopo aggiornamento, rollback e riparazione dei guasti. `127.0.0.1:8081` in PowerShell raggiunge la prova locale; l'IPv4 pubblico raggiunge il server AWS.

Conservare screenshot 2. La pubblicazione Docker ascolta sulle interfacce del server; il security group limita chi può raggiungerla. `127.0.0.1` nel browser del PC indicherebbe il PC, non EC2.

### 4.3 Rilasciare V2

**Dove eseguire i comandi:** nella **EC2 Linux**, aprendo **AWS Console → EC2 → Instances → lab06-SIGLA → Connect → Session Manager → Connect**. Vale anche per chi usa un PC Windows. Questi comandi gestiscono il sito AWS; non incollarli nel PowerShell locale.

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

**Dove eseguire i comandi:** nella **EC2 Linux**, aprendo **AWS Console → EC2 → Instances → lab06-SIGLA → Connect → Session Manager → Connect**. Vale anche per chi usa un PC Windows. Questi comandi gestiscono il sito AWS; non incollarli nel PowerShell locale.

Ripristinare la versione V1 ridistribuendo l'immagine precedente.

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

**Windows — verifica dal PC dopo il rollback:** tornare in PowerShell. Se la finestra è nuova, impostare l'IPv4 pubblico della EC2:

```powershell
$Ec2Ip = Read-Host "IPv4 pubblico della EC2"
```

Richiedere la pagina del server AWS e controllare che mostri V1:

```powershell
curl.exe --noproxy "*" --max-time 5 -i "http://${Ec2Ip}:8080/"
```

Il comando `docker stop lab06-sito` eseguito in PowerShell fermerebbe un container locale omonimo, non quello AWS: per stop/rm/run usare la sessione EC2 indicata sopra.


## 5. Due guasti reali da riparare - 20 minuti

### Guasto A - Il processo si ferma

**Dove eseguire i comandi:** nella **EC2 Linux**, aprendo **AWS Console → EC2 → Instances → lab06-SIGLA → Connect → Session Manager → Connect**. Vale anche per chi usa un PC Windows. Questi comandi gestiscono il sito AWS; non incollarli nel PowerShell locale.

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

**Dove eseguire i comandi:** nella **EC2 Linux**, aprendo **AWS Console → EC2 → Instances → lab06-SIGLA → Connect → Session Manager → Connect**. Vale anche per chi usa un PC Windows. Questi comandi gestiscono il sito AWS; non incollarli nel PowerShell locale.

Nella Console aprire **il solo security group creato per questo lab**. Annotare la regola iniziale e rimuovere temporaneamente l'ingresso TCP 8080.

Aprire una nuova finestra privata sul PC e riprovare il sito. Le connessioni già stabilite possono sopravvivere per il connection tracking; non usare la pagina rimasta in cache come prova. Una richiesta nuova deve fallire dopo l'applicazione della regola.

Dal terminale EC2 verificare invece:

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/
```

La risposta locale deve funzionare. Ripristinare la sola regola TCP 8080 da **My IP /32**, quindi riprovare dal browser.

**La prova è conclusa quando l'accesso esterno torna a funzionare.** Non modificare immagini o container per correggere una regola di rete.

### Sfida finale - 5 minuti, compresi nei 20

Scegliere V1 o V2 e pubblicarla usando il ciclo stop → rm → run già sperimentato. Verificare la pagina, l'immagine utilizzata e una riga di access log generata da una nuova richiesta.

Se resta tempo, creare una V3 con una modifica scelta dallo studente, pubblicarla con un nuovo tag e dimostrare il ritorno alla versione precedente.

## 6. Consegna pratica e cleanup - 25 minuti

Verificare quattro risultati: pagina personalizzata, due immagini in ECR, aggiornamento/rollback e recupero dei due guasti. La scheda finale può essere un foglio o un file di testo: **sigla, ID istanza, nome repository, due digest, risorse rimosse**. Non servono tabelle di previsioni o una relazione Markdown.

Prima di eliminare, salvare i quattro screenshot sul PC. Se si vuole conservare il sorgente, visualizzarlo e copiarlo in un file locale:

```bash
cat index.html
```

### Cleanup sul server

Rimuovere il sito distribuito sul server, per entrambi i percorsi:

```bash
sudo docker rm -f lab06-sito
```

**Solo percorso Linux:** rimuovere anche i container di lavorazione creati sulla EC2. Nel percorso Windows questi due container sono sul PC e si rimuovono nel blocco successivo.

```bash
sudo docker rm -f lab06-autore lab06-controllo
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

### Cleanup Windows - solo per chi ha usato Docker Desktop

Tornare nella finestra PowerShell del PC. I container locali non vengono eliminati terminando EC2.

Rimuovere i due container di lavorazione locali:

```powershell
docker rm -f lab06-autore lab06-controllo
```

Verificarne l'assenza:

```powershell
docker ps -a --filter name=lab06-
```

Rimuovere i riferimenti alle immagini del lab rimasti dopo la sezione 4.1:

```powershell
docker image rm "${RepoUri}:1.0" "${RepoUri}:2.0" officina06:2.0
```

Se una prova è stata interrotta, verificare gli altri tag del lab e rimuovere soltanto quelli creati personalmente; conservare la base condivisa nginx:alpine.

Terminare il login ECR locale:

```powershell
docker logout $Registry
```

Rimuovere le credenziali dalla finestra dedicata:

```powershell
Remove-Item Env:AWS_ACCESS_KEY_ID, Env:AWS_SECRET_ACCESS_KEY, Env:AWS_SESSION_TOKEN -ErrorAction SilentlyContinue
```

Conservare `index.html` e gli screenshot, poi completare anche il cleanup AWS seguente.

### Cleanup in Console

Prima della rete: **ECR → Private repositories → lab06-SIGLA → Delete**. Confermare la cancellazione del proprio repository e delle sue immagini; verificare che non sia più elencato.

Poi eliminare le risorse AWS in quest'ordine:

1. In **EC2 → Instances** selezionare solo `lab06-SIGLA` → **Instance state → Terminate (delete) instance** → confermare. Attendere **Terminated**: Stop non elimina i dischi.
2. In **EC2 → Volumes** cercare l'ID del disco radice annotato all'avvio. Deve essere scomparso grazie a **Delete on termination**. Se è rimasto **Available**, verificarne la proprietà e cancellare solo quel disco con **Actions → Delete volume**.
3. In **EC2 → Network Interfaces** filtrare per l'ID della VPC del lab e attendere il rilascio dell'interfaccia della EC2. Non forzare la cancellazione di interfacce ancora in uso.
4. In **VPC → Security groups** selezionare `lab06-SIGLA-sg` → **Actions → Delete security groups** → confermare.
5. In **VPC → Subnets** selezionare `lab06-SIGLA-public` → **Actions → Delete subnet** → confermare. Controllare che non sia più elencata.
6. In **VPC → Route tables** selezionare `lab06-SIGLA-rt-public`, verificare **Main: No** e nessuna associazione residua → **Actions → Delete route table** → confermare.
7. In **VPC → Internet gateways** selezionare `lab06-SIGLA-igw` → **Actions → Detach from VPC** → confermare. Poi **Actions → Delete internet gateway** → confermare.
8. In **VPC → Your VPCs** selezionare `lab06-SIGLA-vpc` → **Actions → Delete VPC**. Controllare l'elenco delle risorse residue e confermare solo per la propria VPC. I suoi oggetti default vengono rimossi con essa; non cancellare il DHCP option set condiviso.
9. Aggiornare gli elenchi e verificare l'assenza dei propri ID. Se compare **DependencyViolation**, leggere la dipendenza indicata e completare la rimozione della propria risorsa prima di riprovare. Annotare eventuali residui; solo dopo chiudere Learner Lab.

La terminazione elimina anche immagini e file locali sul disco radice. Non è necessario rimuoverli singolarmente prima di terminare l'istanza. L'uso di EC2, EBS, IPv4 pubblico ed ECR consuma budget Academy: completare il cleanup delle risorse al termine del laboratorio.

## Diagnosi rapida

| Sintomo                                        | Verifica utile                                       | Azione                                            |
| ---------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| Session Manager non disponibile                | Profilo, agente, IPv4 pubblico, route e uscita HTTPS | Correggere la configurazione dell'ambiente              |
| ECR AccessDenied                               | ARN LabRole, regione, URI e permessi effettivi       | Non creare nuovi ruoli; conservare l'errore       |
| Tag remoto già esistente                       | Repository con tag immutabili                        | Usare un nuovo tag per il nuovo contenuto         |
| Docker non risponde                            | Stato servizio e uso di sudo                         | Avviare il servizio sulla propria VM              |
| Sito locale corretto, browser esterno bloccato | IP pubblico, TCP 8080, sorgente My IP                | Correggere solo l'accesso di rete                 |
| Pagina Nginx standard                          | Container, porta e COPY effettuata con docker cp     | Controllare quale immagine è stata pubblicata     |
| Sessione terminale riaperta                    | Variabili REPO_URI e REGISTRY assenti                | Riassegnarle con i comandi della sezione 3        |
| Budget o provisioning bloccato                 | Messaggio Academy                                    | Usare una postazione alternativa con ambiente operativo |

## Riferimenti operativi

- [AWS: ciclo di vita delle immagini in ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/getting-started-cli.html)
- [AWS: autenticazione Docker in ECR](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login-password.html)
- [AWS: prerequisiti Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)
- [Docker: commit di un container](https://docs.docker.com/reference/cli/docker/container/commit/)


- [Docker Desktop su Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [AWS CLI: variabili di ambiente in PowerShell](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)

- [AWS: creare una VPC](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html)
- [AWS: creare una subnet](https://docs.aws.amazon.com/vpc/latest/userguide/create-subnets.html)
- [AWS: IPv4 pubblico della subnet](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-public-ip.html)
- [AWS: Internet Gateway e route](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [AWS: associazioni delle route table](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html)
- [AWS: impostazioni DNS](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns-updating.html)
- [AWS: creazione del security group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-security-group.html)
- [AWS: eliminazione della VPC e dipendenze](https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html)
