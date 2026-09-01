# Traefik

Traefik é um reverse proxy e edge router orientado a ambientes dinâmicos, como Docker e Kubernetes.

Ele pode observar os serviços disponíveis e criar regras de roteamento a partir de labels, evitando configurar manualmente um proxy para cada aplicação.

## Quando usar

Traefik é especialmente útil quando você tem:

- várias aplicações rodando em Docker;
- múltiplos serviços que precisam ser acessados por HTTP/HTTPS;
- necessidade de roteamento por domínio ou caminho;
- containers que entram e saem da stack com frequência;
- necessidade de centralizar TLS e entrada HTTP;
- um ambiente local que você quer aproximar de uma arquitetura real.

## O que Traefik é — e o que não é

### Reverse proxy

Recebe uma requisição e encaminha para um serviço interno.

```text
Cliente
   │
   ▼
Traefik
   │
   ▼
Backend
```

### Load balancer

Traefik também pode distribuir requisições entre múltiplas instâncias de um serviço.

```text
             ┌──► backend-1
Traefik ─────┼──► backend-2
             └──► backend-3
```

### API Gateway

Traefik pode assumir algumas funções normalmente associadas a API gateways, como roteamento, TLS e middleware. Porém, "reverse proxy", "load balancer" e "API gateway" são conceitos diferentes e não devem ser tratados como sinônimos.

## Arquitetura básica

```text
                       Internet / Browser
                              │
                              ▼
                       ┌─────────────┐
                       │   Traefik   │
                       └──────┬──────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
          frontend          api          admin-api
```

## Estrutura

```text
traefik/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Template básico

```yaml
services:
  traefik:
    image: traefik:v3.5
    container_name: traefik
    restart: unless-stopped

    command:
      - --api.dashboard=true
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80

    ports:
      - "${TRAEFIK_HTTP_PORT}:80"
      - "${TRAEFIK_DASHBOARD_PORT}:8080"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
```

## Configuração

### Docker provider

```yaml
- --providers.docker=true
```

Faz o Traefik usar o Docker como fonte de descoberta dos serviços.

### `exposedbydefault=false`

```yaml
- --providers.docker.exposedbydefault=false
```

Essa opção é importante para evitar que todos os containers sejam publicados automaticamente pelo Traefik.

Com ela habilitada, o serviço precisa declarar explicitamente:

```yaml
labels:
  - traefik.enable=true
```

Isso torna a exposição de serviços uma decisão explícita.

### EntryPoint

```yaml
- --entrypoints.web.address=:80
```

Cria o ponto de entrada HTTP na porta 80 do container.

### Dashboard

```yaml
- --api.dashboard=true
- --api.insecure=true
```

O dashboard é habilitado para desenvolvimento local.

> `api.insecure=true` não deve ser usado como configuração de produção. Ele expõe a API/dashboard de maneira simplificada e é adequado apenas para laboratório local.

### Docker socket

```yaml
- /var/run/docker.sock:/var/run/docker.sock:ro
```

O Traefik precisa observar o Docker para descobrir containers e suas labels.

O socket é montado como somente leitura (`ro`) neste template.

Mesmo assim, o Docker socket é um recurso sensível. Em ambientes mais rigorosos, considere alternativas de acesso e isolamento apropriadas.

## `.env.example`

```env
TRAEFIK_HTTP_PORT=80
TRAEFIK_DASHBOARD_PORT=8080
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
docker compose logs -f traefik
```

## Dashboard

Com a configuração local deste template:

```text
http://localhost:8080/dashboard/
```

O dashboard permite visualizar routers, services, middlewares e outros elementos descobertos pelo Traefik.

## Roteamento com labels

O ponto forte do provider Docker é declarar o roteamento junto do serviço.

Exemplo:

```yaml
services:
  api:
    image: nginx:alpine

    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`api.localhost`)
      - traefik.http.services.api.loadbalancer.server.port=80
```

O fluxo fica:

```text
http://api.localhost
        │
        ▼
     Traefik
        │
        ▼
   container api
        │
        ▼
       :80
```

## Por que informar a porta do serviço?

A porta publicada pelo Docker e a porta interna do container são conceitos diferentes.

Por exemplo:

```yaml
ports:
  - "8081:8080"
```

significa:

```text
host:8081 → container:8080
```

Mas o Traefik, estando na rede Docker, normalmente acessa o serviço pela porta interna:

```yaml
- traefik.http.services.api.loadbalancer.server.port=8080
```

Nesse cenário, você pode nem precisar publicar a porta da API no host.

## Padrão recomendado com Traefik

Em uma stack onde somente o Traefik deve ser a porta de entrada:

```yaml
services:
  traefik:
    # ...

  api:
    image: minha-api:latest
    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`api.localhost`)
      - traefik.http.services.api.loadbalancer.server.port=8080
```

A API não precisa necessariamente ter:

```yaml
ports:
  - "8080:8080"
```

O Traefik pode acessá-la diretamente pela rede Docker.

Isso reduz a quantidade de portas expostas no host.

## Hostname x Path

### Roteamento por hostname

```yaml
- traefik.http.routers.api.rule=Host(`api.localhost`)
```

Exemplos:

```text
api.localhost
admin.localhost
app.localhost
```

### Roteamento por path

```yaml
- traefik.http.routers.api.rule=PathPrefix(`/api`)
```

Exemplo:

```text
localhost/api/users
```

A escolha depende da arquitetura e de como os serviços serão expostos.

## Cenário: duas APIs

```text
                    Traefik
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
   api.localhost             admin.localhost
          │                         │
          ▼                         ▼
       api:8080                 admin:8080
```

Exemplo:

```yaml
services:
  api:
    image: minha-api:latest
    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`api.localhost`)
      - traefik.http.services.api.loadbalancer.server.port=8080

  admin:
    image: meu-admin:latest
    labels:
      - traefik.enable=true
      - traefik.http.routers.admin.rule=Host(`admin.localhost`)
      - traefik.http.services.admin.loadbalancer.server.port=8080
```

## Rede Docker compartilhada

Quando o Traefik e as aplicações estão em Compose files diferentes, eles precisam compartilhar uma rede Docker.

Crie uma rede:

```bash
docker network create traefik
```

No Traefik:

```yaml
networks:
  traefik:
    external: true
```

E no serviço:

```yaml
networks:
  traefik:
    external: true
```

Assim:

```text
┌────────────────────────────────────┐
│ Docker network: traefik            │
│                                    │
│ Traefik ───────► API               │
│     │                              │
│     └──────────► Admin              │
└────────────────────────────────────┘
```

## Traefik + Spring Boot

Uma API Spring Boot pode ficar atrás do Traefik sem publicar diretamente sua porta para o host.

Exemplo:

```yaml
services:
  api:
    image: minha-api:latest
    expose:
      - "8080"

    labels:
      - traefik.enable=true
      - traefik.http.routers.api.rule=Host(`api.localhost`)
      - traefik.http.services.api.loadbalancer.server.port=8080
```

A aplicação continua escutando em `8080` dentro do container.

O acesso externo passa pelo Traefik.

## Healthcheck

O template possui:

```yaml
healthcheck:
  test: ["CMD", "traefik", "healthcheck", "--ping"]
```

Isso permite acompanhar o estado do container:

```bash
docker compose ps
```

Mas existe uma diferença importante: o healthcheck do Traefik verifica o próprio Traefik. Ele não significa que todas as aplicações atrás dele estão saudáveis.

Cada aplicação deve possuir seu próprio healthcheck quando isso for relevante.

## HTTPS e Let's Encrypt

Em produção, uma configuração comum é:

```text
                     Internet
                        │
                     HTTPS
                        │
                        ▼
                  ┌──────────┐
                  │ Traefik  │
                  └────┬─────┘
                       │
                       ▼
                     API
```

O Traefik pode terminar TLS e trabalhar com certificados obtidos por ACME/Let's Encrypt.

Esse cenário exige configuração adicional de:

- entrypoint HTTPS;
- certificados;
- resolver ACME;
- armazenamento do estado ACME;
- domínio público;
- portas 80/443;
- DNS corretamente configurado.

O template básico deste diretório deliberadamente não habilita HTTPS automático, porque isso depende do ambiente onde será usado.

## Desenvolvimento local x produção

### Local

```text
localhost
   │
   ▼
Traefik :80
   │
   ├── api.localhost
   └── admin.localhost
```

Pode ser suficiente usar o dashboard inseguro e HTTP.

### Produção

```text
Internet
   │
   ▼
HTTPS :443
   │
   ▼
Traefik
   │
   ├── api.example.com
   └── admin.example.com
```

Em produção, avalie TLS, autenticação do dashboard, secrets, redes, firewall e exposição mínima de portas.

## Problemas comuns

### `404 page not found`

O Traefik pode estar funcionando, mas nenhuma regra correspondeu à requisição.

Verifique:

- hostname usado;
- label `traefik.enable=true`;
- regra do router;
- rede compartilhada;
- porta interna configurada no service.

### `502 Bad Gateway`

O Traefik encontrou a rota, mas não conseguiu falar corretamente com o backend.

Confira:

```bash
docker compose logs traefik
```

E valide a porta interna do serviço.

### A API funciona diretamente, mas não pelo Traefik

Se a API funciona em:

```text
localhost:8080
```

mas não em:

```text
api.localhost
```

verifique se o Traefik consegue acessar a API pela rede Docker e se a label aponta para a porta interna correta.

### Usei `localhost` na regra e não funcionou

A regra:

```yaml
Host(`api.localhost`)
```

exige que a requisição tenha esse hostname.

Acesse:

```text
http://api.localhost
```

e não apenas:

```text
http://localhost
```

## Segurança

Este template é principalmente para desenvolvimento.

Para produção, não use `api.insecure=true` para expor o dashboard. Também trate o Docker socket como uma interface sensível e evite expor serviços desnecessariamente.

Avalie ainda:

- HTTPS;
- autenticação;
- secrets;
- redes privadas;
- firewall;
- headers de segurança;
- rate limiting;
- logs e observabilidade;
- exposição mínima de portas.

## Resumo

```text
              Cliente
                 │
                 ▼
             Traefik
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
      API      Admin    Frontend
```

**Reverse proxy:** recebe e encaminha requisições.

**Load balancer:** pode distribuir requisições entre múltiplas instâncias.

**API Gateway:** pode agregar políticas e funcionalidades de entrada de APIs, mas é um conceito mais amplo.

Traefik pode participar de todos esses cenários, mas a função exata depende de como ele é configurado.
