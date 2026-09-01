# Grafana

Grafana é uma plataforma de visualização e exploração de dados de observabilidade.

Neste cookbook, ele será usado principalmente para visualizar métricas coletadas pelo Prometheus.

## Fluxo

```text
Aplicação
   │
   │ métricas
   ▼
Prometheus
   │
   │ consulta
   ▼
Grafana
   │
   ▼
Dashboard
```

## Estrutura

```text
grafana/
├── README.md
├── docker-compose.yml
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
http://localhost:3000
```

## Data source

No Grafana, configure o Prometheus como data source.

Se ambos estiverem em uma mesma rede Docker:

```text
http://prometheus:9090
```

Se estiver usando Prometheus exposto no host, você pode configurar o endereço de acordo com a topologia do ambiente. Em uma stack composta, prefira comunicação pela rede Docker em vez de depender de portas publicadas no host.

## Por que Grafana não coleta as métricas?

Porque essa não é a responsabilidade principal dele neste cenário.

```text
Prometheus
   │
   └── coleta métricas

Grafana
   │
   └── consulta e visualiza métricas
```

Grafana consulta o Prometheus e apresenta os resultados.

## Dashboards

Um dashboard pode acompanhar, por exemplo:

- CPU;
- memória;
- quantidade de requisições;
- latência;
- erros HTTP;
- throughput;
- disponibilidade dos serviços.

As métricas disponíveis dependem do que foi instrumentado na aplicação e dos exporters utilizados.

## Spring Boot

Uma API Spring Boot instrumentada com Micrometer pode expor métricas para o Prometheus.

O Grafana fica uma camada acima:

```text
Spring Boot
     │
     ▼
Prometheus
     │
     ▼
 Grafana
```

Um dashboard pode então transformar séries como contagem de requests e latência em gráficos e indicadores.

## Healthcheck

O template utiliza:

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget --spider -q http://localhost:3000/api/health || exit 1"]
```

Esse endpoint verifica a saúde da instância do Grafana.

## Persistência

O volume:

```yaml
grafana_data:/var/lib/grafana
```

mantém o estado do Grafana, incluindo configurações e dados persistidos da aplicação.

## Grafana + Prometheus em Compose files separados

Para manter os serviços independentes, eles podem continuar em diretórios diferentes e compartilhar uma rede Docker externa.

Crie:

```bash
docker network create observability
```

Conecte os dois serviços à rede:

```yaml
networks:
  observability:
    external: true
```

Depois, no Grafana, o Prometheus poderá ser referenciado pelo nome do serviço na rede.

## Observabilidade de uma API

Um cenário simples:

```text
                   ┌──────────────┐
                   │    Grafana   │
                   └──────┬───────┘
                          │
                          ▼
                   ┌──────────────┐
                   │  Prometheus  │
                   └──────┬───────┘
                          │ scrape
                          ▼
                   ┌──────────────┐
                   │ Spring Boot  │
                   │ /actuator/   │
                   │ prometheus   │
                   └──────────────┘
```

## Métricas, logs e traces

Grafana pode ser usado para explorar diferentes fontes, mas elas representam sinais diferentes.

```text
Metrics → comportamento numérico ao longo do tempo
Logs    → eventos e detalhes de execução
Traces  → caminho de uma requisição distribuída
```

Uma evolução possível deste cookbook é adicionar Loki para logs e Tempo para traces.

## Problemas comuns

### Grafana não conecta no Prometheus

Confira:

- se o Prometheus está rodando;
- se ambos compartilham uma rede Docker;
- hostname e porta;
- se você está usando `prometheus:9090` quando os containers compartilham a rede.

### Dashboard sem dados

Isso geralmente significa que o Prometheus não está coletando as métricas esperadas. Primeiro verifique os targets e as queries no Prometheus.

## Segurança

Este template é destinado ao desenvolvimento local.

Para produção, configure autenticação, HTTPS, gestão de secrets, acesso de rede adequado e políticas de autorização. Não exponha a interface administrativa publicamente sem proteção.
