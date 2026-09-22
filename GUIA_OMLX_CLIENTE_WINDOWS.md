# Guia do oMLX: Mac servidor e Dell Windows cliente

Este guia configura o MacBook como servidor oMLX para um Dell G15 com Windows na mesma rede local. Ele também serve como procedimento de recuperação quando a porta `8000` deixa de responder.

## Arquitetura

- Mac servidor: `192.168.15.135:8000`.
- API OpenAI compatível: `http://192.168.15.135:8000/v1`.
- Cliente: Dell G15 Windows na mesma LAN.
- Autenticação: chave oMLX via cabeçalho `Authorization: Bearer`.
- Modelos: `Ornith-1.5-35B-A3B-MLX-4bit` e `Qwen3.8-27B-4bit`.

> Reserve `192.168.15.135` para o Mac no DHCP do roteador. Se o IP mudar, o servidor não conseguirá usar esse endereço e as URLs do Dell ficarão incorretas.

## 1. Configurar o Mac servidor

No oMLX, use:

- Host: `192.168.15.135` (mais restrito) ou `0.0.0.0` somente em uma LAN confiável.
- Porta: `8000`.
- API key verification: habilitada.
- Memory Guard: `aggressive`.
- Requisições simultâneas: `1`.
- Contexto: `16384`.
- Saída máxima: `4096`.

O oMLX exige uma chave quando escuta em endereço não local. Não desabilite autenticação e não encaminhe a porta `8000` no roteador.

### Iniciar manualmente

```bash
export OMLX_API_KEY="$(security find-generic-password -w -s omlx-api-key)"
omlx serve \
  --host 192.168.15.135 \
  --port 8000 \
  --memory-guard aggressive \
  --max-concurrent-requests 1 \
  --initial-cache-blocks 16 \
  --base-path "$HOME/.omlx"
```

Deixe esse Terminal aberto. Para aceitar conexões em qualquer interface de uma LAN confiável, substitua o host por `0.0.0.0`.

## 2. Verificar o servidor no Mac

```bash
lsof -nP -iTCP:8000 -sTCP:LISTEN
```

Resultado válido para o Dell:

```text
TCP 192.168.15.135:8000 (LISTEN)
```

`127.0.0.1:8000` aceita apenas clientes no próprio Mac e é a causa mais comum da falha remota.

Descubra o IP atual do Wi-Fi:

```bash
ipconfig getifaddr en0
```

Teste localmente sem imprimir a chave:

```bash
curl -H "Authorization: Bearer $OMLX_API_KEY" \
  http://192.168.15.135:8000/v1/models
```

## 3. Firewall e rede

Em **Ajustes do Sistema > Rede > Firewall > Opções**, permita conexões recebidas para o oMLX ou para o executável Python usado por ele.

Confirme também:

- Mac e Dell estão na mesma rede e sub-rede.
- O Wi-Fi não usa isolamento de clientes/AP isolation.
- VPNs estão desligadas durante o diagnóstico.
- Não existe encaminhamento da porta `8000` para a internet.

## 4. Testar pelo Dell

No PowerShell:

```powershell
Test-NetConnection 192.168.15.135 -Port 8000
```

O resultado esperado é `TcpTestSucceeded : True`.

Defina a chave apenas na sessão atual:

```powershell
$env:OMLX_API_KEY = Read-Host "Chave oMLX"
$headers = @{ Authorization = "Bearer $env:OMLX_API_KEY" }
Invoke-RestMethod -Uri "http://192.168.15.135:8000/v1/models" -Headers $headers
```

Teste uma geração:

```powershell
$headers = @{
  Authorization = "Bearer $env:OMLX_API_KEY"
  "Content-Type" = "application/json"
}
$body = @{
  model = "Ornith-1.5-35B-A3B-MLX-4bit"
  messages = @(@{ role = "user"; content = "Responda somente: DELL_OK" })
  max_tokens = 64
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Method Post `
  -Uri "http://192.168.15.135:8000/v1/chat/completions" `
  -Headers $headers `
  -Body $body
```

## 5. OpenCode no Windows

Crie `%USERPROFILE%\.config\opencode\opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "omlx-mac": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "oMLX no Mac",
      "options": {
        "baseURL": "http://192.168.15.135:8000/v1",
        "apiKey": "{env:OMLX_API_KEY}"
      },
      "models": {
        "Ornith-1.5-35B-A3B-MLX-4bit": {
          "limit": { "context": 16384, "output": 4096 }
        },
        "Qwen3.8-27B-4bit": {
          "limit": { "context": 16384, "output": 4096 }
        }
      }
    }
  },
  "model": "omlx-mac/Ornith-1.5-35B-A3B-MLX-4bit",
  "lsp": true
}
```

Abra o OpenCode no mesmo PowerShell em que `OMLX_API_KEY` foi definida.

## 6. Crush no Windows

Crie `%USERPROFILE%\.config\crush\crushrc`:

```bash
provider add omlx-mac --name "oMLX no Mac" --type omlx --base-url "http://192.168.15.135:8000/v1" --api-key "$OMLX_API_KEY"
model large omlx-mac/Ornith-1.5-35B-A3B-MLX-4bit --max-tokens 4096

lsp add typescript --command typescript-language-server --args --stdio \
  --filetypes js --filetypes jsx --filetypes ts --filetypes tsx \
  --root-markers package.json --root-markers tsconfig.json
lsp add java --command jdtls --filetypes java \
  --root-markers pom.xml --root-markers build.gradle --timeout 60
```

Inicie `crush` no PowerShell que contém a chave.

## 7. Pi no Windows

Em `%USERPROFILE%\.pi\agent\models.json`:

```json
{
  "providers": {
    "omlx-mac": {
      "baseUrl": "http://192.168.15.135:8000/v1",
      "api": "openai-completions",
      "apiKey": "OMLX_API_KEY",
      "authHeader": true,
      "models": [
        {
          "id": "Ornith-1.5-35B-A3B-MLX-4bit",
          "name": "Ornith 1.5 35B A3B MLX 4-bit",
          "reasoning": true,
          "input": ["text"],
          "contextWindow": 16384,
          "maxTokens": 4096,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        },
        {
          "id": "Qwen3.8-27B-4bit",
          "name": "Qwen 3.8 27B MLX 4-bit",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 16384,
          "maxTokens": 4096,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

Inicie:

```powershell
pi --model omlx-mac/Ornith-1.5-35B-A3B-MLX-4bit
```

## 8. LSPs de Java e TypeScript

No Dell, instale Node.js LTS e JDK 21 ou superior. Depois:

```powershell
npm install --global typescript typescript-language-server
winget install EclipseAdoptium.Temurin.21.JDK
```

O OpenCode detecta os servidores com `"lsp": true`. O Crush usa as entradas `lsp add`. O Pi precisa de uma extensão LSP de terceiros; revise o código antes de instalar, pois extensões executam com os privilégios do usuário.

## 9. Recuperar a porta 8000

Siga sempre esta ordem:

1. No Dell: `Test-NetConnection 192.168.15.135 -Port 8000`.
2. No Mac: `lsof -nP -iTCP:8000 -sTCP:LISTEN`.
3. No Mac: `ipconfig getifaddr en0` para detectar mudança de IP.
4. Se estiver em `127.0.0.1`, reinicie usando `--host 192.168.15.135`.
5. Se não houver listener, inicie o comando da seção 1.
6. Se o listener estiver correto, revise firewall, VPN e isolamento do Wi-Fi.

Tente primeiro:

```bash
omlx restart
```

Se o servidor foi iniciado diretamente e não for controlado pelo aplicativo:

```bash
lsof -nP -iTCP:8000 -sTCP:LISTEN
kill -TERM PID_EXATO
```

Depois execute novamente o comando completo da seção 1. Encerre somente o PID mostrado para a porta `8000`.

### Interpretação de erros

- `Connection refused`: servidor parado ou ligado a `127.0.0.1`.
- Timeout: firewall, VPN, isolamento Wi-Fi ou redes diferentes.
- HTTP `401`: chave ausente ou incorreta.
- HTTP `404`: use uma rota `/v1/...`, como `/v1/models`.
- Memory Guard: reduza contexto/saída ou libere memória no Mac.
- Primeira chamada lenta: carregamento frio; chamadas seguintes usam cache.

## 10. Instruções para outra IA

Uma IA que opere este ambiente deve:

1. Nunca imprimir nem salvar a chave em arquivos versionados.
2. Confirmar que a rede é privada antes de habilitar acesso remoto.
3. Manter autenticação obrigatória em qualquer endereço não local.
4. Nunca configurar port forwarding para `8000`.
5. Testar `/v1/models` antes de alterar os agentes.
6. Usar contexto `16384`, saída `4096` e uma requisição simultânea.
7. Não manter Ornith e Qwen carregados simultaneamente no Mac de 36 GB.
8. Diagnosticar com `lsof` no Mac e `Test-NetConnection` no Windows.
9. Pedir confirmação antes de alterar firewall, roteador ou serviço persistente.

## Checklist

- [ ] Mac e Dell na mesma rede.
- [ ] IP do Mac reservado no roteador.
- [ ] Listener em `192.168.15.135:8000`, não em `127.0.0.1`.
- [ ] Autenticação habilitada.
- [ ] `Test-NetConnection` retorna `True`.
- [ ] `/v1/models` responde no Dell.
- [ ] Chave armazenada em variável de ambiente.
- [ ] Porta não publicada na internet.
