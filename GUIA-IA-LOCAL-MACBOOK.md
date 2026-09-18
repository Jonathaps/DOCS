# IA local no MacBook com Ollama, Crush e agentes de código

Este guia reproduz no macOS a configuração construída originalmente no Dell com Windows. Ele foi escrito para dois públicos:

- uma pessoa seguindo os comandos manualmente;
- uma IA com acesso ao terminal do Mac, que deve executar cada etapa, validar o resultado e parar diante de divergências.

O cenário principal é um MacBook Pro M3 Pro com 36 GB de memória unificada. O Mac pode operar de duas maneiras:

1. **uso local:** Ollama e o agente de código executam no próprio Mac;
2. **servidor para o Dell:** Ollama executa no Mac, enquanto o agente executa no Dell e usa a API do Mac.

> [!IMPORTANT]
> Um hook pertence ao processo do **agente**, não ao modelo. Se Crush, Cline ou OpenCode executa no Dell, o hook precisa existir no Dell. O Ollama no Mac recebe prompts e devolve tokens; ele não recebe nem executa as chamadas de `git`, shell, edição ou LSP do agente.

## 1. Arquitetura e limites de segurança

### 1.1 Uso inteiramente local no Mac

```text
Crush/Pi/OpenCode no Mac
        │
        ├── hooks, shell, Git, LSP, MCP e arquivos no Mac
        │
        └── http://127.0.0.1:11434 → Ollama → modelo local
```

Nesta arquitetura, instalar o hook no Mac protege as ferramentas executadas pelo agente no Mac.

### 1.2 Dell como cliente de inferência

```text
Dell: agente + hooks + ferramentas ──HTTP──> Mac: Ollama + modelo
```

Aqui, o hook do Mac **não protege** comandos executados no Dell. Mantenha os hooks já instalados no Dell.

### 1.3 Dell como terminal remoto de um agente no Mac

```text
Dell ──SSH/ACP──> Mac: agente + hooks + ferramentas + Ollama
```

Esta é a forma correta de centralizar no Mac tanto a inferência quanto a execução e os guardrails. O Dell passa a ser apenas a interface.

## 2. Pré-requisitos

No Mac, abra o Terminal e confirme a arquitetura:

```bash
uname -m
sw_vers
```

Em um M3, `uname -m` deve retornar `arm64`.

Instale as ferramentas de linha de comando da Apple e o Homebrew, se ainda não existirem:

```bash
xcode-select --install
```

Instale o Homebrew seguindo <https://brew.sh> e carregue-o no `PATH`. Em Apple Silicon, o caminho usual é `/opt/homebrew/bin`.

Depois instale as dependências usadas neste guia:

```bash
brew install git jq node uv
```

Validação:

```bash
git --version
jq --version
node --version
uv --version
```

**Critério de conclusão:** os quatro comandos terminam com código zero.

## 3. Instalar e preparar o Ollama no Mac

Instale o aplicativo oficial em <https://ollama.com/download/mac> e abra-o pelo menos uma vez. Alternativamente, use o método atualmente recomendado na documentação oficial.

Confirme:

```bash
ollama --version
ollama list
```

Baixe pelo menos um modelo adequado a programação. Use o nome exato disponível na biblioteca do Ollama:

```bash
ollama pull NOME_DO_MODELO
ollama run NOME_DO_MODELO
```

Para conferir se a inferência está usando GPU/memória unificada:

```bash
ollama ps
```

O Ollama mantém modelos carregados por cinco minutos por padrão. Para manter um modelo aquecido por mais tempo:

```bash
launchctl setenv OLLAMA_KEEP_ALIVE "30m"
```

Reinicie o aplicativo Ollama depois de alterar variáveis com `launchctl`.

### 3.1 Ajuste conservador para 36 GB

Comece com uma requisição por vez e contexto moderado. Contexto e paralelismo multiplicam o consumo de memória:

```bash
launchctl setenv OLLAMA_NUM_PARALLEL "1"
launchctl setenv OLLAMA_MAX_LOADED_MODELS "1"
launchctl setenv OLLAMA_CONTEXT_LENGTH "32768"
```

Reinicie o Ollama e monitore:

```bash
ollama ps
```

Se houver pressão de memória, reduza primeiro o contexto. Não presuma que o contexto máximo anunciado pelo modelo caberá com bom desempenho.

## 4. Expor o Ollama ao Dell

### Opção A — acesso direto pela LAN

Use somente em uma rede doméstica confiável. A API local do Ollama não deve ser tratada como um serviço autenticado para exposição à internet.

No Mac:

```bash
launchctl setenv OLLAMA_HOST "0.0.0.0:11434"
```

Feche completamente e reabra o Ollama. Descubra o IP do Wi-Fi:

```bash
ipconfig getifaddr en0
```

Se não houver resultado, localize a interface ativa:

```bash
networksetup -listallhardwareports
```

Teste no próprio Mac:

```bash
curl --fail http://127.0.0.1:11434/api/tags
curl --fail http://IP_DO_MAC:11434/api/tags
```

No Dell, em PowerShell, use uma URL normal — sem Markdown, colchetes ou barras invertidas:

```powershell
Invoke-RestMethod -Uri "http://192.168.15.135:11434/api/tags"
$env:OLLAMA_HOST = "http://192.168.15.135:11434"
ollama list
```

Para persistir a variável no Dell:

```powershell
[Environment]::SetEnvironmentVariable(
  "OLLAMA_HOST",
  "http://192.168.15.135:11434",
  "User"
)
```

Abra um terminal novo depois disso.

Se o teste falhar:

1. confirme que o Ollama foi reiniciado após `launchctl setenv`;
2. confirme o IP atual do Mac;
3. confira se Dell e Mac estão na mesma sub-rede;
4. autorize o Ollama no firewall do macOS em **Ajustes do Sistema → Rede → Firewall**;
5. teste a porta no Dell com `Test-NetConnection 192.168.15.135 -Port 11434`.

### Opção B — túnel SSH, recomendada

Esta opção mantém o Ollama escutando apenas em `127.0.0.1` e transporta a conexão por SSH.

No Mac, habilite **Ajustes do Sistema → Geral → Compartilhamento → Acesso Remoto**. No Dell:

```powershell
ssh -N -L 11434:127.0.0.1:11434 USUARIO_DO_MAC@192.168.15.135
```

Enquanto o túnel estiver aberto, em outro terminal do Dell:

```powershell
$env:OLLAMA_HOST = "http://127.0.0.1:11434"
ollama list
```

Se o Dell também executa um Ollama local na porta 11434, use outra porta local:

```powershell
ssh -N -L 11435:127.0.0.1:11434 USUARIO_DO_MAC@192.168.15.135
$env:OLLAMA_HOST = "http://127.0.0.1:11435"
```

**Critério de conclusão:** `ollama list` no Dell apresenta os mesmos modelos do Mac.

## 5. Evitar que o Mac suspenda durante o serviço

Na interface do macOS:

1. abra **Ajustes do Sistema → Bateria → Opções**;
2. ative **Impedir repouso automático no adaptador de energia quando a tela estiver desligada**;
3. configure **Despertar para acesso à rede**;
4. mantenha o Mac ligado à alimentação durante inferências longas.

Para impedir repouso temporariamente enquanto o Ollama estiver sendo usado:

```bash
caffeinate -dimsu
```

O comando mantém a asserção enquanto o terminal estiver aberto. Para envolver um processo específico:

```bash
caffeinate -i ollama serve
```

Inspecione as asserções e o histórico de energia:

```bash
pmset -g assertions
pmset -g log | tail -n 50
```

Evite desativar permanentemente todos os mecanismos de repouso na bateria. A opção gráfica ligada ao adaptador e `caffeinate` são mais fáceis de reverter.

## 6. Instalar e configurar o Crush no Mac

Instale pelo Homebrew:

```bash
brew install charmbracelet/tap/crush
crush --version
mkdir -p ~/.config/crush
```

Crie `~/.config/crush/crushrc`:

```bash
touch ~/.config/crush/crushrc
chmod 600 ~/.config/crush/crushrc
```

Use esta configuração inicial:

```bash
provider add ollama \
  --name "Ollama Local" \
  --type ollama \
  --base-url "http://127.0.0.1:11434/v1/" \
  --discover-models true

option auto-lsp true
```

Execute:

```bash
crush
```

Abra o seletor de modelos e escolha um modelo do provedor `Ollama Local`.

Se os modelos locais desaparecerem:

```bash
curl --fail http://127.0.0.1:11434/api/tags
ollama list
```

Depois confirme que `--discover-models true` permanece no `crushrc`. Uma declaração posterior com o mesmo ID de provedor pode sobrescrever campos anteriores; mantenha uma única fonte de verdade para `provider add ollama`.

### 6.1 Hyper: quando usar

Hyper é o provedor oficial hospedado da Charm para o Crush. Ele oferece modelos remotos, autenticação integrada e uma alternativa quando o modelo local não tem qualidade ou velocidade suficiente. Ele não acelera o Ollama nem transforma o Mac em servidor.

Para trabalho privado e offline, selecione `Ollama Local`. Para tarefas particularmente difíceis, use Hyper conscientemente, sabendo que a inferência deixa de ser local. O Crush permite trocar o modelo sem abandonar a interface.

## 7. LSPs: inteligência estrutural do código

LSP significa **Language Server Protocol**. Um servidor de linguagem fornece definições, referências, símbolos, diagnósticos e informações de tipos. O LSP não é um modelo de IA e não precisa de Ollama.

Instale apenas os servidores das linguagens usadas.

### TypeScript e JavaScript

```bash
npm install -g typescript typescript-language-server
```

No `crushrc`:

```bash
lsp add typescript \
  --command typescript-language-server \
  --args --stdio \
  --filetypes typescript \
  --filetypes javascript \
  --root-markers package.json
```

### Go

O comando `go install ...` exige que a linguagem Go esteja instalada. Go não é necessário para Crush ou Ollama; é necessário apenas para ferramentas escritas/instaladas pelo ecossistema Go, como `gopls`.

```bash
brew install go
go install golang.org/x/tools/gopls@latest
```

Garanta o binário no `PATH`:

```bash
echo 'export PATH="$HOME/go/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

No `crushrc`:

```bash
lsp add go \
  --command "$HOME/go/bin/gopls" \
  --filetypes go \
  --root-markers go.mod
```

**Critério de conclusão:** `command -v typescript-language-server` e, se Go for usado, `command -v gopls` retornam caminhos existentes.

## 8. Semble: busca semântica no código

Semble localiza implementações por significado e pode reduzir buscas repetitivas e contexto enviado ao modelo.

Instalação recomendada:

```bash
uv tool install semble
semble --version
semble install
```

O instalador detecta agentes compatíveis. Para configurar manualmente no Crush, acrescente ao `crushrc`:

```bash
mcp add semble \
  --type stdio \
  --command "$(command -v uvx)" \
  --args --from \
  --args "semble[mcp]" \
  --args semble \
  --timeout 180
```

Teste no repositório desejado:

```bash
cd /caminho/do/projeto
semble search "onde a autenticação é implementada" .
```

O primeiro uso cria o índice; os seguintes reutilizam o cache.

## 9. Skills da Anthropic e outras skills

Skills são diretórios com um arquivo `SKILL.md` que ensina ao agente um fluxo especializado. Elas não treinam nem modificam o modelo.

Clone o repositório:

```bash
mkdir -p ~/.config/crush
git clone https://github.com/anthropics/skills.git \
  ~/.config/crush/anthropic_skills
```

Em vez de depender de um link simbólico ambíguo, registre explicitamente o diretório no `crushrc`:

```bash
option skill-path "$HOME/.config/crush/anthropic_skills/skills"
```

Se preferir o layout do artigo:

```bash
cd ~/.config/crush
ln -s anthropic_skills/skills skills
```

Não execute o `ln -s` novamente se `skills` já existir.

### Muitas skills deixam o modelo local lento?

Podem deixar a experiência mais lenta ou menos precisa, mas não porque todas sejam sempre executadas. Normalmente o agente carrega metadados para descobrir skills e abre o conteúdo completo apenas quando uma skill é escolhida. Os principais custos são:

- mais arquivos para descobrir e indexar na inicialização;
- mais descrições no contexto de seleção;
- maior chance de um modelo local pequeno escolher a skill errada;
- instruções conflitantes entre skills semelhantes.

Comece com um conjunto pequeno e relevante. Desabilite ou remova do caminho de descoberta aquilo que não usa. Para depurar lentidão, inicie o agente sem skills, compare o tempo e reative em grupos.

## 10. RTK: redução de saída de ferramentas

RTK, neste contexto, é **Rust Token Killer**. Ele reescreve comandos suportados para produzir saídas mais compactas. Isso economiza contexto do agente; não aumenta a velocidade de geração do Ollama diretamente.

Instale a versão correta:

```bash
brew install rtk-ai/tap/rtk
rtk --version
rtk gain
```

`rtk gain` deve abrir o painel de economia. Se não abrir, pode ter sido instalado outro projeto também chamado `rtk`.

Integrações oficiais relevantes:

```bash
rtk init --global --agent pi --auto-patch
rtk init --global --opencode --auto-patch
```

Para agentes sem integração oficial, use o hook do Crush apresentado na próxima seção ou uma regra textual. Uma regra textual orienta o modelo; um hook intercepta a ferramenta deterministicamente.

No macOS Apple Silicon, processos iniciados por aplicativos podem não herdar `/opt/homebrew/bin`. Hooks devem usar o caminho absoluto retornado por:

```bash
command -v rtk
```

## 11. Hook global do Crush no Mac

Este hook faz duas coisas:

1. bloqueia operações Git consideradas destrutivas ou remotas;
2. reescreve comandos compatíveis através do RTK.

Crie o diretório:

```bash
mkdir -p ~/.config/agent-hooks
```

Crie `~/.config/agent-hooks/crush-pre-tool-use.sh` com o conteúdo abaixo:

```bash
#!/usr/bin/env bash
set -u

payload="$(cat)"
command_text="$(printf '%s' "$payload" | jq -r '.tool_input.command // empty')"

[[ -z "$command_text" ]] && exit 0

dangerous_git='(^|[;&|][[:space:]]*)git([.]exe)?[[:space:]]+(push([[:space:]]|$)|reset[[:space:]]+--hard([[:space:]]|$)|clean[[:space:]]+[^;&|]*-[^[:space:]]*f|branch[[:space:]]+([^;&|]*[[:space:]])?-D([[:space:]]|$)|checkout[[:space:]]+[.]([[:space:]]|$)|restore[[:space:]]+[.]([[:space:]]|$))'

if [[ "$command_text" =~ $dangerous_git ]]; then
  jq -cn --arg reason "Comando Git destrutivo bloqueado: $command_text" \
    '{decision:"block", reason:$reason}'
  exit 2
fi

rtk_bin="$(command -v rtk 2>/dev/null || true)"
if [[ -z "$rtk_bin" && -x /opt/homebrew/bin/rtk ]]; then
  rtk_bin=/opt/homebrew/bin/rtk
fi
if [[ -z "$rtk_bin" && -x /usr/local/bin/rtk ]]; then
  rtk_bin=/usr/local/bin/rtk
fi

[[ -z "$rtk_bin" ]] && exit 0

rewritten="$($rtk_bin rewrite "$command_text" 2>/dev/null)"
status=$?

if [[ ($status -eq 0 || $status -eq 3) && -n "$rewritten" && "$rewritten" != "$command_text" ]]; then
  jq -cn --arg cmd "$rewritten" \
    '{decision:"allow", updated_input:{command:$cmd}}'
fi

exit 0
```

Torne-o executável:

```bash
chmod 700 ~/.config/agent-hooks/crush-pre-tool-use.sh
```

Acrescente ao `~/.config/crush/crushrc`:

```bash
hook add PreToolUse \
  --matcher "^bash$" \
  --command "$HOME/.config/agent-hooks/crush-pre-tool-use.sh" \
  --name agent-guardrails \
  --timeout 5
```

### Teste isolado do hook

Comando que deve ser bloqueado:

```bash
printf '%s' '{"tool_input":{"command":"git reset --hard HEAD"}}' |
  ~/.config/agent-hooks/crush-pre-tool-use.sh
echo $?
```

O código deve ser `2`.

Comando seguro que pode ser reescrito:

```bash
printf '%s' '{"tool_input":{"command":"git status"}}' |
  ~/.config/agent-hooks/crush-pre-tool-use.sh
echo $?
```

O código deve ser `0`; com RTK ativo, a saída deve conter `updated_input`.

> [!WARNING]
> O hook não é um sandbox. Ele cobre os padrões definidos e pode ser desativado por quem controla a conta do sistema. Para isolamento forte, use permissões do agente, usuário sem privilégios, worktrees, containers ou VMs.

## 12. Hooks em outros agentes no Mac

### Pi

O RTK oferece integração oficial:

```bash
rtk init --global --agent pi --auto-patch
```

O arquivo esperado é `~/.pi/agent/extensions/rtk.ts`. Uma extensão de segurança separada deve interceptar o evento `tool_call` do Bash, mantendo a política independente das atualizações do RTK.

### OpenCode

```bash
rtk init --global --opencode --auto-patch
```

O plugin esperado é `~/.config/opencode/plugins/rtk.ts`. Guardrails adicionais podem usar `tool.execute.before` e rejeitar comandos antes da execução.

### OMP

OMP aceita extensões/hooks por `--hook` ou descoberta em seu diretório de agente. Como versões e caminhos podem mudar, primeiro execute:

```bash
omp --help | grep -E -- '--hook|--extension'
```

Use uma extensão compatível com Pi somente depois de confirmar que a versão instalada expõe a mesma API.

### Qwen Code

Qwen possui `PreToolUse` em `~/.qwen/settings.json`. Registre um hook do tipo `command`, com matcher para `run_shell_command`, e faça o script retornar decisão `deny` e código 2 para operações bloqueadas. Consulte a documentação da versão instalada antes de editar um arquivo que já tenha provedores e credenciais.

### Cline

O Cline CLI descobre hooks globais em `~/.cline/hooks`. A edição do VS Code pode ter uma superfície diferente conforme a versão. Confirme com:

```bash
cline --help | grep -i hook
```

### Continue

Continue não oferece necessariamente uma intercepção global determinística de shell equivalente ao `PreToolUse`. Use uma regra global como defesa orientativa e mantenha aprovações de ferramentas habilitadas. Não classifique uma regra de prompt como barreira de segurança.

### Pool e `ollama launch`

Pool é um cliente ACP. O hook precisa estar no agente/servidor ACP selecionado. Da mesma forma, `ollama launch pi`, `ollama launch omp`, `ollama launch qwen` e comandos semelhantes inicializam outro agente; a capacidade de hook é a capacidade daquele agente, não do Ollama.

## 13. Como colocar o hook “no servidor” para o Dell

Há três interpretações possíveis.

### 13.1 O Dell usa somente a API de inferência do Mac

Não é possível proteger o shell do Dell com um hook no Ollama. Mantenha os hooks no Dell e também instale os hooks no Mac para sessões locais.

### 13.2 O Dell controla um agente que executa no Mac

Instale Crush e o hook no Mac conforme as seções anteriores. No Dell, conecte-se por SSH:

```powershell
ssh USUARIO_DO_MAC@192.168.15.135
```

Na sessão remota:

```bash
cd /caminho/do/projeto
crush
```

Nesse desenho, arquivos, Git, shell, LSP, MCP, RTK e hook executam no Mac. O Dell apenas apresenta o terminal.

Para facilitar, crie no Dell uma função PowerShell:

```powershell
function Enter-MacAI {
  ssh -t USUARIO_DO_MAC@192.168.15.135 'cd /caminho/do/projeto && exec crush'
}
```

### 13.3 O Mac também hospeda o repositório Git remoto

Um hook Git `pre-receive` no Mac pode bloquear force-pushes ou exclusões de branches recebidas pelo servidor. Ele protege o repositório remoto, mas não impede `git reset --hard` ou exclusões locais no Dell.

Em um repositório bare no Mac, crie `REPOSITORIO.git/hooks/pre-receive`:

```bash
#!/usr/bin/env bash
set -euo pipefail

zero=0000000000000000000000000000000000000000

while read -r old new ref; do
  if [[ "$new" == "$zero" ]]; then
    echo "Exclusão de referência bloqueada: $ref" >&2
    exit 1
  fi

  if [[ "$old" != "$zero" ]] && ! git merge-base --is-ancestor "$old" "$new"; then
    echo "Atualização non-fast-forward bloqueada: $ref" >&2
    exit 1
  fi
done
```

Depois:

```bash
chmod 700 REPOSITORIO.git/hooks/pre-receive
```

Teste em um repositório descartável antes de aplicar a um repositório importante.

## 14. Thinking e modelos locais

O modo de thinking depende de três componentes:

1. o modelo precisa suportar raciocínio configurável;
2. o provedor precisa expor o parâmetro correto;
3. o agente precisa permitir alterá-lo.

Não existe um seletor universal no Crush que obrigue qualquer modelo Ollama a raciocinar mais. Alguns agentes oferecem `--thinking low|medium|high`; outros dependem do modelo ou de `extra_body` do provedor. Verifique `crush --help`, a documentação do modelo e os dados enviados ao endpoint antes de assumir que a opção teve efeito.

Thinking maior normalmente aumenta latência e consumo de tokens. Para um M3 Pro de 36 GB, use baixo/médio no trabalho cotidiano e aumente apenas para planejamento, diagnóstico difícil ou refatoração ampla.

## 15. Checklist de validação

Execute na ordem:

```bash
# Ollama e modelo
ollama list
curl --fail http://127.0.0.1:11434/api/tags

# Ferramentas
crush --version
rtk --version
rtk gain
uvx --version

# Configuração
test -r ~/.config/crush/crushrc
test -x ~/.config/agent-hooks/crush-pre-tool-use.sh

# Hook: bloqueado deve retornar 2
printf '%s' '{"tool_input":{"command":"git push origin main"}}' |
  ~/.config/agent-hooks/crush-pre-tool-use.sh
test $? -eq 2

# Hook: seguro deve retornar 0
printf '%s' '{"tool_input":{"command":"git status"}}' |
  ~/.config/agent-hooks/crush-pre-tool-use.sh
test $? -eq 0
```

No Dell:

```powershell
Test-NetConnection 192.168.15.135 -Port 11434
Invoke-RestMethod -Uri "http://192.168.15.135:11434/api/tags"
ollama list
```

Antes de considerar a instalação concluída, abra o Crush no Mac, selecione um modelo Ollama e peça uma operação somente de leitura, como listar arquivos ou executar `git status`. Em seguida, em um repositório descartável, peça explicitamente um comando bloqueado e confirme que a ferramenta falha sem executá-lo.

## 16. Procedimento para uma IA implementar este guia

Uma IA com acesso ao terminal deve seguir estas regras:

1. detectar o que já está instalado antes de instalar;
2. fazer backup dos arquivos de configuração existentes;
3. mesclar configurações, preservando provedores, modelos, MCPs, LSPs e hooks já existentes;
4. usar caminhos absolutos para executáveis chamados por hooks;
5. nunca copiar segredos para logs ou respostas;
6. não expor a porta 11434 fora da LAN sem autenticação e TLS em uma camada intermediária;
7. testar o hook com comandos sintéticos antes de abrir o agente;
8. testar comandos destrutivos apenas em repositórios temporários;
9. relatar separadamente integrações determinísticas e regras apenas orientativas;
10. terminar somente quando todos os critérios da seção 15 passarem ou quando houver um bloqueio concreto documentado.

## 17. Referências

- [Ollama: configuração, rede, contexto e memória](https://docs.ollama.com/faq)
- [Ollama: integrações e `ollama launch`](https://docs.ollama.com/integrations)
- [Crush: configuração](https://github.com/charmbracelet/crush/blob/main/docs/config/README.md)
- [Crush: hooks](https://github.com/charmbracelet/crush/blob/main/docs/hooks/README.md)
- [Crush: instalação e Hyper](https://github.com/charmbracelet/crush/blob/main/README.md)
- [RTK: instalação](https://github.com/rtk-ai/rtk/blob/develop/docs/guide/getting-started/installation.md)
- [RTK: agentes suportados](https://github.com/rtk-ai/rtk/blob/develop/docs/guide/getting-started/supported-agents.md)
- [Semble: instalação](https://github.com/MinishLab/semble/blob/main/docs/installation.md)
- [Apple: ajustes de repouso e acesso pela rede](https://support.apple.com/guide/mac-help/mchle41a6ccd/mac)
