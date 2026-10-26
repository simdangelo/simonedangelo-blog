# Documento tecnico di appoggio

**A chi è rivolto.** A chi deve saper spiegare questi concetti a un colloquio, non a chi li ha già usati per anni. Non do per scontato che tu abbia mai creato un cluster, una task definition o un load balancer. Parto dal problema, introduco un termine alla volta e lo uso subito in un esempio. Il tono resta da senior: ogni termine ha un costo, un limite e una frase da usare. Alla fine trovi le definizioni secche per il ripasso, ma non sono il punto di partenza.

**Come è scritto.** Le prime sezioni sono graduali; le ultime sono più dense. Se un passaggio ti sembra ovvio, puoi saltarlo; se un termine non ti è chiaro, la spiegazione dovrebbe essere già lì, non data per scontata.

**Esempio guida.** Immagina un worker Python che legge un job da DynamoDB (database gestito) e scrive un file Parquet (formato colonnare) su S3 (storage oggetti). Lo hai messo in un container. Vuoi eseguirlo una volta per un job, oppure tenerlo sempre vivo come API. È lo stesso codice, ma sono due modi diversi di usare ECS. Questo esempio torna in tutto il documento.

**Stato.** Nozioni AWS aggiornate a ottobre 2026. Prezzi e quote cambiano; nel colloquio contano i concetti e il criterio di scelta. Dove cito 8 vCPU, 40 GB o 15 minuti, è un dato del progetto o un limite noto, non una stima.

## Indice

1. ECS, EC2, Fargate, task, service e termini collegati

---

## 1. ECS, EC2, Fargate, task, service e termini collegati

### 1.1 Il problema da cui partire

Hai un'applicazione Python. Per evitare il classico “sulla mia macchina funziona”, l'hai chiusa in un **container**. Prima di andare avanti, due parole che spesso vengono usate come sinonimi ma non lo sono:

- **Immagine**: il pacchetto. Contiene il tuo codice, le librerie, il sistema base e le istruzioni per partire. È un artefatto fermo, versionato, che puoi copiare in un **registry**, cioè un archivio di immagini. Docker è il formato più diffuso per costruirle.
- **Container**: il processo che parte da quell'immagine. È l'immagine in esecuzione, con il suo filesystem isolato e le sue variabili. Se il processo esce, il container esce.

Un container non è una macchina virtuale. Non ha un sistema operativo completo da avviare e condivide il **kernel** (il nucleo del sistema operativo) dell'host; per questo parte in meno tempo e consuma meno. Ma resta un processo: se va in crash, non si riavvia da solo; se la macchina si spegne, sparisce; se arriva più traffico, non si moltiplica da solo.

Ora la domanda vera: **chi lo avvia, chi lo tiene vivo, chi gli dà rete e permessi, chi lo aggiorna?** Se hai un solo container e una sola macchina, puoi farlo a mano con Docker. Quando i container diventano più di uno, quando vuoi riavvii automatici, più copie, deploy senza fermare tutto e integrazione con AWS, ti serve un **orchestratore**.

> **Orchestratore**: il software che decide dove e come far girare i container, ne segue lo stato e li sostituisce quando serve. Non esegue il tuo codice al posto loro: coordina chi lo esegue.

**ECS è l'orchestratore AWS.** Non è una macchina e non è il posto dove gira il container: è il livello che prende le decisioni. Il posto dove gira è il **compute**, e può essere EC2, Fargate o ECS Managed Instances. Questa distinzione è il primo passo per non confondere i termini.

### 1.2 Control plane e data plane: chi decide e chi esegue

Due parole che aiutano a capire ECS senza mistero:

- **Control plane**: la parte che prende decisioni. Riceve la richiesta, sceglie dove mettere il task, tiene lo stato, chiama le API (le interfacce di controllo). È ECS.
- **Data plane**: la parte che esegue davvero. È l'istanza EC2, l'unità di compute Fargate o l'istanza gestita da AWS. È lì che girano i tuoi container.

Se l'intervistatore chiede “dove gira ECS?”, la risposta precisa è: **il control plane è gestito da AWS; il tuo codice gira nel data plane**. Non è un server che affitti tu. Questa frase evita l'errore di dire “ECS è una macchina”.

### 1.3 Il viaggio di un job: i termini uno alla volta

Riprendi il worker Python. Il percorso è questo.

**Primo passo: costruisci l'immagine.** Metti il codice in un'immagine Docker e la carichi su **ECR**, il registry delle immagini di AWS. L'immagine ha un nome e un tag, per esempio `report-worker:1.0`.

**Secondo passo: scrivi la task definition.** La **task definition** è la ricetta che dice a ECS come eseguire l'immagine: quale immagine, quanta CPU e memoria, quali variabili d'ambiente, quali segreti, quali permessi AWS, dove scrivere i log, quale rete usare. Non è ancora un container in esecuzione: è il progetto.

**Terzo passo: avvii un task.** Con `run_task` dici a ECS: “esegui una copia di questa ricetta”. ECS sceglie il compute, avvia il container, gli dà un indirizzo di rete e lo fa partire. Quella copia in esecuzione è il **task**. Il task legge il job da DynamoDB, scrive il Parquet su S3 e termina. Se muore, nessuno lo ricrea: era un'operazione one-off.

**Quarto passo: se il worker deve restare vivo, usi un service.** Un'**service** dice a ECS: “voglio sempre N task di questa ricetta”. Se un task muore, il service ne avvia un altro. Se vuoi esporlo in HTTP, lo metti dietro un **load balancer** (il componente che riceve il traffico e lo distribuisce). Se vuoi scalare, aumenti il desired count. Il service è il supervisore, non il processo.

**Quinto passo: tutto vive in un cluster.** Il **cluster** è il raggruppamento logico dentro cui registri task e service. Non è una macchina: è una cartella di orchestrazione. Con Fargate non ha istanze; con EC2 ha le istanze registrate.

Schema mentale:

```
immagine  ->  container
                  ^
task definition --|--> task --> service
                              |
                           cluster
                              |
                  compute: EC2 / Fargate / Managed Instances
```

La task definition è la ricetta; il task è un'esecuzione della ricetta; il service è chi controlla che ci siano N esecuzioni; il cluster è il contesto in cui tutto questo è registrato. Il compute è dove il container gira davvero.

### 1.4 Task definition: la ricetta, non l'esecuzione

La task definition è un documento regionale e versionato. La puoi vedere come la **classe** di un oggetto: descrive la forma, ma non è l'oggetto. Il task è l'istanza.

Contiene, tra le altre cose:

- **family** e **revision**: la famiglia è il nome logico, per esempio `report-worker`; la revision è il numero che aumenta a ogni registrazione, per esempio `report-worker:7`;
- **containerDefinitions**: uno o più container. Un task può avere il container principale e dei **sidecar** (container di supporto), per esempio un proxy o un log router;
- **immagine**: per esempio l'immagine ECR `report-worker:1.0`;
- **CPU e memoria**: su Fargate si dichiarano a livello di task; su EC2 possono essere a livello di task o di container;
- **variabili d'ambiente** e **segreti**: le prime sono configurazione non sensibile, i secondi arrivano da Secrets Manager o SSM, i servizi AWS per conservare valori sensibili e parametri;
- **ruoli IAM**: execution role e task role, due identità con permessi diversi, che vediamo dopo;
- **log**: per esempio il driver `awslogs` verso CloudWatch, il servizio AWS di log e metriche;
- **rete**: per esempio `awsvpc`, che dà al task una scheda di rete e un IP suoi;
- **volumi**: per esempio **EFS**, il filesystem condiviso di AWS, per dati persistenti;
- **health check**, `dependsOn`, `essential`, timeout di avvio e arresto;
- **requiresCompatibilities**: `FARGATE`, `EC2`, `MANAGED_INSTANCES` o combinazioni;
- **runtimePlatform**: sistema operativo e architettura.

La task definition è **immutabile**: se cambi l'immagine o una variabile, registri una nuova revision. Il service punta a una revision; aggiornare la task definition non aggiorna automaticamente il service. È un passaggio separato, e questo è il motivo per cui “ho aggiornato la task definition” non significa “la nuova versione è online”.

**Task definition non è un container.** Un task può avere più container. Se un container marcato `essential` esce, il task intero si ferma.

### 1.5 Task: una esecuzione concreta

Il task è l'unità di scheduling. Nasce da una revision di task definition, riceve un **ARN** — l'identificativo univoco che AWS assegna a ogni risorsa — viene piazzato in una **subnet** (un segmento della rete con un intervallo di IP), ottiene una scheda di rete se usa `awsvpc`, e passa attraverso stati come `PROVISIONING`, `PENDING`, `RUNNING`, `STOPPING`, `STOPPED`.

Un task può essere:

- **standalone**: lo lanci con `run_task` o `start_task` e nessuno lo ricrea se muore;
- **di un service**: il service lo mantiene e lo sostituisce se fallisce;
- **schedulato**: **EventBridge Scheduler** (lo scheduler gestito di AWS) o un evento chiama `run_task` a orario;
- **parte di un workflow**: **Step Functions** (il servizio AWS per orchestrare più passi) può avviarlo e attendere la fine.

Le API che sentirai nominare:

- `run_task`: ECS sceglie la capacità e avvia uno o più task. La risposta contiene `tasks` e `failures`; un task può non partire per capacità, subnet o ENI senza che la chiamata sollevi un'eccezione. La risposta non significa “il job è finito”: il runtime è asincrono.
- `start_task`: avvia il task su un container instance specifico; ha senso nel mondo EC2.
- `stop_task`: chiede lo stop; il task passa a `STOPPED`.
- `describe_tasks`: legge stato, stop reason, exit code dei container e altre informazioni.

Il task ha un `desiredStatus` e un `lastStatus`: il primo è ciò che vuoi, il secondo è ciò che ECS osserva. Un task `RUNNING` non garantisce che l'applicazione sia sana: i container girano, ma l'app può rispondere male o non rispondere. Per quello servono health check e metriche applicative.

### 1.6 Service: il supervisore del desired state

Il service è la risorsa che dice: “voglio N task di questa task definition, possibilmente dietro questo **load balancer**” — il componente che riceve il traffico e lo distribuisce tra i task. È un concetto semplice se lo pensi come un termostato: tu imposti la temperatura, il sistema accende e spegne per mantenerla. Tu imposti il **desired count**, il service avvia e ferma task per mantenerlo.

Cosa fa:

- mantiene il **desired count**;
- sostituisce i task falliti o fermati;
- integra il service con un **target group** (il gruppo di destinazioni che il load balancer controlla con un health check) di un ALB o NLB;
- applica la deployment configuration: `minimumHealthyPercent`, `maximumPercent`, circuit breaker;
- può usare Service Auto Scaling per cambiare il desired count;
- può registrare i task in Cloud Map o usare Service Connect per la comunicazione tra servizi;
- può avere `desiredCount=0`, utile per spegnere un ambiente senza cancellarlo.

Cosa non fa:

- non è un processo: è una risorsa del control plane;
- non è un task: il task è l'istanza, il service è la regola che la mantiene;
- non è obbligatorio per un carico one-off: per un job che deve girare una volta, `run_task` è più adatto;
- non rende automaticamente idempotente il lavoro: se un task viene sostituito, il tuo codice deve gestire il retry.

**Desired count e running count non sono la stessa cosa.** Il primo è quello che dichiari; il secondo è quello che ECS osserva. Se non c'è capacità, i task restano `PENDING` e il service non è “a posto” solo perché il numero desiderato è corretto.

**REPLICA e DAEMON.** Un service `REPLICA` mantiene N copie. Un service `DAEMON` tiene una copia per container instance: ha senso su EC2, per un agente o un collector. Fargate non lo supporta, perché non ci sono container instance da popolare.

**Service e deployment.** Il service è la risorsa persistente; il deployment è il processo con cui una nuova revision prende il posto della vecchia. Puoi avere un service senza deployment in corso, e un deployment che fallisce senza distruggere il service.

### 1.7 Cluster: il contesto, non la macchina

Il cluster è un raggruppamento logico di task e service. È come una cartella o un namespace: serve a separare ambienti o applicazioni, non a fornire CPU.

- con **Fargate** il cluster non ha container instance: è solo un contesto di scheduling e configurazione;
- con **EC2** ha le istanze registrate;
- con **Managed Instances** ha la capacity provider gestita da AWS;
- può contenere una miscela di task Fargate, EC2, Managed Instances ed External;
- è regionale.

Non è una VPC e non è una macchina. La rete vive nelle subnet e nei security group; il cluster è il livello di orchestrazione.

### 1.8 EC2: la macchina che gestisci tu

**EC2** è una macchina virtuale che affitti. Hai accesso al sistema operativo, scegli l'**AMI** (Amazon Machine Image, il modello di sistema operativo e software da cui parte l'istanza), installi ciò che ti serve, applichi le patch, decidi quando accenderla e quando spegnerla.

Con ECS in modalità **EC2 launch type**:

1. accendi una o più istanze EC2;
2. installi l'**agente ECS** (il processo che registra l'istanza nel cluster e riceve i comandi di ECS) e registri l'istanza; l'istanza diventa un **container instance**;
3. ECS piazza i task sulle istanze con capacità libera;
4. se le istanze sono piene, i task restano `PENDING` o falliscono il piazzamento.

Cosa gestisci tu:

- AMI, sistema operativo, patching e hardening;
- dimensione e numero delle istanze, o l'**Auto Scaling Group** (ASG: il gruppo che aggiunge e rimuove istanze in base a una regola);
- agente ECS e ruolo IAM dell'istanza;
- capacità libera e costo delle istanze anche quando non hanno task.

Cosa guadagni:

- accesso a tipi di istanza specifici, GPU, acceleratori, custom AMI e volumi;
- **placement constraint** (regole su dove mettere i task) e strategie come `spread` (distribuisci) o `binpack` (riempi un host prima di passare al successivo);
- reti `bridge`, `host` o `awsvpc`;
- costi più prevedibili ad alta utilizzazione, con Reserved Instances, Savings Plans o Spot;
- controllo su **daemon** (un processo che gira su ogni host, per esempio un agente di monitoraggio) e software di host.

Un container instance può essere messo in stato `DRAINING`: non accetta nuovi task e il service sposta quelli che può. È il modo corretto di togliere un host dal cluster senza uccidere il servizio.

La frase da non dire: “ECS su EC2 è la stessa cosa di Fargate con più lavoro”. Non è solo più lavoro: è un modello diverso, in cui la capacità è un problema tuo e in cambio hai accesso a tutta la superficie di EC2.

### 1.9 Fargate: la scatola di compute che non gestisci

**Fargate** è un motore di compute per container. Tu dichiari CPU e memoria a livello di task; AWS crea la capacità, isola il task e lo esegue. Non vedi l'istanza, non fai SSH, non gestisci l'agente ECS e non paghi capacità inattiva: paghi le risorse del task mentre gira.

Cosa devi sapere:

- **CPU e memoria a livello di task**: le combinazioni sono validate da AWS. Per esempio 8 vCPU richiede Linux platform `1.4.0` o successiva — la platform version è la versione dell'ambiente Fargate — e la memoria va scelta in incrementi validi. Il task del cash flow da 8 vCPU e 40 GB rientra nelle combinazioni Fargate. 8 e 16 vCPU sono opzioni Linux; Fargate supporta anche Windows containers, ma con combinazioni e limiti diversi.
- **awsvpc obbligatorio**: ogni task ha una scheda di rete (ENI), un IP nella subnet e i suoi **security group** (i firewall di rete). Non c'è `host` network, non c'è `bridge`, non c'è port mapping dinamico sull'host.
- **Nessun accesso all'host**: non fai SSH, non monti il filesystem dell'host, non usi container privileged. Per debuggare usi log, metriche ed ECS Exec (il comando che apre una shell nel container).
- **Storage**: disco effimero locale, cioè temporaneo, con dimensione configurabile entro i limiti Fargate; per dati persistenti usi **EFS** (il filesystem condiviso di AWS) o un servizio esterno. Non usare il disco locale come stato permanente.
- **GPU**: Fargate non espone GPU. Per GPU e acceleratori guardi EC2 o ECS Managed Instances.
- **Architettura**: Linux x86_64 e ARM64 (Graviton); ECS Fargate supporta anche Windows containers, ma con limiti e senza GPU.
- **Fargate Spot**: capacità Spot, cioè capacità in eccesso a prezzo ridotto che può essere interrotta, con preavviso di circa due minuti; va usata per workload tolleranti, non per un servizio che deve restare su.
- **Billing**: a consumo, con minimo di un minuto e granularità al secondo; non paghi l'host idle.

Fargate è la scelta giusta quando il carico è variabile, il team è piccolo e il valore sta nell'applicazione, non nell'infrastruttura. Non è la scelta giusta quando ti serve una GPU, un custom AMI, un daemon sull'host, un tipo di istanza particolare o un'utilizzazione molto alta e stabile: lì EC2 o Managed Instances possono costare meno e dare più controllo.

### 1.10 ECS Managed Instances: il compromesso nuovo

Da fine 2025 AWS offre **Amazon ECS Managed Instances**: una **capacity provider** — cioè un meccanismo che fornisce capacità al cluster — in cui AWS gestisce le istanze EC2: provisioning, patching, scaling e lifecycle. Tu mantieni la possibilità di scegliere tipi di istanza, GPU, architetture e caratteristiche EC2. La task definition dichiara `MANAGED_INSTANCES` tra le compatibilità.

Non è Fargate: le istanze sono EC2 e il costo è quello EC2 più un costo di gestione. La capacity provider può usare On-Demand (prezzo pieno, capacità garantita), Spot (capacità in eccesso, interrompibile) o Capacity Reservations (capacità prenotata in una zona). AWS la indica come opzione consigliata per molti workload nuovi, soprattutto quando vuoi la semplicità operativa senza rinunciare a GPU o a tipi di istanza specifici.

Perché la cito: un intervistatore aggiornato può chiedere “Fargate o EC2?” e la risposta “dipende” oggi ha una terza opzione. La frase onesta è:

> “Nei progetti ho usato Fargate e, per il frontend del repricing, EC2 con autoscaling. Oggi AWS ha anche ECS Managed
> Instances, che sposta la gestione delle EC2 su AWS mantenendo accesso a GPU e tipi di istanza. Non l'ho usata, ma so
> collocarla: è il compromesso tra la semplicità di Fargate e la flessibilità di EC2.”

### 1.11 Launch type e capacity provider: due parole che sembrano uguali

Qui molti si perdono. Parti da una domanda: **quando ECS deve avviare un task, come sa dove metterlo?**

- Il **launch type** dichiara il tipo di compute con cui il task è compatibile: EC2, Fargate, External o Managed Instances. Serve anche nella task definition per dire “questa ricetta può girare su Fargate?”.
- Il **capacity provider** è il meccanismo con cui il cluster fornisce capacità: Fargate, Fargate Spot, un **ASG** (Auto Scaling Group, cioè un gruppo di istanze EC2 che scala) o Managed Instances.
- La **capacity provider strategy** è la regola con cui distribuisci i task tra capacity provider, usando `base` e `weight`: `base` è il numero minimo di task da mettere su un provider, `weight` è il peso relativo per il resto. Per esempio una base su Fargate e un peso su Fargate Spot per il resto.

Regole pratiche:

- launch type e capacity provider strategy non si specificano insieme sullo stesso task o service;
- se non specifichi né l'uno né l'altra, vale la strategia di default del cluster;
- una capacity provider strategy non può mescolare tipi diversi: Fargate, ASG e Managed Instances non stanno nella stessa strategia;
- con EC2 launch type ECS non scala l'ASG per te; con una capacity provider ASG e managed scaling, ECS può farlo;
- i capacity provider Fargate e Fargate Spot sono creati automaticamente su ogni cluster;
- la capacity provider strategy è la strada consigliata da AWS per i nuovi workload, mentre il launch type resta utile per dichiarare la compatibilità.

In pratica: nella task definition dichiari `requiresCompatibilities`; quando crei il service scegli la capacity provider strategy. Se ti chiedono “dove gira?”, la risposta completa è “su Fargate, tramite la capacity provider FARGATE”; non è una contraddizione, sono due livelli.

**External, ECS Anywhere, Outposts.** Il launch type `EXTERNAL` serve per registrare istanze fuori da AWS, per esempio server on-premise o edge: si chiama **ECS Anywhere** e richiede l'agente SSM e l'agente ECS sull'istanza. Non è Fargate e non è pensato per il cloud AWS. **ECS on Outposts** è invece ECS su hardware AWS installato on-premise, con EC2 come compute. Sono opzioni di nicchia: le cito per completezza, non le ho usate nei progetti.

### 1.12 Rete: come parla un task

Per capire `awsvpc` servono quattro parole:

- **VPC**: la rete privata virtuale in cui vivono le risorse AWS.
- **Subnet**: un segmento della VPC in una **availability zone** (una zona fisica separata dentro una **region**, cioè un'area geografica); è lì che il task riceve l'IP.
- **ENI**: una scheda di rete virtuale; è ciò che dà a una risorsa un indirizzo IP e dei security group.
- **Security group**: un firewall che decide chi può entrare e uscire.

**awsvpc**:

- ogni task ha una ENI, un IP privato nella subnet e i suoi security group;
- è obbligatorio su Fargate;
- il target group del load balancer usa target type `ip`;
- non c'è port mapping dinamico: il container ascolta sulla porta che dichiari, e il traffico arriva all'IP del task;
- è il modello più pulito per isolamento e sicurezza, ma consuma indirizzi IP e ENI.

**bridge**:

- è il bridge Docker sull'istanza EC2;
- il container espone una porta, l'host la mappa su una porta dinamica;
- il target group usa target type `instance`;
- è comodo su EC2, ma non su Fargate.

**host**:

- il container usa la rete dell'host, con la porta direttamente sull'istanza;
- nessun isolamento di rete per task;
- può avere conflitti di porta;
- solo EC2.

**none**: nessuna rete esterna, casi molto specifici.

Su Fargate, se il task deve uscire verso internet, la subnet deve essere pubblica con IP pubblico assegnato oppure privata con un **NAT Gateway** (il componente che dà uscita internet a una subnet privata). I security group del task devono permettere il traffico in ingresso dal load balancer sulla porta del container e in uscita verso le dipendenze.

Sul load balancer: **ALB** per HTTP/HTTPS, routing per host o path, target type `ip` con `awsvpc`; **NLB** per TCP/UDP, latenza bassa e IP statici. Il **target group** è il gruppo di destinazioni verso cui il load balancer manda il traffico; il **health check** è la prova periodica che una destinazione risponde. In entrambi i casi target group e health check sono parte del contratto tra service e load balancer. Un worker su coda non ha bisogno di un load balancer.

### 1.13 IAM: chi può fare cosa

Un **ruolo IAM** è un'identità con un insieme di permessi. Le risorse AWS lo assumono temporaneamente; non è una password e non è un utente.

Qui c'è un errore che fanno quasi tutti i candidati: usare un solo ruolo per tutto.

- **Execution role**: lo usa l'agente ECS. Serve per pull dell'immagine da ECR, scrittura dei log su CloudWatch, lettura di secret e parametri SSM e decriptazione con KMS (il servizio AWS per le chiavi di cifratura). Non è il ruolo che il codice applicativo usa per chiamare DynamoDB o S3, anche se in un setup pigro capita che i permessi siano lì.
- **Task role**: lo assume il codice dentro il container. È il ruolo per DynamoDB, S3, SQS, Athena, Bedrock o qualunque servizio l'applicazione chiami. Va tenuto al minimo privilegio.
- **EC2 instance role / container instance role**: lo usa l'agente ECS sull'istanza EC2 per registrarsi al cluster e parlare con il control plane. Non è un task role.
- **Service-linked role**: lo crea ECS per gestire risorse collegate, per esempio il load balancer di un service.

La regola: se il container deve leggere da S3, il permesso sta nel task role; se ECS deve scaricare l'immagine, il permesso sta nell'execution role. Se metti tutto nell'execution role, funziona ma hai allargato la superficie e reso il codice dipendente da un ruolo infrastrutturale.

Per i segreti: usi **Secrets Manager** o **SSM Parameter Store**, due servizi AWS per conservare valori sensibili o parametri di configurazione. Il valore può essere iniettato nel container tramite l'execution role, oppure letto dall'applicazione con il task role. Non metti il segreto nell'immagine o in una variabile hardcoded.

### 1.14 Deploy e scaling: due piani diversi

**Deployment ECS rolling**:

- il service avvia task della nuova revision e ne ferma alcuni della vecchia;
- `minimumHealthyPercent` dice quanta capacità sana deve restare durante il deployment;
- `maximumPercent` dice quanta capacità extra può essere creata;
- il circuit breaker può fermare un deployment che non diventa sano e fare rollback;
- `healthCheckGracePeriod` dà tempo al task per diventare sano prima che il load balancer lo consideri fallito;
- `forceNewDeployment` riavvia i task anche senza cambiare revision, utile dopo un cambio di configurazione esterno;
- il controller può essere `ECS`, `CODE_DEPLOY` o `EXTERNAL`, a seconda di chi gestisce il rollout.

**Blue/green**:

- il controller è **CodeDeploy**, il servizio AWS che gestisce i deploy;
- ci sono due target group e due versioni: il traffico viene spostato gradualmente dalla vecchia alla nuova;
- puoi fare **canary** (una piccola percentuale di traffico per un periodo) o **linear** (incrementi regolari), con rollback;
- è più costoso e più complesso del rolling, ma riduce il rischio e permette di testare la nuova versione prima di spostare tutto il traffico.

**Service Auto Scaling**:

- usa **Application Auto Scaling** per cambiare il desired count del service;
- target tracking su CPU, memoria, richieste ALB per target o metriche custom;
- step scaling e scheduled scaling;
- è diverso dall'ASG delle istanze EC2: quello scala la capacità del cluster, questo scala i task.

**EC2 launch type**:

- l'ASG scala le istanze; se non c'è capacità libera, i task restano in `PENDING`;
- con capacity provider ASG e managed scaling ECS può gestire il capacity planning;
- con EC2 launch type “puro” la capacità è un problema tuo.

**Fargate**:

- non c'è un ASG di host da scalare;
- il service scala il numero di task;
- la dimensione del singolo task è decisa nella task definition e non cambia a runtime.

La frase da ricordare: **Service Auto Scaling cambia il numero di task; l'ASG cambia il numero di macchine.** Sono due manopole diverse.

### 1.15 Osservabilità e debug

Cosa guardare quando un task non parte o muore:

- **stato e stop reason** del task: `describe_tasks` mostra `lastStatus`, `desiredStatus`, `stoppedReason`, `stopCode`, exit code e motivo dei container;
- **log** su CloudWatch, se configurato `awslogs` o FireLens;
- **eventi ECS** su EventBridge, per transizioni di stato e fallimenti;
- **metriche** CloudWatch e **Container Insights** (le metriche ECS aggregate su CloudWatch) per CPU, memoria, rete;
- **load balancer** health check e target group per capire se il task è sano ma non riceve traffico;
- **ECS Exec** per aprire una shell nel container, se abilitato; su Fargate non c'è SSH sull'host;
- **task metadata** (l'endpoint locale che espone informazioni del task) per capire rete, ENI e credenziali del task role.

Un task `RUNNING` può essere malato: lo stato ECS dice che i container girano, non che l'applicazione risponda. Per questo il health check e le metriche applicative sono parte del contratto, non un extra.

### 1.16 Costi: il criterio, non la cifra

- **Fargate**: paghi vCPU e memoria del task al secondo, con minimo di un minuto; non paghi capacità inattiva. Per workload variabili o piccoli è spesso la scelta più semplice.
- **EC2**: paghi l'istanza anche quando è idle; ad alta utilizzazione stabile può costare meno di Fargate. Puoi usare Spot (capacità in eccesso, interrompibile), Reserved Instances (impegno di un anno o più) e Savings Plans (impegno di spesa).
- **Fargate Spot**: sconto in cambio di interruzione; bene per batch e job ritentabili, male per un servizio critico.
- **Managed Instances**: costo EC2 più un costo di gestione AWS; è il compromesso quando vuoi la flessibilità EC2 senza gestire le istanze.
- **Costi nascosti**: **NAT Gateway** (il componente che dà uscita internet a una subnet privata), data transfer, load balancer, CloudWatch Logs, ECR storage, tempo di startup e capacità EC2 inattiva. In un conto **TCO** (costo totale di possesso) entrano tutti.

La domanda da fare non è “Fargate o EC2?”, è “qual è il profilo di utilizzazione e quanto vale il tempo del team?”. Un carico che satura le istanze 24/7 può giustificare EC2; un carico a ondate o un team piccolo spesso no.

### 1.17 Quando uso cosa

| Esigenza | Scelta tipica | Perché |
|---|---|---|
| API web long-running | ECS service + ALB, Fargate o EC2 | Il service mantiene i task e il load balancer distribuisce. |
| Job one-off | `run_task` su Fargate | Non serve un desired count; il task gira una volta. |
| Cron | EventBridge Scheduler + `run_task` | Il trigger esterno avvia il task; non serve un service sempre acceso. |
| Worker su coda | ECS service + SQS + autoscaling | Il service scala sul backlog e sostituisce i task. |
| GPU o ML | EC2 o Managed Instances | Fargate non espone GPU. |
| Custom AMI, privileged, host network | EC2 | Fargate non li supporta. |
| Carico variabile, poca ops | Fargate | Nessun host da gestire, paghi a consumo. |
| Utilizzazione alta e stabile | EC2 o Managed Instances | Può essere più conveniente e dà più controllo. |
| Kubernetes | EKS | ECS non è Kubernetes. |
| Funzione breve event-driven | Lambda | Diverso modello, limite di 15 minuti. |
| Tanti job batch | AWS Batch | Aggiunge code e compute environment sopra ECS. |

### 1.18 ECS, EKS, Lambda, Batch, App Runner: differenze in breve

- **ECS**: orchestratore AWS-native per container; più semplice di Kubernetes, meno estensibile.
- **EKS**: Kubernetes gestito. Kubernetes è un orchestratore di container open source; EKS te lo gestisce, ma ti lascia l'ecosistema, le CRD, gli operatori e la portabilità. Più potente e più costoso in competenze.
- **Lambda**: funzioni event-driven, scala automaticamente, limite di 15 minuti, non è pensata per un servizio container long-running.
- **Batch**: servizio AWS per code di job e compute environment; usa ECS sotto il cofano. È la scelta quando hai molti job batch e vuoi scheduling, retry e priorità senza costruirli.
- **App Runner**: servizio AWS che esegue un servizio web containerizzato con meno configurazione di ECS; meno controllo, più semplice.

### 1.19 Il cash flow: un task, non un service

Questo è il punto in cui il vocabolario diventa concreto.

Nel cash flow la Lambda fa da dispatcher: costruisce il payload, lo salva su DynamoDB e chiama `ecs.run_task` su una task definition Fargate. Il task ha 8 vCPU e 40 GB, legge il record del job e esegue il calcolo. Non c'è un service, non c'è un desired count, il task non è dietro un load balancer: il job deve girare una volta.

Perché è un task e non un service:

- un service manterrebbe un numero desiderato di task e li riavvierebbe; per un job one-off sarebbe sbagliato;
- `run_task` avvia un task e restituisce la risposta, ma il runtime continua in modo asincrono;
- se il task muore, nessuno lo ricrea: lo stato applicativo deve essere aggiornato dal codice o da un watcher;
- il task non è dietro un ALB: non serve esporlo, deve solo leggere e scrivere dati.

I failure mode da citare:

- `run_task` può restituire `failures` senza sollevare un'eccezione: capacità, subnet, ENI, quota. La Lambda deve controllare la risposta, non solo stamparla.
- un task può essere ucciso per memoria: il container esce, il task ECS passa a `STOPPED` con una stop reason, ma il record applicativo su DynamoDB può restare `RUNNING` se il codice non intercetta l'evento.
- un task può fallire a metà scrittura: l'output su S3 va scritto per run e con un marker finale, non sovrascritto alla cieca.
- un retry può duplicare il lavoro: serve un identificativo stabile e idempotenza.

Se lo stesso carico diventasse un'API, userei un service con ALB e health check. Se diventasse un cron, userei EventBridge Scheduler + `run_task`. Se diventasse una coda di cento job, userei **SQS** (la coda gestita di AWS) + service worker o AWS Batch, non cento `run_task` incontrollati.

La frase da usare:

> “Nel cash flow il task è l'unità di lavoro, non il servizio. La Lambda non esegue il calcolo: fa dispatch. Il task
> Fargate è one-off, quindi non ha senso un service; il service servirebbe se volessi un numero desiderato di worker
> sempre vivi.”

### 1.20 Frasi da non dire

- “ECS è serverless”: ECS è l'orchestratore; solo Fargate e Managed Instances sono compute gestito.
- “Fargate è un cluster”: Fargate è un motore di compute; il cluster è un'altra cosa.
- “Task definition e task sono la stessa cosa”: la prima è il template, il secondo è l'istanza.
- “Task e container sono sinonimi”: un task può contenere più container, per esempio un main e un sidecar.
- “Service e task sono la stessa cosa”: il service mantiene il desired state; il task è l'unità.
- “Task role ed execution role sono uguali”: uno serve all'app, l'altro all'agente ECS.
- “`run_task` è sincrono”: la chiamata ritorna, il task continua; devi guardare stato e failure.
- “Per un job one-off serve un service”: no, serve un task; il service serve per un desired state.
- “Fargate supporta GPU”: no; per GPU guardi EC2 o Managed Instances.
- “ECS Managed Instances è Fargate”: no; è una capacity provider EC2 gestita da AWS.
- “Fargate supporta host network, privileged o DAEMON”: no.
- “Il cluster è una macchina”: è un confine logico.
- “Un service deve avere un load balancer”: no, il load balancer è opzionale.
- “ECS scala l'ASG con EC2 launch type”: con il launch type puro la capacità è tua; con capacity provider ASG e managed scaling è diverso.
- “ECS è Kubernetes”: i concetti sono simili, ma non è lo stesso modello.

### 1.21 Se te lo chiedono in 60 secondi

> “ECS è l'orchestratore, non il compute. EC2, Fargate e Managed Instances sono i modi di fornire capacità: con EC2
> gestisci le istanze, con Fargate no, con Managed Instances AWS gestisce le EC2 ma mantieni accesso a GPU e tipi di
> istanza. La task definition è il template; il task è l'istanza; il service mantiene il desired count. Il cluster è
> il confine logico. Nel cash flow uso un task Fargate one-off via `run_task`, non un service, perché il job deve
> girare una volta; la Lambda fa solo dispatch. Se fosse un'API userei un service dietro ALB; se fosse un cron,
> EventBridge Scheduler più `run_task`; se fossero cento job, SQS più worker o Batch.”

### 1.22 Domande rapide con risposta

**D. ECS e EC2 sono alternativi?** No. EC2 è compute, ECS è orchestratore. Puoi usare ECS con EC2, con Fargate o con Managed Instances. EC2 senza ECS è un'altra scelta, non un'alternativa allo stesso livello.

**D. ECS e Fargate sono alternativi?** No. Fargate è un launch type/capacity provider di ECS. ECS è l'orchestratore, Fargate è dove girano i task.

**D. Task definition, task e container: che differenza c'è?** La task definition è il modello; il task è l'istanza del modello; il container è uno dei processi dentro il task. Un task può avere più container.

**D. Task e service: che differenza c'è?** Il task è l'unità di esecuzione; il service è la regola che mantiene N task vivi e li sostituisce. Un task standalone non viene ricreato; un task di un service sì.

**D. Cos'è un cluster?** Un raggruppamento logico di task e service, con eventuale capacità EC2 o capacity provider. Non è una macchina e non è una VPC.

**D. Launch type e capacity provider sono la stessa cosa?** No. Il launch type dichiara il tipo di compute e la compatibilità; il capacity provider gestisce la capacità e la strategia. AWS consiglia i capacity provider per i nuovi workload.

**D. Task role ed execution role?** Il task role è per il codice applicativo; l'execution role è per l'agente ECS, che deve pullare l'immagine, scrivere log e leggere secret. Non vanno confusi.

**D. Perché Fargate richiede awsvpc?** Perché ogni task ha una ENI e un IP propri, con security group dedicati; non c'è un host condiviso su cui fare port mapping. È il modello che dà isolamento di rete al task.

**D. Quando uso `run_task` e quando un service?** `run_task` per un job one-off o un cron; service per un carico che deve restare vivo con un desired count, un load balancer o un worker su coda.

**D. Cosa succede se un task di un service muore?** Il service scheduler lo rileva e ne avvia un altro secondo il desired count e la deployment configuration. Se il task è standalone, non viene ricreato.

**D. Come fai il deploy di una nuova versione?** Registri una nuova revision della task definition, aggiorni il service e fai un rolling update o un blue/green con CodeDeploy. La task definition da sola non aggiorna il service.

**D. Come fai debug su Fargate?** Log CloudWatch, metriche, eventi ECS, `describe_tasks` con stop reason ed exit code, ECS Exec se abilitato. Non c'è SSH sull'host.

**D. Fargate o Lambda?** Fargate per container long-running, processi con dipendenze di sistema, job oltre i 15 minuti e servizi con più container; Lambda per funzioni brevi ed event-driven. Nel cash flow la Lambda è il dispatcher, Fargate è il worker.

**D. Fargate o EKS?** Fargate è un compute di ECS/EKS; EKS è Kubernetes gestito. Se ti serve l'ecosistema Kubernetes, EKS; se vuoi un'orchestrazione AWS-native più semplice, ECS. Fargate può essere usato con entrambi, con limiti diversi.

**D. Fargate supporta GPU?** No. Per GPU usi EC2 o ECS Managed Instances, che espone tipi di istanza con acceleratori.

**D. ECS Managed Instances è Fargate?** No. È una capacity provider che usa istanze EC2 gestite da AWS; mantiene accesso a GPU, tipi di istanza e caratteristiche EC2. È il compromesso tra semplicità operativa e flessibilità.

**D. Come passi un segreto a un task?** Con Secrets Manager o SSM Parameter Store, referenziato nella task definition e letto dall'execution role, oppure letto dall'applicazione con il task role. Non metti il segreto nell'immagine o in una variabile hardcoded.

**D. Come esponi un service?** Con un ALB o NLB e un target group. Con `awsvpc` il target type è `ip`; con bridge/host può essere `instance`. Il load balancer non è obbligatorio: un worker su coda non ne ha bisogno.

**D. Come scali?** Con Service Auto Scaling cambi il desired count del service; con EC2 o capacity provider ASG cambi la capacità del cluster. Sono due piani diversi: task e host.

### 1.23 Glossario di ripasso

Usa questa tabella solo dopo aver letto le sezioni precedenti. Se un termine non ti torna, torna alla sezione corrispondente invece di memorizzare la riga.

| Termine | Significato in una frase |
|---|---|
| **Immagine** | Il pacchetto fermo dell'applicazione, con codice e dipendenze. |
| **Container** | Un processo in esecuzione creato da un'immagine. |
| **ECS** | L'orchestratore AWS che decide dove e come far girare i container. |
| **EC2** | Macchina virtuale che gestisci tu. |
| **Fargate** | Compute gestito da AWS: dichiari CPU/memoria del task, non gestisci l'host. |
| **Managed Instances** | Istanze EC2 gestite da AWS, con accesso a tipi di istanza e GPU. |
| **Task definition** | La ricetta: immagine, CPU/memoria, ruoli, rete, log. |
| **Task** | Una esecuzione concreta della task definition. |
| **Service** | Il supervisore che mantiene N task vivi e li sostituisce. |
| **Cluster** | Il namespace logico di task e service. |
| **Container instance** | Una istanza EC2 registrata in un cluster ECS. |
| **Capacity provider** | Il pool di capacità: Fargate, Fargate Spot, ASG EC2 o Managed Instances. |
| **Launch type** | Il tipo di compute/compatibilità: EC2, Fargate, External, Managed Instances. |
| **Task role** | Il ruolo IAM dell'applicazione dentro il container. |
| **Execution role** | Il ruolo IAM dell'agente ECS per immagine, log e secret. |
| **ALB/NLB** | Load balancer per esporre o distribuire un service. |
| **awsvpc** | Modalità di rete con una ENI e un IP per task. |

---

*Prossimi argomenti: da aggiungere in coda, mantenendo questa struttura.*
