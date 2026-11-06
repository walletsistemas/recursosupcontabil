# E-mails automáticos (lembretes)

← [E-mail — lista de conferência](../04-email/conferencia.md) · [Índice do guia](../README.md) · [Testes](../06-construcao/testes.md) →

---

Referência: `backend/motor/core/avisos.py`, `core/email_rotina.py`.

Os lembretes saem por uma conta de envio própria do sistema (separada das caixas pessoais),
configurada pelo administrador.

**Aviso na hora — tarefa nova:** quando alguém cria/repassa uma tarefa para você. Sai em segundo
plano, sem segurar a tela; ao fechar o programa, espera alguns segundos os envios pendentes.

**E-mail do dia — um só por pessoa**, com até três blocos (oito atrasadas = um e-mail, não oito):

1. **Atrasadas** — o prazo passou;
2. **Prazo chegando** — vence hoje ou nos próximos N dias úteis (sábado e domingo não contam;
   feriados não são considerados);
3. **Outras em aberto** — só as de que a pessoa **nunca foi avisada**. É este bloco, junto com o
   registro permanente, que põe o escritório em dia na primeira vez que os lembretes são ligados.

**Quem dispara, sem servidor:** o primeiro computador que abrir o programa no dia ("carteiro"). Se
ninguém abrir, nada sai; para horário garantido, use um agendador (ex.: Agendador de Tarefas do
Windows, cron).

**Duas travas contra repetição** — ✂️ simplificado de `backend/motor/core/avisos.py`

```python
def disparar_lembretes(hoje=None) -> dict:
    """Nunca levanta erro: isto roda ao abrir o programa e não pode atrapalhar quem quer trabalhar.
    · `dia|<e-mail>`   no arquivo do DIA   → no máximo um e-mail por pessoa por dia;
    · `<uid>|<e-mail>` no registro PERMANENTE → cada tarefa comunicada uma vez só."""
    resumo = {"enviados": 0, "pulados": 0, "falhas": 0, "postas_em_dia": 0}
    try:
        hj = hoje or date.today().isoformat()
        ja_avisadas = repositorio.tarefas_avisadas()
        for p in pessoas_ativas_com_email():
            trava_dia = f"dia|{p['email'].lower()}"
            if repositorio.aviso_ja_enviado(hj, trava_dia):
                resumo["pulados"] += 1
                continue
            tarefas = _tarefas_da_pessoa(conn, p["id"])
            novas = [t["uid"] for t in tarefas if chave_tarefa(t["uid"], p["email"]) not in ja_avisadas]
            monte = montar_lembrete(p["nome"], tarefas, hj, uids_novas=novas)
            if monte is None:
                continue                                  # nada a avisar: não marca, não envia
            assunto, texto, html = monte
            # a marca do dia é gravada DEPOIS de o e-mail sair: marcada antes, um programa
            # fechado no meio do envio deixava a marca sem e-mail, e ninguém recebia o lembrete
            if enviar(p["email"], assunto, texto, html):
                repositorio.marcar_aviso(hj, trava_dia, p["nome"])
                repositorio.marcar_tarefas_avisadas([chave_tarefa(t["uid"], p["email"]) for t in tarefas], p["nome"])
                resumo["enviados"] += 1
            else:
                resumo["falhas"] += 1
        repositorio.podar_tarefas_avisadas(uids_que_ainda_existem)   # o registro não cresce para sempre
    except Exception as ex:
        log.error("falha ao disparar os lembretes: %s", ex)
    return resumo
```

- E-mail em HTML **e** texto puro (`multipart/alternative`), com nome do escritório e rodapé
  configuráveis.
- **Pasta na nuvem** (Google Drive etc.) demora alguns segundos para mostrar a marca aos outros
  computadores: se duas pessoas abrirem no mesmo minuto, pode sair um e-mail repetido. É o preço de
  não haver servidor.

**E-mail ao cliente da rotina mensal** — diferente dos lembretes, sai pela **caixa pessoal de quem
está logado** (a mesma da [caixa de e-mail](../04-email/README.md)), com a assinatura dela, cópia em Enviados e respostas para o
endereço escolhido. **Modelos com campos que se preenchem:**
`{empresa} {cnpj} {competencia} {documentos} {responsavel} {setor} {escritorio}` — modo manual (a
pessoa revisa e envia) ou automático a partir do N-ésimo dia útil do mês seguinte.

**Base de teste não manda e-mail automático:**

```python
if os.environ.get("SISTEMA_CONTABIL_SEM_EMAIL") == "1":
    if os.environ.get("SISTEMA_CONTABIL_EMAIL_SIMULADO") == "1":
        return _guardar_ensaio(para, assunto, texto, cc)   # vira um .eml na pasta de teste
    log.info("e-mail NÃO enviado (base de teste) para %s: %s", para, assunto)
    return False
```

O que a pessoa escreve à mão na caixa de e-mail sai mesmo na base de teste. Nos testes automáticos,
o enviador é trocado por uma função falsa (`definir_enviador`).

---

← [E-mail — lista de conferência](../04-email/conferencia.md) · [Índice do guia](../README.md) · [Testes](../06-construcao/testes.md) →
