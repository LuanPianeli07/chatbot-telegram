# Chatbot de clima no Telegram

O bot recebe uma cidade do Brasil no Telegram, consulta a temperatura atual na OpenWeather e devolve uma mensagem curta.

Exemplo de resposta:

`🌤️ A temperatura em Belo Horizonte é de 25°C.`

Se a cidade não existir ou a mensagem estiver fora do formato:

`❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

## Arquivos

- `workflow-chatbot-telegram.json` — workflow para importar no n8n. Não contém chave nem token.
- `docker-compose.yml` — n8n local. Não contém chave nem token.
- `.env` — credenciais de teste, só nesta máquina.

O `.gitignore` ignora o projeto inteiro. Não faça commit nem push destes arquivos.

## Subir o n8n

Na pasta do projeto:

```bash
docker compose up -d
```

Abra `http://localhost:5678` e crie a conta de administrador. Os dados ficam no volume local do Docker.

A imagem é `n8nio/n8n:2.29.0`. O compose deixa `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` porque, no n8n 2, o acesso a `$env` vem bloqueado. Sem isso, o nó HTTP não lê `OPENWEATHER_API_KEY`. O acesso pelo navegador em `http://localhost` usa `N8N_SECURE_COOKIE=false`. Telemetria, aviso de versão e templates do n8n ficam desligados.

## Importar o workflow

1. No n8n, abra **Workflows**.
2. Use **Import from File**.
3. Selecione `workflow-chatbot-telegram.json`.
4. Abra o workflow **Chatbot de Clima no Telegram**.

## Credenciais

### OpenWeather

Variável esperada: `OPENWEATHER_API_KEY`

O `docker compose` carrega o `.env`. O nó **OpenWeather** envia `appid` com a expressão `{{ $env.OPENWEATHER_API_KEY }}`. Não cole a chave no nó.

Se o n8n não for este container, defina a mesma variável no processo do n8n e reinicie.

### Telegram

Variável esperada: `TELEGRAM_BOT_TOKEN`

1. No n8n, crie uma credencial **Telegram API**.
2. Dê o nome `Telegram account`.
3. No campo **Access Token**, cole o valor de `TELEGRAM_BOT_TOKEN` que está no `.env`.
4. Salve.
5. Nos nós **Telegram Trigger**, **Enviar temperatura** e **Enviar erro**, selecione essa credencial.

## Ativar o bot

O Telegram entrega as mensagens num webhook HTTPS. Publique a porta `5678` com um túnel e acrescente a URL pública no `.env`, com barra no final:

```bash
WEBHOOK_URL=https://seu-tunel.exemplo/
```

Recrie o container:

```bash
docker compose up -d
```

Ative o workflow. O n8n registra o webhook no bot. Cada bot aceita um único Telegram Trigger ativo por vez.

## Como testar

Mande uma mensagem no formato `Cidade,UF,BR`.

O nó **Formatar entrada** grava o texto na variável `queue`: tira espaços nas pontas e ao redor das vírgulas, passa para minúsculas, remove acentos e completa o país com `br` quando ele falta. O espaço interno do nome permanece. `Belo Horizonte, MG` vira `belo horizonte,mg,br`. A frase enviada ao usuário usa o nome oficial devolvido pela OpenWeather.

- `São Paulo,SP,BR` — temperatura atual em São Paulo.
- `Belo Horizonte,MG,BR` — temperatura atual em Belo Horizonte.
- `Curitiba,PR,BR` — temperatura atual em Curitiba.
- `CidadeQueNaoExiste,SP,BR` — a mensagem de cidade não encontrada.

## Google Gemini

O nó **Google Gemini** fica entre **Mensagem fallback** e **Escolher mensagem**. A temperatura dele é `0.1` e a saída pedida é JSON `{"message":"...","ok":true}`, em português.

Para ligar:

1. Crie uma credencial **Google Gemini (PaLM) API** no n8n.
2. Dê o nome `Google Gemini account`.
3. No campo **API Key**, cole o valor de `GEMINI_API_KEY` que está no `.env`.
4. Salve e selecione essa credencial no nó **Google Gemini**.

Sem credencial, ou se o nó falhar, **Escolher mensagem** envia o texto de **Mensagem fallback**. Esse texto é determinístico: cidade e temperatura arredondada, no formato do exemplo. A avaliação sem credencial Gemini usa esse fallback.

## Fluxo

1. **Telegram Trigger** recebe a mensagem.
2. **Formatar entrada** grava `queue`.
3. **Formato válido** exige `cidade,uf,br`. Fora isso, **Enviar erro** responde.
4. **OpenWeather** consulta `https://api.openweathermap.org/data/2.5/weather` com `q`, `queue`, `units=metric` (Celsius), `lang=pt_br` e `appid`.
5. **Resposta válida** confere status HTTP de sucesso e a temperatura. Se faltar, **Enviar erro** responde.
6. **Mensagem fallback** monta a frase com a temperatura arredondada.
7. **Google Gemini** tenta reescrever. **Escolher mensagem** só usa essa reescrita quando o JSON vem válido.
8. **Enviar temperatura** manda a frase no mesmo chat.
