# Tarefas — regras de negócio

← [Tarefas — modelo de dados](dados.md) · [Índice do guia](../README.md) · [Tarefas — API](api.md) →

---

## Regras de negócio

**Quem vê:** só os envolvidos (responsáveis e quem criou). Admin/sócio veem todas.

📄 `backend/motor/core/tarefas.py`

```python
def _filtro_visibilidade(usuario):
    """(trecho_sql, parametros) — quais tarefas este usuário enxerga."""
    if acesso_total(usuario):
        return "1=1", []
    uid = usuario.get("id")
    return f"({_SQL_SOU_RESP} OR t.criador_id = ?)", [uid, uid, uid]

def _sou_dono(usuario, t):
    """Só quem é RESPONSÁVEL altera/conclui."""
    return usuario.get("id") in responsaveis_ids(t)

def _posso_mexer(usuario, t):
    """Quem pode alterar a tarefa: os responsáveis, quem criou — e o admin."""
    if acesso_total(usuario):
        return True
    return _sou_dono(usuario, t) or t.get("criador_id") == usuario.get("id")

def pode_excluir(usuario, t):
    """A tarefa inteira só é apagada por quem a criou ou pelo admin/sócio.
    O responsável altera e conclui, mas não some com a tarefa de quem a passou para ele."""
    if not usuario or not t:
        return False
    if acesso_total(usuario):
        return True
    return t.get("criador_id") is not None and t.get("criador_id") == usuario.get("id")
```

| Ação | Quem pode |
|---|---|
| Ver | responsáveis, quem criou, admin/sócio |
| Criar | qualquer pessoa logada |
| Editar textos, empresas, checklist | responsáveis, quem criou, admin/sócio |
| Mudar o prazo ao editar | só admin/sócio (os demais mantêm o prazo original) |
| Concluir/reabrir, marcar item | **só os responsáveis** |
| Comentar | quem pode editar |
| Apagar comentário | quem escreveu, ou admin/sócio |
| Excluir a tarefa (com checklist e comentários) | quem criou, ou admin/sócio |

Cada cartão já vem do servidor com `pode_editar`, `pode_concluir`, `pode_excluir`, `pode_comentar`;
a tela só obedece (e mostra o motivo quando a pessoa clica num botão apagado).

**Vários responsáveis — dois modos na criação:**

📄 `backend/motor/core/tarefas.py` (encurtado)

```python
def criar_tarefa(dados, itens, criador_id=None, usuario=None):
    """"compartilhada" (padrão) → uma tarefa só, com todos os responsáveis;
       "individual"             → uma cópia por pessoa, cada uma com o seu checklist e a sua conclusão."""
    resp = [u for u in (dados.get("responsaveis_ids") or []) if u]
    if dados.get("modo_responsaveis") != "individual" or len(resp) <= 1:
        return [salvar_tarefa(dados, itens, None, criador_id, usuario)]
    criados = []
    for uid in resp:
        copia = dict(dados, responsaveis_ids=[uid], responsavel_id=uid)
        copia_itens = [dict(it, concluido=0) for it in itens]   # o checklist INTEIRO, desmarcado
        criados.append(salvar_tarefa(copia, copia_itens, None, criador_id, usuario))
    return criados
```

**Checklist em dois níveis:**

| Tipo de linha | Como reconhecer | Tem caixinha? |
|---|---|---|
| Item comum | sem empresa, sem filhos | sim |
| Empresa | com `empresa` (pendurada num item ou solta) | sim |
| Item-grupo | sem empresa, **com** filhos | não — mostra "0/2" e fecha sozinho |

O item-grupo **não entra** na conta que fecha a tarefa (senão ela nunca fecharia).

**Fecha e reabre sozinha** — 📄 `backend/motor/core/tarefas.py`

```python
def _ajustar_conclusao(conn, tid, n_antes, f_antes, n_agora, f_agora, concluida):
    """· fecha quando o checklist chega ao fim e a tarefa ainda estava aberta;
       · reabre quando o checklist ESTAVA completo, a tarefa estava concluída,
         e alguém desmarca um item.
    O "reabre" olha o estado ANTERIOR de propósito: quem concluiu à mão, sem
    marcar item por item, não pode ter a tarefa reaberta a cada clique.
    Devolve o novo estado (0/1) ou None quando nada mudou."""
    fechar = (not concluida) and n_agora > 0 and f_agora >= n_agora
    reabrir = bool(concluida) and n_antes > 0 and f_antes >= n_antes and f_agora < n_agora
    if not (fechar or reabrir):
        return None
    novo = 1 if fechar else 0
    conn.execute("UPDATE tarefas SET concluida=?, "
                 "concluida_em=CASE WHEN ?=1 THEN datetime('now','localtime') END WHERE id=?",
                 (novo, novo, tid))
    return novo

def marcar_item(item_id, concluido):
    with _lock, conectar() as conn:
        tid = conn.execute("SELECT tarefa_id FROM tarefa_itens WHERE id=?", (item_id,)).fetchone()["tarefa_id"]
        n_antes, f_antes, concluida = _contagem_checklist(conn, tid)   # o estado de ANTES é LIDO
        conn.execute("UPDATE tarefa_itens SET concluido=? WHERE id=?", (1 if concluido else 0, item_id))
        n, f, _ = _contagem_checklist(conn, tid)
        virou = _ajustar_conclusao(conn, tid, n_antes, f_antes, n, f, concluida)
        conn.commit()
    return virou      # a tela usa isso para redesenhar o cartão
```

Outras regras do checklist:

- **Editar não mexe nos tiques:** o que está marcado no banco fica (alguém pode ter marcado enquanto o
  formulário estava aberto). Marcar é só pelo cartão.
- Empresa tirada da tarefa leva junto a linha dela no checklist.
- Pessoa ou empresa inativa continua aparecendo nas opções quando já está na tarefa (marcada como
  inativa) — senão some da tarefa ao salvar.

**Texto e cor do prazo** — 📄 `backend/motor/core/tarefas.py` (encurtado)

```python
def rotulo_prazo(iso, concluida=False, concluida_em=None, hora=""):
    """Concluída: a conta é contra o DIA DA ENTREGA, nunca contra hoje
    (senão uma tarefa entregue no dia certo diria «Atrasada 12 dias» semanas depois).
    A hora entra no TEXTO, nunca na conta de atraso."""
    d = _para_data(iso)
    if not d:
        return "", "sem"
    if concluida:
        entregue = _para_data(str(concluida_em or "")[:10])
        if not entregue:
            return "Entregue", "entregue"
        atraso = (entregue - d).days
        if atraso <= 0:
            return "Entregue no prazo", "entregue"
        return f"Entregue com {atraso} dia{'s' if atraso > 1 else ''} de atraso", "entregue_atrasada"
    dias = (d - date.today()).days
    sufixo = f" às {hora}" if hora else ""
    if dias < 0:
        return f"Atrasada {-dias} dia{'s' if -dias > 1 else ''}", "atrasada"
    if dias == 0:
        if hora and _hora_ja_passou(hora):
            return f"Venceu hoje às {hora}", "hoje"
        return "Vence hoje" + sufixo, "hoje"
    if dias == 1:
        return "Vence amanhã" + sufixo, "amanha"
    if dias <= 7:
        return f"Em {dias} dias" + sufixo, "futuro"
    return d.strftime("%d/%m/%Y") + sufixo, "futuro"
```

Ordem da lista: abertas antes das concluídas → prioridade (3 → 0) → com prazo antes de sem prazo →
prazo mais próximo → mais antiga primeiro.

---

← [Tarefas — modelo de dados](dados.md) · [Índice do guia](../README.md) · [Tarefas — API](api.md) →
