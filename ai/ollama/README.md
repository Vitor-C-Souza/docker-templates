# Ollama

Ollama permite executar modelos de linguagem localmente e expõe uma API HTTP para que outras aplicações possam consumir esses modelos.

É especialmente útil para desenvolvimento local, prototipagem de aplicações com IA, testes de integração e cenários em que você quer evitar depender de uma API de LLM externa durante o desenvolvimento.

## Quando usar

Ollama faz sentido quando você precisa:

- executar LLMs localmente;
- testar uma integração com IA sem depender de um provedor externo;
- desenvolver um backend que consome uma API compatível com o Ollama;
- experimentar modelos diferentes;
- manter prompts e inferência dentro do ambiente local.

O desempenho depende bastante do hardware disponível. Modelos maiores podem exigir bastante RAM/VRAM.

## Arquitetura

```text
┌──────────────────────────────┐
│ Aplicação                    │
│ Spring Boot / Python / etc.  │
└──────────────┬───────────────┘
               │ HTTP
               ▼
        ┌─────────────┐
        │   Ollama    │
        │   :11434    │
        └──────┬──────┘
               │
               ▼
          LLM local
```

## Estrutura

```text
ollama/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Template básico

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped

    ports:
      - "${OLLAMA_PORT}:11434"

    volumes:
      - ollama_data:/root/.ollama

    healthcheck:
      test: ["CMD-SHELL", "ollama list >/dev/null 2>&1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

volumes:
  ollama_data:
```

### Porta

Ollama usa a porta `11434` por padrão.

Do host:

```text
http://localhost:11434
```

De outro container na mesma rede:

```text
http://ollama:11434
```

### Volume

Os modelos baixados precisam persistir entre recriações do container.

Por isso usamos:

```yaml
volumes:
  - ollama_data:/root/.ollama
```

Sem esse volume, você pode acabar baixando os modelos novamente quando o container for recriado.

## `.env.example`

```env
OLLAMA_PORT=11434
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
docker compose logs -f ollama
```

## Baixando um modelo

Depois que o container estiver funcionando:

```bash
docker exec -it ollama ollama pull llama3.2
```

Liste os modelos instalados:

```bash
docker exec -it ollama ollama list
```

Você pode substituir `llama3.2` por outro modelo compatível com o hardware disponível.

## Testando pelo terminal

Entre no container:

```bash
docker exec -it ollama ollama run llama3.2
```

Ou consuma a API HTTP a partir da aplicação.

## Spring Boot

Em uma aplicação Spring Boot executando no host:

```text
http://localhost:11434
```

Se o Spring Boot estiver em outro container na mesma rede:

```text
http://ollama:11434
```

Exemplo de variável:

```env
OLLAMA_BASE_URL=http://ollama:11434
```

## Exemplo conceitual de integração

```text
Spring Boot
    │
    │ POST /api/generate
    ▼
Ollama
    │
    ▼
Modelo local
    │
    ▼
Resposta
```

O backend pode esconder os detalhes do Ollama atrás de uma interface própria, por exemplo:

```text
POST /api/chat
        │
        ▼
   Spring Boot
        │
        ▼
      Ollama
```

Isso evita acoplar o restante da aplicação diretamente ao provedor de LLM.

## Ollama x API externa de LLM

| Ollama | API externa |
|---|---|
| Inferência local | Inferência remota |
| Pode funcionar sem internet após os modelos estarem disponíveis | Normalmente depende de rede |
| Você controla o ambiente | Provedor controla infraestrutura |
| Custo principal é hardware/energia | Custo normalmente depende de uso |
| Modelos limitados pelo hardware disponível | Pode acessar modelos maiores |
| Ótimo para desenvolvimento e experimentação local | Pode ser melhor para escala e modelos maiores |

Não existe um vencedor universal. Para produção, avalie custo, latência, privacidade, capacidade do hardware, qualidade do modelo e requisitos de escala.

## CPU x GPU

O container básico funciona sem uma configuração específica de GPU, mas a inferência pode ficar muito mais rápida com hardware adequado.

Para usar GPU, a configuração depende do hardware e do runtime disponível. Em máquinas NVIDIA, normalmente isso envolve o NVIDIA Container Toolkit e uma configuração adicional do Compose.

Não coloquei uma configuração de GPU obrigatória neste template porque ela tornaria o exemplo menos portátil.

## Healthcheck

O template usa:

```yaml
healthcheck:
  test: ["CMD-SHELL", "ollama list >/dev/null 2>&1"]
```

Isso verifica se o CLI do Ollama consegue consultar o serviço.

Uma aplicação dependente pode usar:

```yaml
depends_on:
  ollama:
    condition: service_healthy
```

Assim como nos outros templates, isso não substitui tratamento de erro e retry na aplicação.

## Cenário: Spring Boot + Ollama no Docker

```text
┌─────────────────────────────────────┐
│ Docker network                      │
│                                     │
│ ┌──────────────┐     ┌───────────┐  │
│ │ Spring Boot  │────►│  Ollama   │  │
│ │              │     │  :11434   │  │
│ └──────────────┘     └───────────┘  │
└─────────────────────────────────────┘
```

A aplicação usa:

```text
http://ollama:11434
```

e não:

```text
http://localhost:11434
```

porque `localhost` dentro do container da aplicação aponta para o próprio container.

## Cenário: Spring Boot no IntelliJ + Ollama no Docker

```text
Spring Boot / IntelliJ
        │
        │ localhost:11434
        ▼
      Ollama
    (container)
```

Esse é provavelmente um dos cenários mais úteis para desenvolvimento local.

## Problemas comuns

### Modelo não encontrado

Liste os modelos:

```bash
docker exec ollama ollama list
```

E faça o pull caso necessário:

```bash
docker exec ollama ollama pull llama3.2
```

### Modelo desapareceu depois de recriar o container

Verifique se o volume está presente:

```bash
docker volume ls
```

E se o Compose possui:

```yaml
- ollama_data:/root/.ollama
```

### Aplicação em Docker não consegue conectar

Use:

```text
http://ollama:11434
```

em vez de:

```text
http://localhost:11434
```

### Respostas muito lentas

Avalie:

- tamanho do modelo;
- quantidade de contexto;
- RAM disponível;
- VRAM disponível;
- uso de CPU/GPU;
- concorrência;
- quantização do modelo.

## Segurança

O template é voltado para desenvolvimento local.

Se você publicar a API do Ollama além do ambiente local, trate isso como um serviço que precisa de controle de acesso e rede adequada. Não exponha uma instância local diretamente à internet sem entender as implicações.

## Regra prática

```text
Quero experimentar LLM localmente?
              │
              ▼
            Ollama

Preciso de um modelo enorme sem hardware local?
              │
              ▼
       Avalie uma API externa

Preciso desenvolver sem acoplar meu backend a um provedor?
              │
              ▼
   Coloque Ollama atrás de uma interface própria
```
