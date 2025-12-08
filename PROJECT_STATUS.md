# Self-hosted AI Starter Kit – Project Overview & Progress

## Objetivo

Preparar um ambiente local, baseado no template de docker compose da n8n, para:

1. Executar o n8n com recursos de IA/automação (Ollama, Qdrant, Postgres).
2. Montar fluxos prontos para uso, incluindo:
   - Demo original do repositório (chat com Llama 3.2).
   - Workflow customizado para enviar cursos locais ao Google Drive.

## Componentes principais configurados

| Serviço | Papel | Status |
| ------- | ----- | ------ |
| n8n | Editor e executor de fluxos | Rodando em `http://localhost:5678` |
| Postgres | Base para n8n e logging | Tabela `course_uploads` criada |
| Qdrant | Vetor store para fluxos IA | Rodando (`http://localhost:6333`) |
| Ollama | Modelo Llama 3.2 local | Modelo baixado (2 GB) e carregado |
| Pastas compartilhadas | `/data/shared/...` | `shared/cursos` e `shared/tmp` criadas |

## Ações concluídas até agora

### Infraestrutura
- Contêineres `postgres`, `n8n`, `qdrant` e `ollama` estão ativos, com o Demo workflow disponível.
- Variáveis de ambiente básicas (credenciais do n8n/Postgres). Adicionada variável `GOOGLE_DRIVE_COURSES_FOLDER_ID` no `.env`.
- Tabela `course_uploads` criada em Postgres para registrar uploads de cursos.

### Automatização de upload de cursos
1. Estrutura local:
   - `shared/cursos` para armazenar os diretórios de cursos.
   - `shared/tmp` para gerar arquivos `.tar.gz` temporários antes do upload.
2. Documentação detalhada (`COURSE_UPLOAD_WORKFLOW.md`) com todos os nós necessários e instruções de teste.
3. Workflow pronto para importação: `n8n/demo-data/workflows/uploadLocalCourses.json`
   - Cron diário → listar cursos locais → checar Drive e Postgres → criar .tar.gz → enviar para o Google Drive → registrar no banco → limpar temporários.
   - Usa placeholders para as credenciais do Google Drive e Postgres, bastando atribuí-las após importar no n8n.

### Estado atual do repositório

```
.env / .env.example     # Podem receber o ID da pasta no Drive
shared/
 ├─ cursos/             # Onde os cursos devem ser colocados
 └─ tmp/                # Arquivos temporários durante o upload
COURSE_UPLOAD_WORKFLOW.md
n8n/demo-data/workflows/
 ├─ srOnR8PAY3u4RSwb.json           # Workflow demo original
 └─ uploadLocalCourses.json         # Automação de backup para o Drive
PROJECT_STATUS.md
```

## Próximos passos sugeridos

1. **Preencher variáveis e credenciais**  
   - Definir `GOOGLE_DRIVE_COURSES_FOLDER_ID` no `.env`.
   - Criar/atribuir credenciais do Google Drive e Postgres no editor n8n.

2. **Importar e ativar o workflow de upload**  
   - Importar `uploadLocalCourses.json`, mapear as credenciais e testar manualmente com alguns cursos.
   - Validar que `course_uploads` registra cada envio.

3. **Monitorar as primeiras execuções**  
   - Acompanhar a aba *Executions* do n8n e os logs (`docker compose logs`) para garantir que não há conflitos de permissões ou falhas de rede.

4. **Extensões futuras (ideias)**  
   - Notificações (e-mail/Slack) ao concluir uploads.
   - Geração automática de checksums ou logs de versão dos cursos.
   - Enriquecimento das automações do demo usando Qdrant/Ollama com dados reais.

Este documento serve como snapshot do estado atual, facilitando a continuidade do desenvolvimento e o entendimento rápido do que já foi configurado.

