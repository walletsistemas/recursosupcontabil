# E-mail — escrever e enviar

← [E-mail — segurança](seguranca.md) · [Índice do guia](../README.md) · [E-mail — avisos de e-mail novo e API](avisos-e-api.md) →

---

- Modos: novo · responder · responder a todos · encaminhar (leva os anexos originais) · continuar
  rascunho.
- Editor simples (`contentEditable`; negrito, itálico, sublinhado, listas). O HTML de fora é limpo com
  DOMPurify **ao entrar no editor** (o que é colado, a assinatura, o e-mail citado): saem `style`,
  scripts, formulários. O que a pessoa digitou vai como está.
- Para/Cc/Cco com sugestões: últimos 400 enviados + colegas do escritório + e-mails das empresas
  (cache de 10 min).
- Anexos do computador (botão ou arrastar), **limite total de 25 MB**, conferido antes de decodificar.
- Resposta leva `In-Reply-To` e `References` (o Outlook agrupa a conversa).
- Imagem `data:` (ex.: logo da assinatura) vira anexo interno `cid:` — o Gmail não mostra `data:`.
- **Cco** não vai no cabeçalho: entra só na lista de destinatários do SMTP.

📄 `backend/servicos/email_caixa.py` (encurtado)

```python
def _montar(d: dict, u: dict, c: dict, exigir_destino=True):
    para, cc, cco = _enderecos(d.get("para")), _enderecos(d.get("cc")), _enderecos(d.get("cco"))
    if exigir_destino and not (para or cc or cco):
        raise _NaoDa("Informe pelo menos um destinatário.")
    msg = EmailMessage()
    msg["From"] = formataddr((c.get("nome_exibicao") or u.get("nome") or "", c["email"]))
    if para: msg["To"] = ", ".join(formataddr(x) for x in para)
    if cc:   msg["Cc"] = ", ".join(formataddr(x) for x in cc)
    msg["Subject"] = (d.get("assunto") or "").replace("\r", " ").replace("\n", " ").strip()
    msg["Date"] = formatdate(localtime=True)
    msg["Message-ID"] = make_msgid(domain=c["email"].split("@")[-1])
    resposta = d.get("resposta") or {}
    if resposta.get("message_id"):
        mid = str(resposta["message_id"]).replace("\r", "").replace("\n", "")
        msg["In-Reply-To"] = mid
        msg["References"] = (str(resposta.get("references") or "") + " " + mid).strip()
    html = d.get("html") or ""
    msg.set_content(_texto_de_html(html) or " ")                       # versão texto
    msg.add_alternative(f'<!doctype html><html><head><meta charset="utf-8"></head><body>{html}</body></html>',
                        subtype="html")
    for a in d.get("anexos") or []:
        dados = base64.b64decode(str(a.get("dados") or "").split(",", 1)[-1])
        nome = os.path.basename(str(a.get("nome") or "anexo").replace("\\", "/")) or "anexo"
        principal, _, sub = mimetypes_tipo(nome).partition("/")
        msg.add_attachment(dados, maintype=principal, subtype=sub, filename=nome)
    return msg, [e for _n, e in para + cc + cco]                      # Cco só aqui
```

**Envio** — porta 465 usa SSL direto; outra porta usa STARTTLS:

```python
def _smtp(c: dict, msg, destinos) -> tuple[str, list]:
    """(motivo da falha — '' se saiu; quem foi recusado)."""
    try:
        porta = int(c.get("smtp_porta") or 465)
        if porta == 465:
            s = smtplib.SMTP_SSL(c["smtp_servidor"], porta, timeout=TEMPO, context=ssl.create_default_context())
        else:
            s = smtplib.SMTP(c["smtp_servidor"], porta, timeout=TEMPO)
            s.starttls(context=ssl.create_default_context())
        with s:
            s.login(c["email"], c["senha"])
            recusados = s.send_message(msg, from_addr=c["email"], to_addrs=destinos) or {}
    except smtplib.SMTPAuthenticationError:
        return "O servidor de saída recusou o e-mail ou a senha.", []
    except smtplib.SMTPRecipientsRefused as ex:
        return "O servidor recusou: " + ", ".join(ex.recipients), []
    except (smtplib.SMTPException, OSError) as ex:
        return f"Não consegui enviar ({ex}). Confira a internet e o servidor de saída.", []
    return "", list(recusados)
```

**Depois de sair, em segundo plano:** cópia em "Enviados" (`APPEND` com `\Seen`), marca a original
como respondida (`\Answered`) e apaga o rascunho. A tela diz "enviado" assim que o SMTP aceita —
subir a cópia com anexos demora. Se a cópia falhar, **avise a pessoa** (o e-mail saiu, mas não
aparece em Enviados).

```python
def enviar(d: dict) -> dict:
    u = _usuario()
    c = _conta_ou_erro(u)
    try:
        msg, destinos = _montar(d, u, c)
    except _NaoDa as ex:
        return {"ok": False, "mensagem": str(ex)}
    erro, recusados = _smtp(c, msg, destinos)
    if erro:
        return {"ok": False, "mensagem": erro}
    bruto = msg.as_bytes()
    _em_segundo_plano(lambda: _guardar_enviado(u["login"], bruto, d.get("resposta"),
                                               d.get("rascunho_id"), d.get("substitui")))
    return {"ok": True, "aviso": ("Saiu, mas o servidor não entregou para: " + ", ".join(recusados)) if recusados else ""}
```

**Rascunho salvo sozinho** no IMAP (pasta Rascunhos, aparece no Outlook também), 3 segundos depois
da última alteração e ao fechar a janela. Um cabeçalho próprio (`X-Plus-Rascunho: <id>`) faz a nova
versão substituir a anterior em vez de acumular cópias.

📄 `frontend/src/rotas/email/Escrever.tsx` (trecho)

```tsx
useEffect(() => {
  if (primeira.current) return void (primeira.current = false);
  rascunho.current.sujo = true;
  const t = window.setTimeout(() => void guardar(), 3000);   // 3 s parado → salva
  return () => window.clearTimeout(t);
}, [f, html, anexos, originais]);
// fechou a janela (X, Esc, clicou fora): guarda o que faltou
useEffect(() => () => void guardar(), []);
```

---

← [E-mail — segurança](seguranca.md) · [Índice do guia](../README.md) · [E-mail — avisos de e-mail novo e API](avisos-e-api.md) →
