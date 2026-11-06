# Tarefas — modelo de dados

← [Tarefas](README.md) · [Índice do guia](../README.md) · [Tarefas — regras de negócio](regras.md) →

---

## Modelo de dados

Arquivo `tarefas/<uid>.json`:

```json
{
  "uid": "9f0c2b8e-5d4a-4f1e-8b7c-6a5d4e3f2a1b",
  "titulo": "EFD-Reinf — competência 10/2026",
  "descricao": "O que é a tarefa (não muda).",
  "criador": "maria",
  "responsavel": "joao",
  "responsaveis": ["joao", "ana"],
  "empresa": "12345678000190",
  "empresas": ["12345678000190", "98765432000110"],
  "observacao": "",
  "prioridade": 3,
  "prazo": "2026-11-14",
  "prazo_hora": "17:00",
  "concluida": false,
  "concluida_em": null,
  "criado_em": "2026-11-06 09:12:00",
  "atualizado_em": "2026-11-06 09:12:00",
  "comentarios": [
    {"uid": "3c4d5e6f-…", "autor": "ana", "texto": "Falta o extrato.", "criado_em": "2026-11-06 10:01:02.345"}
  ],
  "itens": [
    {"texto": "Empresas mensais", "concluido": false, "ordem": 0, "pai": "", "empresa": ""},
    {"texto": "", "concluido": true, "ordem": 1, "pai": "Empresas mensais", "empresa": "12345678000190"},
    {"texto": "Conferir os recibos", "concluido": false, "ordem": 2, "pai": "", "empresa": ""}
  ]
}
```

Tabelas equivalentes em SQL (📄 `backend/motor/db/banco.py`, resumido):

```sql
CREATE TABLE tarefas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    uid TEXT,                                -- identidade entre computadores
    titulo TEXT NOT NULL, descricao TEXT, observacao TEXT,
    criador_id INTEGER REFERENCES usuarios(id),
    responsavel_id INTEGER REFERENCES usuarios(id),   -- o PRIMEIRO responsável
    empresa_id INTEGER REFERENCES empresas(id) ON DELETE SET NULL,  -- a PRIMEIRA empresa
    prioridade INTEGER NOT NULL DEFAULT 1,   -- 0 baixa · 1 média · 2 alta · 3 urgente
    prazo TEXT,                              -- 'AAAA-MM-DD'
    prazo_hora TEXT,                         -- 'HH:MM' (campo SEPARADO)
    concluida INTEGER NOT NULL DEFAULT 0, concluida_em TEXT,
    resp_login TEXT, criador_login TEXT, empresa_cnpj TEXT,  -- como vieram do arquivo
    criado_em TEXT, atualizado_em TEXT
);
CREATE TABLE tarefa_responsaveis (id INTEGER PRIMARY KEY, tarefa_id INTEGER NOT NULL REFERENCES tarefas(id) ON DELETE CASCADE,
    usuario_id INTEGER REFERENCES usuarios(id), resp_login TEXT, ordem INTEGER NOT NULL DEFAULT 0, UNIQUE(tarefa_id, usuario_id));
CREATE TABLE tarefa_empresas (id INTEGER PRIMARY KEY, tarefa_id INTEGER NOT NULL REFERENCES tarefas(id) ON DELETE CASCADE,
    empresa_id INTEGER REFERENCES empresas(id) ON DELETE CASCADE, empresa_cnpj TEXT, ordem INTEGER NOT NULL DEFAULT 0);
CREATE TABLE tarefa_itens (id INTEGER PRIMARY KEY, tarefa_id INTEGER NOT NULL REFERENCES tarefas(id) ON DELETE CASCADE,
    pai_id INTEGER REFERENCES tarefa_itens(id) ON DELETE CASCADE,          -- item-grupo
    empresa_id INTEGER REFERENCES empresas(id) ON DELETE SET NULL, empresa_cnpj TEXT,
    texto TEXT NOT NULL, concluido INTEGER NOT NULL DEFAULT 0, ordem INTEGER NOT NULL DEFAULT 0);
CREATE TABLE tarefa_comentarios (id INTEGER PRIMARY KEY, uid TEXT, tarefa_id INTEGER NOT NULL REFERENCES tarefas(id) ON DELETE CASCADE,
    usuario_id INTEGER REFERENCES usuarios(id), texto TEXT NOT NULL,
    criado_em TEXT DEFAULT (strftime('%Y-%m-%d %H:%M:%f','now','localtime')));  -- com milésimos
```

Por que cada detalhe existe:

| Decisão | Motivo (já aconteceu) |
|---|---|
| Prazo e hora em campos separados | Os filtros comparam a data com `=` e `<`. Com a hora junto (`2026-11-14 14:30`), a tarefa sumia do dia dela. |
| Vários responsáveis, mas `responsavel` guarda o primeiro | Onde só cabe um nome, e para versões antigas lerem o arquivo. Idem `empresa` × `empresas`. |
| Guardar login/CNPJ como vieram, mesmo desconhecidos | A máquina que não conhecia a pessoa regravava a tarefa **sem dono** — e ela sumia para todo mundo, inclusive do lembrete. |
| Comentários em vez de "campo de observação" | O campo único era apagado sem querer por quem editava depois. |
| Data do comentário com milésimos | Dois recados no mesmo segundo saíam em ordens diferentes em cada computador. |
| Gravar mesclando | Ao gravar, junte com o que um colega gravou nesse meio-tempo (comentários pelo `uid`), em vez de sobrescrever. |

---

← [Tarefas](README.md) · [Índice do guia](../README.md) · [Tarefas — regras de negócio](regras.md) →
