---
modified: 2026-01-18T21:27:14+01:00
---
ESPERIENZE LAVORATIVE

Software Engineer | [Azienda] | [Città] | [Date] - Presente

Re-engineering e modernizzazione applicazioni finanziarie (Repricing + Cashflow)
- Refactoring completo di script Python legacy (1500+ righe monolitiche) in architettura modulare basata su principi SOLID: separazione tra logica di calcolo, persistenza dati e layer di presentazione, implementazione di [Strategy/Factory/Repository] pattern per [gestione simulazioni/creazione oggetti/accesso dati]
- Progettazione e sviluppo di [X] moduli Python con responsabilità ben definite: motore di  calcolo finanziario, gestione state delle simulazioni, integrazione AWS services (S3, DynamoDB, Athena) tramite boto3 SDK con error handling e retry logic
- Implementazione data pipeline applicativa: serializzazione risultati in formato Parquet con compressione Snappy, scrittura batch su S3, aggiornamento metadati su DynamoDB, query engine per analisi comparativa via Athena con [pandas/PyArrow]
- Sviluppo frontend web per configurazione simulazioni parametriche, monitoring stato esecuzione, visualizzazione risultati con tabelle/grafici interattivi e confronto 
  multi-simulazione (deployment su AWS ECS Fargate gestito dal cliente)
- Collaborazione in ambiente cloud-native: applicazione containerizzata con Docker, 
  deployment via CI/CD automatizzata (AWS CodePipeline/CodeBuild), infrastruttura IaC (AWS CDK) gestita dal team del cliente

Ambiente tecnico: Python 3.11, boto3 (AWS SDK), pandas, [pytest], Docker, AWS (ECS Fargate, 
S3, DynamoDB, Athena, Lambda), Parquet, Git



* Trasformazione da applicazione desktop (PyQt5, SQLite) con distribuzione manuale a sistema AWS cloud-native centralizzato. Refactoring integrale della codebase da struttura monolitica ad architettura modulare seguendo le best-practices di design del software.
* Integrazione AWS per orchestrazione simulazioni su ECS Tasks e storage risultati su S3 (Parquet). Sostituiti workflow basati su Excel con un sistema centralizzato di tracking su DynamoDB e query analytics su Athena per storicizzazione e analisi multi-utente
* Sviluppo in ambiente containerizzato Docker con integrazione in pipeline CI/CD (CodePipeline/CodeBuild). Collaborazione con team infrastrutturale su stack AWS gestito tramite CDK: adattamento codice applicativo per deployment su ECS Fargate, configurazione runtime environment e gestione permessi IAM.


* Trasformazione da applicazione desktop (PyQt5, SQLite) a sistema web-based centralizzato su AWS, eliminando la distribuzione manuale degli eseguibili. Refactoring integrale della codebase da struttura monolitica ad architettura modulare.
- Integrazione AWS (boto3) per orchestrazione simulazioni su ECS Tasks e storage risultati su S3 (Parquet). Sostituiti workflow basati su Excel con sistema centralizzato di tracking su DynamoDB e query analytics su Athena per storicizzazione e analisi multi-utente.
- Sviluppo in ambiente containerizzato Docker con integrazione in pipeline CI/CD (CodePipeline/CodeBuild). Collaborazione con team infrastrutturale su stack AWS gestito tramite CDK: adattamento codice applicativo per deployment su ECS Fargate, configurazione runtime environment e gestione permessi IAM.


