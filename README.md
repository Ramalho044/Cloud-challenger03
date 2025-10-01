# Cloud Challenger 03 — Entrega (LorArch)

## ✅ Visão Geral
- **RG:** rg-LorArch (Brazil South)
- **Azure SQL:** servidor `lorach-sql`, base `lorarchdb`
- **ACR:** `lorarch` (`lorarch.azurecr.io`)
- **ACI:** `nginx-demo` (deploy público para evidência)

## 🔹 Banco de Dados (Azure SQL)
- Evidências:
  - Objetos (tabelas/views): `evidencias/sql_objs.txt`
  - TOP 5 MOTOCICLETA: `evidencias/sql_top5_motocicleta.txt`
  - Contagem por tabela: `evidencias/sql_counts.txt`

## 🔹 ACR
- Repositórios: `evidencias/acr_repos.txt`
- Tags `nginx`: `evidencias/acr_tags_nginx.txt`
- Tags `ping`: `evidencias/acr_tags_ping.txt`
- (se existir) Tags `lorarch-api`: `evidencias/acr_tags_lorarch_api.txt`

## 🔹 ACI
- Estado: `evidencias/aci_state.txt`
- FQDN: `evidencias/aci_fqdn.txt`
- Resposta HTTP: `evidencias/aci_nginx_response.txt`

## 🔹 Como reproduzir
1) `az acr login -n lorarch`  
2) `az container show -g rg-LorArch -n nginx-demo --query "ipAddress.fqdn" -o tsv`  
3) Consultas SQL via `sqlcmd` conforme arquivos em `evidencias/`.

> Observação: O fluxo ACR ➜ ACI está validado via Nginx. A imagem `lorarch-api` será publicada quando o build/JAR for concluído.
