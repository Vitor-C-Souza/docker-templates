# Stacks

Composições prontas para cenários comuns de desenvolvimento backend.

Os templates em outras pastas mostram serviços isoladamente. Aqui combinamos serviços para representar ambientes reais.

## Stacks disponíveis

- `spring-postgresql` — API + PostgreSQL
- `spring-mysql` — API + MySQL
- `spring-postgresql-redis` — API + PostgreSQL + Redis
- `spring-postgresql-rabbitmq` — API + PostgreSQL + RabbitMQ
- `spring-postgresql-redis-rabbitmq` — API + banco + cache + mensageria
- `spring-traefik` — Traefik + API
- `spring-observability` — API + Prometheus + Grafana
- `spring-ai-ollama` — API + Ollama
- `whatsapp-evolution` — Evolution Go + PostgreSQL

## Como usar

Cada diretório possui seu próprio `docker-compose.yml` e `.env.example` quando necessário.

Substitua a imagem `example/spring-api:latest` pela imagem da sua aplicação ou adapte o serviço para usar `build`.

A ideia é copiar uma stack, ajustar o mínimo possível e começar a desenvolver.

## Padrões

As stacks priorizam:

- healthchecks para dependências;
- volumes para dados que precisam persistir;
- comunicação entre containers por nome de serviço;
- exposição mínima de portas;
- variáveis de ambiente;
- separação entre desenvolvimento e produção.

## Escolha rápida

```text
Precisa só de persistência?
  ├─ PostgreSQL → spring-postgresql
  └─ MySQL      → spring-mysql

Precisa cache?
  └─ PostgreSQL + Redis

Precisa processamento assíncrono?
  └─ PostgreSQL + RabbitMQ

Precisa cache + mensageria?
  └─ PostgreSQL + Redis + RabbitMQ

Precisa entrada HTTP centralizada?
  └─ Traefik

Precisa métricas e dashboards?
  └─ Prometheus + Grafana

Precisa IA local?
  └─ Ollama

Precisa integração WhatsApp?
  └─ Evolution Go + PostgreSQL
```

Estas stacks são pontos de partida, não configurações universais de produção.
