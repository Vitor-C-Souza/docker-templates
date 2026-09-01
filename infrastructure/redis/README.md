# Redis

Redis é um armazenamento de dados em memória usado principalmente para cenários em que baixa latência e acesso rápido são importantes.

Apesar de ser frequentemente chamado de "cache", Redis pode atender outros padrões, como armazenamento temporário, sessões, contadores e estruturas de dados em memória.

## Quando usar

Redis é uma boa opção para:

- cache de dados;
- sessões distribuídas;
- rate limiting;
- contadores;
- dados temporários com TTL;
- estruturas como listas, conjuntos e hashes;
- reduzir consultas repetitivas ao banco principal.

## Redis não é simplesmente "outro banco"

Um erro comum é colocar Redis no mesmo grupo mental de MySQL, PostgreSQL e MongoDB.

Eles podem até armazenar dados, mas normalmente resolvem problemas diferentes.

```text
MySQL / PostgreSQL
        │
        └── fonte de dados relacional

MongoDB
        │
        └── fonte de dados orientada a documentos

Redis
        │
        ├── cache
        ├── dados temporários
        ├── sessões
        └── estruturas em memória
```

Redis pode ter persistência, mas isso não significa que você deva tratá-lo automaticamente como substituto do banco principal da aplicação.

## Redis x banco principal

Um padrão bastante comum é:

```text
             ┌──────────────┐
             │ Spring Boot  │
             └──────┬───────┘
                    │
              procura no cache
                    │
              ┌─────▼─────┐
              │   Redis   │
              └─────┬─────┘
                    │ cache miss
                    ▼
              ┌─────────────┐
              │ PostgreSQL  │
              └─────────────┘
```

A aplicação primeiro consulta Redis. Se o dado não estiver disponível, consulta o banco principal e pode armazenar o resultado no cache.

## Estrutura

```text
redis/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Configuração

```yaml
services:
  redis:
    image: redis:8
    container_name: redis
    restart: unless-stopped

    command: redis-server --appendonly yes

    ports:
      - "${REDIS_PORT}:6379"

    volumes:
      - redis_data:/data

    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s

volumes:
  redis_data:
```

### `image`

A versão é definida explicitamente para tornar o ambiente previsível.

### `command`

```yaml
command: redis-server --appendonly yes
```

Ativa AOF (Append Only File), permitindo persistência das operações no disco.

Isso é útil para desenvolvimento quando queremos que os dados sobrevivam à recriação do container, mas **persistência não transforma Redis em um substituto automático de um banco relacional ou documental**.

Se o objetivo for um cache totalmente descartável, você pode optar por não usar persistência.

### `ports`

Redis usa a porta `6379` dentro do container.

Do host:

```text
localhost:6379
```

De outro container na mesma rede:

```text
redis:6379
```

### `volumes`

O volume é montado em `/data`, diretório utilizado pelo Redis para seus arquivos persistentes nesse cenário.

### `healthcheck`

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
```

O comando espera uma resposta `PONG` do Redis.

Uma aplicação dependente pode usar:

```yaml
depends_on:
  redis:
    condition: service_healthy
```

## `.env.example`

```env
REDIS_PORT=6379
```

Copie para `.env`:

```powershell
Copy-Item .env.example .env
```

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
docker compose logs -f redis
```

## Testando Redis

Entre no CLI:

```bash
docker exec -it redis redis-cli
```

Teste:

```text
PING
```

Resposta esperada:

```text
PONG
```

Também podemos testar diretamente:

```bash
docker exec redis redis-cli ping
```

## TTL

Uma das características mais úteis para cache é a expiração automática.

Exemplo:

```text
SET user:123 "Vitor" EX 300
```

O valor expira após 300 segundos.

Isso é particularmente útil para dados que não precisam permanecer no cache indefinidamente.

## Exemplo de cache

Imagine uma API que busca um produto no PostgreSQL.

Sem cache:

```text
Request
   │
   ▼
Spring Boot
   │
   ▼
PostgreSQL
   │
   ▼
Response
```

Com Redis:

```text
Request
   │
   ▼
Spring Boot
   │
   ▼
Redis ──► encontrou? ──► Response
   │
   │ não
   ▼
PostgreSQL
   │
   ▼
Redis (salva)
   │
   ▼
Response
```

## Spring Boot

Uma configuração básica pode ser:

```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
```

Se o Spring Boot estiver em outro container:

```properties
spring.data.redis.host=redis
spring.data.redis.port=6379
```

O hostname é o nome do serviço Docker, não `localhost`.

## Redis como cache

Quando usado como cache, considere:

- TTL;
- tamanho dos valores;
- política de eviction;
- estratégia para cache miss;
- invalidação após alterações;
- consistência entre cache e banco principal.

O famoso problema de cache é simples de explicar e chato de resolver:

```text
Banco atualizado
      │
      ▼
Cache ainda possui valor antigo
      │
      ▼
Aplicação retorna informação desatualizada
```

Por isso, adicionar Redis não é apenas instalar um container. É uma decisão de arquitetura.

## Redis x RabbitMQ

Os dois aparecem frequentemente em arquiteturas distribuídas, mas seus objetivos são diferentes.

| Redis | RabbitMQ |
|---|---|
| Cache e dados em memória | Broker de mensagens |
| Excelente para acesso rápido | Excelente para filas e entrega de mensagens |
| TTL é muito útil | Ack/retry e roteamento são centrais |
| Pode armazenar estruturas de dados | Foco em mensagens |
| Pode participar de padrões de comunicação | Projetado especificamente para mensageria |

Se a pergunta é:

> "Preciso guardar temporariamente este resultado e acessá-lo rapidamente?"

Redis pode fazer sentido.

Se a pergunta é:

> "Preciso enviar um trabalho para outro serviço processar de forma assíncrona?"

RabbitMQ costuma ser uma opção mais natural.

## Cenário: Redis somente para desenvolvimento local

```text
Spring Boot / IntelliJ
        │
        │ localhost:6379
        ▼
      Redis
   (container)
```

## Cenário: aplicação + Redis em Docker

```text
┌──────────────────────────────┐
│ Docker network               │
│                              │
│ ┌──────────────┐             │
│ │ Spring Boot  │             │
│ └──────┬───────┘             │
│        │ redis:6379          │
│        ▼                     │
│ ┌──────────────┐             │
│ │    Redis     │             │
│ └──────────────┘             │
└──────────────────────────────┘
```

## Cache descartável x Redis persistente

### Cache descartável

Pode ser adequado quando perder os dados do Redis não é um problema:

```text
Banco principal = fonte da verdade
Redis = cópia temporária
```

Nesse caso, persistência pode não ser necessária.

### Redis persistente

Pode fazer sentido quando o estado armazenado precisa sobreviver a reinicializações, desde que a arquitetura realmente dependa disso.

Neste template, AOF está habilitado para demonstrar esse cenário.

## Problemas comuns

### `connection refused`

Confira:

```bash
docker compose ps
```

E:

```bash
docker compose logs redis
```

### Aplicação em Docker usando `localhost`

Use:

```text
redis:6379
```

### Redis reiniciou e os dados desapareceram

Verifique se existe volume:

```bash
docker volume ls
```

E se o Compose possui:

```yaml
volumes:
  - redis_data:/data
```

Também confirme se o AOF está habilitado quando a persistência for desejada.

### Cache devolvendo dado antigo

Isso normalmente é um problema de estratégia de cache, não de Docker. Avalie TTL e invalidação do cache quando os dados principais forem alterados.

## Segurança

Este template é para desenvolvimento local. Redis não deve ser exposto publicamente sem uma configuração de segurança adequada.

Se a aplicação e Redis estiverem na mesma rede Docker, considere não publicar a porta no host quando não houver necessidade de acesso externo.

## Resumo

| Situação | Hostname | Porta |
|---|---|---:|
| Aplicação no host | `localhost` | `6379` |
| Container na mesma rede | `redis` | `6379` |
| Cliente externo | endereço do host | porta publicada |

### Regra prática

```text
Precisa de dados rápidos e temporários?
                │
                ▼
              Redis

Precisa da fonte principal dos dados?
                │
                ▼
       Banco apropriado ao domínio

Precisa processar mensagens de forma assíncrona?
                │
                ▼
       Avalie RabbitMQ / Kafka
```
