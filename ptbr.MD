# Sistema de Pipeline de Processamento de Dívidas

O Pipeline de Processamento de Dívidas é um sistema abrangente projetado para processar dados de dívidas enviados por clientes por meio de vários métodos. O pipeline lida com o envio, armazenamento, transformação e processamento final dos dados para garantir que as informações sobre as dívidas sejam registradas e rastreadas com precisão do início ao fim, de forma assíncrona, permitindo que o operador do sistema escale os componentes do sistema de acordo com a demanda variável ao longo do dia.

## Motivação

Esta empresa é uma agência de cobrança de dívidas.  
Seus clientes são empresas que desejam cobrar dívidas em atraso de indivíduos que não pagaram voluntariamente. Os clientes enviam as informações de cobrança, que incluem detalhes básicos sobre o devedor que desejam cobrar, como nome, CPF (Cadastro de Pessoas Físicas), o valor devido e a data em que o pagamento foi perdido. Uma vez que a dívida é registrada no sistema da agência de cobrança, as dívidas serão protestadas, o devedor será notificado, e um desconto ou plano de pagamento parcelado pode ser negociado.

## Escopo

Esta descrição do sistema não abordará a fase de cobrança da dívida, mas, para fins de limitação deste escopo, será restrita apenas ao recebimento, processamento e armazenamento dos dados de dívidas que os clientes enviam para a agência. A complexidade se deve à multiplicidade de formatos de dados e arquivos que os clientes podem usar, e como o sistema pode reagir para ingerir esses dados de forma suave e precisa, desde clientes pequenos até muito grandes, exigindo precisão e agilidade.

## Componentes do Sistema

### a. Serviço de Lote (Batch Service)

O serviço de lote é usado para rastrear e manter o status das solicitações de dados de dívidas enviadas pelos clientes à agência.

### b. Serviços de Entrada (Input Services)

Usados como serviços de fronteira, esses componentes são expostos ao cliente para coletar, de forma passiva ou ativa, os dados da dívida. Esses serviços respondem de forma síncrona, recebendo os dados e armazenando-os para processamento intermediário.

### c. Processo de Seleção de Arquivos (File Picker Process)

No caso de arquivos entregues via FTP, ele acionará um evento de notificação do S3 para o SQS. Em seguida, o processo de seleção de arquivos verificará o arquivo, verificará os detalhes do cliente e, a partir daí, dividirá um arquivo grande em vários sub-lotes ou registrará um único lote para o processo intermediário.

### d. Processo Intermediário (Intermediate Process)

Este serviço faz o processamento dos dados brutos, coletados por várias fontes na fase de entrada, realiza trabalho de ETL (Extract, Transform, Load), prepara os dados e os injeta em um DynamoDB que será usado para o processo de registro.

### e. Processo de Registro (Registering Process)

Pega os dados do armazenamento temporário no DynamoDB, processa esses dados de acordo com várias validações necessárias e insere os dados finais da dívida no sistema de armazenamento de dívidas.

### f. Message Broker (Intermediador de Mensagens)

Para este modelo, usamos o Amazon SNS para a comunicação básica entre processos, sinalizando para diferentes componentes, como o serviço de lote, que há um novo arquivo a ser processado pelo sistema intermediário, ou que o serviço de lote precisa atualizar o status do lote.

### g. Monitoramento

Todos os processos emitem métricas de séries temporais para o AWS Timestream para análise posterior.

## Serviço de Processamento de Lotes (Batch Processing Service)

O Serviço de Processamento de Lotes é responsável por gerenciar o status dos envios dos clientes e rastrear seu progresso através das várias etapas do pipeline de processamento de dívidas. Este serviço garante que os dados sejam corretamente manipulados, processados e armazenados no sistema.

### Responsabilidades Principais do Serviço de Processamento de Lotes

#### Criação de Lotes

- Acionado por uma chamada síncrona das etapas de entrada, contém informações básicas de processamento do lote, incluindo status, localização do arquivo, identificação do cliente, número do lote, número de dívidas recebidas, processadas e com falha.
- Uma vez que um lote é criado, uma mensagem é enviada para a fila do processo intermediário.

#### Rastreamento e Atualizações de Status

O Serviço de Processamento de Lotes mantém o status do lote durante todo o pipeline.

#### Conclusão do Lote

- Uma vez que o processo de registro é concluído, o status do lote é finalizado e atualizado de volta ao lote.
- O lote enviará mensagens para o destino que precisa ser notificado sobre os resultados, seja por meio do SES para e-mail ou do EventBridge para webhooks ou outros métodos.

#### Notificação de Falha

- Em caso de falha durante o processamento da entrada, como formato incorreto, uma vez que a mensagem é recebida pelo serviço, ela será enviada ao cliente por meio do SES ou do EventBridge.

## Fase de Entrada (Input Phase)

A Fase de Entrada é responsável por receber, armazenar e registrar os dados recebidos dos clientes em diferentes formatos. Os clientes podem enviar dados por meio de uploads de arquivos ou através de solicitações REST individuais. Uma vez que os dados são enviados, eles passam por um processamento para transformá-los em um formato adequado para as próximas etapas do pipeline de processamento de dívidas.

### 1 - Envio de Dados via REST API usando formato padrão

#### Público-Alvo

- Para clientes dispostos a usar o formato padrão, sejam grandes ou pequenos.
- Requer que os dados estejam em um formato definido.

#### Operação

- O cliente enviará várias linhas de dívida por meio de chamadas REST.
- Os dados são registrados em um banco de dados NoSQL temporário.
- Ao finalizar, o cliente invocará uma solicitação de envio. Em seguida, os dados são gravados no S3 no formato CSV padrão.

#### Próxima Etapa

- Uma mensagem é enviada ao serviço de lote para criar um lote e, em seguida, enviar uma mensagem para a fila do processo intermediário.

### 2 - Envio de Dados via REST API para CSV usando formato padrão

#### Público-Alvo

- Para clientes dispostos a usar o formato padrão, sejam grandes ou pequenos.
- Requer que os dados estejam em um formato definido.

#### Operação

- O cliente enviará um único CSV por meio de uma chamada REST.
- O CSV é armazenado no S3.

#### Próxima Etapa

- Uma mensagem é enviada ao serviço de lote para criar um lote e, em seguida, enviar uma mensagem para a fila do processo intermediário.

### 3 - Envio de Arquivos via FTP

#### Público-Alvo

- Para clientes que enviam arquivos de tamanho variável.
- Pode ser qualquer formato, definido pelo acordo com o cliente, seja formato padrão ou personalizado.
- Os clientes devem ter acesso ao armazenamento FTP.

#### Operação

- O cliente enviará arquivos CSV no formato padrão para o S3 via FTP.

#### Próxima Etapa

- A notificação de evento do S3 envia uma mensagem para a fila SQS do serviço de seleção de arquivos.

### 4 - Autoatendimento para formato personalizado pequeno

#### Público-Alvo

- Para clientes pequenos que possuem um número reduzido de dívidas.
- Requer um arquivo (CSV, TXT, XLS).

#### Operação

- O cliente usará um site que permite o envio do arquivo.
- As primeiras 100 colunas serão lidas, e o usuário será apresentado no navegador com a opção de corresponder as colunas necessárias ao seu arquivo.
- Uma vez acordado, o envio é realizado.
- O arquivo é gravado no S3.

#### Próxima Etapa

- Uma mensagem é enviada ao serviço de lote para criar um lote e, em seguida, enviar uma mensagem para a fila do processo intermediário. Inclui informações básicas sobre o mapeamento.

## Fase de Processamento (Processing Phase)

A fase de processamento transforma a entrada no destino final.

### 1 - Serviço de Seleção de Arquivos (File Picker Service)

#### Entrada

- O serviço de seleção de arquivos monitora a fila SQS de seleção de arquivos.

#### Operação

- Executa em paralelo com várias réplicas.
- Quando uma mensagem chega, ele localiza o arquivo no S3.
- Avalia quem é o cliente e se o arquivo não está corrompido.
- Verifica se o arquivo é grande o suficiente para exigir que seja dividido ou mantido como um único lote.
- Se necessário, divide o arquivo em arquivos menores e os grava no S3.
- Identifica o tipo de processamento do cliente (formato padrão, formato personalizado por código).

#### Próxima Etapa

- Uma mensagem é enviada ao serviço de lote para criar um lote e, em seguida, enviar uma mensagem para a fila do processo intermediário, como um único ou múltiplos sub-lotes.

### 2 - Serviço Intermediário (Intermediate Service)

#### Entrada

- O serviço intermediário monitora a fila SQS intermediária.

#### Operação

- Executa em paralelo com várias réplicas.
- Avalia se o arquivo existe e está correto.
- Verifica na fila o formato do arquivo a ser processado.
- Pode precisar buscar regras de um serviço externo para saber como processar o arquivo.
- Processa os dados no formato esperado pelo DynamoDB para o processo de registro.
  - O método pode ser discutido em várias opções, incluindo:
    - Código personalizado -- Python, Go, etc.
    - AWS Glue + AWS ETL.
    - EMR Hadoop? Há dados tão grandes assim?

#### Próxima Etapa

- Envia uma mensagem para a fila SQS do serviço de lote relatando mudanças no lote.
- Se bem-sucedido, envia uma mensagem para a fila SQS do processo de registro para processar a solicitação final.
- Se mal-sucedido, envia uma mensagem para a fila SQS do serviço de lote informando sobre a falha no processamento do arquivo (formato, corrupção, etc.).

### 3 - Serviço de Registro (Registering Service)

*Não sabemos exatamente qual é o destino final, então isso é apenas uma suposição de que o armazenamento final é uma "caixa preta" que contém dados e espera um formato específico, como um serviço.*

#### Entrada

- O serviço de registro monitora a fila SQS de registro.

#### Operação

- Executa em paralelo com várias réplicas.
- Recebe cada mensagem para processar um lote.
- Avalia os dados do DynamoDB, realiza validações e insere os valores.
- Registra eventuais inconsistências ou dívidas com falha e as grava no S3 para retorno.

#### Próxima Etapa

- Envia uma mensagem para a fila SQS do serviço de lote relatando o número total de processamentos e o resultado final, seja sucesso ou falha.

## Coleta de Métricas

- Cada serviço (Intermediário, seleção de arquivos, registro, lote) registra métricas relevantes em um banco de dados de séries temporais. Esse banco de dados fornece um registro histórico da atividade do sistema, permitindo o monitoramento e análise do desempenho do sistema.
- O Amazon Timestream é usado para armazenar métricas baseadas em tempo para todos os serviços no pipeline. Alternativamente, serviços não gerenciados como o Prometheus podem ser usados.
- As métricas rastreadas incluem o número de linhas processadas por cliente, o número de falhas, o número de dívidas processadas durante cada execução de injeção e o número de rejeições.