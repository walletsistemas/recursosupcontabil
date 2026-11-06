# Tarefas — lista de conferência

← [Tarefas — API](api.md) · [Índice do guia](../README.md) · [Estudos](../03-estudos/README.md) →

---

## Lista de conferência — Tarefas

- [ ] Quem não participa não vê a tarefa nem pela API (`GET /api/tarefas/{id}` dá 403).
- [ ] Só responsável conclui e marca item; quem criou edita e apaga; responsável não apaga.
- [ ] Prazo e hora separados; filtro "Hoje" acha a tarefa com hora.
- [ ] Modo individual cria uma cópia por pessoa com o checklist inteiro.
- [ ] Último item marcado fecha a tarefa; desmarcar reabre; concluída à mão não reabre ao mexer num item.
- [ ] Item-grupo não tem caixinha e não conta no total.
- [ ] Editar não desfaz tique marcado por outra pessoa enquanto o formulário estava aberto.
- [ ] Concluída mostra "Entregue no prazo" / "Entregue com N dias de atraso", e o número não cresce.
- [ ] Comentário tem autor e data; só o autor (ou admin/sócio) apaga.
- [ ] Número do menu (só as minhas em aberto) atualiza depois de concluir/criar.

---

← [Tarefas — API](api.md) · [Índice do guia](../README.md) · [Estudos](../03-estudos/README.md) →
