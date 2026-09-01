# Evolution Go

Evolution Go é uma API de integração com WhatsApp desenvolvida em Go pela Evolution Foundation. Ela fornece API REST, eventos em tempo real, WebSocket, Webhooks e suporte a AMQP/RabbitMQ e NATS. Também possui persistência opcional em PostgreSQL e suporte a armazenamento de mídia via MinIO/S3. citeturn0search2

> Este template é voltado para desenvolvimento local. A imagem, as variáveis de ambiente e a arquitetura podem mudar conforme a versão do Evolution Go. Confira a documentação oficial antes de usar uma versão diferente em produção.

## Quando usar

Evolution Go é útil quando você quer integrar uma aplicação com WhatsApp sem implementar diretamente o protocolo de comunicação.

Exemplos:

- CRM com atendimento via WhatsApp;
- automações de mensagens;
- notificações para clientes;
- integração de WhatsApp com sistemas internos;
- recebimento de eventos via webhook;
- processamento de eventos através de RabbitMQ.

## Arquitetura deste template

O template usa Evolution Go + PostgreSQL:

```text
┌─────────────────────────────────────┐
│ Docker network                      │
│                                     │
│ ┌────────────────┐                  │
│ │ Evolution Go   │                  │
│ │ :4000          │                  │
│ └───────┬────────┘                  │
│         │                            │
│         │ PostgreSQL                 │
│         ▼                            │
│ ┌────────────────┐                  │
│ │ PostgreSQL     │                  │
│ │ :5432          │                  │
│ └────────────────┘                  │
└─────────────────────────────────────┘
```

A configuração oficial do Evolution Go também contempla PostgreSQL, RabbitMQ, NATS e MinIO em uma stack mais completa. citeturn0search0turn0search2

Neste repositório, deixamos o template inicial propositalmente menor para facilitar o uso local. RabbitMQ, Redis e outros serviços já possuem templates próprios no cookbook.

## Estrutura

```text
evolution-go/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Template

```yaml
services:
  evolution-go:
    image: evoapicloud/evolution-go:latest
    container_name: evolution-go
    restart: unless-stopped

    environment:
      SERVER_PORT: 4000
      CLIENT_NAME: ${EVOLUTION_CLIENT_NAME}
      GLOBAL_API_KEY: ${EVOLUTION_GLOBAL_API_KEY}
      POSTGRES_AUTH_DB: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/evogo_auth?sslmode=disable
      POSTGRES_USERS_DB: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/evogo_users?sslmode=disable
      DATABASE_SAVE_MESSAGES: ${DATABASE_SAVE_MESSAGES}
      CONNECT_ON_STARTUP: ${CONNECT_ON_STARTUP}
      WEBHOOK_FILES: ${WEBHOOK_FILES}
      EVENT_IGNORE_GROUP: ${EVENT_IGNORE_GROUP}
      EVENT_IGNORE_STATUS: ${EVENT_IGNORE_STATUS}

    ports:
      - "${EVOLUTION_PORT}:4000"

    volumes:
      - evolution_data:/app/dbdata
      - evolution_logs:/app/logs

    depends_on:
      postgres:
        condition: service_healthy

    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider http://localhost:4000/ || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  postgres:
    image: postgres:15-alpine
    container_name: evolution-postgres
    restart: unless-stopped

    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_AUTH_DATABASE}

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_AUTH_DATABASE}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

volumes:
  evolution_data:
  evolution_logs:
  postgres_data:
```

## `.env.example`

```env
EVOLUTION_PORT=4000
EVOLUTION_CLIENT_NAME=evolution
EVOLUTION_GLOBAL_API_KEY=change-me-to-a-secure-key

POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres_password
POSTGRES_AUTH_DATABASE=evogo_auth

DATABASE_SAVE_MESSAGES=true
CONNECT_ON_STARTUP=false
WEBHOOK_FILES=true
EVENT_IGNORE_GROUP=false
EVENT_IGNORE_STATUS=true
```

Copie para `.env`:

```powershell
Copy-Item .env.example .env
```

Nunca versione a chave real da API ou a senha do PostgreSQL.

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
docker compose logs -f evolution-go
```

## API

Neste template a API é publicada na porta `4000`:

```text
http://localhost:4000
```

A documentação oficial do projeto informa endpoints de criação de instância, QR code, envio de mensagens e consulta de status, além de documentação Swagger. citeturn0search2

## Autenticação

O template usa:

```env
EVOLUTION_GLOBAL_API_KEY=change-me-to-a-secure-key
```

Troque por uma chave forte antes de usar o serviço.

A API deve ser tratada como um serviço protegido, principalmente se a porta for publicada fora da máquina local.

## PostgreSQL

O Evolution Go utiliza PostgreSQL para persistência quando configurado para isso. A documentação do projeto lista PostgreSQL/GORM como parte da stack e indica bancos de autenticação e usuários na configuração. citeturn0search2turn0search0

Neste template temos:

```text
Evolution Go
     │
     ├── evogo_auth
     │
     └── evogo_users
```

As URLs são montadas com o hostname Docker:

```text
postgres:5432
```

Não use `localhost:5432` dentro do container Evolution Go.

## Por que `depends_on: service_healthy`?

O PostgreSQL precisa estar pronto para aceitar conexões antes que o Evolution Go tente inicializar suas dependências.

Por isso usamos:

```yaml
depends_on:
  postgres:
    condition: service_healthy
```

E o PostgreSQL possui:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_AUTH_DATABASE}"]
```

Isso é mais útil do que apenas:

```yaml
depends_on:
  - postgres
```

porque o segundo formato controla a ordem de inicialização, mas não expressa que o banco precisa estar saudável antes da dependência continuar.

Mesmo assim, a aplicação deve possuir retry e tratamento de falhas de conexão.

## Persistência

Temos três volumes:

```text
evolution_data
      │
      └── estado/dados da aplicação

evolution_logs
      │
      └── logs

postgres_data
      │
      └── dados PostgreSQL
```

Se você executar:

```bash
docker compose down
```

os volumes permanecem.

Se executar:

```bash
docker compose down -v
```

os volumes são removidos e os dados persistidos são perdidos.

## Cenário: Spring Boot + Evolution Go

Uma aplicação Java pode atuar como cliente da API:

```text
┌──────────────┐
│ Spring Boot  │
└──────┬───────┘
       │ HTTP
       │ API Key
       ▼
┌──────────────┐
│ Evolution Go │
└──────┬───────┘
       │
       ▼
    WhatsApp
```

Por exemplo, seu backend pode possuir uma camada:

```text
WhatsAppPort
     │
     ▼
EvolutionGoAdapter
     │
     ▼
Evolution Go API
```

Isso evita espalhar detalhes específicos da Evolution Go pelo domínio da aplicação.

## Webhooks

Evolution Go possui suporte a Webhooks e eventos em tempo real. citeturn0search2

Um fluxo típico pode ser:

```text
WhatsApp
   │
   ▼
Evolution Go
   │
   │ webhook
   ▼
Spring Boot
   │
   ▼
Processamento do evento
```

Se houver muito processamento, você pode combinar isso com RabbitMQ:

```text
WhatsApp
   │
   ▼
Evolution Go
   │
   ▼
Spring Boot
   │
   ▼
RabbitMQ
   │
   ▼
Worker
```

Isso permite responder rapidamente ao webhook e deixar processamento pesado para consumidores assíncronos.

## Evolution Go + RabbitMQ

A documentação do Evolution Go lista AMQP/RabbitMQ entre os mecanismos de eventos suportados. citeturn0search2

O projeto oficial também apresenta uma configuração Docker completa com RabbitMQ e outras dependências. citeturn0search0

Como já temos um template RabbitMQ separado neste repositório, podemos posteriormente criar uma stack composta:

```text
stacks/
└── evolution-go-rabbitmq/
```

sem duplicar o conhecimento de RabbitMQ.

## Evolution Go + MinIO

Para mídia, o projeto também suporta armazenamento baseado em MinIO/S3. citeturn0search2turn0search0

Isso é interessante quando o sistema precisa trabalhar com:

- imagens;
- vídeos;
- documentos;
- áudios.

Podemos adicionar um template MinIO posteriormente e depois criar uma stack composta com Evolution Go.

## Problemas comuns

### `connection refused` no PostgreSQL

Verifique:

```bash
docker compose ps
```

O PostgreSQL deve estar `healthy`.

Veja os logs:

```bash
docker compose logs postgres
```

### Evolution Go não inicia

Veja:

```bash
docker compose logs evolution-go
```

Confira principalmente:

- `GLOBAL_API_KEY`;
- URLs do PostgreSQL;
- nomes das variáveis conforme a versão da imagem;
- estado dos volumes.

### API não responde em `localhost:4000`

Verifique:

```bash
docker compose ps
```

E:

```bash
docker compose logs evolution-go
```

### Alterei senha do PostgreSQL mas continua usando a antiga

Se o volume do PostgreSQL já existe, alterar `POSTGRES_PASSWORD` não recria automaticamente o usuário/senha.

Para um ambiente descartável:

```bash
docker compose down -v
docker compose up -d
```

Isso apaga os dados.

## Versões e imagem

Este template usa:

```text
evoapicloud/evolution-go:latest
```

A documentação oficial disponibiliza o projeto e seus exemplos Docker, mas versões e variáveis podem evoluir. citeturn0search0turn0search2

Para ambientes mais controlados, prefira fixar uma versão específica da imagem depois de validar a compatibilidade da configuração.

## Produção

Este template é para desenvolvimento/local.

Para produção, avalie:

- versão fixa da imagem;
- API key armazenada como secret;
- HTTPS;
- reverse proxy, como Traefik;
- firewall;
- PostgreSQL com credenciais seguras;
- backups;
- persistência adequada;
- monitoramento;
- limites de CPU/RAM;
- política de logs;
- webhook protegido;
- RabbitMQ/NATS quando necessário;
- armazenamento de mídia em S3/MinIO quando necessário.

## Evolution Go + Traefik

Uma arquitetura mais próxima de produção pode ficar assim:

```text
Internet
   │
 HTTPS
   ▼
┌───────────┐
│  Traefik  │
└─────┬─────┘
      │
      ▼
┌──────────────┐
│ Evolution Go │
└──────┬───────┘
       │
       ├──► PostgreSQL
       │
       ├──► RabbitMQ
       │
       └──► MinIO/S3
```

Isso aproveita os outros templates deste repositório sem precisar colocar tudo em um único Compose gigante.

## Resumo

| Serviço | Função |
|---|---|
| Evolution Go | Integração com WhatsApp |
| PostgreSQL | Persistência |
| RabbitMQ | Eventos/processamento assíncrono, quando necessário |
| MinIO/S3 | Mídia, quando necessário |
| Traefik | Entrada HTTP/HTTPS, quando necessário |

O objetivo deste template é fornecer uma base local simples e deixar as dependências opcionais separadas, para que você monte a arquitetura conforme o projeto realmente precisar.
