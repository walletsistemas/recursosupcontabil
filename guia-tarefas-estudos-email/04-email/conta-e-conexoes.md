# E-mail — conta, senha e conexões

← [E-mail (caixa dentro do sistema)](README.md) · [Índice do guia](../README.md) · [E-mail — pastas, lista e leitura](pastas-e-leitura.md) →

---

## Conta e senha protegida

- **Cada pessoa conecta a SUA caixa**, uma vez. A conexão é testada antes de salvar; senha errada
  não é guardada.
- **A senha fica só neste computador, cifrada pelo Windows (DPAPI)**, ligada ao usuário do Windows.
  Nunca vai para a base compartilhada, o git ou o código. Trocou de computador → conecta de novo.

📄 `backend/servicos/email_caixa.py`

```python
def _dpapi(dados: bytes, cifrar: bool) -> bytes:
    import ctypes
    from ctypes import wintypes

    class Blob(ctypes.Structure):
        _fields_ = [("tam", wintypes.DWORD), ("dados", ctypes.POINTER(ctypes.c_char))]

    buf = ctypes.create_string_buffer(dados, len(dados))
    entrada = Blob(len(dados), ctypes.cast(buf, ctypes.POINTER(ctypes.c_char)))
    saida = Blob()
    c = ctypes.windll.crypt32
    f = c.CryptProtectData if cifrar else c.CryptUnprotectData
    if not f(ctypes.byref(entrada), None, None, None, None, 0x1, ctypes.byref(saida)):  # 0x1: sem janela
        raise OSError("O Windows não conseguiu proteger/ler a senha do e-mail.")
    try:
        return ctypes.string_at(saida.dados, saida.tam)
    finally:
        ctypes.windll.kernel32.LocalFree(saida.dados)
```

Fora do Windows: `keyring` (Python), Keychain (macOS), Secret Service (Linux). Num sistema web, um
cofre de segredos no servidor, com a senha cifrada por usuário.

Perfil: nome de exibição e assinatura HTML (limite de 2 MB); dá para importar as assinaturas do
Outlook de `%APPDATA%\Microsoft\Signatures` (com as imagens).

## Conexões IMAP

Regras que evitam travar a tela:

1. **Uma conexão aberta por pessoa, reaproveitada.** Abrir IMAP a cada clique é lento.
2. Antes de usar, `NOOP` com **timeout curto (5 s)**: o provedor derruba conexões paradas, e esperar
   o timeout normal (25 s) travaria a tela. Caiu → abre outra sozinho.
3. **Conexão e trava separadas para o trabalho de fundo** (número de não lidas, aviso de e-mail
   novo). Senão apagar/enviar esperava na fila atrás deles.
4. **Conexão avulsa para buscas demoradas** (no texto, em todas as pastas), com timeout de 180 s: o
   servidor leva cerca de 1 min para ler 3 mil e-mails, e o resto da caixa continua funcionando.

📄 `backend/servicos/email_caixa.py` (encurtado)

```python
TEMPO, TESTE_VIVA, FUNDO = 25, 5, "|fundo"

def _abrir(c: dict, tempo=TEMPO):
    m = imaplib.IMAP4_SSL(c["imap_servidor"], int(c["imap_porta"]), timeout=tempo)
    m.login(c["email"], c["senha"])
    return m

def _conexao(login: str, fundo=False):
    chave = login + FUNDO if fundo else login
    m = _conexoes.get(chave)
    if m is not None:
        sock = getattr(m, "sock", None)
        try:
            if sock: sock.settimeout(TESTE_VIVA)
            m.noop()                                  # ainda viva?
            if sock: sock.settimeout(TEMPO)
            return m
        except Exception:
            _conexoes.pop(chave, None)                # caiu: descarta e abre outra
    c = _ler_conta(login)
    if not c:
        raise SemPermissao("Conecte a sua caixa de e-mail primeiro.")
    try:
        m = _abrir(c)
    except imaplib.IMAP4.error:
        raise SemPermissao("O servidor recusou o e-mail ou a senha. Confira em «Conta».")
    except OSError:
        raise SemPermissao("Não consegui falar com o servidor de e-mail. Confira a internet.")
    _conexoes[chave] = m
    return m

def _q(pasta: str) -> str:
    """Nome de pasta entre aspas para o IMAP — sem quebra de linha: nada de comando escondido."""
    if not pasta or "\r" in pasta or "\n" in pasta:
        raise ValueError("Pasta inválida.")
    return '"' + pasta.replace("\\", "\\\\").replace('"', '\\"') + '"'
```

---

← [E-mail (caixa dentro do sistema)](README.md) · [Índice do guia](../README.md) · [E-mail — pastas, lista e leitura](pastas-e-leitura.md) →
