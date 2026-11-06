# Estudos — modelo de dados

← [Estudos](README.md) · [Índice do guia](../README.md) · [Estudos — regras de negócio](regras.md) →

---

## Modelo de dados — um dono por arquivo

```
estudos/<uid>.json               o estudo           → só quem criou grava
estudos/_andamento/<login>.json  o que a pessoa fez → só ela grava
estudos/_anexos/<uid>/<arquivo>  arquivos das aulas e dos relatórios
estudos/_arquivo/<uid>.json      arquivados (lidos só quando alguém pede)
```

Não há mesclagem nem conflito. A **entrega** do relatório fica no andamento da pessoa; a
**avaliação** fica no arquivo do estudo (quem grava é quem criou).

```json
// estudos/<uid>.json
{
  "uid": "6e8252e6fd7c45f2a66a304555c659a0",
  "titulo": "Reforma tributária: IBS e CBS",
  "descricao": "Trilha para a equipe entender a transição.",
  "criador": "maria", "criador_nivel": 6, "criado_em": "2026-11-06 09:00:00",
  "pessoal": false, "para_logins": [], "para_cargos": [],
  "prazo": "2026-11-16", "relatorio": "aprovar",
  "aulas": [
    {"id": "a1", "titulo": "O que muda e quando", "prazo": "", "links": ["https://www.gov.br/fazenda"],
     "resumo": "Linha do tempo.", "anexos": ["apostila.pdf"],
     "itens": [{"id": "i1", "texto": "Assistir ao vídeo"}, {"id": "i2", "texto": "Ler o resumo"}]}
  ],
  "avaliacoes": {"joao": {"aprovado": false, "obs": "Faltou o exemplo.", "por": "maria", "em": "2026-11-10 15:00:00"}},
  "tipo": "trilha",
  "atualizado_em": "2026-11-06 09:00:00"
}
// "tipo" é calculado ao salvar ("trilha" com mais de uma aula, senão "aula").
// "arquivado_em" só aparece quando o estudo vai para o arquivo.
// Aulas e itens ganham "id" = uuid4().hex[:10] quando chegam sem.

// estudos/_andamento/joao.json
{
  "6e8252e6fd7c45f2a66a304555c659a0": {
    "vistas": ["a1"], "itens": ["i1"],
    "relatorio": {"texto": "Resumo do que aprendi…", "anexo": "relatorio - joao - resumo.pdf", "enviado_em": "2026-11-09 18:00:00"},
    "atualizado_em": "2026-11-09 18:00:00"
  }
}
```

Marcar aula (📄 `backend/motor/core/estudos.py`):

```python
def marcar_aula(login, estudo_uid, aula_id, vista):
    d = andamento(login)                        # lê estudos/_andamento/<login>.json
    reg = d.setdefault(estudo_uid, {})
    vistas = set(reg.get("vistas") or [])
    (vistas.add if vista else vistas.discard)(aula_id)
    reg["vistas"] = sorted(vistas)
    reg["atualizado_em"] = _agora()
    R.gravar_json(f"{PASTA}/_andamento/{login}.json", d)
    return d
```

---

← [Estudos](README.md) · [Índice do guia](../README.md) · [Estudos — regras de negócio](regras.md) →
