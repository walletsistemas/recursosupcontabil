# Tarefas — API

← [Tarefas — regras de negócio](regras.md) · [Índice do guia](../README.md) · [Tarefas — lista de conferência](conferencia.md) →

---

## API

```
GET    /api/tarefas?filtro=&busca=&dia=&so_minhas=   lista + contagens + pode_criar
GET    /api/tarefas/mes?ano=&mes=&so_minhas=         dias do mês com total/abertas/atrasadas
GET    /api/tarefas/contador                         nº em aberto em que EU sou responsável (menu)
GET    /api/tarefas/opcoes?pessoas=1,2&empresas=3    pessoas e empresas para escolher (+ as já escolhidas)
POST   /api/tarefas                                  criar → { ok, ids: [...] }
GET    /api/tarefas/{id}                             detalhe com checklist em árvore
PUT    /api/tarefas/{id}                             editar
DELETE /api/tarefas/{id}                             excluir
POST   /api/tarefas/{id}/concluir      {concluido}
POST   /api/tarefas/{id}/itens/{item}  {concluido}   → { ok, tarefa_concluida: true|false|null }
GET    /api/tarefas/{id}/comentarios
POST   /api/tarefas/{id}/comentarios   {texto}
DELETE /api/tarefas/{id}/comentarios/{cid}
```

Filtros: `aberto` (padrão), `atrasadas`, `hoje`, `semana` (próximos 7 dias), `sem_prazo`,
`concluidas`, `todas`.

Tipo do cartão na tela — 📄 `frontend/src/api/tarefas.ts`

```ts
export type Situacao = "sem" | "atrasada" | "hoje" | "amanha" | "futuro" | "entregue" | "entregue_atrasada";

export type Tarefa = {
  id: number; titulo: string; descricao: string; prioridade: number;
  prazo: string; prazo_hora: string; prazo_texto: string; situacao: Situacao; concluida: boolean;
  responsaveis: { id: number; nome: string }[]; criador_id: number | null; criador_nome: string;
  empresas: string[]; n_itens: number; n_feitos: number;
  n_comentarios: number; ultimo_comentario: string; ultimo_comentario_autor: string;
  progresso: number;                                   // 0.0 a 1.0
  pode_editar: boolean; pode_concluir: boolean; pode_excluir: boolean; pode_comentar: boolean;
};

export type ItemChecklist = {
  id: number; texto: string; concluido: boolean;
  empresa_id: number | null; empresa_nome: string;
  grupo: boolean; filhos: ItemChecklist[];
};
```

---

← [Tarefas — regras de negócio](regras.md) · [Índice do guia](../README.md) · [Tarefas — lista de conferência](conferencia.md) →
