# E-mail — pastas, lista e leitura

← [E-mail — conta, senha e conexões](conta-e-conexoes.md) · [Índice do guia](../README.md) · [E-mail — segurança](seguranca.md) →

---

## Pastas

- Reconhecidas pelas marcas especiais do IMAP (`\Sent`, `\Drafts`, `\Trash`, `\Junk`, `\Archive`)
  e, na falta delas, pelo nome ("Sent", "Itens enviados", "Lixo eletrônico"…).
- Ordem fixa: Entrada, Enviados, Rascunhos, Arquivo, Spam, Lixeira; depois as da pessoa.
- Nomes vêm em **UTF-7 modificado** (RFC 3501): converta para mostrar e para gravar.
- A tela pede a lista de pastas a cada 120 s e o servidor sempre pergunta de novo ao IMAP. Só as
  ações internas (mover, excluir, enviar, rascunho) usam uma cópia guardada por 2 minutos;
  criar/renomear/apagar pasta a renovam. As pastas especiais não se apagam.

📄 `backend/servicos/email_caixa.py`

```python
def _de_utf7(nome: str) -> str:
    """«Itens Exclu&AO0-dos» → «Itens Excluídos»."""
    def trocar(m):
        t = m.group(1)
        if not t:
            return "&"
        t = t.replace(",", "/")
        return base64.b64decode(t + "=" * (-len(t) % 4)).decode("utf-16-be")
    return re.sub(r"&([^-]*)-", trocar, nome)

_ESPECIAIS = {"\\inbox": "entrada", "\\sent": "enviados", "\\drafts": "rascunhos",
              "\\trash": "lixeira", "\\junk": "spam", "\\archive": "arquivo"}
_POR_NOME = {"inbox": "entrada", "sent": "enviados", "itens enviados": "enviados", "sent items": "enviados",
             "drafts": "rascunhos", "trash": "lixeira", "itens excluídos": "lixeira", "deleted items": "lixeira",
             "spam": "spam", "junk": "spam", "lixo eletrônico": "spam"}   # (lista encurtada)
```

## Lista e leitura

- Lista **por UID**, 50 por página; "Carregar mais antigas" pede `antes_de=<menor UID visto>`.
  Busca só cabeçalhos, flags e tamanho.
- Filtros IMAP:

  ```python
  FILTROS = {"todas": [], "nao_lidas": ["UNSEEN"], "sinalizadas": ["FLAGGED"],
             "anexo": ["HEADER", "Content-Type", "multipart/mixed"]}
  ```

- Leitura com `BODY.PEEK[]` (não marca como lida sozinho) e depois `STORE +FLAGS (\Seen)` de
  propósito. As 3 últimas mensagens ficam em memória para reabrir rápido.

📄 `backend/servicos/email_caixa.py` (encurtado)

```python
def ler(pasta: str, uid: int) -> dict:
    u = _usuario()
    with _trava:
        msg = _buscar(_conexao(u["login"]), pasta, uid, marcar_lida=True)
    corpo = msg.get_body(preferencelist=("html", "plain"))
    html, texto = "", ""
    if corpo is not None:
        if corpo.get_content_subtype() == "html":
            html = corpo.get_content()
        else:
            texto = corpo.get_content()
    # imagens coladas no corpo (cid:) viram data: para aparecerem
    if html:
        for parte in msg.walk():
            cid = str(parte.get("content-id") or "").strip("<>")
            if cid and parte.get_content_maintype() == "image":
                dado = base64.b64encode(parte.get_payload(decode=True) or b"").decode()
                html = html.replace(f"cid:{cid}", f"data:{parte.get_content_type()};base64,{dado}")
    anexos = [{"indice": i, "nome": a.get_filename() or f"anexo-{i + 1}", "tipo": a.get_content_type(),
               "tamanho": len(a.get_payload(decode=True) or b"")}
              for i, a in enumerate(msg.iter_attachments())]
    return {"uid": uid, "assunto": _texto(msg.get("subject")) or "(sem assunto)",
            "de": _pessoas(msg.get("from")), "para": _pessoas(msg.get("to")), "cc": _pessoas(msg.get("cc")),
            "data": _data(msg.get("date")), "html": html, "texto": texto, "anexos": anexos,
            "responder_para": _pessoas(msg.get("reply-to")) or _pessoas(msg.get("from")),
            "message_id": _texto(msg.get("message-id")), "references": _texto(msg.get("references")),
            "rascunho_id": _texto(msg.get("x-plus-rascunho"))}
```

- **Conversa:** junta as mensagens pelo assunto sem `RE:/RES:/ENC:/FW:/FWD:/TR:`.
- **Ações em lote:** marcar lida/não lida, sinalizar, mover (arrastando), excluir. UIDs agrupados em
  faixas (`1,2,3,7,8` → `1:3,7:8`) para "selecionar a pasta inteira" não estourar o comando.
- **Excluir** vai para a Lixeira; na Lixeira apaga de vez (a tela pergunta antes).
- **Mover sem o comando MOVE** = `COPY` + `\Deleted` + apagar **só estas** (com `UIDPLUS`,
  `UID EXPUNGE`). O `EXPUNGE` comum apagaria também o que o Outlook marcou e ainda não limpou.

---

← [E-mail — conta, senha e conexões](conta-e-conexoes.md) · [Índice do guia](../README.md) · [E-mail — segurança](seguranca.md) →
