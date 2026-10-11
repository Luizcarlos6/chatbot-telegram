# 🌤️ Chatbot Telegram de Temperatura (N8N + OpenWeather)

Bot do Telegram feito no **N8N** que recebe o nome de uma cidade e responde com a temperatura atual, consultando a API da **OpenWeather**.

```
Você:  Belo Horizonte,MG,BR
Bot:   🌤️ A temperatura em Belo Horizonte é de 25°C.

Você:  Cidadeinexistente
Bot:   ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).
```

## Arquivos

| Arquivo | Descrição |
|---|---|
| `workflow-chatbot-telegram.json` | Workflow exportado do N8N (sem credenciais) |
| `workflow-telegram-chatbot.json` | Cópia idêntica (o enunciado cita os dois nomes) |
| `README.md` | Esta documentação |
| `docker-compose.yml` | Opcional: sobe o N8N localmente com as variáveis de ambiente |
| `.env.example` | Modelo das variáveis (o `.env` real fica fora do Git) |

## Como o workflow funciona

```
Telegram Trigger
   └─► Formatar entrada (queue)          Set: cria a variável `queue`
         └─► OpenWeather - Clima atual   HTTP Request (Send Query Parameters)
               └─► Resposta válida?      IF: status 200 + main.temp + name
                     ├─ true ─► Formatar mensagem (fallback)   Code (determinístico)
                     │            └─► Google Gemini (opcional)  desativado por padrão
                     │                  └─► Definir mensagem final   Code
                     │                        └─► Enviar temperatura  Telegram
                     └─ false ─► Enviar erro                    Telegram
```

1. **Telegram Trigger**: recebe as mensagens de texto (`updates: message`).
2. **Formatar entrada (queue)** (Set node): guarda o texto na variável **`queue`** já normalizado:
   - remove comandos como `/start`;
   - remove acentos (`São Paulo` → `sao paulo`) e passa para minúsculas;
   - remove espaços extras e os espaços ao redor das vírgulas (`São Paulo , SP , BR` → `sao paulo,sp,br`), que é o formato `cidade,UF,país` aceito pela OpenWeather.
   - Os espaços **dentro** do nome são mantidos, porque `belohorizonte` não seria encontrado pela API.
3. **OpenWeather - Clima atual** (HTTP Request): `GET https://api.openweathermap.org/data/2.5/weather` com **Send Query Parameters** ativado:

   | Parâmetro | Valor |
   |---|---|
   | `q` | `{{ $json.queue }}`: o parâmetro que a OpenWeather de fato lê |
   | `queue` | `{{ $json.queue }}`: o nome pedido no enunciado (a API ignora parâmetros extras) |
   | `units` | `metric` (graus Celsius) |
   | `lang` | `pt_br` (português do Brasil) |
   | `appid` | `{{ $env.OPENWEATHER_API_KEY }}` |

   O nó usa *Full Response* + *Never Error*, para que o status HTTP chegue ao IF em vez de interromper o fluxo.
4. **Resposta válida?** (IF): segue pelo caminho de sucesso só se `statusCode == 200`, `body.main.temp` existir e `body.name` não estiver vazio. Qualquer outro caso (404 cidade não encontrada, 400 texto vazio, 401 chave inválida) vai para **Enviar erro**.
5. **Formatar mensagem (fallback)** (Code): arredonda a temperatura com `Math.round` e monta `🌤️ A temperatura em {cidade} é de {temp}°C.`
6. **Enviar temperatura** / **Enviar erro** (Telegram Send Message): respondem no mesmo chat.

## Pré-requisitos

- N8N 1.x (testado com a imagem `n8nio/n8n:latest`), via Docker ou n8n Cloud.
- Um bot do Telegram (token do **@BotFather**: `/newbot`).
- Uma API Key da OpenWeather (<https://home.openweathermap.org/api_keys>). Chaves novas podem levar até ~2 h para serem ativadas.
- **URL pública HTTPS** para o N8N: o Telegram só envia mensagens para webhooks HTTPS. No n8n Cloud isso já existe; localmente use ngrok ou cloudflared.

## Variáveis esperadas

| Variável | Uso |
|---|---|
| `OPENWEATHER_API_KEY` | Lida pelo nó HTTP via `{{ $env.OPENWEATHER_API_KEY }}` |
| `TELEGRAM_BOT_TOKEN` | Token do bot, usado na credencial **Telegram API** do N8N |

> ⚠️ Nunca coloque os valores reais no JSON nem no README. O `.env` está no `.gitignore`.

## Passo a passo

### 1. Subir o N8N (opcional, com Docker)

```bash
cp .env.example .env
```

Preencha o `.env` e exponha a porta 5678 em HTTPS, por exemplo:

```bash
ngrok http 5678
```

Coloque a URL gerada em `WEBHOOK_URL` no `.env` e suba o N8N:

```bash
docker compose up -d
```

Acesse <http://localhost:5678> e crie o usuário administrador.

> Sem Docker: defina as variáveis no sistema antes de iniciar (`OPENWEATHER_API_KEY`, `TELEGRAM_BOT_TOKEN`, `WEBHOOK_URL`/`N8N_WEBHOOK_URL` e `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`) e rode `npx n8n`.
>
> No N8N 2.x, depois de importar, clique em **Publish** no topo do workflow para registrar o webhook no Telegram.

### 2. Importar o workflow

1. No N8N, clique em **Create Workflow** (ou abra um workflow vazio).
2. Menu **⋯** (canto superior direito) → **Import from File…**
3. Selecione `workflow-chatbot-telegram.json`.

### 3. Credencial do Telegram (`TELEGRAM_BOT_TOKEN`)

1. Abra o nó **Telegram Trigger** → em *Credential to connect with* escolha **Create New Credential**.
2. Em **Access Token** cole o token do BotFather. Se preferir não digitá-lo, use a expressão `{{ $env.TELEGRAM_BOT_TOKEN }}` (requer a variável definida no ambiente do N8N, como no `docker-compose.yml`).
3. Salve e selecione **a mesma credencial** nos nós **Enviar temperatura** e **Enviar erro**.

### 4. Chave da OpenWeather (`OPENWEATHER_API_KEY`)

Não há credencial a criar: o nó **OpenWeather - Clima atual** já lê `{{ $env.OPENWEATHER_API_KEY }}`. Basta a variável existir no ambiente do N8N (arquivo `.env` + `docker-compose.yml`, ou variável do sistema). No n8n Cloud, use **Settings → Variables** e troque a expressão por `{{ $vars.OPENWEATHER_API_KEY }}`.

### 5. Executar

- **Teste:** clique em **Execute workflow** e envie uma mensagem ao bot (o webhook de teste escuta uma mensagem por vez).
- **Produção:** ative o botão **Active** no topo do workflow; o bot passa a responder sempre.

### 6. O que esperar

| Mensagem enviada | Resposta |
|---|---|
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `belo horizonte` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `Curitiba,PR,BR` | `🌤️ A temperatura em Curitiba é de 16°C.` |
| `Cidadeinexistente` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

(As temperaturas variam conforme o clima do momento.)

### Testes realizados (11/10/2026, N8N 2.42.6)

| Mensagem enviada | `queue` gerada | HTTP | Resposta do bot |
|---|---|---|---|
| `Belo horizonte` | `belo horizonte` | 200 | 🌤️ A temperatura em Belo Horizonte é de 30°C. |
| `Sao paulo, sp, br` | `sao paulo,sp,br` | 200 | 🌤️ A temperatura em São Paulo é de 23°C. |
| `Curitiba, pr, br` | `curitiba,pr,br` | 200 | 🌤️ A temperatura em Curitiba é de 20°C. |
| `São paulo sp br` (sem vírgulas) | `sao paulo sp br` | 404 | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |
| `Sao paulo, sp, r` (país inválido) | `sao paulo,sp,r` | 404 | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |
| `/start` (sem cidade) | *(vazia)* | 400 | ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR). |

Os nós de envio ao Telegram e a chamada à OpenWeather têm **Retry on Fail** (3 tentativas), para que uma falha de rede momentânea não deixe o usuário sem resposta.

## Opcional: Google Gemini

O nó **Google Gemini (opcional)** fica entre **Formatar mensagem (fallback)** e **Definir mensagem final** e vem **desativado**, então o workflow funciona sem nenhuma chave do Gemini.

- **Configuração:** temperatura `0.1`, instrução em português e saída no formato `{"message":"...","ok":true}`.
- **Fallback obrigatório:** o nó Code **Formatar mensagem (fallback)** sempre gera a `message` de forma determinística. O nó **Definir mensagem final** só usa o texto do Gemini se ele for um JSON válido com `ok: true` **e** contiver a mesma temperatura (`25°C`); em qualquer outro caso (nó desativado, sem credencial, erro ou resposta fora do formato) usa o fallback. O campo `fonte` na saída indica qual foi usado (`gemini` ou `fallback`).
- **Como ativar:**
  1. Gere uma chave em <https://aistudio.google.com/app/apikey>.
  2. No N8N: **Credentials → Add Credential → Google Gemini (PaLM) API**, cole a chave em **API Key** e salve.
  3. Abra o nó **Google Gemini (opcional)**, selecione a credencial e o modelo (ex.: `gemini-2.5-flash`).
  4. Clique com o botão direito no nó → **Activate** (ou tecla `D`).
- O nó usa *On Error: Continue*, então uma falha do Gemini nunca impede a resposta ao usuário.

## Segurança

- O JSON exportado **não contém** tokens, chaves nem IDs de credenciais: a chave da OpenWeather é lida de `$env` e a credencial do Telegram é escolhida após a importação.
- Conferência rápida antes de publicar:

```bash
grep -nE "[0-9]{8,10}:[A-Za-z0-9_-]{35}|[a-f0-9]{32}" workflow-chatbot-telegram.json README.md
```

(Nenhum resultado = nenhum token do Telegram nem chave da OpenWeather no arquivo.)
