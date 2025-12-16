# WAHA + Chatwoot (e Calls) – Stack Docker

Este repositório contém um stack Docker mínimo para rodar o **WAHA (WhatsApp HTTP API)** integrado ao **Chatwoot** e com o app de **calls** habilitado (rejeita chamadas e envia mensagem automática).

> Importante: este projeto não contém o código‑fonte do WAHA, apenas a configuração Docker e de ambiente para facilitar o deploy.

---

## Visão geral da arquitetura

- `waha`  
  Container com a imagem oficial do WAHA (`devlikeapro/waha:latest-2025.12.1`), expondo uma API HTTP para controlar sessões de WhatsApp.

- `redis`  
  Usado pelo WAHA para processar jobs dos Apps (Chatwoot, Calls, etc.).

- Integração com **Chatwoot**  
  Feita via App `chatwoot` do próprio WAHA, usando:
  - URL pública do Chatwoot,
  - `accountId`,
  - `accountToken`,
  - `inboxId` ou `inboxIdentifier`.

- App **Calls**  
  Rejeita chamadas (diretas e em grupos) e envia mensagens automáticas informando para o usuário enviar áudio ou texto.

O stack foi pensado para ser usado atrás de um **reverse proxy** (por exemplo, Traefik), portanto o `docker-compose.yml` não expõe portas diretamente.

---

## Estrutura de arquivos

- `docker-compose.yml`  
  Define os serviços `waha` e `redis`, rede e volume de dados do Redis.

- `.env`  
  Arquivo de variáveis de ambiente lido pelo serviço `waha`.  
  Contém:
  - Configurações gerais do WAHA,
  - Dados de integração com o Chatwoot,
  - Textos e flags do app de Calls.

> Recomenda-se não commitar um `.env` com segredos reais. Use este arquivo como modelo e crie um `.env` privado no servidor.

---

## Pré‑requisitos

- Docker e Docker Compose instalados
- Um servidor com:
  - Acesso HTTP/HTTPS (idealmente atrás de um reverse proxy, como Traefik)
  - Acesso ao seu Chatwoot (URL pública)
- Uma instância de Chatwoot já configurada:
  - `accountId` da conta
  - `accountToken` (token de API da conta)
  - `inboxId` ou `inboxIdentifier` da inbox que receberá as mensagens do WhatsApp

---

## Configuração do `.env`

No arquivo `WAHA/.env` existem três grupos principais de variáveis.

### 1. WAHA

```env
WAHA_BASE_URL=http://0.0.0.0:3000
WAHA_PUBLIC_URL=https://waha.seu-dominio.com
WAHA_APPS_ENABLED=True
WAHA_APPS_ON=chatwoot,calls
REDIS_URL=redis://redis:6379
WHATSAPP_DEFAULT_ENGINE=GOWS
WAHA_API_KEY_PLAIN=troque_esta_senha_forte
```

- `WAHA_BASE_URL`  
  Endereço interno de escuta do WAHA dentro do container.

- `WAHA_PUBLIC_URL`  
  URL pública pela qual o WAHA será acessado (por trás do Traefik ou outro proxy).

- `WAHA_APPS_ENABLED` / `WAHA_APPS_ON`  
  Habilitam o sistema de Apps do WAHA e ativam os apps `chatwoot` e `calls`.

- `REDIS_URL`  
  URL do serviço Redis definido no `docker-compose.yml`.

- `WHATSAPP_DEFAULT_ENGINE`  
  Engine padrão de WhatsApp (GOWS é a recomendada nas docs do WAHA).

- `WAHA_API_KEY_PLAIN`  
  Senha em texto usada por alguns Apps. Use um valor forte em produção.

### 2. Chatwoot

```env
CHATWOOT_URL=https://chat.seu-chatwoot.com
CHATWOOT_ACCOUNT_ID=1
CHATWOOT_ACCOUNT_TOKEN=COLOQUE_SEU_ACCOUNT_TOKEN_AQUI
CHATWOOT_INBOX_ID=1
CHATWOOT_INBOX_IDENTIFIER=COLOQUE_SEU_INBOX_IDENTIFIER_AQUI
```

Essas variáveis não são lidas diretamente pelo WAHA, mas servem como fonte de verdade para você configurar o App `chatwoot` (via Dashboard ou API).

- `CHATWOOT_URL`  
  URL pública do Chatwoot.

- `CHATWOOT_ACCOUNT_ID`  
  ID numérico da conta no Chatwoot.

- `CHATWOOT_ACCOUNT_TOKEN`  
  Token de API da conta (não confundir com token de inbox).

- `CHATWOOT_INBOX_ID` / `CHATWOOT_INBOX_IDENTIFIER`  
  Identificadores da inbox (numérico ou string). Use o que for mais conveniente nas configurações do App `chatwoot`.

### 3. Calls (app de chamadas)

```env
CALLS_DM_REJECT=true
CALLS_DM_MESSAGE=📞❌ Não atendemos chamadas agora.\n🎤 Envie áudio ou texto, respondemos em seguida.
CALLS_GROUP_REJECT=true
CALLS_GROUP_MESSAGE=📞❌ Não atendemos chamadas em grupos.\n🎤 Envie áudio ou texto.
```

Esses valores representam a política padrão desejada para o App `calls`:

- Rejeitar chamadas diretas (`dm`) e em grupos (`group`).
- Enviar mensagens automáticas explicando para o usuário enviar áudio ou texto.

Os textos podem ser ajustados livremente conforme a sua necessidade.

---

## Como subir o stack

1. Clone o repositório:

```bash
git clone https://github.com/alaxricardsilva/waha.git
cd waha
```

2. Ajuste o arquivo `.env` com os valores do seu ambiente:

- Domínio real do WAHA em `WAHA_PUBLIC_URL`
- URL do Chatwoot em `CHATWOOT_URL`
- `CHATWOOT_ACCOUNT_ID`, `CHATWOOT_ACCOUNT_TOKEN`, `CHATWOOT_INBOX_ID` ou `CHATWOOT_INBOX_IDENTIFIER`
- `WAHA_API_KEY_PLAIN` com uma senha forte

3. Suba o stack:

```bash
docker compose up -d
```

Isso criará os containers:

- `waha`
- `waha-redis`

4. Configure o reverse proxy (por exemplo, Traefik) para apontar o domínio configurado em `WAHA_PUBLIC_URL` para o serviço `waha` na porta interna `3000`.

---

## Configurando o App Chatwoot no WAHA

Com o WAHA rodando e acessível via `WAHA_PUBLIC_URL`, acesse o dashboard do WAHA e crie um App `chatwoot` para a sessão desejada (por exemplo, `default`).

Um exemplo de configuração (adaptar com seus valores):

```json
{
  "app": "chatwoot",
  "session": "default",
  "config": {
    "linkPreview": "OFF",
    "locale": "pt-BR",
    "url": "https://chat.seu-chatwoot.com",
    "accountId": 1,
    "accountToken": "SEU_CHATWOOT_ACCOUNT_TOKEN",
    "inboxId": 1,
    "inboxIdentifier": "SEU_INBOX_IDENTIFIER",
    "templates": {},
    "commands": {
      "server": true,
      "queue": true
    },
    "conversations": {
      "sort": "created_newest",
      "status": ["open", "pending", "snoozed"]
    }
  },
  "enabled": true
}
```

Essa configuração faz com que:

- Todas as mensagens do WhatsApp da sessão escolhida sejam encaminhadas para a inbox indicada no Chatwoot.
- As conversas sejam ordenadas por mais novas e consideradas nos estados `open`, `pending` e `snoozed`.

Consulte a documentação oficial do WAHA para detalhes adicionais de campos e opções.

---

## Configurando o App de Calls

Da mesma forma, crie um App `calls` para a mesma sessão:

```json
{
  "app": "calls",
  "session": "default",
  "id": "app_default_calls",
  "config": {
    "dm": {
      "reject": true,
      "message": "📞❌ Não atendemos chamadas agora.\n🎤 Envie áudio ou texto, respondemos em seguida."
    },
    "group": {
      "reject": true,
      "message": "📞❌ Não atendemos chamadas em grupos.\n🎤 Envie áudio ou texto."
    }
  },
  "enabled": true
}
```

A partir desse ponto:

- Chamadas recebidas em conversas diretas serão rejeitadas e a mensagem definida em `dm.message` será enviada.
- Chamadas em grupos seguirão a política configurada em `group`.

---

## Fluxo básico de uso

1. Subir o stack com `docker compose up -d`.
2. Acessar o dashboard do WAHA pela URL configurada em `WAHA_PUBLIC_URL`.
3. Criar/usar uma sessão de WhatsApp e escanear o QR Code para conectar o número.
4. Configurar o App `chatwoot` com seus dados reais do Chatwoot.
5. Configurar o App `calls` com a política de chamadas desejada.
6. Enviar uma mensagem de outro celular para o número conectado:
   - A mensagem deve aparecer na inbox configurada do Chatwoot.
7. Fazer uma chamada para o número para validar se a chamada é rejeitada e se a mensagem automática é enviada.

---

## Aviso sobre segurança

- Nunca commite tokens reais, senhas ou dados sensíveis em repositórios públicos.
- Use um `.env` com valores fictícios no repositório e mantenha o `.env` real apenas no servidor.
- Se utilizar tokens em URLs de Git (para automação), prefira tokens de escopo restrito e substitua-os sempre que necessário.
