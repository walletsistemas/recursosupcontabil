# Estudos — regras de negócio

← [Estudos — modelo de dados](dados.md) · [Índice do guia](../README.md) · [Estudos — API](api.md) →

---

## Regras de negócio

**Para quem vai um estudo da equipe** — 📄 `backend/motor/core/estudos.py`

```python
def _destinado_a(estudo, usuario) -> bool:
    """«Para todos» vale só para quem está ABAIXO de quem criou:
    o sócio cria → líder até estagiário; o líder cria → analista até estagiário.
    Quem está no mesmo nível ou acima só acompanha."""
    logins, cargos = estudo.get("para_logins") or [], estudo.get("para_cargos") or []
    if not logins and not cargos:
        return nivel_cargo(usuario.get("cargo")) < int(estudo.get("criador_nivel", 99))
    return usuario.get("login") in logins or (usuario.get("cargo") or "") in cargos

def destinatarios(estudo):
    """[(login, nome)] de quem recebe. Quem entrou na equipe DEPOIS de o estudo
    ser criado não recebe (o novato não herda centenas de aulas antigas)."""
    if estudo.get("pessoal"):
        return []
    gente = ...  # usuários ativos, perfil 'usuario'
    return [(u["login"], u["nome"]) for u in gente
            if u["login"] != estudo.get("criador") and _destinado_a(estudo, u)
            and (estudo.get("criado_em") or "") >= (u.get("criado_em") or "")]
```

**Quem altera:** quem criou; num estudo da equipe, também admin/sócio. Pessoal × equipe não muda
depois de criado. Arquivado não se edita (tira do arquivo antes).

**Regra para concluir** (escolhida por estudo):

| `relatorio` | Texto na tela | Conclui quando |
|---|---|---|
| `nao` | Sem relatório — conclui ao ver as aulas | viu todas as aulas |
| `anexar` | Relatório obrigatório — conclui ao entregar | viu todas **e** entregou o relatório (texto ou arquivo) |
| `aprovar` | Relatório com aprovação de quem criou | viu todas, entregou **e** quem criou aprovou (pode devolver com observação; a pessoa reenvia) |

**Situação de um estudo para uma pessoa** — 📄 `backend/motor/core/estudos.py`

```python
def situacao(estudo, and_pessoa, login):
    feitas, total = progresso(estudo, and_pessoa)
    reg = and_pessoa.get(estudo.get("uid")) or {}
    rel = reg.get("relatorio") or None
    regra = estudo.get("relatorio") or "nao"
    av = (estudo.get("avaliacoes") or {}).get(login)
    if av and rel and av.get("em", "") < rel.get("enviado_em", ""):
        av = None                                  # reenviou depois de devolvido
    if regra == "nao":
        st_rel = ""
    elif not rel:
        st_rel = "falta"
    elif regra == "anexar":
        st_rel = "entregue"
    else:
        st_rel = "aprovado" if av and av["aprovado"] else ("devolvido" if av else "aguardando")
    aulas_ok = total > 0 and feitas >= total
    concluido = aulas_ok and st_rel in ("", "entregue", "aprovado")
    prazos = [a.get("prazo") for a in estudo.get("aulas") or []
              if a.get("prazo") and a["id"] not in set(reg.get("vistas") or [])]
    return {"feitas": feitas, "total": total, "relatorio": st_rel, "rel": rel, "avaliacao": av,
            "concluido": concluido,
            "atrasado": atrasado(estudo.get("prazo"), concluido) or any(atrasado(p, False) for p in prazos)}
```

**Estado do cartão** (nesta ordem): arquivado → acompanhando (não é para mim) → concluído →
atrasado → em andamento (alguma aula vista, ou relatório aguardando/devolvido) → a começar.

**Arquivar sozinho** (`auto_arquivar`):

- roda quando **quem criou** abre a aba, e só para os estudos que essa pessoa criou (quem grava o
  arquivo do estudo é sempre o criador);
- estudo pessoal: arquivado quando ela concluiu; estudo da equipe: só quando **todos** os
  destinatários concluíram (com relatório aprovado, quando a regra pede);
- os N dias (configurável, padrão 7) contam a partir do **último movimento** (última marcação de
  alguém ou última avaliação), não da conclusão;
- se falhar, ignore (é só arrumação). Arquivar e tirar do arquivo também se faz à mão.

**Relatório:** só depois de concluir todas as aulas; relatório já aprovado não é reenviado.

**Anexos — seguro por padrão** — 📄 `backend/servicos/estudos.py` (encurtado)

```python
MAX_ANEXO = 25 * 1024 * 1024
# lista branca (no projeto fica em backend/janela/acoes.py e vale também para o e-mail)
DOCUMENTOS = {".pdf", ".xlsx", ".xls", ".csv", ".ods", ".docx", ".doc", ".odt", ".pptx", ".ppt",
              ".txt", ".ofx", ".xml", ".png", ".jpg", ".jpeg", ".gif", ".bmp", ".webp",
              ".mp4", ".mp3", ".zip", ".rar", ".7z"}

def _uid_valido(uid) -> str:
    uid = str(uid or "")
    if not re.fullmatch(r"[0-9a-f]{32}", uid):          # nada de "..\..\" no caminho
        raise SemPermissao("Identificação de estudo inválida.")
    return uid

def _gravar_anexo(uid, nome, dados_b64, prefixo=""):
    bruto = base64.b64decode(dados_b64.split(",", 1)[-1])
    if len(bruto) > MAX_ANEXO:
        raise ValueError("O arquivo passa de 25 MB.")
    pasta = E.pasta_anexos(os.path.basename(uid))
    os.makedirs(pasta, exist_ok=True)
    base, ext = os.path.splitext(prefixo + os.path.basename(nome or "arquivo"))
    if ext.lower() not in DOCUMENTOS:
        raise ValueError(f"Esse tipo de arquivo ({ext or 'sem extensão'}) não pode ser anexado.")
    final, n = base + ext, 2
    while os.path.exists(os.path.join(pasta, final)):    # nunca sobrescreve
        final, n = f"{base} ({n}){ext}", n + 1
    with open(os.path.join(pasta, final), "wb") as f:
        f.write(bruto)
    return final
```

O relatório de um colega (`relatorio - <login> - …`) só é visto por quem pode alterar o estudo (quem
criou, ou admin/sócio num estudo da equipe); compare sem diferenciar maiúsculas (o Windows não
diferencia).

**Visão da equipe:** quem criou, e admin/sócio/líder num estudo da equipe, vê por pessoa: aulas
feitas e estado do relatório. **Aprovar/devolver** é só de quem pode alterar o estudo (quem criou,
ou admin/sócio). ⚠️ No projeto de referência a tela mostra os botões também para o líder que não
criou o estudo, e o servidor recusa com 403; no seu projeto, esconda os botões nesse caso.

---

← [Estudos — modelo de dados](dados.md) · [Índice do guia](../README.md) · [Estudos — API](api.md) →
