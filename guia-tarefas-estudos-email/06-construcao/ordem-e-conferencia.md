# Ordem de construção e lista de conferência

← [Testes](testes.md) · [Índice do guia](../README.md) · [Armadilhas conhecidas (resumo)](armadilhas.md) →

---

1. **Usuários e permissões** ([Perfis e permissões](../01-base/permissoes.md)) — tudo depende disso.
2. **Tarefas:** modelo → criar/listar/editar → concluir → checklist simples → checklist em dois
   níveis com fechar/reabrir → comentários → filtros e contagens → calendário → número no menu →
   aviso de tarefa nova.
3. **Estudos:** estudo pessoal com aulas → andamento por pessoa → estudo da equipe com
   destinatários → anexos → relatório (`anexar`, depois `aprovar`) → visão da equipe → arquivar.
4. **E-mail:** conectar/testar com senha protegida → pastas → lista paginada → leitura segura →
   anexos → marcar/mover/excluir → escrever/enviar + cópia em Enviados → rascunhos → busca → número
   no menu e pop-up.
5. **Lembretes por e-mail** (precisa de Tarefas e de uma conta de envio).

Antes de dar por pronto, em cada módulo:

- [ ] Toda regra de permissão está no servidor e tem teste.
- [ ] Toda mensagem de erro está em português e diz o que fazer.
- [ ] Nenhum `id` local foi gravado no arquivo/tabela compartilhada no lugar de `uid`/login/CNPJ.
- [ ] Gravações são atômicas; ler mal não vira "apagar tudo".
- [ ] Tema claro e escuro conferidos.
- [ ] Testado numa base de teste, nunca na de produção.
- [ ] Nenhuma senha, token ou dado de cliente no git.

---

← [Testes](testes.md) · [Índice do guia](../README.md) · [Armadilhas conhecidas (resumo)](armadilhas.md) →
