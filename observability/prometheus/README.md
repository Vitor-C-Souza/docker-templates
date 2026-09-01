# Prometheus

Prometheus é um sistema de monitoramento e observabilidade baseado em métricas de séries temporais.

O modelo principal é simples: o Prometheus **faz scrape** de endpoints HTTP expostos pelas aplicações e armazena as métricas coletadas.

## Fluxo

```text
Aplicação
   │
   │ /metrics
   ▼
Prometheus
   │
   │ PromQL
   ▼
Consulta / Grafana
```

## Estrutura

```text
prometheus/
├── README.md
├── docker-compose.yml
├── prometheus.yml
└── .env.example
```

## Subir

```bash
docker compose up -d
```

Verifique:

```bash
docker compose ps
```

Acesse:

```text
http://localhost:9090
```

## Configuração

O arquivo `prometheus.yml` define como o Prometheus coleta métricas.

Exemplo:

```yaml
scrape_configs:
  - job_name: minha-api
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["api:8080"]
```

### `job_name`

Identifica o alvo no Prometheus.

### `metrics_path`

Define o endpoint onde as métricas estão disponíveis.

Para Spring Boot com Actuator + Micrometer, é comum usar:

```text
/actuator/prometheus
```

### `targets`

Define onde o Prometheus deve buscar as métricas.

Se a API estiver no mesmo Docker network:

```text
api:8080
```

Se a API estiver rodando diretamente no host enquanto Prometheus está em Docker:

```text
host.docker.internal:8080
```

## Pull model

Uma diferença importante é que Prometheus normalmente trabalha no modelo **pull**:

```text
Prometheus ──GET──► /metrics
```

A aplicação expõe métricas e o Prometheus decide quando buscá-las.

## PromQL

PromQL é a linguagem usada para consultar as séries armazenadas.

Exemplo:

```promql
up
```

Mostra o estado dos targets.

Outro exemplo:

```promql
process_cpu_usage
```

pode consultar uma métrica exposta por uma aplicação compatível.

As métricas exatas disponíveis dependem dos exporters e bibliotecas utilizadas.

## Healthcheck

O Compose usa:

```yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:9090/-/healthy"]
```

Isso verifica a saúde do próprio Prometheus.

Não significa que todos os serviços monitorados estejam saudáveis.

## Persistência

O volume:

```yaml
prometheus_data:/prometheus
```

mantém os dados do Prometheus entre recriações do container.

Para um ambiente local descartável, o volume pode ser removido.

## Spring Boot

Para instrumentar uma API Spring Boot, a aplicação normalmente usa Actuator e Micrometer com o registry do Prometheus.

O conceito é:

```text
Spring Boot
    │
    └── /actuator/prometheus
              ▲
              │ scrape
              │
         Prometheus
```

O endpoint deve ser exposto de maneira adequada ao ambiente. Evite disponibilizar endpoints de gerenciamento indiscriminadamente na internet.

## Desenvolvimento local

Se sua API roda pelo IntelliJ:

```text
IntelliJ
   │
   ▼
Spring Boot :8080
   ▲
   │ host.docker.internal:8080
   │
Prometheus :9090
```

Se a API também estiver no Docker:

```text
┌─────────────────────────────┐
│ Docker network              │
│                             │
│ Prometheus ───► api:8080    │
│                             │
└─────────────────────────────┘
```

## Problemas comuns

### Target `DOWN`

Confira:

- hostname;
- porta;
- endpoint de métricas;
- rede Docker;
- se a aplicação está realmente rodando;
- se o endpoint está acessível a partir do container do Prometheus.

### Prometheus não atualizou a configuração

Após alterações no `prometheus.yml`, reinicie o serviço:

```bash
docker compose restart prometheus
```

### Métrica não aparece

Verifique primeiro o endpoint da aplicação diretamente:

```text
http://localhost:8080/actuator/prometheus
```

Depois verifique o status dos targets no Prometheus.

## Produção

Este template é para desenvolvimento. Em produção, avalie retenção, armazenamento, segurança, autenticação, TLS, alta disponibilidade e estratégia de armazenamento adequada ao volume de métricas.
