# MySQL

MySQL é um sistema gerenciador de banco de dados relacional (RDBMS). Ele organiza os dados em tabelas relacionadas e utiliza SQL para consultas e manipulação.

Neste template, o MySQL é executado em um container Docker com persistência por volume e configuração externa por variáveis de ambiente.

## Quando usar

MySQL é uma boa escolha quando:

- o domínio da aplicação é naturalmente relacional;
- existem entidades relacionadas por chaves estrangeiras;
- a aplicação trabalha principalmente com CRUD e consultas SQL tradicionais;
- você quer uma tecnologia muito difundida no ecossistema web;
- simplicidade e familiaridade são prioridades.

## Quando considerar PostgreSQL

PostgreSQL também é relacional e pode atender aos mesmos cenários. Vale considerá-lo quando você precisa de recursos relacionais mais avançados, tipos de dados ricos, extensões ou consultas complexas.

Não existe uma regra universal de que PostgreSQL é melhor ou que MySQL é melhor. A escolha deve partir dos requisitos da aplicação.

## Estrutura

```text
mysql/
├── README.md
├── docker-compose.yml
└── .env.example
```

## Configuração

O template utiliza três arquivos/conceitos principais:

- `docker-compose.yml`: define o serviço MySQL;
- `.env.example`: documenta as variáveis necessárias;
- volume Docker: mantém os dados mesmo quando o container é recriado.

### docker-compose.yml

```yaml
services:
  mysql:
    image: mysql:8.4
    container_name: mysql
    restart: unless-stopped

    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}

    ports:
      - "${MYSQL_PORT}:3306"

    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

## Entendendo a configuração

### `image`

```yaml
image: mysql:8.4
```

Define a imagem utilizada pelo container. Fixar uma versão principal/minor conhecida evita depender implicitamente da tag `latest`.

### `container_name`

```yaml
container_name: mysql
```

Define um nome explícito para o container. Isso facilita comandos como:

```bash
docker logs mysql
docker exec -it mysql mysql -u root -p
```

Em stacks maiores, nomes de serviço e redes são normalmente mais importantes que `container_name`; por isso, ele não é obrigatório.

### `environment`

```yaml
environment:
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

Essas variáveis configuram o usuário root, o banco inicial e um usuário da aplicação.

Os valores não ficam diretamente no Compose. O Docker Compose lê as variáveis do arquivo `.env` ou do ambiente do processo.

### `ports`

```yaml
ports:
  - "${MYSQL_PORT}:3306"
```

A porta à esquerda pertence à máquina host. `3306` é a porta usada pelo MySQL dentro do container.

Se `.env` tiver:

```env
MYSQL_PORT=3306
```

então a conexão a partir da máquina host será:

```text
localhost:3306
```

### `volumes`

```yaml
volumes:
  - mysql_data:/var/lib/mysql
```

O diretório de dados do MySQL dentro do container é associado a um volume Docker.

Sem o volume, remover/recriar o container pode fazer você perder os dados armazenados nele.

O volume também permite que o container seja recriado sem recriar o banco do zero.

## `.env.example`

```env
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=app_db
MYSQL_USER=app
MYSQL_PASSWORD=app_password
MYSQL_PORT=3306
```

Copie para `.env`:

```powershell
Copy-Item .env.example .env
```

No Git, o `.env` real não deve ser versionado quando contém credenciais reais. O `.env.example` serve apenas como referência.

## Subindo o MySQL

Dentro desta pasta:

```bash
docker compose up -d
```

Verifique o container:

```bash
docker compose ps
```

Veja os logs:

```bash
docker compose logs -f mysql
```

## Parando o MySQL

Para parar e remover o container, preservando o volume:

```bash
docker compose down
```

Para remover também o volume:

```bash
docker compose down -v
```

> **Atenção:** `down -v` remove os dados armazenados no volume `mysql_data`. Use somente quando essa perda for desejada.

## Host x Container

Essa diferença é fundamental quando a aplicação também roda em Docker.

### Aplicação rodando na máquina

Se o Spring Boot estiver rodando diretamente no IntelliJ ou pelo terminal:

```text
localhost:3306
```

Exemplo:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/app_db
```

### Aplicação rodando em outro container

Se a aplicação estiver na mesma rede Docker, ela deve usar o **nome do serviço**:

```text
mysql:3306
```

Exemplo:

```properties
spring.datasource.url=jdbc:mysql://mysql:3306/app_db
```

Nesse cenário, `localhost` apontaria para o próprio container da aplicação, e não para o container do MySQL.

## Cenário: apenas banco local

Use este template quando você tem uma aplicação executando diretamente no host e precisa somente do banco.

```text
Spring Boot / IntelliJ
        │
        │ localhost:3306
        ▼
     MySQL
   (container)
```

É o cenário mais simples para desenvolvimento.

## Cenário: aplicação + banco em Docker

Quando a aplicação também está containerizada, normalmente os serviços compartilham uma rede Docker.

```text
┌─────────────────────────────┐
│ Docker network              │
│                             │
│  ┌──────────────┐           │
│  │ Spring Boot  │           │
│  └──────┬───────┘           │
│         │ mysql:3306        │
│         ▼                   │
│  ┌──────────────┐           │
│  │    MySQL     │           │
│  └──────────────┘           │
└─────────────────────────────┘
```

Nesse caso, não é necessário expor a porta do MySQL para o host se nenhum processo externo ao Docker precisar acessá-lo.

## Healthcheck

Este template básico não exige um healthcheck para funcionar. Em uma stack com dependências entre serviços, porém, ele pode ser útil para indicar quando o MySQL está pronto para aceitar conexões.

Uma configuração possível é:

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s
  timeout: 5s
  retries: 5
```

Healthcheck e `depends_on` têm responsabilidades diferentes. `depends_on` controla dependências de inicialização; o healthcheck verifica o estado de saúde do serviço.

## Persistência

O volume é deliberado neste template porque bancos de dados normalmente precisam sobreviver à recriação do container.

```bash
docker compose down
```

mantém o volume.

Já:

```bash
docker compose down -v
```

remove o volume.

## Segurança

Este template é voltado para desenvolvimento local. Para ambientes reais, não copie essas credenciais de exemplo.

Evite:

- colocar senhas reais diretamente no repositório;
- usar a conta root na aplicação sem necessidade;
- expor a porta do banco publicamente sem necessidade;
- usar credenciais de desenvolvimento em produção.

Para produção, avalie secrets, controle de acesso, TLS quando aplicável, backups, atualização de imagem e políticas de rede.

## Problemas comuns

### `Communications link failure`

Verifique se o container está ativo:

```bash
docker compose ps
```

Depois confira os logs:

```bash
docker compose logs mysql
```

### Spring Boot não consegue conectar usando `localhost`

Se o Spring Boot estiver em outro container, troque:

```text
localhost:3306
```

por:

```text
mysql:3306
```

### Alterei as credenciais e o MySQL continua usando as antigas

As variáveis de inicialização são usadas durante a inicialização do banco. Se o volume já contém um banco existente, simplesmente alterar o `.env` não significa que o usuário/senha existentes serão recriados.

Em um ambiente descartável, você pode recriar tudo com:

```bash
docker compose down -v
docker compose up -d
```

Isso apaga os dados existentes.

## Resumo

| Situação | Hostname | Porta |
|---|---|---:|
| Spring Boot no host | `localhost` | `3306` |
| Outro container na mesma rede | `mysql` | `3306` |
| Cliente externo ao Docker | endereço do host | porta publicada |

O princípio mais importante deste template é: **a porta publicada serve para comunicação com o host; entre containers, use o nome do serviço e a porta interna.**
