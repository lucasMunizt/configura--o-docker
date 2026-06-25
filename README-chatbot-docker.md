# Ambiente Docker para Chatbot com n8n, WAHA e Redis

Este arquivo apresenta um resumo da configuração Docker utilizada para executar o ambiente do chatbot. A solução utiliza três serviços principais:

- **Redis**: responsável pelo armazenamento temporário de dados, sessões, contexto e buffer de mensagens.
- **n8n**: responsável pela criação e execução dos fluxos automatizados de atendimento.
- **WAHA**: responsável pela integração com o WhatsApp, permitindo receber e enviar mensagens por meio de webhooks.

---

## 1. Estrutura dos arquivos

Crie uma pasta para o projeto e, dentro dela, adicione os seguintes arquivos:

```text
projeto-chatbot/
├── docker-compose.yml
└── .env
```

---

## 2. Arquivo `docker-compose.yml`

O arquivo `docker-compose.yml` define os containers necessários para executar o ambiente.

```yaml
version: "3.9"

services:
  redis:
    image: redis:7-alpine
    container_name: local_redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  n8n:
    image: n8nio/n8n:latest
    container_name: local_n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    env_file:
      - .env
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - redis

  waha:
    image: devlikeapro/waha:latest
    container_name: local_waha
    platform: linux/amd64
    restart: unless-stopped
    ports:
      - "3000:3000"
    env_file:
      - .env
    volumes:
      - waha_sessions:/app/.sessions
      - waha_media:/app/.media
    depends_on:
      - n8n

volumes:
  redis_data:
  n8n_data:
  waha_sessions:
  waha_media:
```

---

## 3. Arquivo `.env`

O arquivo `.env` armazena as variáveis de ambiente utilizadas pelos containers.

```env
# =====================
# Timezone
# =====================

GENERIC_TIMEZONE=America/Fortaleza
TZ=America/Fortaleza

# =====================
# n8n
# =====================

N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=teste123

N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http

# URL usada pelo n8n para gerar webhooks.
# Em ambiente local, pode ser localhost.
# Em produção, substitua pelo domínio público ou IP do servidor.
WEBHOOK_URL=http://localhost:5678

N8N_SECURE_COOKIE=false

# =====================
# WAHA
# =====================

WHATSAPP_DEFAULT_ENGINE=GOWS

# URL interna usada pelo WAHA para enviar eventos ao webhook do n8n.
# Como os containers estão na mesma rede Docker, o nome do serviço "n8n" funciona como host.
WHATSAPP_HOOK_URL=http://n8n:5678/webhook/webhook

WHATSAPP_HOOK_EVENTS=message.any

WAHA_NO_API_KEY=true
WAHA_DASHBOARD_NO_PASSWORD=true
WHATSAPP_SWAGGER_NO_PASSWORD=true

# =====================
# Redis
# =====================

REDIS_HOST=redis
REDIS_PORT=6379
```

> Observação: no arquivo `.env`, evite deixar espaço depois do sinal de igual.
> Exemplo correto: `WEBHOOK_URL=http://localhost:5678`
> Exemplo incorreto: `WEBHOOK_URL= http://localhost:5678`

---

## 4. Explicação dos serviços

### Redis

O Redis é utilizado como banco de dados em memória. No projeto, ele pode ser usado para armazenar temporariamente o histórico das conversas, controlar sessões, salvar buffers de mensagens e registrar chaves de bloqueio quando um atendente humano assume o atendimento.

O container utiliza a imagem `redis:7-alpine`, que é uma versão leve do Redis.

---

### n8n

O n8n é a ferramenta responsável pela automação do fluxo de atendimento. Nele são criados os workflows que recebem mensagens do WhatsApp, tratam os dados, consultam o Redis, acionam o agente de inteligência artificial e enviam respostas ao cliente.

O n8n fica disponível na porta:

```text
http://localhost:5678
```

O acesso básico foi ativado com as variáveis:

```env
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=teste123
```

---

### WAHA

O WAHA é responsável por conectar o sistema ao WhatsApp. Ele recebe e envia mensagens, funcionando como uma ponte entre o WhatsApp e o n8n.

O painel do WAHA fica disponível na porta:

```text
http://localhost:3000
```

A variável abaixo define para onde o WAHA enviará os eventos recebidos do WhatsApp:

```env
WHATSAPP_HOOK_URL=http://n8n:5678/webhook/webhook
```

Nesse caso, `n8n` é o nome do serviço dentro da rede Docker, por isso não é utilizado `localhost`.

---

## 5. Como executar o projeto

### Passo 1: abrir o terminal na pasta do projeto

Acesse a pasta onde estão os arquivos `docker-compose.yml` e `.env`.

Exemplo:

```bash
cd projeto-chatbot
```

---

### Passo 2: subir os containers

Execute o comando:

```bash
docker compose up -d
```

Esse comando baixa as imagens necessárias e inicia os containers em segundo plano.

---

### Passo 3: verificar se os containers estão rodando

Execute:

```bash
docker ps
```

Você deverá ver os containers:

```text
local_redis
local_n8n
local_waha
```

---

### Passo 4: acessar o n8n

Abra o navegador e acesse:

```text
http://localhost:5678
```

---

### Passo 5: acessar o WAHA

Abra o navegador e acesse:

```text
http://localhost:3000
```

No painel do WAHA, acesse a sessão do WhatsApp, gere o QR Code e faça a autenticação com o aplicativo WhatsApp.

---

## 6. Como parar o ambiente

Para parar os containers, execute:

```bash
docker compose down
```

Esse comando para e remove os containers, mas mantém os volumes com os dados salvos.

---

## 7. Como apagar tudo, incluindo os dados

Caso deseje remover os containers e também apagar os volumes, execute:

```bash
docker compose down -v
```

Atenção: esse comando apaga os dados armazenados nos volumes, incluindo dados do n8n, sessões do WAHA e dados do Redis.

---

## 8. Observações importantes

- O `localhost` dentro de um container aponta para o próprio container, não para outro serviço.
- Para comunicação entre containers, use o nome do serviço definido no `docker-compose.yml`, como `n8n` ou `redis`.
- A URL `http://n8n:5678` funciona dentro da rede Docker.
- A URL `http://localhost:5678` funciona no navegador da máquina local.
- Em produção, recomenda-se hospedar o ambiente em uma VPS e configurar um domínio com HTTPS.
- Também é recomendado substituir senhas de teste por credenciais seguras antes de colocar o sistema em produção.
