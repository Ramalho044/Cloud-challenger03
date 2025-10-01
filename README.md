 ☁️ Cloud Challenger 03 — Entrega Final (LorArch)

> **Aluno:** Marcos Ramalho (Ramalho044)  
> **Região Azure:** Brazil South  
> **RG:** `rg-LorArch`  
> **Data da Entrega:** 2025-10-01

---

## ✅ Arquitetura entregue

**Banco de Dados → API → Container → ACR → ACI (público)**

- **Azure SQL**
  - **Servidor:** `lorach-sql.database.windows.net`
  - **Banco:** `lorarchdb`
  - **Usuário app:** `lorarch_app`
- **Azure Container Registry (ACR)**
  - **Nome:** `lorarch`
  - **Login server:** `lorarch.azurecr.io`
  - **Repositórios (evidência):** `nginx`, `ping`, `lorarch-api`
- **Azure Container Instances (ACI)**
  - **Container:** `lorarch-api` (exposição pública na porta 8080)
- **API Java / Spring Boot**
  - Build com **Gradle/Maven**
  - **Imagem publicada:** `lorarch.azurecr.io/lorarch-api:v1`

> Todas as **evidências de execução** estão na pasta `evidencias/` (consultas SQL, ACR, ACI, resposta HTTP).

---

## 📂 Evidências incluídas

- `evidencias/sql_objs.txt` — Objetos do banco (`sys.objects`: tabelas e views)
- `evidencias/sql_top5_motocicleta.txt` — `SELECT TOP 5 * FROM MOTOCICLETA`
- `evidencias/sql_counts.txt` — contagem por tabela
- `evidencias/acr_repos.txt` — repositórios no ACR
- `evidencias/acr_tags_nginx.txt` — tags do repositório `nginx`
- `evidencias/acr_tags_ping.txt` — tags do repositório `ping`
- `evidencias/acr_tags_lorarch_api.txt` — tags do repositório `lorarch-api`
- `evidencias/aci_state.txt` — estado do ACI (`Running`)
- `evidencias/aci_fqdn.txt` — FQDN público do container
- `evidencias/aci_nginx_response.txt` — resposta HTTP de teste

> As capturas de tela adicionais estão na pasta `prints/` (quando aplicável).

---

## 🗄️ Banco de Dados (Azure SQL)

### Objetos criados (resumo)
**Tabelas**: `ESTADO`, `CIDADE`, `UNIDADE_FISICA`, `SETOR_DA_UNIDADE`,  
`MOTOCICLETA`, `DEFEITO_MOTOCICLETA`, `MANUTENCAO_MOTOCICLETA`,  
`RELACAO_MOTOCICLETA_DEFEITO`, `HISTORICO_MOVIMENTACAO_MOTOCICLETA`,  
`LOG_AUDITORIA_MOTOCICLETA`, `LEITOR_RFID`, `DADOS_RFID`, `DADOS_LORA`.

**Views**: `VW_MOTOCICLETA_DEFEITOS_ATIVOS`, `VW_HIST_MOV_MOTOCICLETA_RESUMO`  
**Procedures**: `usp_RegistrarMovimentacao`, `usp_AtualizarStatusMotocicleta`  
**Triggers**: `trg_Moto_Touch`, `trg_Log_Movimentacao`, `trg_Log_Status`  
**Carga inicial**: estados, cidades, unidades, setores, motos, defeitos, manutenções, RFID, LORA.

> O script completo usado na criação/carga está versionado no repositório (ex.: `db/script_bd_mssql.sql`).

### Consultas de evidência (como foram geradas)
```bash
# Objetos (tabelas e views)
sqlcmd -C -S lorach-sql.database.windows.net -d lorarchdb -U <SQL_ADMIN> -P '<SQL_ADMIN_PASSWORD>' \
 -Q "SELECT name,type_desc FROM sys.objects WHERE type IN ('U','V') ORDER BY type;" \
 | tee evidencias/sql_objs.txt

# TOP 5 da tabela MOTOCICLETA
sqlcmd -C -S lorach-sql.database.windows.net -d lorarchdb -U <SQL_ADMIN> -P '<SQL_ADMIN_PASSWORD>' \
 -Q "SELECT TOP 5 * FROM MOTOCICLETA;" \
 | tee evidencias/sql_top5_motocicleta.txt

# Contagem por tabela
sqlcmd -C -S lorach-sql.database.windows.net -d lorarchdb -U <SQL_ADMIN> -P '<SQL_ADMIN_PASSWORD>' \
 -Q "SELECT t.name, p.rows FROM sys.tables t JOIN sys.partitions p ON t.object_id=p.object_id AND p.index_id IN (0,1) GROUP BY t.name,p.rows ORDER BY t.name;" \
 | tee evidencias/sql_counts.txt

🧱 Build da API & Docker
Dockerfile (multi-stage)

# ===== build (Gradle wrapper + JDK 17) =====
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY gradlew ./
COPY gradle ./gradle
COPY build.gradle* settings.gradle* ./
RUN chmod +x ./gradlew
RUN ./gradlew dependencies --no-daemon || true
COPY src ./src
RUN ./gradlew clean bootJar --no-daemon

# ===== runtime (JRE 17) =====
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/build/libs/*.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]

Build local da imagem

# (na raiz do projeto, onde está o Dockerfile)
docker build -t lorarch.azurecr.io/lorarch-api:v1 .

📦 ACR — Azure Container Registry
Login, push e listagens

# Login
az acr login -n lorarch

# Push da imagem
docker push lorarch.azurecr.io/lorarch-api:v1

# Repositórios
az acr repository list -n lorarch -o table | tee evidencias/acr_repos.txt

# Tags (exemplos)
az acr repository show-tags -n lorarch --repository nginx -o table | tee evidencias/acr_tags_nginx.txt
az acr repository show-tags -n lorarch --repository ping  -o table | tee evidencias/acr_tags_ping.txt
az acr repository show-tags -n lorarch --repository lorarch-api -o table | tee evidencias/acr_tags_lorarch_api.txt

    Também validamos fluxo ACR com imagens de teste (nginx, ping).

🚀 ACI — Azure Container Instances (deploy público)
Variáveis de conexão (Spring)

# JDBC da aplicação (substitua as credenciais do app user)
export CONNECTION="jdbc:sqlserver://lorach-sql.database.windows.net:1433;database=lorarchdb;user=lorarch_app;password=<APP_USER_PASSWORD>;encrypt=true;trustServerCertificate=false;loginTimeout=30;"

Credenciais do ACR para o ACI

ACR_USER=$(az acr credential show -n lorarch --query username -o tsv)
ACR_PASS=$(az acr credential show -n lorarch --query passwords[0].value -o tsv)

Criação do container

DNS_LABEL="lorarch-api-$RANDOM"

az container create \
  -g rg-LorArch -l brazilsouth \
  -n lorarch-api \
  --image lorarch.azurecr.io/lorarch-api:v1 \
  --os-type Linux \
  --registry-login-server lorarch.azurecr.io \
  --registry-username "$ACR_USER" \
  --registry-password "$ACR_PASS" \
  --dns-name-label "$DNS_LABEL" \
  --ports 8080 \
  --cpu 1 --memory 1.5 \
  --secure-environment-variables \
    SPRING_DATASOURCE_URL="$CONNECTION" \
    SPRING_DATASOURCE_USERNAME="lorarch_app" \
    SPRING_DATASOURCE_PASSWORD="<APP_USER_PASSWORD>"

Evidências do ACI

# Estado
az container show -g rg-LorArch -n lorarch-api --query "instanceView.state" -o tsv | tee evidencias/aci_state.txt

# FQDN público
az container show -g rg-LorArch -n lorarch-api --query "ipAddress.fqdn" -o tsv | tee evidencias/aci_fqdn.txt

# Teste HTTP (se a aplicação expõe endpoint público)
FQDN=$(az container show -g rg-LorArch -n lorarch-api --query "ipAddress.fqdn" -o tsv)
curl -s "http://$FQDN:8080/actuator/health" | tee evidencias/aci_nginx_response.txt

🔁 Como reproduzir do zero

# 0) Autenticação e assinatura
az login
az account set --subscription "<SUA_ASSINATURA>"

# 1) Banco de Dados (opcional se já estiver criado)
sqlcmd -C -S lorach-sql.database.windows.net -d lorarchdb -U <SQL_ADMIN> -P '<SQL_ADMIN_PASSWORD>' -i db/script_bd_mssql.sql

# 2) Build da API (Gradle)
chmod +x gradlew
./gradlew clean bootJar --no-daemon
ls -lh build/libs/

# 3) Docker build
az acr login -n lorarch
docker build -t lorarch.azurecr.io/lorarch-api:v1 .

# 4) Push para o ACR
docker push lorarch.azurecr.io/lorarch-api:v1

# 5) Deploy no ACI
ACR_USER=$(az acr credential show -n lorarch --query username -o tsv)
ACR_PASS=$(az acr credential show -n lorarch --query passwords[0].value -o tsv)

CONNECTION="jdbc:sqlserver://lorach-sql.database.windows.net:1433;database=lorarchdb;user=lorarch_app;password=<APP_USER_PASSWORD>;encrypt=true;trustServerCertificate=false;loginTimeout=30;"
DNS_LABEL="lorarch-api-$RANDOM"

az container create \
  -g rg-LorArch -l brazilsouth \
  -n lorarch-api \
  --image lorarch.azurecr.io/lorarch-api:v1 \
  --os-type Linux \
  --registry-login-server lorarch.azurecr.io \
  --registry-username "$ACR_USER" \
  --registry-password "$ACR_PASS" \
  --dns-name-label "$DNS_LABEL" \
  --ports 8080 \
  --cpu 1 --memory 1.5 \
  --secure-environment-variables \
    SPRING_DATASOURCE_URL="$CONNECTION" \
    SPRING_DATASOURCE_USERNAME="lorarch_app" \
    SPRING_DATASOURCE_PASSWORD="<APP_USER_PASSWORD>"

# 6) Verificação
az container show -g rg-LorArch -n lorarch-api --query "instanceView.state" -o tsv
az container show -g rg-LorArch -n lorarch-api --query "ipAddress.fqdn" -o tsv

🧾 Checklist de avaliação

Modelagem de dados completa (tabelas, views, procedures, triggers)

Carga inicial e consultas de evidência

API Java buildada com Gradle (ou Maven fallback)

Dockerfile multi-stage (build + runtime)

Push no ACR e listagens de repositórios/tags

Deploy no ACI com FQDN público e testes de saúde

Evidências versionadas em evidencias/

