# PostgreSQL

PostgreSQL é um sistema gerenciador de banco de dados relacional (RDBMS). Assim como MySQL, trabalha com tabelas, relacionamentos e SQL, mas possui um conjunto amplo de recursos avançados para modelagem, consultas e extensibilidade.

## Quando usar

PostgreSQL é uma ótima escolha quando:

- o domínio é relacional;
- existem regras de integridade e relacionamentos importantes;
- consultas complexas fazem parte da aplicação;
- você precisa de tipos de dados ricos;
- extensões do PostgreSQL podem agregar valor;
- consistência e recursos avançados do banco são importantes.

## PostgreSQL x MySQL

Não existe um vencedor universal. Ambos atendem muito bem aplicações web e APIs tradicionais.

Como regra prática:

| Situação | Tendência |
|---|---|
| CRUD relacional tradicional | Ambos |
| Consultas relacionais complexas | PostgreSQL tende a ser uma opção forte |
| Tipos e recursos avançados | PostgreSQL tende a oferecer mais opções |
| Ecossistema web muito difundido | Ambos |
| Projeto já padronizado em MySQL | MySQL |
| Projeto novo sem restrição tecnológica | Avalie os requisitos antes de decidir |

A comparação completa deve considerar o modelo de dados, consultas, equipe, infraestrutura e recursos realmente necessários.

## Estrutura

```text
postgresql/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Configuração

```yaml
services:
  postgresql:
    image: postgres:18
    container_name: postgresql
    restart: unless-stopped

    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    ports:
      - "${POSTGRES_PORT}:5432"

    volumes:
      - postgresql_data:/var/lib/postgresql

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

volumes:
  postgresql_data:
```

### `image`

A versão da imagem é explicitada em vez de depender de `latest`. Isso torna o ambiente mais previsível.

### `environment`

As variáveis configuram o banco inicial, usuário e senha. Os valores ficam no `.env` ou no ambiente do processo, e o `.env.example` serve como documentação.

### `ports`

A porta `5432` é a porta padrão do PostgreSQL dentro do container. A porta do host pode ser alterada pelo `POSTGRES_PORT`.

Se a aplicação estiver no host:

```text
localhost:5432
```

Se estiver em outro container na mesma rede:

```text
postgresql:5432
```

### `volumes`

O volume preserva os dados do PostgreSQL quando o container é recriado.

> A localização do diretório de dados pode variar conforme a versão da imagem oficial. Este template acompanha o layout da imagem escolhida; se a imagem for alterada, confirme o caminho de persistência correspondente.

### `healthcheck`

O `pg_isready` verifica se o servidor PostgreSQL está pronto para aceitar conexões.

Em uma aplicação dependente:

```yaml
depends_on:
  postgresql:
    condition: service_healthy
```

Isso é diferente de simplesmente declarar:

```yaml
depends_on:
  - postgresql
```

O healthcheck informa o estado do serviço; `service_healthy` permite que a dependência de inicialização considere esse estado.

## `.env.example`

```env
POSTGRES_DB=app_db
POSTGRES_USER=app
POSTGRES_PASSWORD=app_password
POSTGRES_PORT=5432
```

Copie para `.env`:

```powershell
Copy-Item .env.example .env
```

Não versione o `.env` quando ele contiver credenciais reais.

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
docker compose logs -f postgresql
```

## Parando

```bash
docker compose down
```

Isso remove o container, mas preserva o volume.

Para remover também os dados:

```bash
docker compose down -v
```

## Spring Boot

### Spring Boot no IntelliJ

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/app_db
spring.datasource.username=app
spring.datasource.password=app_password
```

### Spring Boot em Docker

Use o nome do serviço:

```properties
spring.datasource.url=jdbc:postgresql://postgresql:5432/app_db
```

`localhost` dentro do container do Spring Boot aponta para o próprio container, não para o PostgreSQL.

## Cenário: apenas banco local

```text
Spring Boot / IntelliJ
        │
        │ localhost:5432
        ▼
   PostgreSQL
   (container)
```

## Cenário: aplicação + banco em Docker

```text
┌─────────────────────────────┐
│ Docker network              │
│                             │
│  ┌──────────────┐           │
│  │ Spring Boot  │           │
│  └──────┬───────┘           │
│         │ postgresql:5432   │
│         ▼                   │
│  ┌──────────────┐           │
│  │ PostgreSQL   │           │
│  └──────────────┘           │
└─────────────────────────────┘
```

Se nenhum processo no host precisar acessar o banco, a publicação da porta pode ser removida da stack e o serviço continuar acessível pela rede Docker.

## Quando preferir PostgreSQL

Considere PostgreSQL especialmente quando a aplicação depende bastante de recursos relacionais avançados, consultas complexas, tipos de dados específicos ou extensões.

Isso não significa que MySQL seja inadequado para esses projetos em todos os casos. A decisão deve ser baseada nas necessidades reais.

## Problemas comuns

### `connection refused`

Verifique o estado:

```bash
docker compose ps
```

E os logs:

```bash
docker compose logs postgresql
```

### Aplicação containerizada usando `localhost`

Troque `localhost` pelo nome do serviço:

```text
postgresql:5432
```

### Alterar senha não altera um banco existente

As variáveis de inicialização não recriam automaticamente um banco já existente no volume. Para um ambiente descartável:

```bash
docker compose down -v
docker compose up -d
```

Isso apaga os dados.

## Segurança

Este template é voltado para desenvolvimento. Para produção, avalie secrets, controle de acesso, backups, rede, TLS quando aplicável, atualização de imagem e políticas de segurança.

## Resumo

| Situação | Hostname | Porta |
|---|---|---:|
| Aplicação no host | `localhost` | `5432` |
| Container na mesma rede | `postgresql` | `5432` |
| Cliente externo ao Docker | endereço do host | porta publicada |

O ponto central é o mesmo do template de MySQL: **portas publicadas são para acesso pelo host ou rede externa; containers na mesma rede usam o nome do serviço e a porta interna.**
