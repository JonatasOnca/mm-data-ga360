# Relatório de Adequação ao Padrão — Projeto GCP `mm-data-ga360`

> **Data da análise:** 2026-03-19  
> **Analista:** jonatas.onca-santo@madeiramadeira.com.br  
> **Projeto:** `mm-data-ga360` (nº 239438646710, criado em 2022-07-06)  
> **Status do projeto:** ACTIVE | Pasta pai: `740742580703`

---

## Resumo Executivo

O projeto `mm-data-ga360` está funcional e em produção, mas apresenta **gaps significativos** em segurança, governança, observabilidade e padronização de infraestrutura. Os problemas mais críticos são a política de IAM excessivamente permissiva (usuários individuais com `roles/owner` e `roles/editor`) e a ausência completa de metadados nos recursos de BigQuery e Cloud Storage.

| Pilar | Severity | Nº de Achados |
|---|---|---|
| IAM & Segurança | 🔴 Crítico | 4 |
| Governança BigQuery | 🟠 Alto | 5 |
| Cloud Storage | 🟡 Médio | 4 |
| Compute & Orquestração | 🟡 Médio | 3 |
| Observabilidade | 🔴 Crítico | 2 |
| Tagging & Rastreabilidade | 🟡 Médio | 3 |
| **TOTAL** | | **21** |

---

## 1. IAM & Segurança 🔴

### 1.1 — [CRÍTICO] 9 usuários individuais com `roles/owner`
**Achado:** Os seguintes usuários possuem a role primitiva `roles/owner` atribuída diretamente, dando acesso irrestrito ao projeto:
- `david.ortiz`, `felipe.schmidt@bulkylog.com.br`, `gil.pinto-solution`, `guilherme.dio`, `guilherme.selonke`, `jorge.sanfelice`, `marcio.borges`, `ruthe.nazareth`, `walter.costa`

> [!CAUTION]
> `roles/owner` concede controle total incluindo modificação de IAM, faturamento e exclusão do projeto. Jamais deve ser atribuída a usuários individuais em produção.

**Ação requerida:**
1. Criar grupo `gcp-owners-mmdataga360@madeiramadeira.com.br` e adicionar apenas 1-2 responsáveis.
2. Remover a role `owner` de todos os usuários individuais.
3. Substituir pelos papéis granulares necessários (ex: `roles/bigquery.admin`, `roles/dataform.editor`).

```bash
# Remover owner de um usuário (repetir para cada um)
gcloud projects remove-iam-policy-binding mm-data-ga360 \
  --member="user:guilherme.selonke@madeiramadeira.com.br" \
  --role="roles/owner"
```

---

### 1.2 — [CRÍTICO] 8 usuários individuais com `roles/editor`
**Achado:** Os usuários abaixo possuem `roles/editor` (acesso de leitura/escrita em todos os recursos):
- `adriane.graciano`, `alexandre.gonzaga`, `augusto.tanaka`, `cleverson.casagrande`, `guilherme.selonke`, `marcus.mendes`, `matheus.lopes-solution`, `stella.silva`

> [!WARNING]
> `roles/editor` concede permissão para modificar qualquer recurso do projeto, incluindo dados de produção. Viola o princípio do menor privilégio.

**Ação requerida:** Substituir `roles/editor` por roles granulares por função:
- Analistas BQ → `roles/bigquery.dataViewer` + `roles/bigquery.jobUser`
- Engenheiros de dados → `roles/bigquery.dataEditor` + `roles/bigquery.jobUser`
- Devs Cloud Run/Functions → roles específicas do serviço

---

### 1.3 — [ALTO] 1 usuário externo com `roles/owner`
**Achado:** `felipe.schmidt@bulkylog.com.br` é de domínio externo (`bulkylog.com.br`) e possui `roles/owner`.

**Ação requerida:** Verificar se o acesso é legítimo. Se não for mais necessário, revogar imediatamente. Se for, criar SA dedicada com escopo mínimo.

---

### 1.4 — [MÉDIO] Service Account desabilitada em uso histórico
**Achado:** `teste-bq-looker@mm-data-ga360.iam.gserviceaccount.com` está **DISABLED** mas existiu como SA de teste.

**Ação requerida:** Deletar a SA `teste_bq_looker` após confirmar que não há chaves ativas nem integrações dependentes.

```bash
gcloud iam service-accounts delete teste-bq-looker@mm-data-ga360.iam.gserviceaccount.com
```

---

## 2. Governança BigQuery 🟠

O projeto possui **51 datasets** no BigQuery. Nenhum deles possui `labels`, `description` ou política de expiração configurada.

### 2.1 — [ALTO] Ausência de labels em todos os datasets (51/51)
**Achado:** Nenhum dataset possui labels (`labels: None`). Isso impede rastreamento de custos, automações e categorização.

**Ação requerida:** Definir e aplicar taxonomia de labels obrigatória. Sugestão:
```
env: production | staging | development
layer: raw | staging | intermediate | output | sandbox
owner: <squad>
```
```bash
bq update --set_label env:production mm-data-ga360:digital_analytics
bq update --set_label layer:output mm-data-ga360:digital_analytics
```

---

### 2.2 — [ALTO] Ausência de description em todos os datasets
**Achado:** Nenhum dos 51 datasets possui descrição, dificultando o entendimento do propósito de cada dataset.

**Ação requerida:** Documentar cada dataset com pelo menos: propósito, fonte de dados, time responsável e frequência de atualização.

---

### 2.3 — [ALTO] Datasets de staging/temp sem política de expiração
**Achado:** Os datasets abaixo claramente são temporários mas não possuem `defaultTableExpirationMs`:
- `sessions_temp`, `search_temp`, `temp_spark_cubo_midia`, `temp_spark_cubo_midia_attributed_events`, `temp_spark_cubo_midia_product_views`, `temp_spark_cubo_midia_sessao`, `sandbox_optim`

**Ação requerida:** Configrar expiração de tabelas para datasets temporários:
```bash
# 30 dias de expiração = 2592000000 ms
bq update --default_table_expiration 2592000000 mm-data-ga360:sessions_temp
bq update --default_table_expiration 2592000000 mm-data-ga360:search_temp
```

---

### 2.4 — [MÉDIO] Datasets de "teste" em ambiente de produção
**Achado:** Datasets com naming de teste/sandbox:
- `teste_uncover`, `cubo_de_estudos`, `sandbox_optim`, `black_friday`, `black_friday_2023_app`

**Ação requerida:** Avaliar se esses datasets ainda são necessários. Se não forem, remover. Se estiverem ativos, renomear e documentar adequadamente.

---

### 2.5 — [MÉDIO] Ausência de nomenclatura padronizada por camada
**Achado:** Não existe um padrão claro de nomes. Coexistem: `analytics_ga360`, `digital_analytics`, `prd_analytics_ga360`, `analytics_217280870`, `analytics_448128076` sem hierarquia clara.

**Ação requerida:** Definir convenção de nomenclatura por camada (ex: `raw_`, `stg_`, `int_`, `out_`, `prd_`) e aplicar nos novos datasets. Documentar datasets legados.

---

## 3. Cloud Storage (GCS) 🟡

O projeto possui **18 buckets**. Nenhum possui versionamento, lifecycle, logging ou labels.

### 3.1 — [ALTO] Nenhum bucket com versionamento habilitado
**Achado:** `Versioning enabled: None` em todos os buckets analisados (`cubo_midia`, `outbound_ga`, `ga4_purchase_source_medium`, etc.).

**Ação requerida:** Habilitar versionamento nos buckets com dados críticos:
```bash
gsutil versioning set on gs://cubo_midia
gsutil versioning set on gs://outbound_ga
```

---

### 3.2 — [MÉDIO] Nenhum bucket com política de ciclo de vida
**Achado:** `Lifecycle configuration: None` em todos os buckets. Dados antigos nunca são movidos para classes mais baratas ou deletados.

**Ação requerida:** Criar políticas de lifecycle. Exemplo — mover para Coldline após 90 dias:
```json
{
  "rule": [{
    "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
    "condition": {"age": 90}
  }]
}
```

---

### 3.3 — [MÉDIO] Nenhum bucket com labels
**Achado:** `Labels: None` em todos os buckets. Impossível rastrear custos por produto/squad.

**Ação requerida:**
```bash
gsutil label ch -l env:production -l layer:raw gs://cubo_midia
```

---

### 3.4 — [BAIXO] Buckets com nome sem prefixo de projeto
**Achado:** Nomes como `gs://blackfriday_2021/` e `gs://blackfriday_2023/` não seguem convenção `mm-` ou `mm-data-` que identifica o projeto.

**Ação requerida:** Para novos buckets, adotar convenção `mm-data-<propósito>-<env>`.

---

## 4. Compute & Orquestração 🟡

### 4.1 — [MÉDIO] 4 Cloud Functions ainda em 1st gen (obsoleto)
**Achado:** Das 5 Cloud Functions, 4 estão em 1st gen:
- `print` (HTTP, 1st gen)
- `send_alert_data_transfer` (Event, 1st gen)
- `send_slack_notification-dev` (HTTP, 1st gen)
- `teste-billing` (HTTP, 1st gen)

> [!WARNING]
> O Google anunciou deprecação do Cloud Functions 1st gen. A migração para 2nd gen (ou Cloud Run) é recomendada.

**Ação requerida:** Migrar as functions para 2nd gen ou consolidar em Cloud Run (já em uso para `send-google-chat-notification` e `ga4-etl-schedules`).

---

### 4.2 — [MÉDIO] Cloud Function de desenvolvimento (`send_slack_notification-dev`) em produção
**Achado:** Existe uma function com sufixo `-dev` no projeto de produção `mm-data-ga360`.

**Ação requerida:** Remover recursos de desenvolvimento do projeto de produção. Criar projeto separado `mm-data-ga360-dev`.

---

### 4.3 — [BAIXO] Container Registry legado em uso paralelo ao Artifact Registry
**Achado:** O projeto usa tanto `gs://us.artifacts.mm-data-ga360.appspot.com/` (Container Registry legado, deprecated) quanto Artifact Registry (`cloud-run-source-deploy`, `gcf-artifacts`).

**Ação requerida:** Verificar se ainda há imagens sendo publicadas no Container Registry legado e migrar todo o fluxo para Artifact Registry.

---

## 5. Observabilidade 🔴

### 5.1 — [CRÍTICO] Nenhuma política de alertas de monitoramento configurada
**Achado:** O Cloud Monitoring não possui nenhuma política de alerta configurada (0 alertas).

> [!CAUTION]
> Sem alertas, falhas em pipelines, excesso de custos ou indisponibilidades não serão detectados proativamente.

**Ação requerida:** Criar alertas mínimos de:
- Falhas em Cloud Functions / Cloud Run (error rate > 0)
- Erro em BigQuery Data Transfer
- Gasto acima do orçamento esperado (via Budget Alerts no Billing)

---

### 5.2 — [CRÍTICO] Ausência de Secret Manager para credenciais
**Achado:** O Secret Manager está habilitado (`secretmanager.googleapis.com`) mas possui 0 secrets cadastrados.

**Pergunta crítica:** Como as credenciais sensíveis (tokens de API, senhas de BD, chaves de integração Segment, Uncover, Grafana etc.) estão sendo gerenciadas?

**Ação requerida:**
- Auditar todas as Cloud Functions, Cloud Run services e SAs para localizar credenciais hard-coded ou em variáveis de ambiente.
- Migrar todas as credenciais para o Secret Manager.
- Usar referências a secrets nas funções/services em vez de valores diretos.

---

## 6. Tagging & Rastreabilidade 🟡

### 6.1 — [MÉDIO] Projeto sem tag `environment`
**Achado:** O GCP emitiu aviso explícito: _"Project 'mm-data-ga360' lacks an 'environment' tag"_.

**Ação requerida:**
```bash
gcloud resource-manager tags bindings create \
  --tag-value=tagValues/production \
  --parent=//cloudresourcemanager.googleapis.com/projects/239438646710
```

---

### 6.2 — [MÉDIO] Labels do projeto não propagadas para os recursos
**Achado:** O projeto possui labels (`cc: 10104612`, `ccdesc: data_analytics`, `dev_ops: anderson_lepeco`, `dir: rafael_castellar`, `em: milena_ramirez`, `sup: sup-65165`), mas nenhum recurso interno (BQ, GCS, Functions) espelha essas labels para rastreamento de custos.

**Ação requerida:** Definir política de labels obrigatória e aplicar via automação (Terraform, script de bootstrap).

---

### 6.3 — [BAIXO] Repositório Git local vazio
**Achado:** O repositório Git em `/mm-data-ga360` não possui nenhum commit. O projeto existe desde 2022 sem versionamento de código no repositório.

**Ação requerida:** Definir o que deve ser versionado neste repositório:
- Configurações Terraform/IaC dos recursos GCP?
- DAGs do Airflow/Composer?
- Código do Dataform?

Fazer o commit inicial com a estrutura acordada.

---

## Plano de Priorização

| Prioridade | Achado | Esforço |
|---|---|---|
| 🔴 P0 | Remover `roles/owner` de usuários individuais | Baixo |
| 🔴 P0 | Remover `roles/editor` de usuários individuais | Baixo |
| 🔴 P0 | Revogar acesso externo (`bulkylog.com.br`) | Baixo |
| 🔴 P0 | Criar alertas de monitoramento mínimos | Médio |
| 🔴 P0 | Auditar e migrar credenciais para Secret Manager | Alto |
| 🟠 P1 | Aplicar labels e descriptions nos 51 datasets BQ | Médio |
| 🟠 P1 | Configurar expiração em datasets temp/sandbox | Baixo |
| 🟡 P2 | Habilitar versionamento nos buckets críticos | Baixo |
| 🟡 P2 | Migrar Cloud Functions para 2nd gen | Médio |
| 🟡 P2 | Adicionar tag `environment` ao projeto | Baixo |
| 🟡 P2 | Configurar lifecycle nos buckets GCS | Médio |
| 🟢 P3 | Padronizar nomenclatura dos datasets BQ | Alto |
| 🟢 P3 | Migrar do Container Registry legado para Artifact Registry | Baixo |
| 🟢 P3 | Popular e estruturar repositório Git | Alto |

---

## Contexto do Projeto (Para Referência)

| Atributo | Valor |
|---|---|
| Project ID | `mm-data-ga360` |
| Project Number | `239438646710` |
| Criado em | 2022-07-06 |
| Origem dos dados | GA360 + GA4 + Google Ads + Segment + Merchant Center |
| Tecnologias em uso | BigQuery, Dataform, Cloud Run, Cloud Functions, GCS, Vertex AI, BigQuery Omni (AWS) |
| Integrações externas | Databricks (Unity Catalog), Grafana, Looker, Metabase, DataHub, Segment, Uncover |
| Datasets BigQuery | 51 |
| GCS Buckets | 18 |
| Service Accounts | 22 (1 desabilitada) |
| Cloud Functions | 5 (4 em 1st gen, 1 em 2nd gen) |
| Cloud Run Services | 2 |
