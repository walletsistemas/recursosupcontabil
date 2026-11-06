# Padrões de código

← [Arquitetura](arquitetura.md) · [Índice do guia](../README.md) · [Perfis e permissões](permissoes.md) →

---

## Rota fina (API)

📄 `backend/api/tarefas.py` (trecho)

```python
from fastapi import APIRouter
from pydantic import BaseModel
from servicos import tarefas as T

rota = APIRouter(prefix="/api/tarefas", tags=["tarefas"])

class Item(BaseModel):
    texto: str = ""
    concluido: bool | None = None      # None = "mantenha o que já está no banco"
    empresa_id: int | None = None
    empresas: list[int] = []           # empresas penduradas neste item (item-grupo)

class Tarefa(BaseModel):
    titulo: str = ""
    descricao: str = ""
    prioridade: int = 1
    prazo: str = ""                    # "AAAA-MM-DD"
    prazo_hora: str = ""               # "HH:MM"
    responsaveis_ids: list[int] = []
    empresas_ids: list[int] = []
    modo_responsaveis: str = "compartilhada"   # ou "individual"
    itens: list[Item] = []

class Marca(BaseModel):
    concluido: bool

@rota.get("")
def listar(filtro: str = "aberto", busca: str = "", dia: str = "", so_minhas: bool = False):
    return T.listar(filtro, busca, dia, so_minhas)

@rota.post("/{tid}/concluir")
def concluir(tid: int, d: Marca):
    return T.concluir(tid, d.concluido)
```

## Permissão no serviço → 403 com frase em português

📄 `backend/servicos/empresas.py` e `backend/app.py`

```python
class SemPermissao(Exception):
    pass

# no app:
@app.exception_handler(SemPermissao)
async def sem_permissao(_request, ex):
    return JSONResponse({"detail": str(ex)}, status_code=403)
```

📄 `backend/servicos/tarefas.py` (trecho)

```python
def marcar_item(tid: int, item_id: int, concluido: bool) -> dict:
    u = _usuario()                       # quem está logado (ou SemPermissao "Entre no sistema de novo.")
    t = _tarefa(tid)                     # ou SemPermissao "Essa tarefa não existe mais..."
    if not (T._pode_editar(u) and T._sou_dono(u, t)):
        raise SemPermissao("Só quem é responsável marca o checklist.")
    ...                                  # confere também que o item é DESTA tarefa
    virou = T.marcar_item(item_id, concluido)
    return {"ok": True, "tarefa_concluida": None if virou is None else bool(virou)}
```

Convenção de respostas:

- erro de preenchimento → `200 {"ok": false, "erro": "Dê um título para a tarefa."}` (a tela mostra num aviso);
- "você não pode" → `403 {"detail": "Só quem é responsável marca o checklist."}`;
- deu certo → `{"ok": true, ...}`.

Exceção no módulo de e-mail: as rotas passam por um `_chamar` que transforma `SemPermissao` em 403 e
`ValueError` em **400** (ex.: "E-mail inválido"), e as respostas de erro usam a chave `mensagem`
em vez de `erro` (`{"ok": false, "mensagem": "Informe pelo menos um destinatário."}`).

## API só para este computador (senha interna)

📄 `backend/app.py` (encurtado). A API escuta só em `127.0.0.1`, numa porta sorteada, e exige uma
senha gerada a cada abertura (`secrets.token_urlsafe(24)`), que a janela recebe no endereço
(`#api=...&t=...`).

```python
@app.middleware("http")
async def exigir_token(request: Request, chamar):
    if (request.url.path.startswith("/api")
            and request.method != "OPTIONS"
            and not hmac.compare_digest(request.headers.get("x-token") or "", token)):
        return JSONResponse({"detail": "sem permissão"}, status_code=401)
    resposta = await chamar(request)
    if request.url.path.startswith("/api"):
        resposta.headers["Cache-Control"] = "no-store"   # dado nunca fica guardado no navegador
    return resposta

# só atende quem chama por 127.0.0.1/localhost
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["127.0.0.1", "localhost"])
```

O trecho acima deixa de fora três detalhes do original: `/api/docs` e `/api/openapi.json` não pedem
senha (e só existem em modo de desenvolvimento); a senha também é aceita no endereço (`?t=`), **só
em GET** e só em alguns caminhos (fotos, anexos de estudo), porque `<img src>` não manda cabeçalho;
e pedidos acima de 60 MB recebem 413.

Num sistema web comum, isto vira o login com sessão/JWT de sempre.

## Cliente da API na tela

📄 `frontend/src/api/cliente.ts` (trecho)

```ts
async function chamar<T>(metodo: string, caminho: string, corpo?: unknown): Promise<T> {
  let resposta: Response;
  try {
    resposta = await fetch(API + caminho, {
      method: metodo,
      cache: "no-store",
      headers: { "Content-Type": "application/json", "X-Token": TOKEN },
      body: corpo === undefined ? undefined : JSON.stringify(corpo),
    });
  } catch {
    throw new ErroDaApi("O sistema não respondeu. Feche e abra o programa de novo.");
  }
  if (!resposta.ok) {
    // o Python manda o motivo em «detail» (ex.: «Só o administrador apaga empresas.»)
    let motivo = `Erro ${resposta.status} ao falar com o sistema.`;
    try {
      const d = await resposta.json();
      if (typeof d?.detail === "string") motivo = d.detail;
    } catch { /* sem corpo */ }
    throw new ErroDaApi(motivo);
  }
  return resposta.json() as Promise<T>;
}

export const api = {
  get: <T>(c: string) => chamar<T>("GET", c),
  post: <T>(c: string, corpo?: unknown) => chamar<T>("POST", c, corpo ?? {}),
  put: <T>(c: string, corpo?: unknown) => chamar<T>("PUT", c, corpo ?? {}),
  del: <T>(c: string) => chamar<T>("DELETE", c),
};
```

## Buscar, gravar e recarregar (TanStack Query)

📄 `frontend/src/rotas/tarefas/TelaTarefas.tsx` (trecho)

```tsx
const consultas = useQueryClient();
const buscaAdiada = useDeferredValue(busca);          // não pesquisa a cada tecla

const q = useQuery({
  queryKey: ["tarefas", "lista", filtro, buscaAdiada, dia, soMinhas],
  queryFn: () => tarefasApi.listar(filtro, buscaAdiada, dia, soMinhas),
  placeholderData: keepPreviousData,                  // a lista não "pisca" ao trocar o filtro
});

const recarregar = () => {
  consultas.invalidateQueries({ queryKey: ["tarefas"] });
  consultas.invalidateQueries({ queryKey: ["seu-dia"] }); // o número do menu
};

async function concluir(t: Tarefa, v: boolean) {
  try {
    await tarefasApi.concluir(t.id, v);
    if (v) toast.success("Tarefa concluída.");
    recarregar();
  } catch (ex) {
    toast.error((ex as Error).message);               // a frase em português que veio do servidor
  }
}
```

Preferência de tela **por pessoa** (mais de uma pessoa usa o mesmo computador):

```ts
const daPessoa = (chave: string) => `${chave}:${usuarioAtual()?.id ?? 0}`;
localStorage.setItem(daPessoa("tarefas_filtro"), "atrasadas");
```

---

← [Arquitetura](arquitetura.md) · [Índice do guia](../README.md) · [Perfis e permissões](permissoes.md) →
