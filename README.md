Anotaccoes

b2b058963066d51fcca1841efa4317edbed62075150806c6e55d0233d3e4c6fb


$env:LLAMA_SERVER = "http://192.168.15.135:8000/v1"
$env:LLAMA_API_KEY = "COLE_A_CHAVE_DO_MAC"

curl.exe "$env:LLAMA_SERVER/models" `
  -H "Authorization: Bearer $env:LLAMA_API_KEY"


Configure este servidor llama-cpp-python para que clientes OpenAI compatíveis recebam chamadas de ferramenta estruturadas em `choices[0].message.tool_calls`, em vez de texto literal como `<tool_call>...</tool_call>`.

O modelo atual responde a chat, mas falha num teste com `tool_choice: "required"`: descreve a chamada no campo `content` e deixa `tool_calls` vazio. Verifique o modelo GGUF carregado, os metadados do chat template, a versão do llama-cpp-python e o `chat_format` selecionado. Ajuste o template ou a configuração do servidor para suportar tool calling no formato OpenAI; mantenha o servidor acessível na rede local e preserve a autenticação por API key.

Depois de reiniciar, teste uma requisição que exija uma função simples `run_command` com o argumento `command="echo OK"`. Não execute essa função: confirme apenas que a resposta contém `tool_calls` estruturado e mostre o comando/configuração final usada para iniciar o servidor.
