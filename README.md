# Docker Templates

Templates reutilizáveis de Docker Compose para serviços que aparecem com frequência em projetos de desenvolvimento.

A ideia deste repositório não é apenas guardar arquivos `docker-compose.yml`. Ele funciona como um **cookbook de infraestrutura**: cada serviço possui templates prontos, explicações sobre seu funcionamento e configurações para diferentes situações.

## Objetivos

- Ter configurações Docker Compose reutilizáveis.
- Entender o papel de cada serviço antes de utilizá-lo.
- Documentar configurações comuns e suas alternativas.
- Registrar diferenças entre tecnologias semelhantes.
- Facilitar a criação de ambientes locais e de desenvolvimento.
- Servir como material de consulta para projetos futuros.

## Estrutura

```text
docker-templates/
├── databases/
│   ├── mysql/
│   ├── postgresql/
│   └── mongodb/
│
├── infrastructure/
│   ├── redis/
│   ├── rabbitmq/
│   └── traefik/
│
├── observability/
│   ├── prometheus/
│   └── grafana/
│
├── stacks/
│   ├── spring-mysql/
│   ├── spring-postgresql/
│   ├── spring-mongodb/
│   └── spring-traefik/
│
└── patterns/
    ├── environment-variables/
    ├── volumes/
    ├── networks/
    ├── healthchecks/
    └── depends-on/
```

## Serviços

| Serviço | Categoria | Uso principal |
|---|---|---|
| MySQL | Database | Banco de dados relacional |
| PostgreSQL | Database | Banco relacional com recursos avançados |
| MongoDB | Database | Banco orientado a documentos |
| Redis | Infrastructure | Cache e armazenamento em memória |
| RabbitMQ | Infrastructure | Mensageria assíncrona |
| Traefik | Infrastructure | Reverse proxy e roteamento |
| Prometheus | Observability | Coleta de métricas |
| Grafana | Observability | Dashboards e visualização |

## Como utilizar

Cada serviço deve poder ser executado de forma independente.

Exemplo:

```bash
cd databases/mysql
docker compose up -d
```

Para remover os containers:

```bash
docker compose down
```

Para remover também os volumes persistentes:

```bash
docker compose down -v
```

> **Atenção:** `docker compose down -v` pode apagar dados persistidos pelo serviço. Use com cuidado.

## Padrão dos templates

Sempre que possível, os templates seguem estes princípios:

- Credenciais configuradas por variáveis de ambiente.
- Arquivo `.env.example` documentando as variáveis necessárias.
- Persistência através de Docker volumes quando aplicável.
- Healthchecks quando fizerem sentido.
- `restart: unless-stopped` para serviços locais que devem permanecer disponíveis.
- Nomes de serviços consistentes para facilitar conexões entre containers.
- Documentação do motivo de cada configuração relevante.
- Separação entre configuração básica, desenvolvimento e cenários mais completos.

## Host x Container

Uma regra importante deste repositório é diferenciar conexões feitas **a partir da máquina host** das conexões feitas **entre containers**.

Por exemplo, se o PostgreSQL estiver exposto assim:

```yaml
ports:
  - "5432:5432"
```

Uma aplicação executando diretamente na máquina pode utilizar:

```text
localhost:5432
```

Já uma aplicação executando em outro container da mesma rede deve utilizar o nome do serviço:

```text
postgresql:5432
```

Isso será documentado individualmente em cada template.

## Desenvolvimento x Produção

Os templates deste repositório são principalmente voltados para **desenvolvimento local, testes e ambientes de estudo**.

Uma configuração que funciona muito bem localmente não deve ser automaticamente considerada adequada para produção.

Em ambientes de produção, devem ser avaliados, entre outros pontos:

- gerenciamento de secrets;
- TLS/HTTPS;
- backups;
- políticas de acesso;
- redes e exposição de portas;
- recursos de CPU e memória;
- observabilidade;
- alta disponibilidade;
- atualização de imagens;
- estratégia de persistência;
- segurança das credenciais.

## Comparações

Uma parte importante do projeto é explicar **quando escolher uma tecnologia em vez de outra**.

Exemplos:

- MySQL x PostgreSQL
- PostgreSQL x MongoDB
- Redis x banco de dados tradicional
- RabbitMQ x comunicação síncrona
- Traefik x Nginx

A intenção não é definir uma tecnologia universalmente melhor, mas relacionar cada escolha ao problema que ela resolve.

## Roadmap

### Databases

- [ ] MySQL
- [ ] PostgreSQL
- [ ] MongoDB

### Infrastructure

- [ ] Redis
- [ ] RabbitMQ
- [ ] Traefik
- [ ] MinIO
- [ ] Kafka

### Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] Loki
- [ ] Jaeger

### Stacks

- [ ] Spring Boot + MySQL
- [ ] Spring Boot + PostgreSQL
- [ ] Spring Boot + MongoDB
- [ ] Spring Boot + Redis
- [ ] Spring Boot + RabbitMQ
- [ ] Angular + Spring Boot + Traefik

### Docker Compose Patterns

- [ ] Environment variables
- [ ] Volumes
- [ ] Networks
- [ ] Healthchecks
- [ ] depends_on
- [ ] Secrets
- [ ] Multi-container applications

## Filosofia

> Não basta saber copiar um `docker-compose.yml`. O objetivo é entender o que cada configuração faz, quando ela é necessária e quais consequências ela traz.
