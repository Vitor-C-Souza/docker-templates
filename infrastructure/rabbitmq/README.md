# RabbitMQ

RabbitMQ é um message broker. Ele permite que aplicações publiquem mensagens e que outros serviços as consumam de forma assíncrona.

A ideia central é desacoplar quem **produz** um trabalho de quem **processa** esse trabalho.

## Quando usar

RabbitMQ é uma boa opção para cenários como:

- processamento assíncrono;
- filas de tarefas;
- comunicação entre microsserviços;
- envio de e-mails em background;
- processamento de arquivos;
- integração entre sistemas;
- retry de mensagens;
- workloads que não precisam bloquear a requisição HTTP principal.

## Exemplo: processamento assíncrono

Imagine uma API que precisa enviar um e-mail.

Sem mensageria:

```text
Cliente
  │
  ▼
API
  │
  ├── gera resposta
  │
  └── envia e-mail
          │
          ▼
       SMTP
```

A requisição fica dependente do processamento do e-mail.

Com RabbitMQ:

```text
Cliente
  │
  ▼
API
  │
  └── publica mensagem
          │
          ▼
      RabbitMQ
          │
          ▼
       Worker
          │
          ▼
       SMTP
```

A API pode responder sem precisar executar todo o trabalho antes de devolver a resposta.

## Conceitos principais

```text
Producer
   │
   │ publica
   ▼
Exchange
   │
   │ roteia
   ▼
Queue
   │
   │ entrega
   ▼
Consumer
```

### Producer

É quem publica a mensagem.

### Exchange

Recebe mensagens do producer e decide para quais filas elas devem ser encaminhadas.

### Queue

Armazena as mensagens até que sejam consumidas.

### Consumer

É o serviço que recebe e processa as mensagens.

## Por que usar Exchange?

O producer normalmente não precisa conhecer diretamente o consumer.

Isso ajuda a reduzir o acoplamento:

```text
Producer
   │
   ▼
Exchange
  /   \
 ▼     ▼
Q1     Q2
│      │
▼      ▼
Worker A  Worker B
```

O mesmo evento pode ser roteado para diferentes consumidores dependendo da configuração.

## Estrutura

```text
rabbitmq/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Configuração

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    container_name: rabbitmq
    restart: unless-stopped

    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_DEFAULT_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_DEFAULT_PASS}

    ports:
      - "${RABBITMQ_PORT}:5672"
      - "${RABBITMQ_MANAGEMENT_PORT}:15672"

    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

volumes:
  rabbitmq_data:
```

### Imagem `management`

A variante `management` inclui a interface web de gerenciamento do RabbitMQ.

Isso é muito útil no desenvolvimento para visualizar:

- exchanges;
- queues;
- mensagens;
- consumers;
- conexões;
- bindings.

### Portas

RabbitMQ usa principalmente:

| Porta | Função |
|---:|---|
| `5672` | Protocolo AMQP para aplicações |
| `15672` | Interface web de gerenciamento |

Do host:

```text
AMQP: localhost:5672
Management: http://localhost:15672
```

De outro container na mesma rede:

```text
rabbitmq:5672
```

A interface de gerenciamento normalmente continua sendo acessada pelo host usando a porta publicada `15672`.

### Volume

O volume mantém o estado do RabbitMQ entre recriações do container.

```yaml
volumes:
  - rabbitmq_data:/var/lib/rabbitmq
```

### Healthcheck

O template usa:

```yaml
healthcheck:
  test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
```

Isso verifica se o RabbitMQ está respondendo.

Um worker dependente pode declarar:

```yaml
depends_on:
  rabbitmq:
    condition: service_healthy
```

Assim como nos templates de bancos e Redis, isso não elimina a necessidade de retry e tratamento de falhas na aplicação.

## `.env.example`

```env
RABBITMQ_DEFAULT_USER=app
RABBITMQ_DEFAULT_PASS=app_password
RABBITMQ_PORT=5672
RABBITMQ_MANAGEMENT_PORT=15672
```

Copie para `.env`:

```powershell
Copy-Item .env.example .env
```

Não versione credenciais reais.

## Subindo

```bash
docker compose up -d
```

Verifique:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f rabbitmq
```

## Management UI

Com o container rodando, abra no navegador:

```text
http://localhost:15672
```

Use as credenciais definidas no `.env`.

> Em produção, não exponha a interface de gerenciamento publicamente sem uma estratégia de segurança adequada.

## Spring Boot

Para Spring Boot com Spring AMQP, uma configuração local pode usar:

```properties
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=app
spring.rabbitmq.password=app_password
```

Se a aplicação também estiver no Docker:

```properties
spring.rabbitmq.host=rabbitmq
spring.rabbitmq.port=5672
```

O hostname passa a ser `rabbitmq`, porque esse é o nome do serviço na rede Docker.

## RabbitMQ x Redis

Essa é uma distinção importante.

| Redis | RabbitMQ |
|---|---|
| Cache e armazenamento rápido | Message broker |
| TTL é muito comum | Filas e entrega de mensagens são centrais |
| Estruturas de dados em memória | Exchanges, queues e bindings |
| Excelente para cache | Excelente para processamento assíncrono |
| Pode participar de comunicação | Projetado especificamente para mensageria |

Pergunte:

> "Quero guardar um valor para acessar rapidamente?"

Considere Redis.

> "Quero colocar um trabalho numa fila para outro serviço processar?"

Considere RabbitMQ.

## Ack e processamento

O consumer pode confirmar o processamento da mensagem usando acknowledgement.

Um fluxo simplificado:

```text
Queue
  │
  ▼
Consumer recebe
  │
  ├── processamento OK ──► ACK
  │
  └── falha ─────────────► mensagem pode ser reprocessada
```

Isso é uma das razões pelas quais RabbitMQ é interessante para workloads que precisam de processamento assíncrono controlado.

## Retry e mensagens problemáticas

Não trate retry como simplesmente "tentar infinitamente".

Um sistema pode acabar assim:

```text
Mensagem
   │
   ▼
Consumer
   │
   ├── falha
   ▼
Retry
   │
   ├── falha
   ▼
Retry
   │
   └── ...
```

Em uma arquitetura real, considere estratégias como limite de tentativas, atraso entre tentativas e uma dead-letter queue para mensagens que não podem ser processadas.

## Cenário: API + RabbitMQ + Worker

```text
┌─────────────────────────────────────────┐
│ Docker network                          │
│                                         │
│ ┌──────────────┐     ┌──────────────┐   │
│ │ Spring Boot  │────►│   RabbitMQ   │   │
│ │ API          │     │              │   │
│ └──────────────┘     └──────┬───────┘   │
│                             │            │
│                             ▼            │
│                     ┌──────────────┐     │
│                     │    Worker    │     │
│                     └──────────────┘     │
└─────────────────────────────────────────┘
```

## Exemplo de stack dependente

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  worker:
    image: minha-api-worker:latest
    depends_on:
      rabbitmq:
        condition: service_healthy
```

## Problemas comuns

### `connection refused`

Confira:

```bash
docker compose ps
```

E:

```bash
docker compose logs rabbitmq
```

### Aplicação containerizada usando `localhost`

Use:

```text
rabbitmq:5672
```

### Não consigo acessar o Management UI

Confirme se a porta `15672` está publicada:

```bash
docker compose ps
```

E acesse:

```text
http://localhost:15672
```

### Alterei usuário ou senha e nada mudou

Assim como em outros serviços inicializados por variáveis de ambiente, o comportamento inicial pode estar ligado ao estado persistido no volume. Para um ambiente descartável, remova o volume e recrie a stack.

```bash
docker compose down -v
docker compose up -d
```

Isso remove os dados persistidos.

## Segurança

Este template é voltado para desenvolvimento local.

Para produção, avalie:

- credenciais armazenadas como secrets;
- TLS;
- usuários e permissões adequados;
- exposição da porta de gerenciamento;
- redes privadas;
- políticas de retenção;
- monitoramento;
- limites de recursos;
- alta disponibilidade quando necessária.

## Resumo

```text
5672  → aplicação / AMQP
15672 → Management UI
```

```text
Producer
   │
   ▼
Exchange
   │
   ▼
Queue
   │
   ▼
Consumer
```

RabbitMQ é especialmente interessante quando você quer **desacoplar produção e processamento de trabalho**, permitindo que uma aplicação continue responsiva enquanto outro serviço processa a tarefa de forma assíncrona.
