# Testes

← [E-mails automáticos (lembretes)](../05-lembretes/README.md) · [Índice do guia](../README.md) · [Ordem de construção e lista de conferência](ordem-e-conferencia.md) →

---

Referência: `tests/api/test_tarefas.py` (6 testes), `test_estudos.py` (4), `test_email.py` (30),
com pytest e o `TestClient` do FastAPI, numa **base nova e temporária** (nunca na base real).

## Base temporária e troca de usuário

📄 `tests/api/test_tarefas.py` (trecho)

```python
H = {"x-token": "t"}

@pytest.fixture(scope="module")
def c():
    base = tempfile.mkdtemp(prefix="plus_tarefas_")
    B.autorizar_base_nova(base)
    B.definir_servidor_path(base)
    ...
    cli = TestClient(app.criar_api("t"))
    cli.post("/api/sessao/primeiro-usuario", headers=H,
             json={"nome": "Chefe Teste", "login": "chefe", "senha": "segredo1"})
    # cria Ana, Bia e Caio como usuários comuns
    return cli

def _entrar(c, login):
    S._logado["usuario"] = None
    S._zerar_freio()          # senão os logins repetidos batem no freio de tentativas
    assert c.post("/api/sessao/entrar", headers=H, json={"login": login, "senha": "segredo1"}).json()["ok"]
```

## Servidor de e-mail falso (sem rede, sem conta)

📄 `tests/api/test_email.py` (encurtado)

```python
class Falso:
    """O mínimo de um servidor IMAP que o sistema usa."""
    capabilities = ("IMAP4REV1", "MOVE")

    def __init__(self):
        self.caixas = {"INBOX": {}, "INBOX.Lixeira": {}, "INBOX.Enviados": {}, "INBOX.Rascunhos": {},
                       "INBOX.Itens Exclu&AO0-dos.Antigos": {}}          # testa o UTF-7 também
        # mensagens de exemplo, inclusive uma com <script> e outra com virus.exe
        ...

    def noop(self): return "OK", [b""]
    def logout(self): return "BYE", [b""]
    def append(self, q, flags, _data, bruto): ...
    def list(self, *a): ...
    # select, uid(SEARCH/FETCH/STORE/MOVE/COPY/EXPUNGE), status, create, rename, delete…

@pytest.fixture
def servidor(monkeypatch):
    f = Falso()
    def abrir(conta, tempo=None):
        if conta["senha"] != "certa":
            raise imaplib.IMAP4.error("AUTHENTICATIONFAILED")
        return f
    monkeypatch.setattr(EM, "_abrir", abrir)                       # troca a conexão de verdade
    monkeypatch.setattr(EM, "_em_segundo_plano", lambda f: f())    # cópia em Enviados na hora
    EM._conexoes.clear()
    return f
```

O SMTP é trocado do mesmo jeito (`monkeypatch.setattr(smtplib, "SMTP_SSL", SmtpFalso)`).

## O que vale testar

| Módulo | Testes |
|---|---|
| Tarefas | cada linha da tabela de permissões ([Tarefas — regras de negócio](../02-tarefas/regras.md)); checklist fecha e reabre (incluindo "concluí à mão e desmarquei um item" → não reabre); modo individual; "editar não desfaz tique"; prazo com hora no filtro "Hoje" |
| Estudos | destinatários por nível, por cargo e por login; novato não recebe estudo antigo; situação em cada regra de relatório; anexo com extensão proibida, nome com `..\`, arquivo repetido |
| E-mail | senha errada não guarda; senha cifrada no PC; pastas e não lidas; ler marca como lida; `.exe` não abre direto; excluir → Lixeira → apaga; pasta com quebra de linha recusada; enviar com anexo + cópia em Enviados; responder liga a conversa; encaminhar leva anexos; sem MOVE não apaga o que o Outlook marcou; rascunho troca e some ao enviar; conexão caída é trocada |
| Lembretes | um e-mail por pessoa por dia; tarefa comunicada uma vez; base de teste não envia |

O teste com uma conta de e-mail de verdade é feito por uma pessoa, na tela, digitando a própria senha.

---

← [E-mails automáticos (lembretes)](../05-lembretes/README.md) · [Índice do guia](../README.md) · [Ordem de construção e lista de conferência](ordem-e-conferencia.md) →
