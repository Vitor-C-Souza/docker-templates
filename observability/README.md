# Observabilidade

Este diretório reúne templates para monitorar aplicações e infraestrutura localmente.

## Prometheus + Grafana

O par mais comum deste cookbook é:

```text
Aplicações / serviços
        │
        │ métricas
        ▼
   Prometheus
        │
        │ PromQL
        ▼
     Grafana
```

### Prometheus

Prometheus coleta e armazena métricas de séries temporais. Ele consulta endpoints de métricas dos serviços em intervalos definidos pela configuração de scrape.

### Grafana

Grafana é a camada de visualização. Ele consulta fontes de dados, como Prometheus, e transforma os dados em dashboards e gráficos.

Uma forma simples de lembrar:

```text
Prometheus = coleta + armazenamento + consulta de métricas
Grafana    = visualização + dashboards + exploração
```

## Métricas x logs x traces

Observabilidade não é apenas monitoramento.

```text
Métricas
  └── "Quantos erros aconteceram?"

Logs
  └── "O que aconteceu nesta execução?"

Traces
  └── "Por onde esta requisição passou?"
```

Prometheus é focado em **métricas**. Grafana pode centralizar a visualização de diferentes fontes, inclusive logs e traces quando outros componentes forem adicionados à stack.

## Estrutura

```text
observability/
├── README.md
├── prometheus/
│   ├── README.md
│   ├── docker-compose.yml
│   ├── prometheus.yml
│   └── .env.example
└── grafana/
    ├── README.md
    ├── docker-compose.yml
    └── .env.example
```

## Como usar

Os templates são independentes, mas para um ambiente local completo suba os dois.

Primeiro Prometheus:

```bash
cd prometheus
docker compose up -d
```

Depois Grafana:

```bash
cd ../grafana
docker compose up -d
```

Acesse:

```text
Prometheus: http://localhost:9090
Grafana:    http://localhost:3000
```

No Grafana, adicione o Prometheus como data source usando, quando ambos estiverem em uma mesma rede Docker:

```text
http://prometheus:9090
```

Se forem containers em Compose files separados, crie uma rede externa compartilhada e conecte os dois serviços a ela.

## Spring Boot

Uma aplicação Spring Boot pode expor métricas no formato que o Prometheus entende usando Actuator + Micrometer Prometheus.

Exemplo conceitual:

```text
Spring Boot
    │
    │ GET /actuator/prometheus
    ▼
Prometheus
    │
    │ PromQL
    ▼
Grafana
```

No `prometheus.yml` deste template há um exemplo comentado de scrape para uma aplicação Spring Boot.

Quando o Spring Boot roda diretamente no host e o Prometheus roda em Docker, o exemplo usa:

```text
host.docker.internal:8080
```

Quando ambos estão no Docker, prefira o nome do serviço na rede:

```text
api:8080
```

## Healthcheck

Prometheus usa o endpoint interno de healthcheck:

```text
/-/healthy
```

Grafana usa:

```text
/api/health
```

Os dois Compose files possuem healthchecks para facilitar a composição com outros serviços.

## Persistência

Prometheus mantém dados no volume:

```text
prometheus_data
```

Grafana mantém configurações, usuários e dashboards no volume:

```text
grafana_data
```

Remover os volumes apaga o estado persistido desses serviços.

## Por que não colocar tudo em um único Compose?

O cookbook mantém Prometheus e Grafana separados porque você pode querer:

- usar somente Prometheus;
- usar um Grafana apontando para diferentes ambientes;
- trocar a fonte de métricas;
- combinar esses serviços com outras stacks;
- entender cada componente isoladamente.

Depois, nas `stacks/`, teremos uma composição pronta com a rede e as dependências configuradas.

## Próxima evolução

Uma stack de observabilidade mais completa pode evoluir para:

```text
                  ┌────────────┐
                  │  Grafana   │
                  └─────┬──────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
        Prometheus     Loki      Tempo
             ▲          ▲          ▲
             │          │          │
          metrics      logs      traces
```

Neste cookbook, começamos com Prometheus + Grafana para manter o primeiro passo simples e reutilizável.
