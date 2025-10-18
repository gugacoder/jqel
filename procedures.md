# Procedures JQEL - Guia Completo

**Audiência:** Backend/SQL Developers

Este documento explica a nomenclatura, mapeamento e implementação de stored procedures compatíveis com JQEL.

---

## Índice

1. [Conceitos Fundamentais](#conceitos-fundamentais)
2. [Anatomia Completa de uma Procedure](#anatomia-completa-de-uma-procedure)
3. [Nomenclatura e Convenções](#nomenclatura-e-convenções)
4. [Mapeamento de Entidades](#mapeamento-de-entidades)
5. [Exemplos Práticos Completos](#exemplos-práticos-completos)
6. [Best Practices](#best-practices)
7. [Otimização Avançada: Campos Pesados](#otimização-avançada-campos-pesados)
8. [Descoberta e Ferramentas](#descoberta-e-ferramentas)
9. [Referência Rápida](#referência-rápida)

---

## Conceitos Fundamentais

### O que é uma JQEL Procedure?

Uma **JQEL Procedure** é uma stored procedure SQL Server que:

1. **Recebe requisições em formato JSON** (através do parâmetro `@jqel`)
2. **Processa operações de dados** (select, insert, update, delete, ações customizadas)
3. **Retorna respostas padronizadas** no formato JResult (JSON)
4. **Mapeia automaticamente** queries JQEL para procedures específicas

### Por que JQEL Procedures?

**Vantagens:**
- **Consistência**: Todas as procedures seguem o mesmo padrão de entrada/saída
- **Segurança**: Validação centralizada, proteção contra SQL injection
- **Auditoria**: Contexto do usuário (`@user`) sempre disponível
- **Manutenibilidade**: Nomenclatura previsível e auto-documentada
- **Integração**: N8N workflows invocam procedures dinamicamente sem hardcode

**Arquitetura:**
```
Frontend → N8N Workflow → JQEL Procedure → SQL Server → JResult → Workflow → Frontend
```

**⚠️ Regra Fundamental:** Toda query JQEL **DEVE** incluir o campo `schema` (ex: `"schema": "sac"`). Não existe schema padrão.

---

## Anatomia Completa de uma Procedure

Toda procedure JQEL segue um padrão rígido com **cinco componentes obrigatórios**:

### 1. Assinatura Padronizada

```sql
CREATE OR ALTER PROCEDURE {schema}.jqel__{operation}__{entity}[__{action}]
    @user NVARCHAR(MAX) = NULL,  -- Contexto do usuário (opcional, JSON)
    @jqel NVARCHAR(MAX)          -- Query JQEL (obrigatório, JSON)
AS
```

**Componentes do nome:**
- `{schema}`: **OBRIGATÓRIO**. Namespace do banco (Exemplos: `sac`, `jqel`, `api`)
- `{operation}`: `select` ou `mutate`
- `{entity}`: Nome da entidade em snake_case (pode conter `_` simples)
- `{action}`: Ação específica em snake_case (pode conter `_` simples)
  - **Para `select`**: Opcional (ex: `dashboard`, `relatorio_mensal`, `estatistica`)
  - **Para `mutate`**: **Obrigatório** (ex: `insert`, `update`, `delete`, `ativar`, `transferir`)

**Separador:** Duplo underscore (`__`) entre componentes principais

**Exemplos válidos:**
```sql
sac.jqel__select__usuario                           -- Select básico
sac.jqel__select__usuario__dashboard                -- Select com action
sac.jqel__mutate__usuario__insert                   -- Mutate (action obrigatório)
sac.jqel__mutate__chamado__transferir               -- Action customizada
tom.jqel__select__chamado_anexo__estatistica_consumo -- Entity e action com underscore
```

---

### 2. Parâmetros

#### **`@user` (opcional)**

**Tipo:** `NVARCHAR(MAX)` (JSON)

**Propósito:** Identificar o usuário que está executando a operação.

**Casos de uso:**
- **Auditoria**: Registrar quem executou a operação
- **Controle de acesso**: Verificar permissões antes de executar
- **Filtros automáticos**: Limitar dados à empresa/departamento do usuário
- **Lógica de negócio**: Ex: atribuir criador/modificador do registro

**Estrutura esperada:**
```json
{
  "id_usuario": 2,
  "email": "admin@processa.com",
  "nome_exibicao": "Administrador",
  "is_admin": true,
  "roles": [
    { "codigo_papel": "administrador" }
  ],
  "ativo": true,
  "iat": 1759735542,
  "exp": 1759737342
}
```

**⚠️ Importante:** `@user` pode ser `NULL` em contextos sem autenticação (ex: webhooks públicos, jobs agendados). Exemplos de extração estão nos exemplos práticos completos.

---

#### **`@jqel` (obrigatório)**

**Tipo:** `NVARCHAR(MAX)` (JSON)

**Propósito:** Contém a query JQEL completa com todos os parâmetros de filtro, ordenação, valores, etc.

**Estrutura:**
```json
{
  "where": { /* condições de filtro */ },
  "values": { /* dados para insert/update */ },
  "options": {
    "limit": 100,
    "offset": 0,
    "orderBy": [{"field": "nome", "direction": "asc"}]
  }
}
```

**Extração de seções:**
```sql
-- Extrair seções principais (objetos/arrays JSON)
DECLARE @where NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.where');
DECLARE @values NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.values');
DECLARE @options NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.options');
DECLARE @orderBy NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.options.orderBy');
```

**Extração de valores escalares:**
```sql
-- Extrair valores primitivos (int, string, bool)
DECLARE @limit INT = JSON_VALUE(@jqel, '$.options.limit');
DECLARE @offset INT = JSON_VALUE(@jqel, '$.options.offset');

-- Com valores padrão
DECLARE @limit INT = ISNULL(JSON_VALUE(@jqel, '$.options.limit'), 100);
DECLARE @offset INT = ISNULL(JSON_VALUE(@jqel, '$.options.offset'), 0);
```

**Extração de campos individuais de `values`:**
```sql
DECLARE @values NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.values');

-- Extrair campos do values
DECLARE @nome_completo NVARCHAR(255) = JSON_VALUE(@values, '$.nome_completo');
DECLARE @email NVARCHAR(255) = JSON_VALUE(@values, '$.email');
DECLARE @ativo BIT = CAST(JSON_VALUE(@values, '$.ativo') AS BIT);
DECLARE @data_nascimento DATE = CAST(JSON_VALUE(@values, '$.data_nascimento') AS DATE);
```

**⚠️ Diferença crítica:**
- **`JSON_VALUE()`**: Retorna valores escalares (string, int, bool) → Use para primitivos
- **`JSON_QUERY()`**: Retorna objetos/arrays JSON → Use para estruturas complexas

---

### 3. Corpo da Procedure

```sql
BEGIN
    SET NOCOUNT ON;  -- Evita retornar contagem de linhas afetadas

    BEGIN TRY
        -- 1. Extrair e validar parâmetros
        DECLARE @id_usuario INT = JSON_VALUE(@user, '$.id_usuario');
        DECLARE @where NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.where');
        DECLARE @limit INT = ISNULL(JSON_VALUE(@jqel, '$.options.limit'), 100);

        -- 2. Validações de negócio
        IF @id_usuario IS NULL
        BEGIN
            -- Retornar erro e SAIR
            SELECT
                401 AS code,
                'Usuário não autenticado' AS message,
                NULL AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 3. Executar lógica principal
        SELECT TOP (@limit)
            t.DFid_usuario AS id_usuario,
            t.DFnome_completo AS nome_completo,
            t.DFemail AS email
        INTO #temp
        FROM sac.TBusuario t WITH (NOLOCK)
        WHERE t.DFativo = 1;

        -- 4. Retornar JResult de sucesso
        SELECT
            200 AS code,
            'Usuários recuperados com sucesso' AS message,
            (SELECT * FROM #temp FOR JSON PATH) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        -- 5. Tratamento de erros
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
```

---

### 4. Formato de Resposta: JResult

**Toda procedure JQEL DEVE retornar um único objeto JSON no formato JResult.**

**Estrutura obrigatória:**
```json
{
  "code": 200,
  "message": "Operação realizada com sucesso",
  "data": [
    {"id_usuario": 1, "nome": "João Silva", "email": "joao@example.com"},
    {"id_usuario": 2, "nome": "Maria Santos", "email": "maria@example.com"}
  ],
  "warning": []  // Opcional: Array de avisos não críticos
}
```

**Campos:**
- **`code`** (obrigatório): HTTP status code (int)
- **`message`** (obrigatório): Mensagem descritiva (string)
- **`data`** (opcional): Dados retornados (array, object, ou NULL)
- **`warning`** (opcional): Array de JResults com avisos não críticos

**Códigos de status comuns:**

| Código | Significado | Uso |
|--------|-------------|-----|
| `200` | OK | SELECT bem-sucedido |
| `201` | Created | INSERT bem-sucedido |
| `204` | No Content | DELETE/UPDATE sem retorno de dados |
| `400` | Bad Request | Validação falhou, campo obrigatório faltando |
| `401` | Unauthorized | Usuário não autenticado |
| `403` | Forbidden | Usuário autenticado mas sem permissão |
| `404` | Not Found | Registro não encontrado |
| `409` | Conflict | Violação de unique constraint |
| `422` | Unprocessable Entity | Regra de negócio violada |
| `500` | Internal Server Error | Erro inesperado no servidor |

**Exemplo de erro com campo específico:**
```json
{
  "code": 400,
  "message": "Campo 'email' é obrigatório",
  "data": {
    "field": "email",
    "value": null
  }
}
```

**Exemplo com warnings:**
```json
{
  "code": 200,
  "message": "Usuário criado com sucesso",
  "data": {"id_usuario": 123},
  "warning": [
    {
      "code": 200,
      "message": "Email de boas-vindas não pôde ser enviado",
      "data": null
    }
  ]
}
```

**⚠️ CRÍTICO:** A procedure DEVE retornar **exatamente um** JResult. Não retorne múltiplos resultsets ou texto simples.

---

### 5. Tratamento de Erros Obrigatório

**Sempre use `TRY/CATCH`:**

```sql
BEGIN TRY
    -- Lógica da procedure

    -- Retorno de sucesso
    SELECT
        200 AS code,
        'Operação realizada' AS message,
        (SELECT * FROM #temp FOR JSON PATH) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

END TRY
BEGIN CATCH
    -- Capturar erro e retornar JResult padronizado
    SELECT
        500 AS code,
        ERROR_MESSAGE() AS message,
        JSON_QUERY((
            SELECT
                ERROR_NUMBER() AS error_number,
                ERROR_SEVERITY() AS severity,
                ERROR_STATE() AS state,
                ERROR_PROCEDURE() AS procedure_name,
                ERROR_LINE() AS line_number
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
END CATCH
```

**⚠️ Segurança:** Em produção, não exponha detalhes internos no `message`. Use mensagens genéricas e log detalhes no servidor.

---

## Nomenclatura e Convenções

### Estrutura do Nome

**Formato geral:**
```
{schema}.jqel__{operation}__{entity}[__{action}]
```

**Regras:**

1. **Schema** (Exemplo: `sac`)
   - Namespace do banco de dados
   - Define o contexto da entidade

2. **Separador**: Duplo underscore `__` entre componentes principais
   - Permite que `entity` e `action` contenham underscore simples `_`
   - Exemplo: `chamado_anexo` (entity) + `estatistica_consumo` (action)

3. **Operation**: `select` ou `mutate`
   - `select`: Operações de leitura (queries)
   - `mutate`: Operações de escrita (insert, update, delete)

4. **Entity**: Nome da entidade em snake_case
   - Singular, português, minúsculas
   - Pode conter `_` simples para separar palavras
   - Exemplos: `usuario`, `atendente`, `chamado_anexo`, `status_chamado`

5. **Action**: Ação específica em snake_case
   - **Para `select`**: Opcional
     - Use para views especializadas, relatórios, estatísticas
     - Exemplos: `dashboard`, `relatorio_mensal`, `estatistica_consumo`
   - **Para `mutate`**: **OBRIGATÓRIO**
     - Sempre especifique a ação
     - Exemplos: `insert`, `update`, `delete`, `ativar`, `transferir`, `upsert`

---

### Convenções de Case e Ordem

#### ✅ Ordem Consistente

**Sempre seguir:** `schema` → `jqel` → `operação` → `entidade` → `ação`

**Correto:**
```sql
sac.jqel__select__usuario__relatorio
sac.jqel__mutate__atendimento__transferir
sac.jqel__select__chamado_anexo__estatistica_consumo
```

---

#### ✅ Snake_case em Todas as Partes

- **Operações**: minúsculas (`select`, `mutate`)
- **Entidades**: minúsculas, separadas por `_` (`usuario`, `chamado_anexo`, `status_chamado`)
- **Actions**: minúsculas, separadas por `_` (`insert`, `relatorio_mensal`, `estatistica_consumo`)

**Correto:**
```sql
fechamento_mensal
estatistica_consumo
ranking_avaliacao
```

---

### Schemas Disponíveis

**⚠️ IMPORTANTE:** O campo `schema` é **OBRIGATÓRIO** em todas as queries JQEL. Não existe schema padrão - toda query deve especificar explicitamente qual schema usar.

**Schemas válidos:** `sac`, `jqel`, `api`

#### `sac` - Schema principal (Sistema de Atendimento ao Cliente)

**Propósito:** Schema compartilhado entre aplicativos do Coletivos Processa

**Entidades típicas:**
- Usuários e autenticação: `TBusuario`, `TBpapel`, `TBpermissao`
- Helpdesk: `TBchamado`, `TBatendente`, `TBcliente`, `TBcategoria`
- Estrutura organizacional: `TBempresa`, `TBdepartamento`

**Exemplos de procedures:**
```sql
sac.jqel__select__usuario
sac.jqel__mutate__chamado__insert
sac.jqel__select__atendente__ranking_avaliacao
```

---

### Prefixos de Tabelas e Colunas

#### Tabelas: Prefixo `TB`

**Formato:** `TBnome_entidade` (singular, snake_case)

```sql
TBusuario
TBatendente
TBdepartamento
TBempresa
TBchamado
TBchamado_anexo
TBstatus_chamado
```

**Rationale:** Facilita identificação de tabelas vs views, procedures, etc.

---

#### Colunas: Prefixo `DF`

**Formato:** `DFnome_campo` (descritivo, snake_case)

```sql
DFid_usuario
DFnome_completo
DFemail
DFdata_cadastro
DFusuario_criacao   -- FK com contexto
DFid_empresa        -- FK simples
DFativo
```

**Rationale:** Evita conflitos com palavras reservadas SQL e facilita identificação de origem.

---

## Mapeamento de Entidades

### JQEL Query → Procedure

Queries JQEL mapeiam automaticamente para procedures seguindo as regras de nomenclatura:

| Query JQEL | Procedure Mapeada |
|-----------|-------------------|
| `{"schema": "sac", "select": "usuario"}` | `sac.jqel__select__usuario` |
| `{"schema": "sac", "select": "usuario", "action": "dashboard"}` | `sac.jqel__select__usuario__dashboard` |
| `{"schema": "sac", "mutate": "usuario", "action": "insert"}` | `sac.jqel__mutate__usuario__insert` |
| `{"schema": "sac", "mutate": "usuario", "action": "ativar"}` | `sac.jqel__mutate__usuario__ativar` |
| `{"schema": "sac", "mutate": "atendimento", "action": "transferir"}` | `sac.jqel__mutate__atendimento__transferir` |

**⚠️ Campos obrigatórios:**
- **`schema`**: OBRIGATÓRIO. Especifica o schema do banco (`sac`, `jqel`, `api`)
- **`action` para mutate**: OBRIGATÓRIO. Toda operação `mutate` deve especificar `action`

---

### Procedure → JQEL Query (Reverso)

Dado o nome da procedure, é possível inferir a query JQEL:

| Procedure | Query JQEL Inferida |
|-----------|---------------------|
| `sac.jqel__select__usuario` | `{"schema": "sac", "select": "usuario"}` |
| `sac.jqel__mutate__usuario__insert` | `{"schema": "sac", "mutate": "usuario", "action": "insert"}` |
| `sac.jqel__select__chamado_anexo__estatistica_consumo` | `{"schema": "sac", "select": "chamado_anexo", "action": "estatistica_consumo"}` |

**Exemplo de parsing (JavaScript):**

```javascript
function parseProcedureName(fullName) {
  // fullName: 'sac.jqel__select__chamado_anexo__estatistica_consumo'
  const [schema, procedureName] = fullName.split('.');
  const parts = procedureName.split('__');

  return {
    schema,              // 'sac'
    prefix: parts[0],    // 'jqel'
    operation: parts[1], // 'select'
    entity: parts[2],    // 'chamado_anexo'
    action: parts[3] || null // 'estatistica_consumo' ou null
  };
}

// Uso:
const result = parseProcedureName('sac.jqel__select__chamado_anexo__estatistica_consumo');
console.log(result);
/*
{
  schema: 'sac',
  prefix: 'jqel',
  operation: 'select',
  entity: 'chamado_anexo',
  action: 'estatistica_consumo'
}
*/
```

---

## Exemplos Práticos Completos

### Exemplo 1: SELECT com Filtros e Paginação

```sql
CREATE OR ALTER PROCEDURE sac.jqel__select__usuario
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- 1. Extrair parâmetros do JQEL
        DECLARE @where NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.where');
        DECLARE @limit INT = ISNULL(JSON_VALUE(@jqel, '$.options.limit'), 100);
        DECLARE @offset INT = ISNULL(JSON_VALUE(@jqel, '$.options.offset'), 0);

        -- Limitar máximo de resultados
        IF @limit > 1000 SET @limit = 1000;

        -- 2. Extrair filtros específicos (exemplo)
        DECLARE @status NVARCHAR(50) = JSON_VALUE(@where, '$.status.eq');
        DECLARE @email_like NVARCHAR(255) = JSON_VALUE(@where, '$.email.like');
        DECLARE @id_empresa INT = JSON_VALUE(@where, '$.id_empresa.eq');

        -- 3. Executar query com filtros dinâmicos
        SELECT TOP (@limit)
            t.DFid_usuario AS id_usuario,
            t.DFnome_completo AS nome_completo,
            t.DFemail AS email,
            t.DFstatus AS status,
            t.DFdata_cadastro AS data_cadastro,
            t.DFativo AS ativo
        INTO #temp
        FROM sac.TBusuario t WITH (NOLOCK)
        WHERE
            (@status IS NULL OR t.DFstatus = @status)
            AND (@email_like IS NULL OR t.DFemail LIKE '%' + @email_like + '%')
            AND (@id_empresa IS NULL OR t.DFid_empresa = @id_empresa)
        ORDER BY t.DFid_usuario
        OFFSET @offset ROWS;

        -- 4. Contar total de registros (sem paginação)
        DECLARE @total INT = (
            SELECT COUNT(*)
            FROM sac.TBusuario t WITH (NOLOCK)
            WHERE
                (@status IS NULL OR t.DFstatus = @status)
                AND (@email_like IS NULL OR t.DFemail LIKE '%' + @email_like + '%')
                AND (@id_empresa IS NULL OR t.DFid_empresa = @id_empresa)
        );

        -- 5. Retornar JResult com metadados de paginação
        SELECT
            200 AS code,
            'Usuários recuperados com sucesso' AS message,
            JSON_QUERY((
                SELECT
                    (SELECT * FROM #temp FOR JSON PATH) AS items,
                    @total AS total,
                    @limit AS limit,
                    @offset AS offset,
                    CEILING(CAST(@total AS FLOAT) / @limit) AS total_pages
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
            )) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

**Query JQEL:**
```json
{
  "schema": "sac",
  "select": "usuario",
  "where": {
    "status": {"eq": "ativo"},
    "email": {"like": "joao"}
  },
  "options": {
    "limit": 20,
    "offset": 0
  }
}
```

---

### Exemplo 2: INSERT com Validações Completas

```sql
CREATE OR ALTER PROCEDURE sac.jqel__mutate__usuario__insert
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- 1. Extrair values
        DECLARE @values NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.values');

        -- 2. Extrair campos individuais
        DECLARE @nome_completo NVARCHAR(255) = JSON_VALUE(@values, '$.nome_completo');
        DECLARE @email NVARCHAR(255) = JSON_VALUE(@values, '$.email');
        DECLARE @status NVARCHAR(50) = JSON_VALUE(@values, '$.status');
        DECLARE @id_empresa INT = JSON_VALUE(@values, '$.id_empresa');
        DECLARE @senha_hash NVARCHAR(255) = JSON_VALUE(@values, '$.senha_hash');

        -- 3. Extrair contexto do usuário
        DECLARE @id_usuario_criacao INT = JSON_VALUE(@user, '$.id_usuario');

        -- 4. Validar campos obrigatórios
        IF @nome_completo IS NULL OR LTRIM(RTRIM(@nome_completo)) = ''
        BEGIN
            SELECT
                400 AS code,
                'Campo "nome_completo" é obrigatório' AS message,
                JSON_QUERY((
                    SELECT 'nome_completo' AS field
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        IF @email IS NULL OR LTRIM(RTRIM(@email)) = ''
        BEGIN
            SELECT
                400 AS code,
                'Campo "email" é obrigatório' AS message,
                JSON_QUERY((
                    SELECT 'email' AS field
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 5. Validar formato de email
        IF @email NOT LIKE '%_@__%.__%'
        BEGIN
            SELECT
                400 AS code,
                'Formato de email inválido' AS message,
                JSON_QUERY((
                    SELECT 'email' AS field, @email AS value
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 6. Verificar email duplicado
        IF EXISTS (SELECT 1 FROM sac.TBusuario WHERE DFemail = @email)
        BEGIN
            DECLARE @existing_id INT = (
                SELECT TOP 1 DFid_usuario
                FROM sac.TBusuario
                WHERE DFemail = @email
            );

            SELECT
                409 AS code,
                'Email já cadastrado no sistema' AS message,
                JSON_QUERY((
                    SELECT
                        'email' AS field,
                        @email AS value,
                        @existing_id AS existing_id
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 7. Validar foreign key (empresa)
        IF @id_empresa IS NOT NULL
            AND NOT EXISTS (SELECT 1 FROM sac.TBempresa WHERE DFid_empresa = @id_empresa)
        BEGIN
            SELECT
                400 AS code,
                'Empresa não encontrada' AS message,
                JSON_QUERY((
                    SELECT
                        'id_empresa' AS field,
                        @id_empresa AS value
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 8. Validar enum (status)
        IF @status IS NOT NULL AND @status NOT IN ('ativo', 'inativo', 'pendente')
        BEGIN
            SELECT
                400 AS code,
                'Status inválido. Valores permitidos: ativo, inativo, pendente' AS message,
                JSON_QUERY((
                    SELECT
                        'status' AS field,
                        @status AS value
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 9. Inserir registro
        DECLARE @id_usuario INT;

        INSERT INTO sac.TBusuario (
            DFnome_completo,
            DFemail,
            DFsenha_hash,
            DFstatus,
            DFid_empresa,
            DFativo,
            DFdata_cadastro,
            DFusuario_criacao
        )
        VALUES (
            @nome_completo,
            @email,
            @senha_hash,
            ISNULL(@status, 'pendente'),
            @id_empresa,
            1,  -- Ativo por padrão
            GETDATE(),
            @id_usuario_criacao
        );

        SET @id_usuario = SCOPE_IDENTITY();

        -- 10. Retornar registro criado (201)
        SELECT
            201 AS code,
            'Usuário criado com sucesso' AS message,
            JSON_QUERY((
                SELECT
                    DFid_usuario AS id_usuario,
                    DFnome_completo AS nome_completo,
                    DFemail AS email,
                    DFstatus AS status,
                    DFid_empresa AS id_empresa,
                    DFativo AS ativo,
                    DFdata_cadastro AS data_cadastro
                FROM sac.TBusuario
                WHERE DFid_usuario = @id_usuario
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
            )) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

**Query JQEL:**
```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "insert",
  "values": {
    "nome_completo": "Ana Silva",
    "email": "ana.silva@example.com",
    "senha_hash": "$2b$10$...",
    "status": "ativo",
    "id_empresa": 5
  }
}
```

---

### Exemplo 3: DELETE com Soft Delete

```sql
CREATE OR ALTER PROCEDURE sac.jqel__mutate__usuario__delete
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- 1. Extrair where
        DECLARE @where NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.where');
        DECLARE @id_usuario INT = JSON_VALUE(@where, '$.id_usuario.eq');

        IF @id_usuario IS NULL
        BEGIN
            SELECT
                400 AS code,
                'ID do usuário é obrigatório' AS message,
                NULL AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 2. Verificar se registro existe
        IF NOT EXISTS (SELECT 1 FROM sac.TBusuario WHERE DFid_usuario = @id_usuario)
        BEGIN
            SELECT
                404 AS code,
                'Usuário não encontrado' AS message,
                JSON_QUERY((
                    SELECT @id_usuario AS id_usuario
                    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
                )) AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 3. Verificar permissão (exemplo: não deletar admin)
        DECLARE @is_admin BIT = (
            SELECT DFis_admin
            FROM sac.TBusuario
            WHERE DFid_usuario = @id_usuario
        );

        IF @is_admin = 1
        BEGIN
            SELECT
                403 AS code,
                'Não é permitido deletar usuários administradores' AS message,
                NULL AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 4. Extrair contexto do usuário
        DECLARE @id_usuario_modificacao INT = JSON_VALUE(@user, '$.id_usuario');

        -- 5. Soft delete (marcar como inativo)
        UPDATE sac.TBusuario
        SET
            DFativo = 0,
            DFstatus = 'excluido',
            DFdata_exclusao = GETDATE(),
            DFusuario_exclusao = @id_usuario_modificacao
        WHERE DFid_usuario = @id_usuario;

        -- 6. Retornar confirmação (204 No Content)
        SELECT
            204 AS code,
            'Usuário excluído com sucesso' AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

**Query JQEL:**
```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "delete",
  "where": {"id_usuario": {"eq": 123}}
}
```

---

### Exemplo 4: Action Customizada com Transaction

```sql
CREATE OR ALTER PROCEDURE sac.jqel__mutate__chamado__transferir
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

            -- 1. Extrair parâmetros
            DECLARE @values NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.values');
            DECLARE @id_chamado INT = JSON_VALUE(@values, '$.id_chamado');
            DECLARE @id_atendente_destino INT = JSON_VALUE(@values, '$.id_atendente_destino');
            DECLARE @motivo NVARCHAR(500) = JSON_VALUE(@values, '$.motivo');

            DECLARE @id_usuario INT = JSON_VALUE(@user, '$.id_usuario');

            -- 2. Validações
            IF @id_chamado IS NULL OR @id_atendente_destino IS NULL
            BEGIN
                SELECT
                    400 AS code,
                    'ID do chamado e atendente de destino são obrigatórios' AS message,
                    NULL AS data
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
                ROLLBACK TRANSACTION;
                RETURN;
            END

            -- Verificar se chamado existe
            IF NOT EXISTS (SELECT 1 FROM sac.TBchamado WHERE DFid_chamado = @id_chamado)
            BEGIN
                SELECT
                    404 AS code,
                    'Chamado não encontrado' AS message,
                    NULL AS data
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
                ROLLBACK TRANSACTION;
                RETURN;
            END

            -- Verificar se atendente existe
            IF NOT EXISTS (SELECT 1 FROM sac.TBatendente WHERE DFid_atendente = @id_atendente_destino)
            BEGIN
                SELECT
                    404 AS code,
                    'Atendente de destino não encontrado' AS message,
                    NULL AS data
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
                ROLLBACK TRANSACTION;
                RETURN;
            END

            -- Verificar se atendente está ativo
            DECLARE @atendente_ativo BIT = (
                SELECT DFativo FROM sac.TBatendente WHERE DFid_atendente = @id_atendente_destino
            );

            IF @atendente_ativo = 0
            BEGIN
                SELECT
                    422 AS code,
                    'Atendente de destino está inativo' AS message,
                    NULL AS data
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
                ROLLBACK TRANSACTION;
                RETURN;
            END

            -- 3. Obter atendente atual
            DECLARE @id_atendente_origem INT = (
                SELECT DFid_atendente FROM sac.TBchamado WHERE DFid_chamado = @id_chamado
            );

            -- 4. Atualizar chamado
            UPDATE sac.TBchamado
            SET
                DFid_atendente = @id_atendente_destino,
                DFdata_modificacao = GETDATE(),
                DFusuario_modificacao = @id_usuario
            WHERE DFid_chamado = @id_chamado;

            -- 5. Registrar histórico de transferência
            INSERT INTO sac.TBchamado_historico (
                DFid_chamado,
                DFtipo_evento,
                DFdescricao,
                DFid_atendente_origem,
                DFid_atendente_destino,
                DFdata_evento,
                DFusuario_evento
            )
            VALUES (
                @id_chamado,
                'transferencia',
                ISNULL(@motivo, 'Chamado transferido'),
                @id_atendente_origem,
                @id_atendente_destino,
                GETDATE(),
                @id_usuario
            );

        COMMIT TRANSACTION;

        -- 6. Retornar sucesso
        SELECT
            200 AS code,
            'Chamado transferido com sucesso' AS message,
            JSON_QUERY((
                SELECT
                    @id_chamado AS id_chamado,
                    @id_atendente_origem AS id_atendente_origem,
                    @id_atendente_destino AS id_atendente_destino
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
            )) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

**Query JQEL:**
```json
{
  "schema": "sac",
  "mutate": "chamado",
  "action": "transferir",
  "values": {
    "id_chamado": 456,
    "id_atendente_destino": 789,
    "motivo": "Especialidade do atendente"
  }
}
```

---

## Best Practices

### Segurança

#### ✅ Sempre usar queries parametrizadas

**BOM: Parametrizado com `sp_executesql`**
```sql
DECLARE @status NVARCHAR(50) = JSON_VALUE(@where, '$.status.eq');
DECLARE @sql NVARCHAR(MAX) = N'
    SELECT * FROM TBusuario
    WHERE DFstatus = @status';

EXEC sp_executesql @sql, N'@status NVARCHAR(50)', @status;
```

---

#### ✅ Validar e sanitizar inputs

```sql
-- Validar formato de email
IF @email NOT LIKE '%_@__%.__%'
BEGIN
    SELECT
        400 AS code,
        'Formato de email inválido' AS message,
        JSON_QUERY((SELECT 'email' AS field FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Validar range de valores
IF @idade < 0 OR @idade > 150
BEGIN
    SELECT
        400 AS code,
        'Idade deve estar entre 0 e 150' AS message,
        JSON_QUERY((SELECT 'idade' AS field FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Validar enums (whitelist)
IF @status NOT IN ('ativo', 'inativo', 'pendente')
BEGIN
    SELECT
        400 AS code,
        'Status inválido. Valores permitidos: ativo, inativo, pendente' AS message,
        JSON_QUERY((SELECT 'status' AS field, @status AS value FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Validar comprimento de strings
IF LEN(@nome_completo) > 255
BEGIN
    SELECT
        400 AS code,
        'Nome completo deve ter no máximo 255 caracteres' AS message,
        JSON_QUERY((SELECT 'nome_completo' AS field FOR JSON PATH, WITHOUT_ARRAY_WRAPPER)) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Sanitizar strings (remover espaços)
SET @email = LTRIM(RTRIM(LOWER(@email)));
SET @nome_completo = LTRIM(RTRIM(@nome_completo));
```

---

#### ✅ Verificar permissões do usuário

```sql
-- Verificar autenticação
IF @user IS NULL OR JSON_VALUE(@user, '$.id_usuario') IS NULL
BEGIN
    SELECT
        401 AS code,
        'Usuário não autenticado' AS message,
        NULL AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Verificar role específico
DECLARE @user_role NVARCHAR(50) = JSON_VALUE(@user, '$.roles[0].codigo_papel');
IF @user_role NOT IN ('administrador', 'gerente')
BEGIN
    SELECT
        403 AS code,
        'Você não tem permissão para executar esta operação' AS message,
        NULL AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Verificar permissão granular
DECLARE @is_admin BIT = CAST(JSON_VALUE(@user, '$.is_admin') AS BIT);
IF @is_admin = 0
BEGIN
    -- Lógica adicional de verificação
    -- Ex: verificar se usuário tem permissão específica
END
```

---

#### ✅ Filtrar por contexto do usuário (Row-Level Security)

```sql
-- Adicionar filtro automático de empresa
DECLARE @id_empresa INT = JSON_VALUE(@user, '$.id_empresa');

-- Usuários só veem dados da própria empresa
SELECT
    t.DFid_chamado,
    t.DFtitulo,
    t.DFdescricao
FROM sac.TBchamado t WITH (NOLOCK)
WHERE t.DFid_empresa = @id_empresa  -- Filtro automático
    AND (@where IS NULL OR /* aplicar outros filtros */);

-- Exceção para administradores
DECLARE @is_admin BIT = CAST(JSON_VALUE(@user, '$.is_admin') AS BIT);

SELECT
    t.DFid_chamado,
    t.DFtitulo,
    t.DFdescricao
FROM sac.TBchamado t WITH (NOLOCK)
WHERE
    (@is_admin = 1 OR t.DFid_empresa = @id_empresa)  -- Admin vê tudo
    AND (@where IS NULL OR /* aplicar outros filtros */);
```

---

#### ✅ Não expor detalhes internos em mensagens de erro

**BOM: Mensagem genérica em produção**
```sql
BEGIN CATCH
    -- Log detalhes internos (servidor)
    INSERT INTO sac.TBlog_erro (DFprocedure, DFerror_message, DFerror_line, DFdata_erro)
    VALUES (OBJECT_NAME(@@PROCID), ERROR_MESSAGE(), ERROR_LINE(), GETDATE());

    -- Retornar mensagem genérica ao cliente
    SELECT
        500 AS code,
        'Erro interno ao processar requisição. Contate o suporte.' AS message,
        JSON_QUERY((
            SELECT NEWID() AS error_id  -- ID para rastreamento
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
END CATCH
```

---

### Validação

#### ✅ Validar campos obrigatórios

```sql
IF @nome_completo IS NULL OR LTRIM(RTRIM(@nome_completo)) = ''
BEGIN
    SELECT
        400 AS code,
        'Campo "nome_completo" é obrigatório' AS message,
        JSON_QUERY((
            SELECT 'nome_completo' AS field
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END
```

---

#### ✅ Validar constraints antes de inserir

```sql
-- Verificar unique constraint
IF EXISTS (SELECT 1 FROM sac.TBusuario WHERE DFemail = @email)
BEGIN
    DECLARE @existing_id INT = (
        SELECT TOP 1 DFid_usuario
        FROM sac.TBusuario
        WHERE DFemail = @email
    );

    SELECT
        409 AS code,
        'Email já cadastrado no sistema' AS message,
        JSON_QUERY((
            SELECT
                'email' AS field,
                @email AS value,
                @existing_id AS existing_id
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END
```

---

#### ✅ Validar foreign keys

```sql
-- Verificar se empresa existe
IF @id_empresa IS NOT NULL
    AND NOT EXISTS (SELECT 1 FROM sac.TBempresa WHERE DFid_empresa = @id_empresa)
BEGIN
    SELECT
        400 AS code,
        'Empresa não encontrada' AS message,
        JSON_QUERY((
            SELECT
                'id_empresa' AS field,
                @id_empresa AS value
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END
```

---

#### ✅ Validar regras de negócio

```sql
-- Validar lógica de negócio: data fim posterior a data início
IF @data_fim < @data_inicio
BEGIN
    SELECT
        422 AS code,
        'Data fim deve ser posterior à data início' AS message,
        JSON_QUERY((
            SELECT
                'data_fim' AS field,
                @data_fim AS value
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Validar limite de registros relacionados
DECLARE @qtd_usuarios_empresa INT = (
    SELECT COUNT(*) FROM sac.TBusuario WHERE DFid_empresa = @id_empresa
);

IF @qtd_usuarios_empresa >= 100
BEGIN
    SELECT
        422 AS code,
        'Empresa atingiu limite de 100 usuários' AS message,
        JSON_QUERY((
            SELECT
                @qtd_usuarios_empresa AS current_count,
                100 AS max_count
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END

-- Validar transição de status
DECLARE @status_atual NVARCHAR(50) = (
    SELECT DFstatus FROM sac.TBchamado WHERE DFid_chamado = @id_chamado
);

IF @status_atual = 'finalizado' AND @status_novo <> 'reaberto'
BEGIN
    SELECT
        422 AS code,
        'Chamado finalizado só pode ser reaberto' AS message,
        NULL AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    RETURN;
END
```

---

### Performance

#### ✅ Usar índices apropriados

```sql
-- Criar índices em colunas de filtro frequente
CREATE NONCLUSTERED INDEX IX_TBusuario_status
    ON sac.TBusuario (DFstatus)
    INCLUDE (DFnome_completo, DFemail, DFativo);

-- Índice composto para queries com múltiplos filtros
CREATE NONCLUSTERED INDEX IX_TBchamado_empresa_status
    ON sac.TBchamado (DFid_empresa, DFstatus)
    INCLUDE (DFtitulo, DFdata_abertura);

-- Índice para ordenação
CREATE NONCLUSTERED INDEX IX_TBchamado_data_abertura
    ON sac.TBchamado (DFdata_abertura DESC)
    INCLUDE (DFid_chamado, DFtitulo, DFstatus);

-- Índice para busca textual
CREATE NONCLUSTERED INDEX IX_TBusuario_email
    ON sac.TBusuario (DFemail)
    INCLUDE (DFid_usuario, DFnome_completo);
```

---

#### ✅ Usar WITH (NOLOCK) em leituras

```sql
-- Reduz locks e melhora concorrência (read uncommitted)
SELECT
    DFid_usuario,
    DFnome_completo,
    DFemail
FROM sac.TBusuario WITH (NOLOCK)
WHERE DFstatus = 'ativo';

-- ⚠️ Atenção: Só use NOLOCK em leituras onde dirty reads são aceitáveis
-- NÃO use em operações críticas ou transações
```

---

#### ✅ Limitar resultados

```sql
-- Aplicar limite padrão se não especificado
DECLARE @limit INT = ISNULL(JSON_VALUE(@jqel, '$.options.limit'), 100);
DECLARE @offset INT = ISNULL(JSON_VALUE(@jqel, '$.options.offset'), 0);

-- Limitar máximo para prevenir abuse
IF @limit > 1000
    SET @limit = 1000;

-- Usar OFFSET/FETCH para paginação eficiente
SELECT
    t.DFid_usuario,
    t.DFnome_completo
FROM sac.TBusuario t WITH (NOLOCK)
WHERE t.DFativo = 1
ORDER BY t.DFid_usuario
OFFSET @offset ROWS
FETCH NEXT @limit ROWS ONLY;
```

---

#### ✅ Evitar SELECT *

**BOM: Especificar colunas**
```sql
SELECT
    DFid_usuario,
    DFnome_completo,
    DFemail,
    DFstatus
FROM sac.TBusuario WITH (NOLOCK)
WHERE DFstatus = 'ativo';
```

---

#### ✅ Usar EXISTS ao invés de COUNT

**BOM: EXISTS (mais rápido)**
```sql
-- Para de verificar assim que encontra 1 registro
IF EXISTS (SELECT 1 FROM sac.TBusuario WHERE DFemail = @email)
BEGIN
    -- Email já existe
END
```

---

#### ✅ Usar JOINs eficientes

```sql
-- Preferir INNER JOIN quando possível (mais rápido)
SELECT
    c.DFid_chamado,
    c.DFtitulo,
    a.DFnome_completo AS atendente_nome
FROM sac.TBchamado c WITH (NOLOCK)
INNER JOIN sac.TBatendente a WITH (NOLOCK) ON c.DFid_atendente = a.DFid_atendente
WHERE c.DFstatus = 'aberto';

-- LEFT JOIN apenas quando necessário
SELECT
    c.DFid_chamado,
    c.DFtitulo,
    a.DFnome_completo AS atendente_nome  -- Pode ser NULL
FROM sac.TBchamado c WITH (NOLOCK)
LEFT JOIN sac.TBatendente a WITH (NOLOCK) ON c.DFid_atendente = a.DFid_atendente
WHERE c.DFstatus = 'aberto';

-- Evitar subqueries correlacionadas: use JOIN
SELECT
    c.DFid_chamado,
    a.DFnome_completo AS atendente
FROM sac.TBchamado c
INNER JOIN sac.TBatendente a ON c.DFid_atendente = a.DFid_atendente;
```

---

### Transactions

#### ✅ Usar transactions para operações relacionadas

```sql
BEGIN TRY
    BEGIN TRANSACTION;

        -- Operação 1: Inserir usuário
        INSERT INTO sac.TBusuario (DFnome_completo, DFemail, DFdata_cadastro)
        VALUES (@nome_completo, @email, GETDATE());

        DECLARE @id_usuario INT = SCOPE_IDENTITY();

        -- Operação 2: Inserir permissões padrão (relacionada)
        INSERT INTO sac.TBusuario_permissao (DFid_usuario, DFid_permissao)
        SELECT @id_usuario, DFid_permissao
        FROM sac.TBpermissao
        WHERE DFcodigo_permissao IN ('read_chamado', 'create_chamado');

        -- Operação 3: Registrar auditoria (relacionada)
        INSERT INTO sac.TBauditoria (
            DFtabela_afetada,
            DFid_registro_afetado,
            DFtipo_acao,
            DFdata_hora_acao,
            DFid_usuario
        )
        VALUES (
            'TBusuario',
            @id_usuario,
            'CREATE',
            GETDATE(),
            @id_usuario_criacao
        );

    COMMIT TRANSACTION;

    -- Retornar sucesso
    SELECT
        201 AS code,
        'Usuário criado com sucesso' AS message,
        JSON_QUERY((
            SELECT @id_usuario AS id_usuario
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
        )) AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

END TRY
BEGIN CATCH
    -- Rollback em caso de erro
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    -- Retornar erro
    SELECT
        500 AS code,
        ERROR_MESSAGE() AS message,
        NULL AS data
    FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
END CATCH
```

**⚠️ Importante:**
- Sempre faça `ROLLBACK` no `CATCH` se `@@TRANCOUNT > 0`
- Não abra transactions muito longas (lock contention)
- Evite transactions em procedures de leitura

---

#### ✅ Definir isolation level apropriado

```sql
BEGIN TRY
    -- READ COMMITTED (padrão): Lê apenas dados committed
    SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
    BEGIN TRANSACTION;
        -- Operações...
    COMMIT TRANSACTION;

    -- SERIALIZABLE: Mais seguro mas mais lento
    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    BEGIN TRANSACTION;
        -- Operações críticas que não podem ter phantom reads
    COMMIT TRANSACTION;

END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
END CATCH
```

---

### Auditoria

#### ✅ Registrar operações críticas

```sql
-- Após operação bem-sucedida, registrar auditoria
INSERT INTO sac.TBauditoria (
    DFtabela_afetada,
    DFid_registro_afetado,
    DFtipo_acao,
    DFvalores_anteriores,
    DFvalores_novos,
    DFdata_hora_acao,
    DFid_usuario,
    DFendereco_ip
)
VALUES (
    'TBusuario',
    @id_usuario,
    'UPDATE',
    (SELECT * FROM #dados_antigos FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
    (SELECT * FROM #dados_novos FOR JSON PATH, WITHOUT_ARRAY_WRAPPER),
    GETDATE(),
    JSON_VALUE(@user, '$.id_usuario'),
    JSON_VALUE(@user, '$.ip_address')
);
```

---

## Otimização Avançada: Campos Pesados

**⚠️ NOTA IMPORTANTE:** Esta é uma **otimização avançada e rara**. A grande maioria das procedures JQEL **NÃO precisa** implementar essa lógica. Leia esta seção apenas se sua tabela contém campos pesados (BLOBs, imagens, documentos, textos longos).

---

### Campos `output` e `except` no JQEL

O payload JQEL pode conter opcionalmente os campos `output` (lista de campos a incluir) e `except` (lista de campos a excluir). Esses campos **estão disponíveis** no parâmetro `@jqel` para casos específicos onde a procedure precisa considerar quais campos retornar.

**Porém, na maioria dos casos, procedures NÃO devem tratar esses campos!**

---

### Quando NÃO tratar `output` e `except`

**Em geral, procedures JQEL NÃO precisam implementar lógica de `output` e `except`.**

O workflow N8N que invoca as procedures já realiza a filtragem de campos na resposta final. A procedure pode simplesmente retornar todos os campos da entidade, e o workflow se encarrega de incluir/excluir conforme solicitado.

**Vantagem:** Simplifica o código das procedures e centraliza a lógica de projeção no workflow.

**Quando isso é suficiente:**
- ✅ Todos os campos da tabela são primitivos e pequenos (`INT`, `DATETIME`, `VARCHAR(50)`)
- ✅ A tabela tem poucos registros (< 1000)
- ✅ Não há campos `NVARCHAR(MAX)`, `VARBINARY(MAX)`, ou similar
- ✅ A latência da query está dentro do aceitável (< 200ms)

**Exemplo de tabela segura:**
```sql
CREATE TABLE sac.TBusuario (
    DFid_usuario INT PRIMARY KEY,
    DFnome_completo NVARCHAR(255),
    DFemail NVARCHAR(255),
    DFativo BIT,
    DFdata_cadastro DATETIME2
);
```

**Nenhum campo pesado → procedure pode ignorar `output` e `except`.**

---

### Quando DEVE tratar `output` e `except`

**⚠️ CRÍTICO:** Quando a tabela contém **campos pesados**, a procedure **DEVE** validar `output` e `except` antes do SELECT.

**Campos pesados incluem:**
- **Imagens** (`VARBINARY(MAX)`, `IMAGE`)
- **Documentos** (`VARBINARY(MAX)`)
- **BLOBs** (Binary Large Objects)
- **Texto longo** (`NVARCHAR(MAX)`, `TEXT`)
- **JSON complexos** com muitos dados aninhados
- **XML volumosos**

**Por quê?**

Carregar campos pesados do banco de dados é **extremamente custoso** em termos de:
- **I/O do disco**: Leitura de megabytes/gigabytes desnecessários
- **Memória do SQL Server**: Buffer pool ocupado com dados temporários
- **Largura de banda da rede**: Tráfego entre SQL Server → N8N → Frontend
- **Latência da resposta**: Queries que levavam 50ms passam a levar 5s+

**Não adianta o workflow filtrar depois** se os dados já foram carregados do banco. A otimização precisa acontecer **no SELECT**, evitando trazer dados desnecessários.

---

### Implementação: Verificação de Campos Pesados

```sql
CREATE OR ALTER PROCEDURE sac.jqel__select__documento
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- 1. Extrair output e except
        DECLARE @output NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.output');
        DECLARE @except NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.except');

        -- 2. Flags para campos pesados
        DECLARE @incluir_conteudo BIT = 0;
        DECLARE @incluir_preview BIT = 0;

        -- 3. Determinar quais campos pesados incluir
        IF @output IS NOT NULL
        BEGIN
            -- Modo inclusão: apenas campos listados
            IF @output LIKE '%"conteudo_arquivo"%' SET @incluir_conteudo = 1;
            IF @output LIKE '%"preview_imagem"%' SET @incluir_preview = 1;
        END
        ELSE IF @except IS NOT NULL
        BEGIN
            -- Modo exclusão: todos exceto os listados
            SET @incluir_conteudo = CASE WHEN @except LIKE '%"conteudo_arquivo"%' THEN 0 ELSE 1 END;
            SET @incluir_preview = CASE WHEN @except LIKE '%"preview_imagem"%' THEN 0 ELSE 1 END;
        END
        ELSE
        BEGIN
            -- Sem output/except: NÃO incluir campos pesados por padrão
            SET @incluir_conteudo = 0;
            SET @incluir_preview = 0;
        END;

        -- 4. SELECT condicional
        SELECT
            d.DFid_documento AS id_documento,
            d.DFnome_arquivo AS nome_arquivo,
            d.DFtamanho_bytes AS tamanho_bytes,
            d.DFtipo_mime AS tipo_mime,
            d.DFdata_upload AS data_upload,

            -- Campos pesados: apenas se solicitados
            CASE WHEN @incluir_conteudo = 1
                 THEN d.DFconteudo_arquivo
                 ELSE NULL
            END AS conteudo_arquivo,

            CASE WHEN @incluir_preview = 1
                 THEN d.DFpreview_imagem
                 ELSE NULL
            END AS preview_imagem
        INTO #temp
        FROM sac.TBdocumento d WITH (NOLOCK);

        -- 5. Retornar JResult
        SELECT
            200 AS code,
            'Documentos recuperados com sucesso' AS message,
            (SELECT * FROM #temp FOR JSON PATH) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

---

### Exemplos de Query JQEL para Campos Pesados

```json
// ✅ Solicita apenas metadados (sem campos pesados)
{
  "schema": "sac",
  "select": "documento",
  "output": ["id_documento", "nome_arquivo", "tamanho_bytes"]
}

// ❌ Sem output: campos pesados NÃO incluídos por padrão
{
  "schema": "sac",
  "select": "documento"
}

// ✅ Solicita campo pesado explicitamente
{
  "schema": "sac",
  "select": "documento",
  "output": ["id_documento", "nome_arquivo", "conteudo_arquivo"]
}

// ✅ Exclui campo pesado
{
  "schema": "sac",
  "select": "documento",
  "except": ["conteudo_arquivo", "preview_imagem"]
}
```

---

### Boas Práticas para Campos Pesados

1. **Nunca inclua campos pesados por padrão** quando `output` e `except` são omitidos
2. **Use CASE WHEN para SELECT condicional** baseado nas flags
3. **Documente quais campos são considerados pesados** na entidade (comentários, README)
4. **Considere criar actions específicas** para buscar campos pesados:
   ```json
   {"schema": "sac", "select": "documento", "action": "download"}
   {"schema": "sac", "select": "documento", "action": "preview"}
   ```
5. **Crie índices parciais** em tabelas com BLOBs para otimizar queries de metadados:
   ```sql
   CREATE NONCLUSTERED INDEX IX_TBdocumento_metadados
       ON sac.TBdocumento (DFid_documento)
       INCLUDE (DFnome_arquivo, DFtamanho_bytes, DFtipo_mime);
   ```
6. **Monitore o tamanho médio** das respostas em produção usando:
   ```sql
   SELECT
       AVG(DATALENGTH(DFconteudo_arquivo)) AS avg_size_bytes,
       MAX(DATALENGTH(DFconteudo_arquivo)) AS max_size_bytes
   FROM sac.TBdocumento;
   ```
7. **Considere armazenamento externo** (S3, Azure Blob Storage) para arquivos > 1MB
8. **Use streaming** para downloads de arquivos grandes ao invés de carregar tudo na memória

---

### Exemplo Avançado: Procedure com Download Action

```sql
CREATE OR ALTER PROCEDURE sac.jqel__select__documento__download
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- 1. Extrair where
        DECLARE @where NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.where');
        DECLARE @id_documento INT = JSON_VALUE(@where, '$.id_documento.eq');

        IF @id_documento IS NULL
        BEGIN
            SELECT
                400 AS code,
                'ID do documento é obrigatório' AS message,
                NULL AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 2. Verificar se documento existe
        IF NOT EXISTS (SELECT 1 FROM sac.TBdocumento WHERE DFid_documento = @id_documento)
        BEGIN
            SELECT
                404 AS code,
                'Documento não encontrado' AS message,
                NULL AS data
            FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
            RETURN;
        END

        -- 3. Retornar APENAS o arquivo (não outros campos)
        SELECT
            200 AS code,
            'Documento recuperado com sucesso' AS message,
            JSON_QUERY((
                SELECT
                    DFid_documento AS id_documento,
                    DFnome_arquivo AS nome_arquivo,
                    DFtipo_mime AS tipo_mime,
                    DFconteudo_arquivo AS conteudo_arquivo  -- Base64 encoded
                FROM sac.TBdocumento
                WHERE DFid_documento = @id_documento
                FOR JSON PATH, WITHOUT_ARRAY_WRAPPER
            )) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

**Query JQEL:**
```json
{
  "schema": "sac",
  "select": "documento",
  "action": "download",
  "where": {"id_documento": {"eq": 123}}
}
```

---

## Descoberta e Ferramentas

### Listar Procedures JQEL Disponíveis

```sql
-- Listar todas as procedures JQEL do schema sac
SELECT
    SCHEMA_NAME(schema_id) AS schema_name,
    name AS procedure_name,
    create_date,
    modify_date
FROM sys.procedures
WHERE name LIKE 'jqel\_\_%' ESCAPE '\'
  AND SCHEMA_NAME(schema_id) = 'sac'
ORDER BY name;
```

**Resultado:**
```
schema_name | procedure_name                          | create_date         | modify_date
------------|-----------------------------------------|---------------------|--------------------
sac         | jqel__mutate__usuario__ativar          | 2025-01-15 10:00:00 | 2025-01-15 10:00:00
sac         | jqel__mutate__usuario__insert          | 2025-01-15 10:00:00 | 2025-01-15 10:00:00
sac         | jqel__mutate__usuario__update          | 2025-01-15 10:00:00 | 2025-01-15 10:00:00
sac         | jqel__select__usuario                  | 2025-01-15 10:00:00 | 2025-01-15 10:00:00
sac         | jqel__select__usuario__dashboard       | 2025-01-15 10:00:00 | 2025-01-15 10:00:00
```

---

### Verificar se Operação Existe

```sql
-- Verificar se procedure específica existe
IF EXISTS (
    SELECT 1
    FROM sys.procedures
    WHERE name = 'jqel__select__usuario'
      AND SCHEMA_NAME(schema_id) = 'sac'
)
BEGIN
    PRINT 'Procedure exists';
END
ELSE
BEGIN
    PRINT 'Procedure does not exist';
END

-- Contar procedures por schema
SELECT
    SCHEMA_NAME(schema_id) AS schema_name,
    COUNT(*) AS procedure_count
FROM sys.procedures
WHERE name LIKE 'jqel\_\_%' ESCAPE '\'
GROUP BY SCHEMA_NAME(schema_id)
ORDER BY schema_name;
```

---

### Listar Actions de uma Entidade

```sql
-- Listar todas as actions de "usuario"
SELECT
    name AS procedure_name,
    CASE
        WHEN name LIKE 'jqel\_\_select\_\_%' ESCAPE '\' THEN 'select'
        WHEN name LIKE 'jqel\_\_mutate\_\_%' ESCAPE '\' THEN 'mutate'
    END AS operation,
    CASE
        WHEN name LIKE '%\_\_%\_\_%\_\_%' ESCAPE '\' THEN
            SUBSTRING(name, LEN('jqel__') + LEN('select__usuario__') + 1, LEN(name))
        ELSE NULL
    END AS action
FROM sys.procedures
WHERE (name LIKE 'jqel\_\_select\_\_usuario%' ESCAPE '\'
    OR name LIKE 'jqel\_\_mutate\_\_usuario%' ESCAPE '\')
  AND SCHEMA_NAME(schema_id) = 'sac'
ORDER BY operation, name;
```

**Resultado:**
```
procedure_name                  | operation | action
--------------------------------|-----------|-------------------
jqel__mutate__usuario__ativar   | mutate    | ativar
jqel__mutate__usuario__desativar| mutate    | desativar
jqel__mutate__usuario__insert   | mutate    | insert
jqel__mutate__usuario__update   | mutate    | update
jqel__select__usuario           | select    | NULL
jqel__select__usuario__dashboard| select    | dashboard
jqel__select__usuario__relatorio| select    | relatorio
```

---

### Geração Dinâmica de Nome de Procedure

**JavaScript/TypeScript:**

```javascript
function buildProcedureName(query) {
  // Schema é OBRIGATÓRIO
  if (!query.schema) {
    throw new Error('Campo "schema" é obrigatório na query JQEL');
  }

  const schema = query.schema;
  const operation = query.select ? 'select' : 'mutate';
  const entity = query.select || query.mutate;
  const action = query.action || null;

  const parts = ['jqel', operation, entity];
  if (action) parts.push(action);

  return `${schema}.${parts.join('__')}`;
}

// Exemplos:
buildProcedureName({schema: 'sac', select: 'usuario'})
// → 'sac.jqel__select__usuario'

buildProcedureName({schema: 'sac', mutate: 'usuario', action: 'insert'})
// → 'sac.jqel__mutate__usuario__insert'

buildProcedureName({schema: 'sac', select: 'chamado_anexo', action: 'estatistica_consumo'})
// → 'sac.jqel__select__chamado_anexo__estatistica_consumo'
```

---

### Validação de Query JQEL

```javascript
function validateJsqlQuery(query) {
  const errors = [];

  // Validar schema obrigatório
  if (!query.schema) {
    errors.push('Campo "schema" é obrigatório');
  }

  // Validar schema permitido
  const allowedSchemas = ['sac', 'jqel', 'api'];
  if (query.schema && !allowedSchemas.includes(query.schema)) {
    errors.push(`Schema inválido: ${query.schema}. Permitidos: ${allowedSchemas.join(', ')}`);
  }

  // Validar que possui select ou mutate
  if (!query.select && !query.mutate) {
    errors.push('Query deve conter "select" ou "mutate"');
  }

  // Validar que não possui ambos
  if (query.select && query.mutate) {
    errors.push('Query não pode conter "select" e "mutate" simultaneamente');
  }

  // Validar action obrigatório para mutate
  if (query.mutate && !query.action) {
    errors.push('Query "mutate" deve especificar "action"');
  }

  // Validar values obrigatório para mutate
  if (query.mutate && !query.values) {
    errors.push('Query "mutate" deve conter "values"');
  }

  return {
    valid: errors.length === 0,
    errors
  };
}

// Exemplos:
validateJsqlQuery({schema: 'sac', select: 'usuario'})
// → {valid: true, errors: []}

validateJsqlQuery({select: 'usuario'})
// → {valid: false, errors: ['Campo "schema" é obrigatório']}

validateJsqlQuery({schema: 'sac', mutate: 'usuario'})
// → {valid: false, errors: ['Query "mutate" deve especificar "action"', ...]}

validateJsqlQuery({schema: 'sac', select: 'usuario', mutate: 'atendente'})
// → {valid: false, errors: ['Query não pode conter "select" e "mutate" simultaneamente']}
```

---

## Referência Rápida

### Checklist de Implementação

Ao criar uma nova JQEL Procedure, verifique:

**Estrutura básica:**
- [ ] Assinatura padrão: `@user NVARCHAR(MAX) = NULL, @jqel NVARCHAR(MAX)`
- [ ] Nomenclatura correta: `{schema}.jqel__{operation}__{entity}[__{action}]`
- [ ] Action obrigatório para mutate
- [ ] `SET NOCOUNT ON;` no início
- [ ] `TRY/CATCH` implementado

**Extração de dados:**
- [ ] Usar `JSON_QUERY()` para objetos/arrays
- [ ] Usar `JSON_VALUE()` para valores escalares
- [ ] Valores padrão com `ISNULL()` quando apropriado

**Validações:**
- [ ] Campos obrigatórios validados
- [ ] Formato de dados validado (email, data, enum)
- [ ] Foreign keys validados
- [ ] Unique constraints verificados antes de inserir
- [ ] Regras de negócio aplicadas

**Segurança:**
- [ ] Queries parametrizadas (sem concatenação de strings)
- [ ] Verificação de permissões (quando aplicável)
- [ ] Filtros por contexto do usuário (row-level security)
- [ ] Mensagens de erro genéricas (não expor detalhes internos em produção)

**Performance:**
- [ ] Índices criados em colunas de filtro/ordenação
- [ ] `WITH (NOLOCK)` em SELECTs (quando apropriado)
- [ ] Limite padrão e máximo aplicados
- [ ] `EXISTS` ao invés de `COUNT` para verificações
- [ ] SELECT especifica colunas (evita `SELECT *`)

**Otimizações avançadas:**
- [ ] Campos pesados verificados em `output`/`except` (se aplicável)
- [ ] Transactions usadas para operações relacionadas
- [ ] Auditoria de operações críticas

**Retorno:**
- [ ] **Retorna envelope JResult correto**
- [ ] Código HTTP status apropriado (`200`, `201`, `400`, `404`, `500`, etc.)
- [ ] Mensagem descritiva
- [ ] `data` formatado corretamente (JSON)

**Documentação:**
- [ ] Comentários explicando lógica complexa
- [ ] Campos pesados documentados
- [ ] Exemplos de query JQEL documentados (README, comentários)

---

### Resumo das Regras

| Aspecto | Regra |
|---------|-------|
| **Separador principal** | Duplo underscore `__` |
| **Separador interno** | Underscore simples `_` |
| **Ordem** | jqel → operação → entidade → ação |
| **Case** | snake_case (minúsculas) |
| **Schema** | **OBRIGATÓRIO** (`sac`, `jqel`, `api`) |
| **Action para select** | Opcional (views, relatórios) |
| **Action para mutate** | **Obrigatório** (insert, update, delete, etc.) |
| **Prefixo tabelas** | `TB` |
| **Prefixo colunas** | `DF` |
| **Formato de resposta** | JResult (JSON com code, message, data) |
| **Tratamento de erros** | `TRY/CATCH` obrigatório |
| **Código status sucesso** | `200` (SELECT), `201` (INSERT), `204` (DELETE) |
| **Código status erro** | `400` (validação), `404` (não encontrado), `500` (erro interno) |

---

### Templates Rápidos

#### Template: SELECT Básico

```sql
CREATE OR ALTER PROCEDURE sac.jqel__select__{entity}
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        DECLARE @limit INT = ISNULL(JSON_VALUE(@jqel, '$.options.limit'), 100);
        DECLARE @offset INT = ISNULL(JSON_VALUE(@jqel, '$.options.offset'), 0);

        IF @limit > 1000 SET @limit = 1000;

        SELECT TOP (@limit)
            -- Campos aqui
        INTO #temp
        FROM sac.TB{entity} t WITH (NOLOCK)
        ORDER BY t.DFid_{entity}
        OFFSET @offset ROWS;

        SELECT
            200 AS code,
            '{Entity} recuperados com sucesso' AS message,
            (SELECT * FROM #temp FOR JSON PATH) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

---

#### Template: INSERT

```sql
CREATE OR ALTER PROCEDURE sac.jqel__mutate__{entity}__insert
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        DECLARE @values NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.values');
        -- Extrair campos...

        -- Validações...

        INSERT INTO sac.TB{entity} (...)
        VALUES (...);

        DECLARE @id INT = SCOPE_IDENTITY();

        SELECT
            201 AS code,
            '{Entity} criado com sucesso' AS message,
            (SELECT * FROM sac.TB{entity} WHERE DFid_{entity} = @id FOR JSON PATH, WITHOUT_ARRAY_WRAPPER) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

---

#### Template: UPDATE

```sql
CREATE OR ALTER PROCEDURE sac.jqel__mutate__{entity}__update
    @user NVARCHAR(MAX) = NULL,
    @jqel NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        DECLARE @where NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.where');
        DECLARE @values NVARCHAR(MAX) = JSON_QUERY(@jqel, '$.values');

        DECLARE @id INT = JSON_VALUE(@where, '$.id_{entity}.eq');

        -- Validar existe...

        UPDATE sac.TB{entity}
        SET
            -- Campos = ISNULL(@novo_valor, campo_atual)
            DFdata_modificacao = GETDATE()
        WHERE DFid_{entity} = @id;

        SELECT
            200 AS code,
            '{Entity} atualizado com sucesso' AS message,
            (SELECT * FROM sac.TB{entity} WHERE DFid_{entity} = @id FOR JSON PATH, WITHOUT_ARRAY_WRAPPER) AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;

    END TRY
    BEGIN CATCH
        SELECT
            500 AS code,
            ERROR_MESSAGE() AS message,
            NULL AS data
        FOR JSON PATH, WITHOUT_ARRAY_WRAPPER;
    END CATCH
END;
GO
```

---

### Códigos HTTP Status Comuns

| Código | Nome | Quando Usar |
|--------|------|-------------|
| **2xx** | **Sucesso** | |
| `200` | OK | SELECT bem-sucedido, UPDATE sem retornar dados |
| `201` | Created | INSERT bem-sucedido |
| `204` | No Content | DELETE bem-sucedido, UPDATE sem dados de retorno |
| **4xx** | **Erro do Cliente** | |
| `400` | Bad Request | Validação falhou, campo obrigatório faltando, formato inválido |
| `401` | Unauthorized | Usuário não autenticado (`@user` é NULL) |
| `403` | Forbidden | Usuário autenticado mas sem permissão |
| `404` | Not Found | Registro não encontrado |
| `409` | Conflict | Violação de unique constraint, registro já existe |
| `422` | Unprocessable Entity | Regra de negócio violada |
| **5xx** | **Erro do Servidor** | |
| `500` | Internal Server Error | Erro inesperado, exception não tratada |

---

### Próximos Passos

- **[Sintaxe JQEL](sintaxe.md)** - Estrutura completa de queries JQEL (where, options, values)
- **[Mapeamento](mapeamento.md)** - Mapeamento de entidades e campos
- **[README JQEL](README.md)** - Visão geral do sistema JQEL

---

**Versão do documento:** 2.0
**Última atualização:** 2025-10-06
**Autor:** Coletivos Team
