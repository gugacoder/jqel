# Sintaxe JQEL - Guia Completo

**Audiência:** Frontend/API Developers

Este documento é o guia completo de JQEL, cobrindo estrutura, operações (select/mutate), sintaxe (where, options, output/except), formato de resposta e boas práticas.

---

## Estrutura Básica de uma Query JQEL

Toda query JQEL possui campos fundamentais que definem o que será executado e onde.

```json
{
  "schema": "",
  ( "select" | "mutate" ): <entity>,
  "action": "",
  "values": {},
  "where": {},
  "options": {
    "limit": 0,
    "offset": 0,
    "orderBy": []
  },
  "output": [],
  "except": []
}
```

### Schema

Define o schema do banco de dados onde a operação será executada:

```json
{
  "schema": "sac"
}
```

**Esquema comum aos aplicativos do sistema coletivos:** `sac`

---

### Operações: Select e Mutate

#### SELECT - Consultas (leitura)

Usado para buscar dados. Especifica a entidade que será consultada:

```json
{
  "schema": "sac",
  "select": "usuario"
}
```

**Executa:** `sac.jqel__select__usuario`

#### MUTATE - Modificações (escrita)

Usado para inserir, atualizar ou deletar dados. Requer `action`:

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "insert"
}
```

**Executa:** `sac.jqel__mutate__usuario__insert`

---

### Entity (Entidade)

Nome da entidade (tabela) alvo da operação. Sempre no **singular**:

```json
{
  "select": "usuario"     // ✅ Correto
}
```

```json
{
  "select": "usuarios"    // ❌ Errado
}
```

**Exemplos de entidades:** `usuario`, `atendente`, `chamado`, `cliente`, `categoria`

---

### Action (Ação)

Define a ação específica a ser executada.

#### Para SELECT (Opcional)

Action em `select` é opcional e usado para transformar o resultado (views, estatísticas, relatórios):

```json
{
  "select": "atendimento",
  "action": "dashboard"
}
```

**Exemplos de actions para select:**
- `dashboard` - Métricas agregadas
- `relatorio_mensal` - Relatório customizado
- `resumo_compra` - Sumário de compra
- `impressora_zebra` - Formato para impressora

#### Para MUTATE (Obrigatório)

Action em `mutate` é **obrigatório** e define a operação de modificação:

```json
{
  "mutate": "usuario",
  "action": "create"
}
```

**Actions padrão para mutate:**

| Action | Descrição | SQL Equivalente |
|--------|-----------|-----------------|
| `insert` | Inserir novo registro | `INSERT INTO` |
| `update` | Atualizar registro existente | `UPDATE` |
| `delete` | Deletar registro | `DELETE FROM` |
| `upsert` | Atualizar ou inserir | `MERGE` |

**Actions customizadas para mutate:**
- `ativar` - Ativar registros
- `desativar` - Desativar com motivo
- `transferir` - Transferir atendimentos
- `reset_senha` - Reset de senha
- `gerar_doc` - Gerar documento

---

### Exemplo Completo Mínimo

**SELECT:**
```json
{
  "schema": "sac",
  "select": "usuario"
}
```

**MUTATE:**
```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "insert",
  "values": {
    "nome_completo": "João Silva",
    "email": "joao@example.com"
  }
}
```

---

## Formato de Saída (JResult)

Todas as operações JQEL retornam um envelope padronizado chamado **JResult**.

### Estrutura Completa

```json
{
  "code": 200,
  "message": "",
  "field": "",
  "data": [],
  "warnings": [
    { "code": 200, "message": "", "field": "", "data": [] }
  ]
}
```

---

### Campos do Envelope

#### `code` (obrigatório)

**Tipo:** `number`

Status HTTP da operação. Único campo obrigatório.

**Valores comuns:**
- `200` - OK (select, update, delete)
- `201` - Created (insert, upsert quando cria)
- `400` - Bad Request (validação falhou)
- `404` - Not Found (registro não encontrado)
- `409` - Conflict (constraint violada)
- `422` - Unprocessable Entity (erro semântico)
- `500` - Internal Server Error

---

#### `message` (opcional)

**Tipo:** `string`

Mensagem descritiva. Geralmente acompanha falhas.

**Comportamento:**
- **Sucesso (200/201)**: vazio `""` ou omitido
- **Erro (400+)**: descrição do erro

**Exemplos:**
```json
"Campo 'email' é obrigatório"
"Registro não encontrado"
"Email já cadastrado no sistema"
```

---

#### `field` (opcional)

**Tipo:** `string`

Campo específico relacionado ao erro ou validação.

**Uso principal:** Erros de validação (400) e conflitos (409).

**Exemplo:**
```json
{
  "code": 400,
  "message": "Email inválido",
  "field": "email"
}
```

---

#### `data` (condicional)

**Tipo:** `array`

Resultado da operação ou informação adicional.

**Array mesmo quando retornando um único valor:**
```json
{
  "code": 201,
  "data": [{
    "id_usuario": 123,
    "nome": "João"
  }]
}
```

**Pode ser omitido quando não há dados:**
```json
{ "code": 200 }
```

**Ou array vazio em listagens sem resultado:**
```json
{ "code": 200, "data": [] }
```

---

#### `warnings` (opcional)

**Tipo:** `array` de objetos JResult

Coleção de alertas não-bloqueantes. Operação é concluída mesmo com warnings.

**Estrutura de cada warning:**
```json
{
  "code": <number>,
  "message": <string>,
  "field": <string>,
  "data": <array>
}
```

**Exemplo:**
```json
{
  "code": 200,
  "data": [{ "id": 1, "nome": "João" }],
  "warnings": [
    {
      "code": 299,
      "message": "Campo 'telefone_fixo' será removido em 2025-06-01",
      "field": "telefone_fixo"
    }
  ]
}
```

---

### Códigos de Status HTTP

#### Códigos de Sucesso (2xx)

##### 200 OK

Operação bem-sucedida.

**Quando usar:**
- SELECT encontrou dados ou não encontrou (retorna `[]`)
- UPDATE bem-sucedido
- DELETE bem-sucedido
- UPSERT que atualizou registro existente

**Exemplos:**
```json
// SELECT com dados
{ "code": 200, "data": [{...}] }

// SELECT sem dados
{ "code": 200, "data": [] }

// UPDATE
{ "code": 200, "data": [{...}] }

// DELETE
{ "code": 200 }
```

---

##### 201 Created

Novo registro criado com sucesso.

**Quando usar:**
- INSERT bem-sucedido
- UPSERT que criou novo registro

**Exemplo:**
```json
{
  "code": 201,
  "data": [{
    "id_usuario": 123,
    "nome": "João Silva"
  }]
}
```

---

##### 299 Deprecation Warning

Alerta de funcionalidade que será removida (usado em `warnings`).

**Exemplo:**
```json
{
  "code": 200,
  "data": [...],
  "warnings": [
    {
      "code": 299,
      "message": "Campo 'telefone_fixo' será removido em 2025-06-01",
      "field": "telefone_fixo"
    }
  ]
}
```

---

#### Códigos de Redirecionamento (3xx)

##### 301 Moved Permanently

Recurso movido permanentemente para nova URL.

**Exemplo:**
```json
{
  "code": 301,
  "message": "Recurso movido permanentemente",
  "data": [{ "see": "https://api.exemplo.com/v2/usuarios/123" }]
}
```

---

#### Códigos de Erro do Cliente (4xx)

##### 400 Bad Request

Requisição inválida devido a erro de sintaxe ou validação.

**Quando usar:**
- Sintaxe JSON inválida
- Campos obrigatórios ausentes
- Valores inválidos (formato, tipo, range)
- Operadores desconhecidos
- Estrutura de payload incorreta

**Exemplos:**
```json
// Campo obrigatório ausente
{
  "code": 400,
  "message": "Campo 'email' é obrigatório",
  "field": "email"
}

// Valor inválido
{
  "code": 400,
  "message": "Email inválido",
  "field": "email"
}

// Operador desconhecido
{
  "code": 400,
  "message": "Operador 'contains' não é suportado",
  "field": "where.nome"
}
```

---

##### 401 Unauthorized

Autenticação necessária ou credenciais inválidas.

**Quando usar:**
- Token JWT ausente ou inválido
- Credenciais incorretas
- Token expirado

**Exemplo:**
```json
{
  "code": 401,
  "message": "Token de autenticação inválido ou expirado"
}
```

---

##### 403 Forbidden

Autenticado, mas sem permissão para o recurso.

**Quando usar:**
- Usuário sem permissão para a operação
- Tentativa de acessar recurso de outra empresa
- Operação bloqueada por regra de negócio

**Exemplo:**
```json
{
  "code": 403,
  "message": "Você não tem permissão para deletar usuários"
}
```

---

##### 404 Not Found

Recurso não encontrado.

**Quando usar:**
- Entidade não existe
- Registro específico não encontrado (quando esperado)
- Procedure JQEL não existe

**Exemplos:**
```json
{
  "code": 404,
  "message": "Usuário com id 999 não encontrado"
}

{
  "code": 404,
  "message": "Procedure jqel__select__xyz não encontrada"
}
```

**Nota:** SELECT sem resultados retorna `200` com array vazio, não `404`.

---

##### 409 Conflict

Conflito com estado atual do recurso.

**Quando usar:**
- Violação de constraint UNIQUE
- Violação de constraint FOREIGN KEY
- Violação de regra de negócio que impede a operação

**Exemplos:**
```json
// Unique constraint
{
  "code": 409,
  "message": "Email já cadastrado no sistema",
  "field": "email",
  "data": [{ "existing_id": 456 }]
}

// Foreign key
{
  "code": 409,
  "message": "Não é possível deletar. Existem 5 registros dependentes.",
  "data": [{
    "dependent_count": 5,
    "dependent_entity": "atendimento"
  }]
}

// Business rule
{
  "code": 409,
  "message": "Usuário não pode ser desativado enquanto houver atendimentos em andamento",
  "data": [{ "active_atendimentos": 3 }]
}
```

---

##### 422 Unprocessable Entity

Requisição bem formada, mas com erros semânticos.

**Quando usar:**
- Dados válidos sintaticamente mas inválidos semanticamente
- Validações de negócio complexas

**Exemplos:**
```json
{
  "code": 422,
  "message": "Data de término deve ser posterior à data de início",
  "field": "data_termino"
}

{
  "code": 422,
  "message": "Não é possível realizar esta operação no status atual",
  "data": [{
    "current_status": "finalizado",
    "required_status": ["pendente", "em_andamento"]
  }]
}
```

---

##### 429 Too Many Requests

Limite de taxa de requisições excedido.

**Exemplo:**
```json
{
  "code": 429,
  "message": "Limite de requisições excedido. Tente novamente em 60 segundos.",
  "data": [{ "retry_after": 60 }]
}
```

---

#### Códigos de Erro do Servidor (5xx)

##### 500 Internal Server Error

Erro interno do servidor.

**Quando usar:**
- Erro não tratado na procedure
- Erro de lógica interna
- Exceção inesperada

**Exemplo:**
```json
{
  "code": 500,
  "message": "Erro interno ao processar requisição"
}
```

**Nota:** Em produção, evite expor detalhes internos. Logue a stack trace internamente.

---

##### 503 Service Unavailable

Serviço temporariamente indisponível.

**Quando usar:**
- Banco de dados inacessível
- Manutenção programada
- Sobrecarga temporária

**Exemplo:**
```json
{
  "code": 503,
  "message": "Serviço temporariamente indisponível. Tente novamente em instantes.",
  "data": [{ "retry_after": 30 }]
}
```

---

### Diretrizes de Uso

#### Quando usar 200 vs 404 em SELECT

**200 com array vazio:**
```json
// Query válida, nenhum registro atende ao filtro
{
  "schema": "sac",
  "select": "usuario",
  "where": { "status": { "eq": "pendente" } }
}
→ { "code": 200, "data": [] }
```

**404:**
```json
// Recurso específico esperado não existe
{
  "schema": "sac",
  "select": "usuario",
  "where": { "id_usuario": { "eq": 999 } }
}
→ { "code": 404, "message": "Usuário 999 não encontrado" }
```

**Regra geral:** Use `404` quando a ausência do registro é considerada erro. Use `200` com `data: []` em listagens.

---

#### Quando usar 400 vs 422

**400 - Erro de sintaxe/estrutura:**
```json
// JSON inválido, campo obrigatório ausente, tipo errado
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "insert"
  // "values" obrigatório ausente
}
→ { "code": 400, "message": "Campo 'values' é obrigatório" }
```

**422 - Erro semântico/lógico:**
```json
// Estrutura correta, mas valores incompatíveis
{
  "schema": "sac",
  "mutate": "evento",
  "action": "insert",
  "values": {
    "data_inicio": "2024-01-31",
    "data_fim": "2024-01-01"  // Antes do início
  }
}
→ { "code": 422, "message": "Data fim deve ser posterior ao início" }
```

---

#### Quando usar 409 vs 422

**409 - Conflito com estado/constraint:**
```json
// Constraint UNIQUE violada
{ "code": 409, "message": "Email já existe" }

// FK constraint
{ "code": 409, "message": "Não é possível deletar. Registros dependentes existem." }
```

**422 - Validação de regra de negócio:**
```json
// Regra de negócio impede operação
{ "code": 422, "message": "Não é possível alterar status de 'finalizado' para 'pendente'" }
```

---

### Tabela de Referência Rápida

| Código | Nome | Uso em JQEL |
|--------|------|-------------|
| 200 | OK | SELECT, UPDATE, DELETE bem-sucedidos |
| 201 | Created | INSERT, UPSERT (criação) bem-sucedidos |
| 299 | Deprecation | Warning de depreciação (em warnings) |
| 301 | Moved Permanently | Recurso movido |
| 400 | Bad Request | Validação, sintaxe, campos obrigatórios |
| 401 | Unauthorized | Autenticação ausente/inválida |
| 403 | Forbidden | Sem permissão |
| 404 | Not Found | Entidade/registro não encontrado |
| 409 | Conflict | Violação de constraint, conflito |
| 422 | Unprocessable | Erro semântico, validação de negócio |
| 429 | Too Many Requests | Rate limit excedido |
| 500 | Internal Server Error | Erro interno não tratado |
| 503 | Service Unavailable | Serviço indisponível |

---

## SELECT - Operações de Leitura

### Estrutura Básica

```json
{
  "schema": "<schema>",             // OBRIGATÓRIO
  "select": "<entidade>",
  "action": "<ação>",               // Opcional — view/transformação
  "where": { /* filtros */ },       // Opcional
  "options": { /* configs */ },     // Opcional
  "output": [ /* campos */ ],       // Opcional (projeção por inclusão)
  "except": [ /* campos */ ]        // Opcional (projeção por exclusão)
}
```

**Observação:** `output` e `except` são mutuamente exclusivos.

---

### Exemplos de Select

#### Listar Todos

```json
{
  "schema": "sac",
  "select": "usuario",
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 1, "nome_completo": "Maria Silva", "email": "maria@example.com" },
    { "id_usuario": 2, "nome_completo": "João Santos", "email": "joao@example.com" }
  ]
}
```

---

#### Buscar por ID

```json
{
  "schema": "sac",
  "select": "usuario",
  "where": { "id_usuario": { "eq": 123 } },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Resposta (sempre array):**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 123, "nome_completo": "Carlos Oliveira", "email": "carlos@example.com" }
  ]
}
```

**Não encontrado:**
```json
{
  "code": 404
}
```

---

#### Projeção por Exclusão (Except)

```json
{
  "schema": "sac",
  "select": "usuario",
  "where": { "status": { "eq": "ativo" } },
  "except": ["senha_hash", "token_recuperacao"]
}
```

Retorna todos os campos, exceto os listados.

---

#### Paginação

```json
{
  "schema": "sac",
  "select": "usuario",
  "where": { "status": { "eq": "ativo" } },
  "options": {
    "limit": 10,
    "offset": 0,
    "orderBy": [{ "data_criacao": "desc" }]
  },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

---

#### Com Joins (Campos Relacionados)

```json
{
  "schema": "sac",
  "select": "atendimento",
  "where": { "status": { "eq": "finalizado" } },
  "options": {
    "limit": 20,
    "orderBy": [{ "data_atendimento": "desc" }]
  },
  "output": [
    "id_atendimento",
    "data_atendimento",
    "cliente.nome",
    "cliente.telefone",
    "atendente.nome_completo"
  ]
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    {
      "id_atendimento": 1523,
      "data_atendimento": "2024-01-31T16:45:00Z",
      "cliente": {
        "nome": "Empresa XYZ Ltda",
        "telefone": "11987654321"
      },
      "atendente": {
        "nome_completo": "Maria Santos"
      }
    }
  ]
}
```

---

### Actions Customizadas (Select)

Actions permitem operações específicas além da busca padrão. O resultado de um select com action tende a ser específico. Consulte a documentação da action para entender o formato.

#### `ultimasAcoes` - Histórico do usuário

```json
{
  "schema": "sac",
  "select": "usuario",
  "action": "ultimasAcoes",
  "where": { "id_usuario": { "eq": 123 } },
  "options": { "limit": 10 }
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    {
      "data_acao": "2024-01-15T14:30:00Z",
      "tipo_acao": "login",
      "ip_origem": "192.168.1.100"
    }
  ]
}
```

---

#### `dashboard` - Métricas agregadas

```json
{
  "schema": "sac",
  "select": "atendimento",
  "action": "dashboard",
  "where": {
    "data_atendimento": {
      "gte": "2024-01-01",
      "lte": "2024-01-31"
    }
  }
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    {
      "total_atendimentos": 1543,
      "atendimentos_finalizados": 1420,
      "tempo_medio_minutos": 45,
      "satisfacao_media": 4.2
    }
  ]
}
```

**Nota:** Mesmo agregações únicas retornam array.

---

#### `relatorio` - Relatórios customizados

```json
{
  "schema": "sac",
  "select": "vendas",
  "action": "relatorioMensal",
  "where": {
    "mes": { "eq": 1 },
    "ano": { "eq": 2024 }
  },
  "options": {
    "orderBy": [{ "total_vendas": "desc" }]
  }
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    {
      "vendedor": "Carlos Silva",
      "total_vendas": 450000.00,
      "quantidade_vendas": 85,
      "ticket_medio": 5294.12
    }
  ]
}
```

---

## MUTATE - Operações de Escrita

### Estrutura Básica

```json
{
  "schema": "<schema>",       // OBRIGATÓRIO
  "mutate": "<entidade>",
  "action": "<ação>",         // Obrigatório: insert, update, delete, upsert, etc
  "where": { /* filtros */ }, // Para update/delete
  "values": { /* dados */ },  // Para insert/update/upsert
  "output": [ /* campos */ ], // Opcional (projeção por inclusão)
  "except": [ /* campos */ ]  // Opcional (projeção por exclusão)
}
```

---

### INSERT - Criar Registro

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "insert",
  "values": {
    "nome_completo": "Pedro Alves",
    "email": "pedro@example.com",
    "telefone": "11999887766",
    "status": "ativo"
  },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Resposta - Sucesso:**
```json
{
  "code": 201,
  "data": [{
    "id_usuario": 456,
    "nome_completo": "Pedro Alves",
    "email": "pedro@example.com"
  }]
}
```

**Resposta - Validação:**
```json
{
  "code": 400,
  "field": "email",
  "message": "Campo 'email' é obrigatório"
}
```

**Resposta - Conflito (email duplicado):**
```json
{
  "code": 409,
  "field": "email",
  "message": "Email já cadastrado no sistema",
  "data": [{ "existing_id": 123 }]
}
```

---

### UPDATE - Atualizar Registro(s)

#### Atualizar um registro

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "update",
  "where": { "id_usuario": { "eq": 123 } },
  "values": {
    "telefone": "11999887766",
    "status": "inativo"
  },
  "output": ["id_usuario", "telefone", "status"]
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 123, "telefone": "11999887766", "status": "inativo" }
  ]
}
```

---

#### Atualizar múltiplos registros

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "update",
  "where": {
    "status": { "eq": "pendente" },
    "data_cadastro": { "lt": "2023-01-01" }
  },
  "values": { "status": "inativo" },
  "output": ["id_usuario", "status"]
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 10, "status": "inativo" },
    { "id_usuario": 25, "status": "inativo" },
    { "id_usuario": 43, "status": "inativo" }
  ]
}
```

---

### DELETE - Remover Registro(s)

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "delete",
  "where": { "id_usuario": { "eq": 123 } }
}
```

**Resposta:**
```json
{ "code": 200 }
```

**Ou com contador:**
```json
{
  "code": 200,
  "data": [{ "deleted_count": 5 }]
}
```

**Não encontrado:**
```json
{
  "code": 404,
  "message": "Nenhum registro encontrado para deletar"
}
```

**Nota:** `output`/`except` são ignorados em delete.

---

### UPSERT - Update ou Insert

```json
{
  "schema": "sac",
  "mutate": "configuracao",
  "action": "upsert",
  "values": {
    "chave": "tema_padrao",
    "valor": "dark",
    "id_usuario": 123
  },
  "output": ["id_configuracao", "chave", "valor"]
}
```

**Resposta - Criado:**
```json
{
  "code": 201,
  "data": [{
    "id_configuracao": 789,
    "chave": "tema_padrao",
    "valor": "dark"
  }]
}
```

**Resposta - Atualizado:**
```json
{
  "code": 200,
  "data": [
    { "id_configuracao": 789, "chave": "tema_padrao", "valor": "dark" }
  ]
}
```

---

### Actions Customizadas (Mutate)

#### `ativar` - Ativar registros

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "ativar",
  "where": { "id_usuario": { "in": [10, 20, 30] } }
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 10, "status": "ativo" },
    { "id_usuario": 20, "status": "ativo" },
    { "id_usuario": 30, "status": "ativo" }
  ]
}
```

---

#### `desativar` - Desativar com motivo

```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "desativar",
  "where": { "id_usuario": { "eq": 123 } },
  "values": {
    "motivo_desativacao": "Solicitação do usuário",
    "data_desativacao": "2024-01-15T10:00:00Z"
  }
}
```

---

#### `transferir` - Transferir atendimentos

```json
{
  "schema": "sac",
  "mutate": "atendimento",
  "action": "transferir",
  "where": {
    "id_atendente_origem": { "eq": 10 },
    "status": { "in": ["pendente", "em_andamento"] }
  },
  "values": {
    "id_atendente_destino": 25,
    "motivo_transferencia": "Redistribuição de carga"
  }
}
```

---

## WHERE - Condições de Filtro

Define condições para filtrar registros. Se ausente, a operação se aplica a todos os registros.

### Operadores Lógicos

#### AND (implícito) - Todas as condições verdadeiras

**Entre campos diferentes:**
```json
{
  "status": { "eq": "ativo" },
  "idade": { "gte": 18 }
}
```

**SQL:** `WHERE status = 'ativo' AND idade >= 18`

**No mesmo campo (múltiplos operadores):**
```json
{
  "idade": { "gte": 18, "lte": 65 }
}
```

**SQL:** `WHERE idade >= 18 AND idade <= 65`

---

#### `or` - Pelo menos uma condição verdadeira

```json
{
  "or": [
    { "status": { "eq": "ativo" } },
    { "status": { "eq": "pendente" } }
  ]
}
```

**SQL:** `WHERE status = 'ativo' OR status = 'pendente'`

---

#### `not` - Negação

```json
{
  "not": { "status": { "eq": "inativo" } }
}
```

**SQL:** `WHERE NOT (status = 'inativo')`

---

### Operadores Relacionais

| Operador | Significado | Exemplo | SQL |
|----------|-------------|---------|-----|
| `eq` | Igual | `{"status": {"eq": "ativo"}}` | `status = 'ativo'` |
| `ne` | Diferente | `{"status": {"ne": "inativo"}}` | `status != 'inativo'` |
| `gt` | Maior que | `{"idade": {"gt": 18}}` | `idade > 18` |
| `gte` | Maior ou igual | `{"idade": {"gte": 18}}` | `idade >= 18` |
| `lt` | Menor que | `{"prioridade": {"lt": 5}}` | `prioridade < 5` |
| `lte` | Menor ou igual | `{"prioridade": {"lte": 5}}` | `prioridade <= 5` |

---

### Intervalos (Múltiplos Operadores)

Você pode combinar múltiplos operadores relacionais no **mesmo campo** para criar intervalos no estilo BETWEEN. Eles são aplicados com **AND implícito**.

#### Intervalo inclusivo

```json
{
  "data_cadastro": {
    "gte": "2024-01-01",
    "lte": "2024-12-31"
  }
}
```

**SQL:** `WHERE data_cadastro >= '2024-01-01' AND data_cadastro <= '2024-12-31'`

#### Intervalo exclusivo

```json
{
  "preco": {
    "gt": 100,
    "lt": 500
  }
}
```

**SQL:** `WHERE preco > 100 AND preco < 500`

#### Intervalo misto

```json
{
  "idade": {
    "gte": 18,
    "lt": 65
  }
}
```

**SQL:** `WHERE idade >= 18 AND idade < 65`

#### Combinação com exclusão

```json
{
  "status_codigo": {
    "gte": 200,
    "ne": 204
  }
}
```

**SQL:** `WHERE status_codigo >= 200 AND status_codigo != 204`

---

### Operador de Conjunto

#### `in` - Está em lista

```json
{ "status": { "in": ["ativo", "pendente", "processando"] } }
```

**SQL:** `WHERE status IN ('ativo', 'pendente', 'processando')`

---

### Operador de Padrão

#### `like` - Padrão de texto

```json
{ "nome": { "like": "%Silva%" } }
```

**SQL:** `WHERE nome LIKE '%Silva%'`

**Wildcards:**
- `%` - 0 ou mais caracteres
- `_` - Exatamente 1 caractere

**Nota:** Case-insensitive no SQL Server (collation padrão).

---

### Valores Especiais

#### NULL

```json
{ "telefone": { "eq": null } }  // IS NULL
{ "telefone": { "ne": null } }  // IS NOT NULL
```

#### Boolean

```json
{ "ativo": { "eq": true } }   // WHERE ativo = 1
{ "ativo": { "eq": false } }  // WHERE ativo = 0
```

#### Datas (ISO 8601)

```json
{ "data_cadastro": { "eq": "2024-01-15" } }
{ "data_hora": { "gte": "2024-01-15T10:00:00Z" } }
```

---

### Campos Relacionados

Use notação de ponto:

```json
{
  "cliente.cidade": { "eq": "São Paulo" },
  "atendente.status": { "eq": "disponivel" }
}
```

---

### Exemplo Complexo

```json
{
  "schema": "sac",
  "select": "usuario",
  "where": {
    "or": [
      { "status": { "eq": "ativo" } },
      { "status": { "eq": "pendente" } }
    ],
    "data_cadastro": { "gte": "2024-01-01" },
    "empresa.id_empresa": { "in": [10, 20, 30] }
  }
}
```

**SQL:**
```sql
WHERE (DFstatus = 'ativo' OR DFstatus = 'pendente')
  AND DFdata_cadastro >= '2024-01-01'
  AND DFempresa.DFid_empresa IN (10, 20, 30)
```

---

## OPTIONS - Configurações de Query

Configura paginação, ordenação e parâmetros customizados.

### Parâmetros Comuns

#### `limit` - Número máximo de registros

```json
{ "options": { "limit": 50 } }
```

**SQL:** `SELECT TOP 50 ...`

**Recomendação:** Máximo de 1000 registros.

---

#### `offset` - Deslocamento inicial (paginação)

```json
{ "options": { "limit": 10, "offset": 20 } }
```

**SQL:** `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY`

**Importante:** `offset` requer `orderBy` no SQL Server.

---

#### `orderBy` - Ordenação

Array de objetos com campo e direção (`asc` ou `desc`).

```json
{
  "orderBy": [
    { "data_criacao": "desc" },
    { "nome_completo": "asc" }
  ]
}
```

**SQL:** `ORDER BY DFdata_criacao desc, DFnome_completo asc`

---

### Paginação Completa

```json
{
  "schema": "sac",
  "select": "usuario",
  "where": { "status": { "eq": "ativo" } },
  "options": {
    "limit": 10,
    "offset": 10,
    "orderBy": [{ "data_criacao": "desc" }]
  },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Cálculo do offset:**
```javascript
const page = 2; // Página desejada (1-indexed)
const pageSize = 10;
const offset = (page - 1) * pageSize; // (2 - 1) * 10 = 10
```

---

### Cursor-Based Pagination (Alternativa)

Para grandes volumes, use filtros em vez de offset, quando possível:

```json
// Primeira página
{
  "schema": "sac",
  "select": "usuario",
  "options": {
    "limit": 10,
    "orderBy": [{ "id_usuario": "asc" }]
  }
}
// Resposta: último id_usuario = 10

// Próxima página
{
  "schema": "sac",
  "select": "usuario",
  "where": { "id_usuario": { "gt": 10 } },
  "options": {
    "limit": 10,
    "orderBy": [{ "id_usuario": "asc" }]
  }
}
```

**Vantagens:**
- Performance constante
- Não pula registros em inserções concorrentes
- Mais eficiente para tabelas grandes

---

## Projeção de Campos (output/except)

Controla os campos retornados na resposta.

### `output` - Inclusão explícita

Especifica quais campos retornar.

#### Estrutura

Array de strings com nomes dos campos:

```json
{
  "schema": "sac",
  "select": "usuario",
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 1, "nome_completo": "Maria Silva", "email": "maria@example.com" }
  ]
}
```

---

### `except` - Exclusão explícita

Retorna todos os campos da entidade, exceto os listados:

```json
{
  "schema": "sac",
  "select": "usuario",
  "except": ["senha_hash", "token_recuperacao"]
}
```

#### Campos aninhados

Use notação de ponto para subcampos. Excluir o pai remove o objeto inteiro; excluir um subcampo preserva os demais.

```json
{
  "schema": "sac",
  "select": "atendimento",
  "except": [
    "cliente",                 // remove todo objeto cliente
    "atendente.telefone"       // remove só o telefone do atendente
  ]
}
```

---

### Regras e Interação

- `output` e `except` são **mutuamente exclusivos**. Não use ambos na mesma query.
- Se ambos forem enviados, a requisição deve falhar com `400` (mensagem: "Não use 'output' e 'except' juntos").
- Se nenhum for enviado:
  - **SELECT**: Retorna todos os campos da entidade (ou conforme lógica da action);
  - **INSERT/UPDATE**: Retorna campos conforme definido pela procedure.

---

### Campos Relacionados

Use notação de ponto:

```json
{
  "output": [
    "id_atendente",
    "nome_completo",
    "empresa.id_empresa",
    "empresa.nome_fantasia",
    "usuario_responsavel.email"
  ]
}
```

**Resposta aninhada:**
```json
{
  "code": 200,
  "data": [
    {
      "id_atendente": 1,
      "nome_completo": "Carlos Oliveira",
      "empresa": {
        "id_empresa": 10,
        "nome_fantasia": "Tech Solutions"
      },
      "usuario_responsavel": {
        "email": "carlos@techsolutions.com"
      }
    }
  ]
}
```

---

### Performance

#### ✅ Especificar apenas campos necessários

```json
// Bom: apenas o necessário
{ "output": ["id_usuario", "email"] }

// Ruim: campos desnecessários
{ "output": ["id_usuario", "email", "nome_completo", "telefone", "endereco", "cidade", "estado", "cep", "foto", "biografia"] }
```

#### ✅ Evitar campos grandes (BLOBs) sem necessidade

```json
// Se não precisa do campo `foto`, não inclua
{ "output": ["id_usuario", "nome_completo"] }
```

#### ✅ Usar projeção em listagens (output ou except)

```json
{
  "schema": "sac",
  "select": "usuario",
  "options": { "limit": 100 },
  "output": ["id_usuario", "nome_completo", "status"]
}
```

Ou, por exclusão:

```json
{
  "schema": "sac",
  "select": "usuario",
  "options": { "limit": 100 },
  "except": ["foto_blob", "biografia"]
}
```

---

## Boas Práticas

### WHERE

✅ **AND é implícito no objeto; use `or` para OU**
```json
{ "or": [
  { "status": { "eq": "ativo" }, "idade": { "gte": 18 } },
  { "data_cadastro": { "gte": "2024-01-01", "lte": "2024-12-31" } }
]}
```

✅ **Múltiplos operadores no mesmo campo para intervalos**
```json
{ "data_cadastro": { "gte": "2024-01-01", "lte": "2024-12-31" } }
```

✅ **Usar IN para múltiplos valores**
```json
{ "status": { "in": ["ativo", "pendente"] } }  // Melhor que múltiplos OR
```

---

### OPTIONS

✅ **Sempre usar orderBy com offset**
```json
{ "options": { "limit": 10, "offset": 20, "orderBy": [{ "id": "asc" }] } }
```

✅ **Ordenar por campos indexados**
```json
{ "orderBy": [{ "id_usuario": "asc" }] }  // PK, indexada
```

---

### PROJEÇÃO

✅ **Sempre especificar projeção em listagens** (via `output` ou `except`)
```json
{ "schema": "sac", "select": "log_acesso", "options": { "limit": 50 }, "output": ["id", "usuario", "data_hora", "acao"] }
```

✅ **Incluir IDs de entidades relacionadas**
```json
{ "output": ["id_atendimento", "id_cliente", "id_atendente", "status"] }
```

---

### SELECT

✅ **Sempre especificar projeção em listagens**
```json
{ "schema": "sac", "select": "usuario", "output": ["id_usuario", "nome_completo", "email"] }
```

✅ **Usar paginação em queries grandes**
```json
{ "options": { "limit": 100, "orderBy": [{ "id": "asc" }] } }
```

✅ **Filtrar sempre que possível**
```json
{ "where": { "data_atendimento": { "gte": "2024-01-01" } } }
```

---

### MUTATE

✅ **Sempre usar WHERE em update/delete**
```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "update",
  "where": { "id_usuario": { "eq": 123 } },
  "values": { "status": "ativo" }
}
```

✅ **Especificar output para confirmar mudanças**
```json
{
  "schema": "sac",
  "mutate": "usuario",
  "action": "update",
  "where": { "id_usuario": { "eq": 123 } },
  "values": { "status": "ativo" },
  "output": ["id_usuario", "status", "data_atualizacao"]
}
```

---

## Referência Rápida

### Query Completa

```json
{
  "schema": "sac",
  "select": "usuario",
  "where": {
    "status": { "eq": "ativo" },
    "idade": { "gte": 18 }
  },
  "options": {
    "limit": 10,
    "offset": 0,
    "orderBy": [{ "nome_completo": "asc" }]
  },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

---

### Operadores WHERE

| Categoria | Operadores |
|-----------|-----------|
| Lógicos | `or`, `not` (AND é implícito) |
| Relacionais | `eq`, `ne`, `gt`, `gte`, `lt`, `lte` |
| Conjunto | `in` |
| Padrão | `like` |

**Intervalos:** Use múltiplos operadores no mesmo campo (AND implícito)
```json
{ "campo": { "gte": valor1, "lte": valor2 } }
```

---

### Códigos de Status HTTP

| Código | Significado |
|--------|-------------|
| `200` | OK - Operação bem-sucedida |
| `201` | Created - Registro criado |
| `400` | Bad Request - Validação falhou |
| `401` | Unauthorized - Autenticação ausente/inválida |
| `403` | Forbidden - Sem permissão |
| `404` | Not Found - Não encontrado |
| `409` | Conflict - Constraint violada |
| `422` | Unprocessable Entity - Erro semântico |
| `500` | Internal Server Error |

---

### Limites Recomendados

- **`limit`**: Máximo 1000 registros
- **`offset`**: Evitar valores muito altos (usar cursor-based)
- **`orderBy`**: Máximo 5 campos simultâneos

---

**Próximos passos:**
- [Procedures](procedures.md) - Implementação de stored procedures JQEL
