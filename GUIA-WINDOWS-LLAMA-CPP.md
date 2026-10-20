# Cliente Windows para o servidor llama.cpp do Mac

## Estado do servidor

- API OpenAI compatível: `http://192.168.15.135:8000/v1`
- A porta `8000` exige a chave existente no Mac em
  `~/.config/llama-cpp-server/api-key`.
- O IP atual do Mac é `192.168.15.135`; confirme-o novamente após trocar ou
  reiniciar o roteador com `ipconfig getifaddr en0` no Mac.
- Não use `localhost`, `127.0.0.1`, porta `11434` ou porta `8080` no Windows:
  esses endereços apontam para o próprio Windows ou para serviços que não estão
  sendo usados neste setup.

Nunca publique a porta `8000` na Internet. Ela deve ficar restrita à rede local.

## 1. Testar rede e autenticação

Copie a chave do Mac para o Windows por um meio seguro e, no PowerShell:

```powershell
$env:LLAMA_CPP_API_KEY = "COLE_A_CHAVE_AQUI"
$env:LLAMA_CPP_BASE_URL = "http://192.168.15.135:8000/v1"

Test-NetConnection 192.168.15.135 -Port 8000
Invoke-RestMethod `
  -Uri "$env:LLAMA_CPP_BASE_URL/models" `
  -Headers @{ Authorization = "Bearer $env:LLAMA_CPP_API_KEY" }
```

O segundo comando deve listar estes IDs:

- `qwen3.8-27b-iq3_s`
- `qwen3.8-27b-ud-q4_k_m`
- `ornith-1.5-9b-q6_k`
- `gemma-4-12b-it-q5_k_m`
- `gpt-oss-20b-q4_k_m`
- `ornith-1.5-35b-a3b-q4_k_m`

Se `TcpTestSucceeded` for falso, confirme que o Mac está ligado, na mesma rede,
com o serviço ativo, e permita conexões de entrada para Python no Firewall do
macOS. Não desative o firewall inteiro.

## 2. OpenCode

Crie ou ajuste `%USERPROFILE%\.config\opencode\opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llama-cpp": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama.cpp no Mac",
      "options": {
        "baseURL": "http://192.168.15.135:8000/v1",
        "apiKey": "{env:LLAMA_CPP_API_KEY}"
      },
      "models": {
        "qwen3.8-27b-iq3_s": {
          "name": "Qwen 3.8 27B IQ3_S",
          "limit": { "context": 16384, "output": 4096 }
        },
        "ornith-1.5-9b-q6_k": {
          "name": "Ornith 1.5 9B Q6_K",
          "limit": { "context": 32768, "output": 4096 }
        }
      }
    }
  },
  "model": "llama-cpp/qwen3.8-27b-iq3_s",
  "small_model": "llama-cpp/ornith-1.5-9b-q6_k"
}
```

Feche e reabra o terminal depois de definir a variável. Use `/models` no
OpenCode para selecionar um modelo.

## 3. Pi

Em `%USERPROFILE%\.pi\agent\models.json`, registre um provedor OpenAI Chat
Completions. Neste cliente, `apiKey` é o nome da variável de ambiente:

```json
{
  "providers": {
    "llama-cpp": {
      "api": "openai-completions",
      "apiKey": "LLAMA_CPP_API_KEY",
      "baseUrl": "http://192.168.15.135:8000/v1",
      "models": [
        {
          "id": "qwen3.8-27b-iq3_s",
          "name": "Qwen 3.8 27B IQ3_S",
          "contextWindow": 16384,
          "maxTokens": 4096,
          "input": ["text"],
          "reasoning": false
        },
        {
          "id": "ornith-1.5-9b-q6_k",
          "name": "Ornith 1.5 9B Q6_K",
          "contextWindow": 32768,
          "maxTokens": 4096,
          "input": ["text"],
          "reasoning": false
        }
      ]
    }
  }
}
```

Valide com:

```powershell
pi --list-models llama-cpp
pi --provider llama-cpp --model qwen3.8-27b-iq3_s
```

## 4. Crush

No arquivo `%USERPROFILE%\.config\crush\crushrc`, use:

```bash
provider add llama-cpp --name "llama.cpp no Mac" --type openai-compat --base-url "http://192.168.15.135:8000/v1" --api-key "${LLAMA_CPP_API_KEY:?defina LLAMA_CPP_API_KEY}" --discover-models true
model add llama-cpp/qwen3.8-27b-iq3_s --name "Qwen 3.8 27B IQ3_S" --context-window 16384 --default-max-tokens 4096
model add llama-cpp/ornith-1.5-9b-q6_k --name "Ornith 1.5 9B Q6_K" --context-window 32768 --default-max-tokens 4096
model large llama-cpp/qwen3.8-27b-iq3_s --max-tokens 4096
model small llama-cpp/ornith-1.5-9b-q6_k --max-tokens 4096
```

Depois execute `crush models`.

## 5. Cline

Selecione um provedor `OpenAI Compatible` e informe:

- Base URL: `http://192.168.15.135:8000/v1`
- API key: o conteúdo de `~/.config/llama-cpp-server/api-key` do Mac
- Model ID: `qwen3.8-27b-iq3_s`
- Context window: `16384`

Não selecione `Ollama`, pois este servidor é a API do `llama-cpp-python`.

## 6. OMP e SmolCoder

No OMP, adicione o mesmo provedor `openai-completions`, com a URL do Mac, e
inicie-o passando a chave quando necessário:

```powershell
omp --model llama-cpp/qwen3.8-27b-iq3_s --api-key $env:LLAMA_CPP_API_KEY
```

O SmolCoder descobre servidores remotos pela tela `/models`. Informe
`http://192.168.15.135:8000`. Se a versão instalada não oferecer um campo para
API key, não remova a autenticação do servidor: use Cline, Pi, OpenCode, Crush
ou OMP, que suportam a chave explicitamente.

## Diagnóstico rápido

- Modelos não aparecem: URL/porta errada ou o cliente ainda está apontando para
  Ollama (`11434`).
- `401 Invalid API key`: chave ausente ou diferente da chave do Mac.
- O modelo imprime `<tool_call>`: confirme que o servidor do Mac foi reiniciado
  com o handler `native-tools-normalized` e teste novamente.
- O comando não roda apesar de `tool_calls`: habilite as ferramentas e aprove a
  execução no cliente; o servidor propõe a chamada, mas quem executa é o agente
  no Windows.
- Erro de contexto: use o limite real do modelo; o modelo Ornith 35B está
  configurado com apenas `8192` tokens neste Mac.
