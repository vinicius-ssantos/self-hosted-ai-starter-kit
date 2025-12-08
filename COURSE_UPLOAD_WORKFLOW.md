# Workflow: Upload automático de cursos para o Google Drive

Este guia descreve como montar no n8n um fluxo que empacota cada curso da pasta compartilhada, envia o pacote para uma pasta do Google Drive e registra o upload em uma tabela Postgres (`course_uploads`). Os passos assumem que você já criou as pastas `shared/cursos` e `shared/tmp`, que o docker-compose está ativo e que o ID da pasta de destino no Drive foi definido na variável `GOOGLE_DRIVE_COURSES_FOLDER_ID` do `.env`.

## Pré‑requisitos

1. **Credencial Google Drive:** em *Settings → Credentials*, crie uma credencial “Google Drive OAuth2 API” com as permissões padrão `https://www.googleapis.com/auth/drive`.
2. **Tabela de log:** o comando abaixo já foi executado e fica como referência caso precise recriar.

   ```bash
   docker compose --profile cpu exec postgres \
     psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" \
     -c "CREATE TABLE IF NOT EXISTS course_uploads (
            id SERIAL PRIMARY KEY,
            course_name TEXT UNIQUE NOT NULL,
            drive_file_id TEXT NOT NULL,
            uploaded_at TIMESTAMPTZ DEFAULT NOW()
        );"
   ```

3. **Estrutura local:** coloque cada curso dentro de `shared/cursos/<nome-do-curso>`. O n8n monta essa pasta como `/data/shared/cursos` nos nós de “Execute Command” e “Read Binary Files”.

## Nós e conexões

Monte o workflow seguindo a ordem abaixo (posicione os nós para ficar mais fácil visualizar o fluxo):

1. **Cron** – `n8n-nodes-base.cron`
   - Modo: “Every Day” (ajuste conforme preferir).
   - Hora recomendada: durante a madrugada para não disputar rede.

2. **Listar cursos** – `n8n-nodes-base.executeCommand`
   - Comando: `ls -1 /data/shared/cursos || true`
   - Continua mesmo que a pasta esteja vazia.

3. **Preparar itens** – `n8n-nodes-base.function`
   - Código:
     ```javascript
     const stdout = (items[0].json.stdout || '').trim();
     if (!stdout) {
       return [];
     }
     return stdout
       .split('\n')
       .map(line => line.trim())
       .filter(Boolean)
       .map(course => ({ json: { course } }));
     ```

4. **Verificar uploads existentes (Postgres)** – `n8n-nodes-base.postgres`
   - Operação: *Execute Query*.
   - Query:
     ```sql
     SELECT drive_file_id
     FROM course_uploads
     WHERE course_name = $1
     ```
   - Query Parameters: `{{$json["course"]}}`
   - Opções: habilite **Continue On Fail** para não interromper o fluxo caso não haja registros.

5. **IF – Já registrado?**
   - Condição: `{{$json["drive_file_id"] !== undefined}}`
   - Ramo **true** → nó **Marcar como concluído** (Set) apenas para log:
     - `status = "Pulado (já está no Postgres)"`
   - Ramo **false** segue para checar o Drive.

6. **Buscar no Drive** – `n8n-nodes-base.googleDrive`
   - Operação: *List*.
   - Query:
     ```
     name = '{{$json["course"]}}.tar.gz'
     and '{{$env["GOOGLE_DRIVE_COURSES_FOLDER_ID"]}}' in parents
     and trashed = false
     ```

7. **IF – Já existe no Drive?**
   - Condição: `{{ Array.isArray($json["files"]) && $json["files"].length > 0 }}`
   - Ramo **true** → `Set` para marcar `status = "Pulado (já está no Drive)"`.
   - Ramo **false** continua para o upload.

8. **Criar pacote .tar.gz** – `n8n-nodes-base.executeCommand`
   - Comando:
     ```
     COURSE={{$json["course"]}}
     ARCHIVE="/data/shared/tmp/${COURSE}.tar.gz"
     SRC="/data/shared/cursos/${COURSE}"
     mkdir -p /data/shared/tmp
     tar -czf "$ARCHIVE" -C "/data/shared/cursos" "$COURSE"
     echo "$ARCHIVE"
     ```
   - Ative *Shell* `/bin/bash`.

9. **Ler arquivo** – `n8n-nodes-base.readBinaryFile`
   - Property Name: `data`
   - File Path: `{{$json["stdout"].trim()}}`

10. **Upload para Drive** – `n8n-nodes-base.googleDrive`
    - Operação: *Upload*.
    - Binary Property: `data`
    - Nome do arquivo: `{{$json["course"]}}.tar.gz`
    - Pasta: `{{$env["GOOGLE_DRIVE_COURSES_FOLDER_ID"]}}`

11. **Registrar no Postgres** – `n8n-nodes-base.postgres`
    - Operação: *Execute Query*.
    - Query:
      ```sql
      INSERT INTO course_uploads (course_name, drive_file_id)
      VALUES ($1, $2)
      ON CONFLICT (course_name) DO NOTHING
      ```
    - Params:
      1. `{{$json["course"]}}`
      2. `{{$json["id"]}}` (ID do arquivo retornado pelo nó do Drive).

12. **Remover pacote temporário** – `n8n-nodes-base.executeCommand`
    - Comando: `rm -f /data/shared/tmp/{{$json["course"]}}.tar.gz`
    - Continue On Fail habilitado para não quebrar caso o arquivo já tenha sido removido.

13. **Set – Sucesso**
    - Campos:
      - `status = "Upload concluído"`
      - `drive_file_id = {{$json["id"]}}`

## Testes

1. Desative o `Cron` e troque temporariamente por um `Manual Trigger` para validar com poucos cursos.
2. Coloque 1–2 cursos de teste em `shared/cursos`. Execute o workflow e confirme:
   - Arquivo `.tar.gz` criado em `shared/tmp`.
   - Arquivo enviado ao Drive na pasta configurada.
   - Registro inserido em `course_uploads`.
3. Reexecute: os itens pulados devem aparecer com status “já está no Postgres/Drive”.

## Produção

1. Reative o `Cron` e defina o horário final.
2. Deixe o workflow *Active*. Acompanhe em *Executions* e nos logs do container n8n (erro de permissão, etc.).
3. Opcional: use um nó `E-mail` ou `Slack` ao final para notificar o resultado por execução.

Com isso, o fluxo automatiza a verificação diária, o empacotamento e o envio das pastas de cursos para o Google Drive, evitando duplicações e mantendo o histórico em Postgres.

