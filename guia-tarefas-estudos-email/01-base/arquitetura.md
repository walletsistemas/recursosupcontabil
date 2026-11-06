# Arquitetura

← [Stack, bibliotecas e frameworks](stack.md) · [Índice do guia](../README.md) · [Padrões de código](padroes-de-codigo.md) →

---

## Camadas

```
tela (React)  →  API (FastAPI, só repassa)  →  serviço (regras + permissão)  →  motor (dados)
```

| Camada | Arquivos de referência | O que faz |
|---|---|---|
| Motor | `backend/motor/core/tarefas.py`, `core/estudos.py`, `core/avisos.py` | Lê/grava os dados, regras puras. |
| Serviço | `backend/servicos/tarefas.py`, `estudos.py`, `email_caixa.py` | Junta o que a tela precisa e **confere quem pode o quê**. |
| API | `backend/api/tarefas.py`, `estudos.py`, `email.py` | Rotas finas com Pydantic; nenhuma regra. |
| Tela | `frontend/src/rotas/tarefas/`, `estudos/`, `email/` | Uma pasta por módulo. |
| Cliente | `frontend/src/api/cliente.ts`, `api/tarefas.ts`… | Um arquivo por assunto, com os tipos. |

## Onde os dados moram

Pasta compartilhada, em JSON, com um SQLite local descartável (`cache\sessao.db`) só para as telas
serem rápidas:

```
<base>\
├── usuarios\<login>.json
├── tarefas\<uid>.json              uma tarefa por arquivo (checklist e comentários dentro)
├── estudos\<uid>.json              o estudo (só quem criou grava)
├── estudos\_andamento\<login>.json  o que ESTA pessoa marcou (só ela grava)
├── estudos\_anexos\<uid>\...       arquivos das aulas e dos relatórios
├── estudos\_arquivo\<uid>.json     estudos arquivados
├── _avisos\<dia>.<computador>.json     marca de "lembrete já enviado hoje" (um arquivo POR
│                                       computador; a leitura junta todos)
└── _avisos\_tarefas.<computador>.json  registro permanente das tarefas já avisadas
```

O e-mail **não** fica na base: as mensagens ficam no servidor de e-mail.

## Regras de ouro

- **Permissão conferida no servidor, sempre.** A tela só esconde botões.
- **Identidade estável entre computadores:** tarefa e comentário por `uid` (`str(uuid4())`, 36
  caracteres com hífens), estudo por `uid` (`uuid4().hex`, 32 caracteres), pessoa por `login`,
  empresa por CNPJ. Nunca grave no arquivo compartilhado o `id`
  numérico de um banco local: ele muda de máquina para máquina.
- **Gravação atômica:** escreve num temporário (com o nome do computador e do processo) e só então
  troca pelo definitivo. Se a rede cair no meio, o arquivo antigo continua inteiro.

  ✂️ versão mínima:

  ```python
  import json, os, socket

  def gravar_json(caminho: str, dados) -> None:
      tmp = f"{caminho}.{socket.gethostname()}.{os.getpid()}.tmp"
      with open(tmp, "w", encoding="utf-8") as f:
          json.dump(dados, f, ensure_ascii=False, indent=1)
          f.flush()
          os.fsync(f.fileno())
      os.replace(tmp, caminho)          # troca atômica
  ```

- **Separe "não existe" de "não consegui ler".** Arquivo corrompido ou meio gravado tratado como
  vazio já apagou trabalho de colega. Na dúvida, não sobrescreva.
- **Um dono por arquivo** sempre que der (ver Estudos e os avisos, cada computador com o seu): sem
  conflito de gravação, sem mesclar. Quando todos gravavam o mesmo `_avisos/<dia>.json`, a marca de
  um computador sumia e o lembrete saía de novo.
- **Mensagens de erro dizem o que fazer:** "Conecte a sua caixa de e-mail primeiro."

---

← [Stack, bibliotecas e frameworks](stack.md) · [Índice do guia](../README.md) · [Padrões de código](padroes-de-codigo.md) →
