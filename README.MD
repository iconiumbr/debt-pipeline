# Debt Processing Pipeline System

## Language Selection
- [English](#english)
- [Português](#portugues)

## English Version {#english}

<section id="english">

### Overview
The Debt Processing Pipeline is a comprehensive system designed to process debt data submitted by clients through various methods. It handles data submission, storage, transformation, and final processing to ensure accurate recording and tracking of debt information in an asynchronous manner. This allows for scalable system operations based on variable demand.

### Reasoning
The company is a debt collection bureau that helps businesses collect overdue debts. Clients submit debt collection information, including debtor details, CPF (individual taxpayer registry number), amount owed, and due date. Once the debt is registered, further actions such as protests, notifications, discounts, or installment plans may follow.

### Scope
This system focuses on receiving, processing, and storing debt data submitted by clients. It does not cover the debt collection phase. The complexity arises from handling multiple data and file formats, ensuring smooth ingestion for both small and large clients with precision and efficiency.

### System Components

#### Batch Service
Tracks and maintains the status of debt data requests submitted by clients.

#### Input Services
Border services exposed to clients to collect debt data passively or actively, storing raw data for intermediate processing.

#### File Picker Process
- Detects file submissions via FTP using S3 notifications to SQS.
- Validates file details and client identity.
- Splits large files into multiple sub-batches or registers single batches for processing.

#### Intermediate Process
- Gathers raw data from various input sources.
- Performs ETL operations.
- Prepares and injects data into DynamoDB for registration.

#### Registering Process
- Takes data from DynamoDB.
- Validates and processes debt data.
- Inputs finalized data into the debt storage system.

#### Message Broker
- Uses Amazon SNS for inter-process messaging.
- Notifies batch service and intermediate systems of new files.

#### Monitoring
- All processes emit time-series metrics to AWS Timestream for analysis.

</section>

## Versão em Português {#portugues}

<section id="portugues">

### Visão Geral
O Pipeline de Processamento de Dívidas é um sistema abrangente projetado para processar dados de dívida enviados por clientes por vários métodos. Ele lida com a submissão, armazenamento, transformação e processamento final dos dados para garantir o registro e rastreamento precisos das informações de dívida de forma assíncrona. Isso permite operações escaláveis do sistema com base na demanda variável.

### Justificativa
A empresa é um bureau de cobrança que auxilia empresas na recuperação de dívidas vencidas. Os clientes enviam informações de cobrança, incluindo detalhes do devedor, CPF (Cadastro de Pessoa Física), valor devido e data de vencimento. Uma vez registrada a dívida, outras ações, como protestos, notificações, descontos ou planos de parcelamento, podem ser realizadas.

### Escopo
Este sistema se concentra no recebimento, processamento e armazenamento dos dados de dívida enviados pelos clientes. Ele não cobre a fase de cobrança da dívida. A complexidade surge da necessidade de lidar com vários formatos de dados e arquivos, garantindo uma ingestão suave e eficiente tanto para clientes pequenos quanto grandes.

### Componentes do Sistema

#### Serviço de Lotes
Rastreia e mantém o status das solicitações de dados de dívida enviadas pelos clientes.

#### Serviços de Entrada
Serviços de fronteira expostos aos clientes para coletar dados de dívida de forma passiva ou ativa, armazenando dados brutos para processamento intermediário.

#### Processo de Seleção de Arquivos
- Detecta submissões de arquivos via FTP usando notificações do S3 para o SQS.
- Valida detalhes do arquivo e identidade do cliente.
- Divide arquivos grandes em vários sub-lotes ou registra lotes únicos para processamento.

#### Processo Intermediário
- Coleta dados brutos de várias fontes de entrada.
- Realiza operações de ETL.
- Prepara e injeta os dados no DynamoDB para registro.

#### Processo de Registro
- Obtém dados do DynamoDB.
- Valida e processa os dados da dívida.
- Insere os dados finalizados no sistema de armazenamento de dívidas.

#### Corretor de Mensagens
- Usa Amazon SNS para mensagens interprocessuais.
- Notifica o serviço de lotes e sistemas intermediários sobre novos arquivos.

#### Monitoramento
- Todos os processos emitem métricas de séries temporais para o AWS Timestream para análise.

</section>
