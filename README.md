# 🌤️ Chatbot de Temperatura no Telegram com n8n

Chatbot do Telegram, construído no n8n, que informa a temperatura atual de qualquer cidade do Brasil. O usuário envia o nome da cidade e do estado, o workflow consulta a API gratuita da OpenWeather e responde com uma mensagem curta e amigável.

```
Usuário: Belo Horizonte,MG,BR
Bot:     🌤️ A temperatura em Belo Horizonte é de 25°C.
```

## Como o workflow funciona

Arquivo: [`workflow-chatbot-telegram.json`](workflow-chatbot-telegram.json)

| # | Nó | Tipo | Função |
|---|----|------|--------|
| 1 | **Telegram Trigger** | Telegram Trigger | Recebe as mensagens de texto enviadas ao bot. |
| 2 | **Normalizar Cidade** | Set | Grava o texto na variável `queue`: remove acentos, converte para minúsculas, remove espaços extras e espaços ao redor das vírgulas e acrescenta `,br` se o país não foi informado. Também guarda o `chatId`. |
| 3 | **Consultar OpenWeather** | HTTP Request | `GET https://api.openweathermap.org/data/2.5/weather` com *Send Query Parameters*: `q`/`queue` = `{{ $json.queue }}`, `units` = `metric` (°C), `lang` = `pt_br`, `appid` = `{{ $env.OPENWEATHER_API_KEY }}`. Retorna a resposta completa (status HTTP + corpo) sem interromper o fluxo em caso de erro. |
| 4 | **Resposta Válida?** | IF | Verifica se o status HTTP é `200` e se o corpo contém `main.temp` e `name`. |
| 5 | **Formatar Mensagem** | Code | Extrai a cidade e a temperatura, arredonda a temperatura e monta a mensagem final. |
| 6 | **Enviar Temperatura** | Telegram (Send Message) | Envia `🌤️ A temperatura em <Cidade> é de <N>°C.` |
| 7 | **Enviar Erro** | Telegram (Send Message) | Ramo *false* do IF. Envia `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

> A API da OpenWeather lê a cidade no parâmetro `q`. O parâmetro `queue` também é enviado, com o mesmo valor, para seguir a especificação do desafio; a API ignora parâmetros desconhecidos.

Exemplo de normalização: `  São   Paulo , SP ` → `sao paulo,sp,br`.

## Variáveis esperadas

| Variável | Onde é usada | Como obter |
|----------|--------------|------------|
| `OPENWEATHER_API_KEY` | Variável de ambiente do n8n, lida no nó HTTP Request via `{{ $env.OPENWEATHER_API_KEY }}` | Crie uma conta gratuita em [openweathermap.org](https://openweathermap.org/api) e copie a chave em *My API keys*. Chaves novas podem levar até 2 horas para ativar. |
| `TELEGRAM_BOT_TOKEN` | Credencial *Telegram API* do n8n | Converse com o [@BotFather](https://t.me/BotFather), envie `/newbot` e copie o token gerado. |

**Nenhuma chave ou token está no repositório.** O JSON do workflow não contém credenciais; elas são configuradas apenas na sua instância do n8n.

## Como executar

### 1. Subir o n8n

**Com Docker (opcional):** crie um arquivo `.env` (não o versione) ao lado do `docker-compose.yml`:

```env
OPENWEATHER_API_KEY=sua_chave_openweather
WEBHOOK_URL=https://seu-endereco-publico.ngrok-free.app/
```

```bash
docker compose up -d
```

O n8n abre em `http://localhost:5678`.

**Sem Docker:** defina as variáveis de ambiente antes de iniciar o n8n:

```bash
export OPENWEATHER_API_KEY=sua_chave_openweather
export N8N_BLOCK_ENV_ACCESS_IN_NODE=false
n8n start
```

> `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` é necessário para que os nós possam ler `$env`.
>
> O Telegram Trigger usa webhook, então o n8n precisa estar acessível por uma URL pública HTTPS. Em ambiente local, use um túnel (por exemplo, `ngrok http 5678`) e defina essa URL em `WEBHOOK_URL`.

### 2. Importar o workflow

1. No n8n, clique em **Create Workflow** (ou abra um workflow vazio).
2. Abra o menu **⋯** no canto superior direito e escolha **Import from File...**.
3. Selecione `workflow-chatbot-telegram.json`.

### 3. Configurar a credencial do Telegram

1. Abra o nó **Telegram Trigger**.
2. Em **Credential to connect with**, escolha **Create new credential**.
3. Cole o valor de `TELEGRAM_BOT_TOKEN` no campo **Access Token** e salve.
4. Selecione essa mesma credencial nos nós **Enviar Temperatura** e **Enviar Erro**.

### 4. Configurar a OpenWeather

A chave é lida da variável de ambiente `OPENWEATHER_API_KEY` (passo 1). Não é preciso criar credencial no n8n nem colar a chave no nó.

### 5. Ativar e testar

1. Salve o workflow e ative-o (**Active**).
2. No Telegram, abra o seu bot e envie uma cidade no formato `Cidade,UF,BR`.

| Mensagem enviada | Resposta esperada |
|------------------|-------------------|
| `Belo Horizonte,MG,BR` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `Recife, PE` | `🌤️ A temperatura em Recife é de 29°C.` |
| `Cidadequenaoexiste,XX,BR` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

Os valores de temperatura acima são ilustrativos; o bot retorna a temperatura atual.

## Estrutura do repositório

```
├── workflow-chatbot-telegram.json   # workflow exportado do n8n (sem credenciais)
├── docker-compose.yml               # opcional: n8n local com Docker
└── README.md
```
