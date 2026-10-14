Anotaccoes

b2b058963066d51fcca1841efa4317edbed62075150806c6e55d0233d3e4c6fb


$env:LLAMA_SERVER = "http://192.168.15.135:8000/v1"
$env:LLAMA_API_KEY = "COLE_A_CHAVE_DO_MAC"

curl.exe "$env:LLAMA_SERVER/models" `
  -H "Authorization: Bearer $env:LLAMA_API_KEY"
