# Coolify Multi-Server Backup com Restic e S3

Script de backup automatizado para servidores que fazem parte de uma infraestrutura gerenciada pelo [Coolify](https://coolify.io/).

O script foi desenvolvido para ser reutilizado em múltiplos servidores. Cada máquina envia seus dados para um repositório Restic independente dentro de um armazenamento compatível com S3, utilizando automaticamente o nome do servidor como prefixo.

Ele realiza backup de:

* todo o conteúdo de `/srv`;
* dumps de bancos armazenados em `/srv/backups`;
* arquivos persistentes de aplicações;
* bind mounts organizados dentro de `/srv`;
* configurações e dados locais do Coolify, quando uma instalação é detectada;
* `APP_KEY` da instância do Coolify;
* chaves SSH utilizadas pelo Coolify;
* configurações do proxy;
* certificados e demais dados armazenados em `/data/coolify`.

---

## Visão geral

Em uma infraestrutura Coolify com múltiplos servidores, o painel centraliza o gerenciamento dos recursos, mas os dados persistentes permanecem fisicamente em cada servidor.

Por exemplo:

```text
Servidor coolify-manager
├── Instância principal do Coolify
├── APP_KEY
├── Chaves SSH
├── Configurações do proxy
└── /srv

Servidor apps-01
├── Aplicações
├── Bind mounts
├── Arquivos persistentes
└── /srv

Servidor database-01
├── Dumps dos bancos
└── /srv/backups
```

O script deve ser instalado em cada servidor.

Cada máquina cria e utiliza um repositório Restic próprio:

```text
Bucket S3
└── contabo-server-backups
    ├── coolify-manager
    ├── apps-01
    ├── apps-02
    └── database-01
```

Dessa forma, os backups e as políticas de retenção de um servidor não interferem nos demais.

---

## Principais funcionalidades

* Detecção automática do nome do servidor.
* Criação automática de um prefixo S3 por servidor.
* Inicialização automática do repositório Restic.
* Backup completo do diretório `/srv`.
* Detecção automática de uma instalação local do Coolify.
* Backup adicional de `/data/coolify` no servidor principal.
* Validação da presença da `APP_KEY`.
* Validação da existência das chaves SSH do Coolify.
* Política automática de retenção.
* Suporte a S3, Cloudflare R2, MinIO, Backblaze B2 S3 e outros serviços compatíveis.
* Execução segura com `set -Eeuo pipefail`.
* Identificação dos snapshots pelo hostname.
* Possibilidade de personalizar o nome do servidor.
* Possibilidade de desabilitar o `restic check` diário.

---

## Estratégia de backup

O script utiliza o Restic para criar backups:

* criptografados;
* incrementais;
* deduplicados;
* versionados;
* armazenados fora do servidor.

O backup não copia imagens Docker, pois elas podem ser reconstruídas a partir do repositório Git ou baixadas novamente do registry.

O foco do script são os dados que não podem ser reconstruídos facilmente:

* bancos de dados;
* arquivos enviados por usuários;
* mídia;
* configurações;
* segredos;
* chaves;
* estado persistente das aplicações;
* arquivos dos agentes;
* bind mounts;
* dumps;
* dados da instância do Coolify.

---

## Estrutura recomendada no servidor

A estratégia pressupõe que os dados persistentes sejam organizados dentro de `/srv`.

Exemplo:

```text
/srv
├── ai
│   └── hermes
│       ├── mari
│       └── outros-agentes
├── applications
├── backups
│   ├── postgres
│   ├── mariadb
│   ├── mongodb
│   └── sqlite
├── belfiore-admin
│   └── shared
│       ├── db.sqlite3
│       └── media
├── databases
├── infrastructure
└── monitoring
```

Todo o conteúdo de `/srv` será enviado ao backup.

---

## Bancos de dados

Este script não executa diretamente comandos como:

```text
pg_dump
mysqldump
mariadb-dump
mongodump
sqlite3 .backup
```

Cada banco deve possuir sua própria rotina de backup consistente.

Os dumps gerados devem ser armazenados dentro de:

```text
/srv/backups
```

Como `/srv` inteiro é incluído no backup, todos os dumps encontrados nesse diretório serão enviados automaticamente ao armazenamento S3.

### Exemplo com PostgreSQL

```bash
pg_dump \
    --format=custom \
    --file=/srv/backups/postgres/minha-aplicacao.dump \
    minha_aplicacao
```

### Exemplo com MariaDB

```bash
mariadb-dump \
    --single-transaction \
    --routines \
    --triggers \
    minha_aplicacao \
    > /srv/backups/mariadb/minha-aplicacao.sql
```

### Exemplo com SQLite

```bash
sqlite3 /caminho/do/banco.sqlite3 \
    ".backup '/srv/backups/sqlite/banco.sqlite3'"
```

Para bancos gerenciados pelo próprio Coolify, também é possível configurar os backups nativos para gerar os arquivos diretamente em armazenamento S3 ou em um diretório local.

---

## Detecção da instância do Coolify

O script verifica se o servidor atual executa uma instalação do Coolify.

A detecção ocorre de duas formas.

### Estrutura de diretórios

O servidor é considerado um host Coolify quando existe:

```text
/data/coolify
```

ou:

```text
/data/coolify/source/.env
```

### Containers Docker

Como verificação adicional, o script procura containers com nomes como:

```text
coolify
coolify-db
coolify-redis
coolify-realtime
coolify-proxy
```

Se uma instalação for detectada, o diretório completo abaixo será incluído:

```text
/data/coolify
```

---

## Dados do Coolify incluídos

Ao salvar `/data/coolify`, o backup inclui, conforme a estrutura existente na instalação:

```text
/data/coolify/source/.env
/data/coolify/ssh/keys
/data/coolify/proxy
/data/coolify/ssl
/data/coolify/applications
/data/coolify/databases
```

Entre os dados mais importantes estão:

### APP_KEY

Normalmente encontrada em:

```text
/data/coolify/source/.env
```

A `APP_KEY` é utilizada pelo Coolify para criptografar dados sensíveis.

Sem a chave original, uma instância restaurada pode não conseguir descriptografar credenciais e variáveis armazenadas no banco interno do Coolify.

### Chaves SSH

Normalmente armazenadas em:

```text
/data/coolify/ssh/keys
```

Essas chaves permitem que o Coolify se conecte aos servidores remotos gerenciados.

Sem elas, uma instância restaurada pode conhecer os servidores cadastrados, mas não conseguir acessá-los.

### Configurações do proxy

O backup também preserva configurações locais do proxy e certificados armazenados dentro da estrutura do Coolify.

---

## Importante: banco interno do Coolify

O diretório `/data/coolify` não substitui o backup consistente do banco PostgreSQL interno do Coolify.

O banco interno contém informações como:

* projetos;
* ambientes;
* aplicações;
* bancos cadastrados;
* servidores remotos;
* variáveis de ambiente;
* domínios;
* configurações de deploy;
* usuários;
* equipes;
* destinos;
* credenciais criptografadas.

É necessário manter também o backup nativo da instância do Coolify ou gerar um dump consistente do banco `coolify-db`.

O procedimento recomendado é:

1. Configurar o backup nativo da instância no painel do Coolify.
2. Armazenar os backups em S3 ou Cloudflare R2.
3. Executar este script para preservar `/data/coolify`, incluindo a `APP_KEY` e as chaves SSH.

Uma restauração completa do Coolify depende das duas partes:

```text
Banco interno do Coolify
+
APP_KEY e arquivos de /data/coolify
```

---

## Requisitos

O servidor deve possuir:

* Linux;
* Bash;
* Restic;
* acesso ao armazenamento S3;
* hostname configurado;
* privilégios de root;
* Docker, quando for um servidor Coolify.

### Rocky Linux

```bash
sudo dnf install -y epel-release
sudo dnf install -y restic
```

### Debian e Ubuntu

```bash
sudo apt update
sudo apt install -y restic
```

### Verificar a instalação

```bash
restic version
```

---

## Instalação

Clone o repositório:

```bash
git clone https://github.com/SEU-USUARIO/SEU-REPOSITORIO.git
cd SEU-REPOSITORIO
```

Copie o script:

```bash
sudo cp backup-server /usr/local/sbin/backup-server
```

Aplique permissões restritas:

```bash
sudo chown root:root /usr/local/sbin/backup-server
sudo chmod 700 /usr/local/sbin/backup-server
```

---

## Configuração

Crie o diretório de configuração:

```bash
sudo mkdir -p /etc/restic
sudo chmod 700 /etc/restic
```

Crie o arquivo:

```bash
sudo nano /etc/restic/env
```

Exemplo:

```bash
export AWS_ACCESS_KEY_ID="SEU_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="SEU_SECRET_ACCESS_KEY"

export RESTIC_REPOSITORY_BASE="s3:https://SEU_ENDPOINT_S3/contabo-server-backups"

export RESTIC_PASSWORD_FILE="/etc/restic/password"

# Opcional.
# Quando não definido, o hostname curto será utilizado.
# export BACKUP_SERVER_NAME="coolify-manager"

# Opcional.
# Quando true, executa restic check após cada backup.
export RUN_RESTIC_CHECK="true"
```

Proteja o arquivo:

```bash
sudo chown root:root /etc/restic/env
sudo chmod 600 /etc/restic/env
```

---

## Senha do repositório

Crie uma senha forte:

```bash
sudo nano /etc/restic/password
```

Exemplo:

```text
uma-senha-longa-aleatoria-e-exclusiva
```

Proteja o arquivo:

```bash
sudo chown root:root /etc/restic/password
sudo chmod 600 /etc/restic/password
```

A senha do Restic é necessária para descriptografar e restaurar os dados.

Perder essa senha significa perder o acesso aos backups.

Mantenha uma cópia em um gerenciador de senhas ou cofre seguro fora do servidor.

---

## Configuração com Cloudflare R2

No Cloudflare R2, crie um bucket, por exemplo:

```text
contabo-server-backups
```

Crie um token com acesso de leitura e gravação ao bucket.

O endpoint normalmente segue este formato:

```text
https://ACCOUNT_ID.r2.cloudflarestorage.com
```

A configuração ficaria semelhante a:

```bash
export AWS_ACCESS_KEY_ID="ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="SECRET_KEY"

export RESTIC_REPOSITORY_BASE="s3:https://ACCOUNT_ID.r2.cloudflarestorage.com/contabo-server-backups"

export RESTIC_PASSWORD_FILE="/etc/restic/password"
```

---

## Nome do servidor

Por padrão, o script utiliza:

```bash
hostname -s
```

Exemplo:

```bash
hostname -s
```

Resultado:

```text
apps-01
```

Nesse caso, o repositório final será:

```text
s3:https://ENDPOINT/contabo-server-backups/apps-01
```

### Personalização manual

O nome pode ser definido no arquivo `/etc/restic/env`:

```bash
export BACKUP_SERVER_NAME="servidor-belfiore"
```

Isso é útil quando:

* o hostname não é descritivo;
* o hostname contém caracteres inadequados;
* existe um padrão próprio de nomenclatura;
* o nome do servidor no provedor é diferente do hostname.

O nome é normalizado automaticamente:

* convertido para letras minúsculas;
* espaços são substituídos;
* caracteres não permitidos são removidos ou substituídos;
* apenas letras, números, ponto, hífen e underscore são preservados.

---

## Inicialização automática

Não é necessário executar manualmente:

```bash
restic init
```

O script testa o repositório com:

```bash
restic cat config
```

Se o repositório ainda não existir, ele executa automaticamente:

```bash
restic init
```

Esse comportamento permite instalar o mesmo script em vários servidores sem inicializar cada repositório manualmente.

---

## Execução manual

Execute:

```bash
sudo /usr/local/sbin/backup-server
```

Exemplo de saída:

```text
[2026-07-21 02:30:00] Servidor identificado: apps-01
[2026-07-21 02:30:00] Repositório Restic: s3:https://endpoint/bucket/apps-01
[2026-07-21 02:30:00] Incluindo todo o conteúdo de /srv
[2026-07-21 02:30:00] Nenhuma instância local do Coolify foi detectada
[2026-07-21 02:30:01] Repositório Restic já inicializado
[2026-07-21 02:30:01] Iniciando backup Restic
[2026-07-21 02:35:20] Aplicando política de retenção
[2026-07-21 02:36:02] Verificando integridade do repositório
[2026-07-21 02:38:17] Backup concluído com sucesso
```

Em um servidor Coolify:

```text
[2026-07-21 02:30:00] Instância local do Coolify detectada
[2026-07-21 02:30:00] Incluindo /data/coolify no backup
[2026-07-21 02:30:00] APP_KEY encontrada no arquivo de ambiente do Coolify
[2026-07-21 02:30:00] Chaves SSH do Coolify encontradas
```

---

## Política de retenção

O script mantém:

```text
7 backups diários
4 backups semanais
6 backups mensais
```

A política é aplicada com:

```bash
restic forget \
    --host "$SERVER_NAME" \
    --tag automatico \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 6 \
    --prune
```

O parâmetro `--prune` remove do repositório os blocos que não pertencem mais a nenhum snapshot retido.

Cada servidor possui seu próprio repositório. Portanto, a retenção é aplicada apenas aos backups daquela máquina.

---

## Tags dos snapshots

Cada snapshot recebe as tags:

```text
automatico
servidor-NOME_DO_SERVIDOR
```

Exemplo:

```text
automatico
servidor-apps-01
```

Também é utilizado:

```bash
--host "$SERVER_NAME"
```

Isso facilita consultas, filtros e auditorias.

---

## Verificação do repositório

Por padrão, o script executa:

```bash
restic check
```

Essa operação verifica a integridade estrutural do repositório.

Em repositórios grandes, a execução diária pode consumir tempo e recursos.

Ela pode ser desativada no arquivo de ambiente:

```bash
export RUN_RESTIC_CHECK="false"
```

Nesse caso, recomenda-se criar uma rotina separada para executar a verificação semanalmente ou mensalmente.

Exemplo manual:

```bash
sudo bash -c '
    source /etc/restic/env
    SERVER_NAME=$(hostname -s | tr "[:upper:]" "[:lower:]")
    export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/${SERVER_NAME}"
    restic check
'
```

---

## Agendamento com systemd

Crie o serviço:

```bash
sudo nano /etc/systemd/system/backup-server.service
```

Conteúdo:

```ini
[Unit]
Description=Backup do servidor com Restic
Documentation=https://github.com/SEU-USUARIO/SEU-REPOSITORIO
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/backup-server

Nice=10
IOSchedulingClass=best-effort
IOSchedulingPriority=7

PrivateTmp=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
```

Crie o timer:

```bash
sudo nano /etc/systemd/system/backup-server.timer
```

Conteúdo:

```ini
[Unit]
Description=Executa o backup diário do servidor

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=10m

[Install]
WantedBy=timers.target
```

Recarregue o systemd:

```bash
sudo systemctl daemon-reload
```

Ative o timer:

```bash
sudo systemctl enable --now backup-server.timer
```

Verifique:

```bash
systemctl status backup-server.timer
```

Liste os próximos timers:

```bash
systemctl list-timers backup-server.timer
```

---

## Execução com atraso aleatório

A opção:

```ini
RandomizedDelaySec=10m
```

faz com que o backup seja iniciado em algum momento dentro dos dez minutos seguintes ao horário configurado.

Isso evita que todos os servidores da infraestrutura iniciem o backup exatamente ao mesmo tempo.

Em uma infraestrutura com vários servidores:

```text
02:30: servidor apps-01
02:33: servidor apps-02
02:37: servidor database-01
02:39: servidor coolify-manager
```

Os horários variam automaticamente.

---

## Logs

Consulte os logs da última execução:

```bash
sudo journalctl -u backup-server.service
```

Acompanhe em tempo real:

```bash
sudo journalctl -u backup-server.service -f
```

Consulte apenas a execução atual:

```bash
sudo journalctl -u backup-server.service --since today
```

Verifique o resultado da última execução:

```bash
systemctl status backup-server.service
```

---

## Listar snapshots

Carregue as variáveis e construa o endereço do repositório:

```bash
sudo bash -c '
    source /etc/restic/env

    SERVER_NAME="${BACKUP_SERVER_NAME:-$(hostname -s)}"
    SERVER_NAME=$(printf "%s" "$SERVER_NAME" |
        tr "[:upper:]" "[:lower:]" |
        sed \
            -e "s/[^a-z0-9._-]/-/g" \
            -e "s/^-*//" \
            -e "s/-*$//")

    export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/${SERVER_NAME}"

    restic snapshots
'
```

---

## Listar o conteúdo de um snapshot

```bash
sudo bash -c '
    source /etc/restic/env

    SERVER_NAME="${BACKUP_SERVER_NAME:-$(hostname -s)}"
    SERVER_NAME=$(printf "%s" "$SERVER_NAME" |
        tr "[:upper:]" "[:lower:]" |
        sed \
            -e "s/[^a-z0-9._-]/-/g" \
            -e "s/^-*//" \
            -e "s/-*$//")

    export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/${SERVER_NAME}"

    restic ls latest
'
```

---

## Restaurar um backup

Crie um diretório temporário:

```bash
sudo mkdir -p /srv/restore-test
```

Restaure o snapshot mais recente:

```bash
sudo bash -c '
    source /etc/restic/env

    SERVER_NAME="${BACKUP_SERVER_NAME:-$(hostname -s)}"
    SERVER_NAME=$(printf "%s" "$SERVER_NAME" |
        tr "[:upper:]" "[:lower:]" |
        sed \
            -e "s/[^a-z0-9._-]/-/g" \
            -e "s/^-*//" \
            -e "s/-*$//")

    export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/${SERVER_NAME}"

    restic restore latest --target /srv/restore-test
'
```

O conteúdo será restaurado preservando os caminhos originais:

```text
/srv/restore-test
├── srv
│   ├── ai
│   ├── applications
│   ├── backups
│   └── ...
└── data
    └── coolify
```

---

## Restaurar apenas um diretório

Exemplo para restaurar somente o agente Hermes Mari:

```bash
sudo bash -c '
    source /etc/restic/env

    SERVER_NAME="${BACKUP_SERVER_NAME:-$(hostname -s)}"
    SERVER_NAME=$(printf "%s" "$SERVER_NAME" |
        tr "[:upper:]" "[:lower:]" |
        sed \
            -e "s/[^a-z0-9._-]/-/g" \
            -e "s/^-*//" \
            -e "s/-*$//")

    export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/${SERVER_NAME}"

    restic restore latest \
        --target /srv/restore-test \
        --include /srv/ai/hermes/mari
'
```

---

## Restaurar somente os dados do Coolify

No servidor principal:

```bash
sudo bash -c '
    source /etc/restic/env

    SERVER_NAME="${BACKUP_SERVER_NAME:-$(hostname -s)}"
    SERVER_NAME=$(printf "%s" "$SERVER_NAME" |
        tr "[:upper:]" "[:lower:]" |
        sed \
            -e "s/[^a-z0-9._-]/-/g" \
            -e "s/^-*//" \
            -e "s/-*$//")

    export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/${SERVER_NAME}"

    restic restore latest \
        --target /srv/restore-test \
        --include /data/coolify
'
```

Antes de sobrescrever uma instalação ativa do Coolify, pare os containers e valide cuidadosamente os arquivos restaurados.

---

## Recuperação de um servidor completo

Em caso de perda de uma VPS:

1. Crie um novo servidor.
2. Instale o sistema operacional.
3. Configure o hostname esperado.
4. Instale o Restic.
5. Restaure `/etc/restic/env`.
6. Restaure `/etc/restic/password`.
7. Execute a restauração para um diretório temporário.
8. Instale Docker e demais dependências.
9. Recrie as aplicações pelo Coolify.
10. Pare os serviços stateful.
11. Restaure os arquivos de `/srv`.
12. Restaure os bancos usando os dumps de `/srv/backups`.
13. Ajuste proprietários e permissões.
14. Inicie os serviços.
15. Valide aplicações, bancos, domínios e integrações.

---

## Recuperação do servidor principal do Coolify

Em caso de perda do servidor que hospeda o painel:

1. Crie uma nova VPS.
2. Instale o Coolify.
3. Pare a nova instância.
4. Restaure o banco interno do Coolify.
5. Restaure a `APP_KEY` original.
6. Restaure as chaves de `/data/coolify/ssh/keys`.
7. Restaure demais arquivos necessários de `/data/coolify`.
8. Inicie o Coolify.
9. Valide o acesso aos servidores remotos.
10. Confirme projetos, ambientes e variáveis.
11. Teste um deploy controlado.

A restauração deve preservar a mesma `APP_KEY` usada no momento em que o backup do banco foi criado.

---

## Exclusão de caches

O script utiliza:

```bash
--exclude-caches
```

O Restic ignora diretórios marcados com arquivos de cache compatíveis com a convenção CACHEDIR.TAG.

Isso ajuda a evitar o armazenamento de caches que podem ser recriados.

O parâmetro não exclui automaticamente qualquer diretório chamado `cache`. Ele depende da marcação apropriada do diretório.

---

## Atenção ao diretório `/srv/backups`

Como o script salva `/srv` inteiro, o diretório:

```text
/srv/backups
```

também será enviado ao Restic.

Isso é intencional, pois os dumps dos bancos serão armazenados ali.

Não configure o Restic para criar o próprio repositório local dentro de `/srv/backups`, pois isso poderia fazer o backup incluir sua própria estrutura de repositório.

O destino principal deve permanecer externo, como S3 ou R2.

---

## Bind mounts

Bind mounts apontando para diretórios dentro de `/srv` são incluídos automaticamente.

Exemplo:

```yaml
services:
  hermes-mari:
    volumes:
      - /srv/ai/hermes/mari:/app/data
```

O script salva:

```text
/srv/ai/hermes/mari
```

Não é necessário adicionar esse caminho individualmente.

---

## Docker named volumes

Volumes Docker nomeados não ficam necessariamente dentro de `/srv`.

Exemplo:

```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

O conteúdo físico normalmente fica abaixo de:

```text
/var/lib/docker/volumes
```

Esse diretório não é incluído pelo script.

Para bancos, a estratégia recomendada é utilizar dumps consistentes em `/srv/backups`.

Para volumes que não sejam bancos, existem três opções:

1. Migrar o armazenamento para um bind mount dentro de `/srv`.
2. Criar uma rotina separada de exportação do volume.
3. Estender o script para exportar volumes selecionados antes do Restic.

Não é recomendado copiar diretamente arquivos ativos de bancos em `/var/lib/docker/volumes`.

---

## Testes de restauração

Backups devem ser testados regularmente.

Recomenda-se:

* verificar snapshots após a primeira execução;
* restaurar arquivos para uma pasta temporária;
* validar dumps de bancos;
* executar testes mensais;
* documentar o procedimento de desastre;
* confirmar que a senha do Restic está armazenada fora da VPS.

Exemplo:

```bash
sudo mkdir -p /srv/restore-test
```

Depois:

```bash
restic restore latest --target /srv/restore-test
```

Para SQLite:

```bash
sqlite3 \
    /srv/restore-test/srv/backups/sqlite/banco.sqlite3 \
    "PRAGMA integrity_check;"
```

Resultado esperado:

```text
ok
```

---

## Segurança

Os arquivos abaixo contêm informações sensíveis:

```text
/etc/restic/env
/etc/restic/password
/data/coolify/source/.env
/data/coolify/ssh/keys
```

Utilize permissões restritas:

```bash
sudo chmod 700 /etc/restic
sudo chmod 600 /etc/restic/env
sudo chmod 600 /etc/restic/password
sudo chmod 700 /usr/local/sbin/backup-server
```

Não publique:

* senhas;
* credenciais S3;
* `APP_KEY`;
* chaves SSH;
* tokens;
* arquivos `.env` reais.

O arquivo `/etc/restic/env` não deve fazer parte do repositório Git.

---

## Exemplo de `.gitignore`

```gitignore
.env
env
*.key
*.pem
password
secrets/
restore/
backups/
```

---

## Permissões S3

A credencial utilizada deve ter acesso apenas ao bucket ou prefixo necessário.

Permissões esperadas:

* listar objetos;
* criar objetos;
* ler objetos;
* remover objetos.

O Restic precisa remover objetos durante operações como:

```bash
restic forget --prune
```

Uma credencial apenas de escrita pode impedir a retenção e a verificação do repositório.

---

## Uso em múltiplos servidores

Instale o mesmo script em todas as máquinas:

```text
coolify-manager
apps-01
apps-02
database-01
workers-01
```

Use o mesmo bucket:

```bash
export RESTIC_REPOSITORY_BASE="s3:https://ENDPOINT/contabo-server-backups"
```

Cada servidor acrescentará automaticamente seu próprio nome:

```text
contabo-server-backups/coolify-manager
contabo-server-backups/apps-01
contabo-server-backups/apps-02
contabo-server-backups/database-01
contabo-server-backups/workers-01
```

As credenciais S3 e a senha do Restic podem ser iguais em todos os servidores, mas uma estratégia de maior isolamento pode usar:

* credenciais diferentes por servidor;
* senhas diferentes por servidor;
* buckets diferentes por ambiente;
* políticas IAM limitadas por prefixo.

---

## Ambientes separados

Uma possível organização:

```text
backups
├── production
│   ├── coolify-manager
│   ├── apps-01
│   └── database-01
└── staging
    ├── coolify-staging
    └── apps-staging
```

Para produção:

```bash
export RESTIC_REPOSITORY_BASE="s3:https://ENDPOINT/backups/production"
```

Para homologação:

```bash
export RESTIC_REPOSITORY_BASE="s3:https://ENDPOINT/backups/staging"
```

---

## Diagnóstico de problemas

### Arquivo de ambiente não encontrado

Erro:

```text
Arquivo de configuração não encontrado: /etc/restic/env
```

Verifique:

```bash
sudo ls -l /etc/restic/env
```

---

### Senha não encontrada

Verifique:

```bash
sudo ls -l /etc/restic/password
```

E confirme:

```bash
grep RESTIC_PASSWORD_FILE /etc/restic/env
```

---

### Falha de autenticação no S3

Verifique:

* `AWS_ACCESS_KEY_ID`;
* `AWS_SECRET_ACCESS_KEY`;
* endpoint;
* nome do bucket;
* permissões do token;
* relógio do sistema;
* conectividade de rede.

Teste:

```bash
sudo /usr/local/sbin/backup-server
```

---

### Repositório não encontrado

Na primeira execução, o script deve inicializá-lo automaticamente.

Se a inicialização falhar, verifique:

```bash
source /etc/restic/env
```

Depois defina manualmente o repositório final:

```bash
export RESTIC_REPOSITORY="${RESTIC_REPOSITORY_BASE%/}/$(hostname -s)"
```

Teste:

```bash
restic init
```

---

### Coolify não detectado

Verifique:

```bash
sudo ls -la /data/coolify
```

E:

```bash
docker ps -a --format '{{.Names}}' | grep coolify
```

Se o Coolify estiver instalado em um caminho não padrão, ajuste as variáveis no script:

```bash
COOLIFY_ROOT="/caminho/personalizado"
COOLIFY_ENV="${COOLIFY_ROOT}/source/.env"
COOLIFY_SSH_KEYS="${COOLIFY_ROOT}/ssh/keys"
```

---

### APP_KEY não encontrada

Verifique:

```bash
sudo grep '^APP_KEY=' /data/coolify/source/.env
```

O valor não deve ser exibido em logs públicos nem compartilhado.

---

### Chaves SSH não encontradas

Verifique:

```bash
sudo find /data/coolify/ssh/keys -maxdepth 2 -type f
```

---

### Falta de espaço local

O Restic envia os dados diretamente para o destino remoto e não cria uma cópia completa local.

Entretanto, dumps de bancos armazenados em `/srv/backups` ocupam espaço até serem removidos ou rotacionados.

Configure uma política local de retenção dos dumps.

Exemplo:

```bash
find /srv/backups/postgres \
    -type f \
    -mtime +7 \
    -delete
```

Use esse comando somente após validar a política desejada.

---

## Boas práticas

* Utilize armazenamento externo à VPS.
* Mantenha a senha do Restic fora do servidor.
* Preserve a `APP_KEY` do Coolify.
* Preserve as chaves SSH.
* Configure dumps consistentes dos bancos.
* Teste restaurações.
* Monitore falhas do timer systemd.
* Evite volumes anônimos para dados importantes.
* Prefira bind mounts organizados dentro de `/srv`.
* Não dependa exclusivamente de snapshots do provedor.
* Não considere um backup válido apenas porque o comando terminou sem erro.
* Valide periodicamente arquivos e bancos restaurados.
* Mantenha documentação de recuperação de desastre.
* Use credenciais S3 com privilégio mínimo.

---

## Limitações

O script não realiza:

* dumps automáticos de bancos;
* restauração automática;
* backup de volumes Docker fora de `/srv`;
* backup de imagens Docker;
* backup de dados armazenados somente na memória;
* replicação entre servidores;
* sincronização de diretórios de aplicações stateful;
* validação funcional das aplicações restauradas;
* monitoramento ou envio de alertas;
* backup de arquivos fora de `/srv` e `/data/coolify`.

Novos caminhos podem ser adicionados ao array:

```bash
BACKUP_PATHS=(
    "$SRV_ROOT"
    "/outro/diretorio"
)
```

---

## Atualização do script

Depois de atualizar o repositório:

```bash
git pull
```

Copie novamente:

```bash
sudo cp backup-server /usr/local/sbin/backup-server
sudo chmod 700 /usr/local/sbin/backup-server
```

Teste:

```bash
sudo /usr/local/sbin/backup-server
```

---

## Checklist inicial

Antes de ativar o timer:

* [ ] Restic instalado.
* [ ] Bucket criado.
* [ ] Credencial S3 criada.
* [ ] `/etc/restic/env` configurado.
* [ ] `/etc/restic/password` criado.
* [ ] Permissões restritas aplicadas.
* [ ] Hostname validado.
* [ ] Backup manual executado.
* [ ] Snapshot listado.
* [ ] Arquivo restaurado em diretório temporário.
* [ ] Dumps de bancos configurados em `/srv/backups`.
* [ ] Backup nativo do banco interno do Coolify configurado.
* [ ] Timer systemd ativado.
* [ ] Logs verificados.

---

## Checklist de recuperação

* [ ] Credenciais S3 disponíveis.
* [ ] Senha do Restic disponível.
* [ ] Nome original do servidor conhecido.
* [ ] Snapshot correto identificado.
* [ ] Dados restaurados para diretório temporário.
* [ ] Dumps dos bancos validados.
* [ ] Aplicações paradas antes de sobrescrever arquivos.
* [ ] Proprietários e permissões corrigidos.
* [ ] Banco interno do Coolify restaurado.
* [ ] `APP_KEY` original restaurada.
* [ ] Chaves SSH restauradas.
* [ ] Aplicações e integrações testadas.

