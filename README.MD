# Sistema de Pipeline de Processamento de Dívidas  

O Sistema de Pipeline de Processamento de Dívidas é um sistema abrangente projetado para processar dados de dívida enviados por clientes por meio de vários métodos. O pipeline gerencia a submissão, armazenamento, transformação e processamento final dos dados para garantir que as informações da dívida sejam registradas e rastreadas com precisão do início ao fim, de forma assíncrona, permitindo que o operador do sistema dimensione os componentes do sistema de acordo com a demanda variável ao longo do dia.

## Justificativa  
Esta empresa é um bureau de cobrança de dívidas.  

Seus clientes são empresas que desejam cobrar dívidas vencidas de indivíduos que não efetuaram o pagamento voluntariamente. Os clientes enviam as informações de cobrança da dívida, que incluem detalhes básicos sobre o devedor do qual desejam cobrar, como nome, CPF (Cadastro de Pessoas Físicas), valor devido e data do vencimento não pago. Assim que a dívida é registrada no sistema do bureau de cobrança, as dívidas serão protestadas, o devedor será notificado e um desconto ou plano de parcelamento poderá ser negociado.

## Escopo  
Esta descrição do sistema não abordará a fase de cobrança da dívida, mas, para fins de delimitação deste escopo, será restrita apenas ao recebimento, processamento e armazenamento dos dados de dívida enviados pelos clientes ao bureau. A complexidade se deve à multiplicidade de formatos de dados e arquivos que os clientes podem utilizar, e como o sistema deve reagir para ingerir esses dados de maneira fluida e precisa, atendendo tanto pequenos quanto grandes clientes, garantindo precisão e agilidade.

## Componentes do Sistema  

### Serviço de Lotes  
O serviço de lotes é utilizado para rastrear e manter o status das solicitações de dados de dívida que os clientes enviam ao bureau.

### Serviços de Entrada  
Utilizados como serviços de borda, esses componentes são expostos ao cliente para coletar, de forma passiva ou ativa, os dados de dívida. Esses serviços respondem de maneira síncrona, recebendo a entrada e armazenando os dados brutos para processamento intermediário.

### Processo de Captura de Arquivos  
No caso de arquivos entregues via FTP, um evento na notificação S3 será acionado para a SQS. Em seguida, o processo de captura de arquivos verificará o arquivo, identificará os detalhes do cliente e, a partir daí, poderá dividir um arquivo grande em vários sublotes ou registrar um único lote para o processo intermediário.

### Processo Intermediário  
Este serviço processa os dados brutos coletados por diversas fontes na fase de entrada, realiza ETL, prepara os dados e os injeta em um banco DynamoDB que será utilizado para o processo de registro.

### Processo de Registro  
Extrai os dados do armazenamento temporário no DynamoDB, processa esses dados de acordo com diversas validações exigidas e os insere no sistema de armazenamento de dívidas.

### Broker de Mensagens  
Para este modelo, utilizamos o Amazon SNS para retransmissão básica de mensagens entre processos, sinalizando para diferentes componentes, como o serviço de lotes, que há um novo arquivo a ser processado pelo sistema intermediário, ou que o serviço de lotes precisa atualizar o status do lote.

### Monitoramento  
Todos os processos emitem métricas de séries temporais para o AWS Timestream para análise posterior.

## Serviço de Processamento de Lotes  

O Serviço de Processamento de Lotes é responsável por gerenciar o status das submissões dos clientes e rastrear seu progresso nas várias etapas do pipeline de processamento de dívidas. Este serviço garante que os dados sejam tratados, processados e armazenados corretamente no sistema.

### Responsabilidades Principais do Serviço de Processamento de Lotes  

#### Criação de Lotes  
- Invocado por uma chamada síncrona das etapas de entrada, contém informações básicas de processamento do lote, incluindo status, localização do arquivo, identificação do cliente, número do lote, número de dívidas recebidas, processadas e falhas.  
- Assim que um lote é criado, uma mensagem é enviada para a fila do processo intermediário.

#### Rastreamento e Atualizações de Status  
- O Serviço de Processamento de Lotes mantém o status do lote ao longo de todo o pipeline.

#### Conclusão do Lote  
- Após a conclusão do processo de registro, o status do lote é finalizado e atualizado.  
- O lote enviará mensagens para o destino que precisa ser notificado sobre os resultados, seja por SES para e-mail ou Event Bridge para webhook ou outros métodos.

#### Notificação de Falha  
- Em caso de falha no processamento da entrada, como formato inválido, assim que a mensagem for recebida pelo serviço, esta será enviada ao cliente por meio do SES ou Event Bridge.

## Fase de Entrada  

A Fase de Entrada é responsável por receber, armazenar e registrar os dados enviados pelos clientes em diferentes formatos. Os clientes podem submeter dados por upload de arquivos ou por meio de requisições REST individuais.

### 1 - Submissão de Dados via REST API usando Formato Padrão  
**Público-alvo:** Para clientes que desejam utilizar o formato padrão, sejam eles pequenos ou grandes.  
**Requisitos:** Os dados devem estar em um formato definido.  
**Operação:**  
- O cliente enviará várias linhas de dívida via chamadas REST.  
- Os dados são gravados em um banco de dados NoSQL temporário.  
- Após a finalização, o cliente invocará uma solicitação de submissão e os dados serão gravados em um CSV no S3.  
**Próximo passo:**  
- Uma mensagem é enviada ao serviço de lotes para criar o lote e posteriormente enviá-lo para a fila do processo intermediário.

### 2 - Submissão de Dados via REST API para CSV  
**Público-alvo:** Para clientes que desejam utilizar o formato padrão, sejam eles pequenos ou grandes.  
**Requisitos:** Os dados devem estar em um formato definido.  
**Operação:**  
- O cliente enviará um único arquivo CSV via chamada REST.  
- O CSV é armazenado no S3.  
**Próximo passo:**  
- Uma mensagem é enviada ao serviço de lotes para criar o lote e enviá-lo ao processo intermediário.

### 3 - Submissão de Arquivos via FTP  
**Público-alvo:** Para clientes que enviam arquivos de tamanho variável.  
**Operação:**  
- Os clientes enviam arquivos para o S3 via FTP em diversos formatos, seja padrão ou personalizado.  
**Próximo passo:**  
- A notificação de evento S3 envia uma mensagem para a fila SQS do serviço de captura de arquivos.

### 4 – Formato Personalizado para Pequenos Clientes  
**Público-alvo:** Pequenos clientes com poucas dívidas.  
**Operação:**  
- O cliente faz o upload de um arquivo (CSV, TXT, XLS) via portal web.  
- As primeiras 100 colunas são analisadas, e o usuário mapeia os campos.  
- O arquivo é gravado no S3.  
**Próximo passo:**  
- Uma mensagem é enviada ao serviço de lotes para criação do lote e envio ao processo intermediário.

## Fase de Processamento

### Serviço de Captura de Arquivos 

**Entrada**
- Monitora a fila SQS do serviço de captura de arquivos.  

**Operação**
- Executa em paralelo com múltiplas réplicas.  
- Verifica arquivos corrompidos e informações do cliente.  
- Divide arquivos grandes em sublotes, se necessário.  
**Próximo passo:**  
- Envia mensagens para o serviço de lotes e para a fila do processo intermediário.  

## Serviço Intermediário  

**Entrada**
- Monitora a fila SQS intermediária.  

**Operação**
- Executa em paralelo com múltiplas réplicas.  
- Obtém regras de um serviço externo, se necessário.  
- Processa os dados para o formato exigido pelo DynamoDB.  

**Próximo passo**
- Relata sucesso/falha à fila do serviço de lotes.  
- Se bem-sucedido, encaminha para a fila do processo de registro.  

## Serviço de Registro  

**Entrada**
- Monitora a fila SQS de registro.  

**Operação**
- Executa em paralelo com múltiplas réplicas.  
- Recupera os dados do DynamoDB, valida e os grava no sistema de armazenamento.  
- Registra inconsistências para retorno posterior.  

**Próximo passo**
- Relata os resultados finais do processamento à fila do serviço de lotes.  


## Coleta de Métricas  

Cada serviço (Intermediário, Captura, Registro, Lotes) registra métricas relevantes em um banco de dados de séries temporais. Esse banco fornece um histórico das atividades do sistema, permitindo o monitoramento e a análise do desempenho.

O **Amazon Timestream** é utilizado para armazenar métricas de tempo para todos os serviços do pipeline. Alternativamente, serviços não gerenciados como **Prometheus** podem ser usados.

Métricas monitoradas incluem:  

- Número de dívidas processadas.  
- Número de falhas.  
- Desempenho do processamento por cliente.  
