# Healthchecks no Docker Compose

Healthchecks ajudam a diferenciar duas situações que parecem iguais, mas não são:

- o container foi iniciado;
- o serviço dentro do container está pronto para ser utilizado.

Isso é especialmente importante quando um serviço depende de outro, como uma API que precisa do banco de dados antes de iniciar.

## `depends_on` não significa "serviço pronto"

Um exemplo simples:

```yaml
services:
  backend:
    depends_on:
      - mysql

  mysql:
    image: mysql:8.4
```

Nesse formato, o Compose conhece a dependência entre os serviços, mas isso não deve ser interpretado como uma verificação de que o MySQL terminou sua inicialização e está aceitando conexões.

## Healthcheck + `service_healthy`

Quando a aplicação precisa esperar o serviço ficar saudável, podemos combinar um `healthcheck` com uma condição de dependência:

```yaml
services:
  backend:
    image: minha-api:latest
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.4
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
```

O `healthcheck` verifica a saúde real do serviço. A condição `service_healthy` permite que o Compose aguarde esse estado antes de iniciar o serviço dependente.

## Componentes do healthcheck

### `test`
Define o comando usado para verificar a saúde do serviço.

### `interval`
Define o intervalo entre verificações.

### `timeout`
Define quanto tempo uma verificação pode levar antes de falhar.

### `retries`
Define quantas falhas consecutivas são necessárias para o serviço ser considerado `unhealthy`.

### `start_period`
Dá ao serviço um período inicial para inicialização antes de contabilizar as falhas normalmente.

## Exemplos

### MySQL

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 5s
  timeout: 5s
  retries: 10
  start_period: 10s
```

### PostgreSQL

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  interval: 5s
  timeout: 5s
  retries: 10
  start_period: 10s
```

### Redis

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 3s
  retries: 5
```

### RabbitMQ

```yaml
healthcheck:
  test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
  interval: 10s
  timeout: 5s
  retries: 5
```

### MongoDB

O comando deve ser compatível com o shell disponível na versão da imagem escolhida. Por exemplo, imagens que incluem `mongosh` podem usar uma verificação baseada em `ping` do servidor:

```yaml
healthcheck:
  test: ["CMD-SHELL", "mongosh --eval 'db.adminCommand({ ping: 1 })' --quiet"]
  interval: 10s
  timeout: 5s
  retries: 5
```

## Quando usar

Healthchecks são especialmente úteis quando:

- uma API depende de um banco;
- um worker depende de RabbitMQ;
- uma aplicação depende de Redis;
- uma stack possui vários serviços que precisam iniciar em determinada ordem;
- você quer visualizar o estado do serviço com `docker compose ps`.

## Healthcheck não substitui retry na aplicação

Mesmo usando `service_healthy`, a aplicação deve estar preparada para indisponibilidade temporária de suas dependências. Healthcheck organiza a inicialização da stack, mas não transforma uma dependência externa em uma garantia permanente de disponibilidade.

## Verificando o estado

```bash
docker compose ps
```

Estados comuns incluem:

```text
healthy
unhealthy
starting
```

Para investigar um serviço:

```bash
docker inspect mysql
```

## Regra prática

```text
Preciso que A dependa de B?
          │
          ▼
      B possui
     healthcheck?
       │       │
      não     sim
       │       │
       ▼       ▼
Avaliar retry  usar
na aplicação   service_healthy
```

O objetivo não é adicionar healthchecks indiscriminadamente, mas tornar explícito o estado real das dependências.
