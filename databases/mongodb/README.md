# MongoDB

MongoDB é um banco de dados NoSQL orientado a documentos. Em vez de organizar os dados principalmente em tabelas e linhas, ele trabalha com documentos BSON agrupados em collections.

## Quando usar

MongoDB pode ser uma boa escolha quando:

- o modelo de dados é naturalmente orientado a documentos;
- a estrutura dos documentos muda com frequência;
- dados relacionados podem ser mantidos juntos em um documento;
- o domínio não depende fortemente de joins relacionais;
- você se beneficia do modelo flexível de documentos.

MongoDB não é simplesmente "um banco sem schema". A aplicação ainda precisa de uma boa modelagem, validações, índices e regras de consistência.

## MongoDB x MySQL x PostgreSQL

A escolha deve partir do modelo e dos requisitos, não da popularidade da tecnologia.

| Característica | MySQL | PostgreSQL | MongoDB |
|---|---|---|---|
| Modelo | Relacional | Relacional | Documentos |
| Linguagem principal | SQL | SQL | Query API / documentos |
| Relacionamentos | Forte | Forte | Possíveis, mas modelados de outra forma |
| Schema | Estruturado | Estruturado | Flexível |
| Joins | Sim | Sim | Possíveis, mas não são o centro do modelo |
| Dados aninhados | Limitado ao modelo relacional | Recursos específicos | Natural |
| Consultas relacionais complexas | Boa opção | Excelente opção em muitos cenários | Geralmente não é o motivo principal para escolhê-lo |

### Regra prática

```text
O domínio é fortemente relacional?
        │
   ┌────┴────┐
   │         │
  sim       não
   │         │
   ▼         ▼
MySQL/PG   O dado é naturalmente um documento?
                │
           ┌────┴────┐
           │         │
          sim       não
           │         │
           ▼         ▼
       MongoDB   avalie outros modelos
```

## Exemplo de modelagem

Imagine um pedido com seus itens.

Em um modelo relacional, normalmente teremos entidades relacionadas:

```text
orders
   │
   └── order_items
```

No MongoDB, dependendo do acesso necessário, os itens podem fazer parte do próprio documento:

```json
{
  "customerId": "123",
  "status": "PAID",
  "items": [
    {
      "productId": "p1",
      "quantity": 2,
      "price": 49.90
    }
  ]
}
```

Isso pode simplificar leituras que sempre precisam do pedido completo, mas não significa que toda relação deve ser embutida.

## Estrutura

```text
mongodb/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Configuração

```yaml
services:
  mongodb:
    image: mongo:8
    container_name: mongodb
    restart: unless-stopped

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_DATABASE}

    ports:
      - "${MONGO_PORT}:27017"

    volumes:
      - mongodb_data:/data/db

    healthcheck:
      test: ["CMD-SHELL", "mongosh --quiet --eval \"db.adminCommand({ ping: 1 }).ok\" | grep 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s

volumes:
  mongodb_data:
```

### `image`

A versão da imagem é definida explicitamente para evitar mudanças inesperadas causadas por `latest`.

### `environment`

As variáveis `MONGO_INITDB_ROOT_USERNAME` e `MONGO_INITDB_ROOT_PASSWORD` configuram o usuário administrativo inicial.

`MONGO_INITDB_DATABASE` define o banco usado no processo de inicialização da imagem. Ele não significa que toda conexão futura ficará limitada a esse banco.

### `ports`

A porta padrão do MongoDB dentro do container é `27017`.

Do host:

```text
localhost:27017
```

De outro container na mesma rede:

```text
mongodb:27017
```

### `volumes`

O volume mantém os dados fora do ciclo de vida do container.

```yaml
volumes:
  mongodb_data:
```

E é montado em:

```text
/data/db
```

### `healthcheck`

O template usa `mongosh` para executar um `ping` administrativo no servidor MongoDB.

Isso permite usar o serviço em uma stack dependente:

```yaml
depends_on:
  mongodb:
    condition: service_healthy
```

O comando do healthcheck precisa ser compatível com a versão da imagem utilizada. Se você trocar a imagem por uma variante que não contenha `mongosh`, ajuste o teste.

## `.env.example`

```env
MONGO_ROOT_USERNAME=root
MONGO_ROOT_PASSWORD=root_password
MONGO_DATABASE=app_db
MONGO_PORT=27017
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
docker compose logs -f mongodb
```

## Parando

```bash
docker compose down
```

Os dados continuam no volume.

Para remover o volume e os dados:

```bash
docker compose down -v
```

> Isso é destrutivo para o ambiente local.

## Conexão

### MongoDB Compass ou aplicação no host

```text
mongodb://root:root_password@localhost:27017/?authSource=admin
```

### Aplicação em outro container

```text
mongodb://root:root_password@mongodb:27017/?authSource=admin
```

O `authSource=admin` é importante neste template porque o usuário root é criado no banco de autenticação `admin`.

## Spring Boot

Com Spring Data MongoDB, uma configuração no host pode ser:

```properties
spring.data.mongodb.uri=mongodb://root:root_password@localhost:27017/app_db?authSource=admin
```

Dentro de uma stack Docker:

```properties
spring.data.mongodb.uri=mongodb://root:root_password@mongodb:27017/app_db?authSource=admin
```

Mais uma vez: dentro do container, `localhost` aponta para o próprio container da aplicação.

## Cenário: MongoDB local + Spring Boot no IntelliJ

```text
Spring Boot
    │
    │ localhost:27017
    ▼
 MongoDB
(container)
```

## Cenário: aplicação + MongoDB em Docker

```text
┌───────────────────────────────┐
│ Docker network                │
│                               │
│  ┌──────────────┐             │
│  │ Spring Boot  │             │
│  └──────┬───────┘             │
│         │ mongodb:27017       │
│         ▼                     │
│  ┌──────────────┐             │
│  │   MongoDB    │             │
│  └──────────────┘             │
└───────────────────────────────┘
```

## MongoDB não é automaticamente substituto de MySQL/PostgreSQL

O erro comum é pensar:

> "Se MongoDB é mais flexível, então é melhor para qualquer aplicação."

Flexibilidade não elimina decisões de modelagem.

Se o domínio possui muitos relacionamentos, transações complexas e consultas fortemente relacionais, um banco relacional pode representar melhor o problema.

Se o domínio é naturalmente orientado a documentos e os padrões de acesso combinam com esse modelo, MongoDB pode simplificar a solução.

## Índices

Mesmo com schema flexível, índices continuam sendo fundamentais.

Exemplo conceitual:

```javascript
db.users.createIndex({ email: 1 }, { unique: true })
```

Não crie índices indiscriminadamente. Eles melhoram determinadas leituras, mas também têm custo de armazenamento e escrita.

## Problemas comuns

### `Authentication failed`

Verifique:

- usuário;
- senha;
- `authSource=admin`;
- se o volume já existia quando as variáveis foram alteradas.

### Alterei a senha no `.env`, mas continua usando a antiga

As variáveis de inicialização são aplicadas durante a criação inicial do banco. Se o volume já contém dados, alterar o `.env` não recria automaticamente o usuário.

Em ambiente descartável:

```bash
docker compose down -v
docker compose up -d
```

Isso apaga os dados.

### `connection refused`

Confira:

```bash
docker compose ps
```

E:

```bash
docker compose logs mongodb
```

### Aplicação em Docker usando `localhost`

Use:

```text
mongodb:27017
```

em vez de:

```text
localhost:27017
```

## Produção

Este template é voltado para desenvolvimento e estudo. Para produção, avalie autenticação adequada, secrets, rede, TLS quando necessário, backups, monitoramento, limites de recursos, política de atualização e arquitetura de replicação/alta disponibilidade.

## Resumo

| Situação | Hostname | Porta |
|---|---|---:|
| Aplicação no host | `localhost` | `27017` |
| Container na mesma rede | `mongodb` | `27017` |
| Cliente externo | endereço do host | porta publicada |

A principal diferença conceitual em relação a MySQL e PostgreSQL não é apenas a tecnologia usada no Docker: é o **modelo de dados** que você está escolhendo para sua aplicação.
