# JQEL - Mapa de Conhecimento

**Tagline:** Write JSON, Query Data

JQEL é uma notação otimizada para frontend JavaScript realizar consultas e mutações de dados no backend. Permite executar operações de dados através de payloads JSON padronizados, sem expor detalhes de implementação.

---

## 📚 Documentação por Feature

### Para Desenvolvedores Frontend

Você vai **consumir** JQEL via API para buscar e modificar dados.

 - [**Sintaxe**](sintaxe.md) - Como construir queries (where, options, output/except)
- [**Operações**](operacoes.md) - Select e Mutate (consultas e modificações)
- [**Respostas**](respostas.md) - Interpretar envelope e status codes

### Para Implementadores Backend

Você vai **implementar** operações JQEL no backend.

- [**Mapeamento**](mapeamento.md) - Regras JSON ↔ Operações
- [**Implementação**](procedures.md) - Como criar handlers compatíveis

### Para Consulta Rápida

- [**Referência**](referencia.md) - Operadores, exemplos e convenções

---

## 🚀 Quick Start

### Consulta (Select)

```json
{
  "schema": "vendapp",
  "select": "usuario",
  "where": { "status": { "eq": "ativo" } },
  "options": { "limit": 10, "orderBy": [{ "field": "nome_completo", "direction": "asc" }] },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Resposta:**
```json
{
  "code": 200,
  "data": [
    { "id_usuario": 1, "nome_completo": "Ana Silva", "email": "ana@example.com" }
  ]
}
```

### Modificação (Mutate)

```json
{
  "schema": "vendapp",
  "mutate": "usuario",
  "action": "insert",
  "values": { "nome_completo": "Carlos Oliveira", "email": "carlos@example.com", "status": "ativo" },
  "output": ["id_usuario", "nome_completo", "email"]
}
```

**Resposta:**
```json
{
  "code": 201,
  "data": [{ "id_usuario": 123, "nome_completo": "Carlos Oliveira", "email": "carlos@example.com" }]
}
```

### Projeção por exclusão (Except)

```json
{
  "schema": "vendapp",
  "select": "usuario",
  "where": { "status": { "eq": "ativo" } },
  "options": { "limit": 10 },
  "except": ["senha_hash", "token_recuperacao"]
}
```

Retorna todos os campos da entidade, exceto os listados em `except`.

---

## 💡 Principais Conceitos

- **`select`**: Leitura de dados (consultas, relatórios)
- **`mutate`**: Escrita de dados (insert, update, delete)
- **`where`**: Filtros declarativos com operadores padronizados
- **`output` / `except`**: Projeção granular de campos
- **`options`**: Paginação, ordenação e limites
- **Response padronizado**: Sempre envelope com `code`, `message`, `data`
