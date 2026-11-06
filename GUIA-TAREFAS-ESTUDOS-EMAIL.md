# Guia de referência — Tarefas, Estudos e E-mail

> **Para quem é:** uma IA (ou pessoa) que vai construir estas funções em OUTRO projeto.
> **O que é:** o retrato de como elas funcionam no *Sistema Contábil Plus* da UP Contábil,
> com as regras de negócio e os erros que já custaram caro. Não precisa copiar igual —
> use como ponto de partida e adapte à sua stack e ao seu cliente.
>
> Referência viva: repositório `sistema-contabil-plus` (arquivos citados entre crases).

---

## Sumário

1. [Ideias que valem para as três funções](#1-ideias-que-valem-para-as-três-funções)
2. [Perfis e permissões](#2-perfis-e-permissões)
3. [Tarefas](#3-tarefas)
4. [Estudos](#4-estudos)
5. [E-mail (caixa dentro do sistema)](#5-e-mail-caixa-dentro-do-sistema)
6. [E-mails automáticos (lembretes)](#6-e-mails-automáticos-lembretes)
7. [Ordem sugerida de construção e testes](#7-ordem-sugerida-de-construção-e-testes)

---

## 1. Ideias que valem para as três funções

### 1.1 Camadas

```
tela (React)  →  API (FastAPI, só repassa)  →  serviço (regras + permissão)  →  motor (dados)
```

| Camada | No projeto de referência | O que faz |
|---|---|---|
| Motor | `backend/motor/core/tarefas.py`, `core/estudos.py` | Lê/grava os dados, regras puras. |
| Serviço | `backend/servicos/tarefas.py`, `estudos.py`, `email_caixa.py` | Junta o que a tela precisa e **confere quem pode o quê**. |
| API | `backend/api/tarefas.py`, `estudos.py`, `email.py` | Rotas finas com Pydantic; não têm regra. |
| Tela | `frontend/src/rotas/tarefas/`, `estudos/`, `email/` | React + TanStack Query + Tailwind + Motion + Sonner. |

Stack de referência: Python + FastAPI (só em `127.0.0.1`, porta sorteada, senha interna) aberto numa
janela própria com pywebview/WebView2; frontend React + Vite + Tailwind + Radix/shadcn + Lucide.
Nada disso é obrigatório — as regras abaixo valem em qualquer stack.

### 1.2 Regras de ouro

- **Permissão é conferida no servidor, sempre.** A tela só esconde botões. O serviço recebe o usuário
  logado, confere e responde 403 com uma frase em português ("Só quem é responsável marca o checklist.").
  Já houve falha de auditoria por regra que só existia na tela.
- **Mensagens de erro em linguagem de gente**, dizendo o que fazer ("Dê um título para a tarefa.",
  "Conecte a sua caixa de e-mail primeiro.").
- **Respostas de gravação no formato `{ ok: true }` ou `{ ok: false, erro: "..." }`** para erros de
  preenchimento (a tela mostra num aviso); 403 para "você não pode".
- **Identidade estável entre computadores:** tarefa e estudo por `uid` (32 hex, `uuid4().hex`),
  pessoa por `login`, empresa por CNPJ. Nunca grave o `id` autoincremento de um banco local no arquivo
  compartilhado — ele muda de máquina para máquina.
- **Gravação atômica:** grava num temporário e troca pelo definitivo. Se a rede cair no meio, o arquivo
  antigo continua inteiro.
- **Separe "não existe" de "não consegui ler".** Arquivo corrompido/meio-gravado tratado como vazio
  já apagou trabalho de colega. Na dúvida, não sobrescreva.
- **Um dono por arquivo** sempre que der (ver Estudos): elimina conflito de gravação sem precisar mesclar.
- **Atualização da tela:** depois de gravar, invalide as consultas do assunto
  (`invalidateQueries({ queryKey: ["tarefas"] })`) e também o contador do menu.
- **Preferências da tela por pessoa** (filtro escolhido, modo lista/calendário) no `localStorage` com
  o id do usuário na chave — mais de uma pessoa usa o mesmo computador.

### 1.3 Onde os dados moram (no projeto de referência)

Pasta compartilhada do escritório, em JSON, com um SQLite local descartável só para as telas serem
rápidas:

```
<base>\
├── usuarios\<login>.json
├── tarefas\<uid>.json             uma tarefa por arquivo (checklist e comentários dentro)
├── estudos\<uid>.json             o estudo (só quem criou grava)
├── estudos\_andamento\<login>.json o que ESTA pessoa marcou (só ela grava)
├── estudos\_anexos\<uid>\...      arquivos das aulas e relatórios
├── estudos\_arquivo\<uid>.json    estudos arquivados
└── _avisos\<data>.json            marca de "lembrete já enviado hoje"
```

O e-mail **não** fica na base: as mensagens ficam no servidor de e-mail (IMAP).

Se o seu projeto usa um banco de dados de verdade (Postgres etc.), troque arquivos por tabelas — as
regras continuam as mesmas.

---

## 2. Perfis e permissões

```
perfil: "admin" | "usuario"
cargo (níveis): estagiário 1 · auxiliar 2 · assistente 3 · analista 4 · líder 5 · sócio 6
                (admin conta como 7 quando o nível importa)
```

| Função auxiliar | Quem | Usada para |
|---|---|---|
| `acesso_total(u)` | admin ou sócio | ver/alterar todas as tarefas; mudar prazo; apagar qualquer tarefa |
| `cuida_do_cadastro(u)` | admin, sócio ou líder | criar estudo compartilhado; acompanhar a equipe |
| `pode_editar(u)` | qualquer pessoa logada (ponto único para criar um perfil "só consulta" no futuro) | criar, marcar, comentar |

---

## 3. Tarefas

### 3.1 O que é

Um quadro de tarefas do escritório: título, descrição, prioridade, prazo (data + hora opcional),
um ou vários responsáveis, empresas ligadas, checklist e comentários.

### 3.2 Modelo de dados

Arquivo `tarefas/<uid>.json` (o que vai para a pasta compartilhada):

```json
{
  "uid": "9f0c…",
  "titulo": "REINF de outubro",
  "descricao": "O que é a tarefa (não muda).",
  "criador": "maria",
  "responsavel": "joao",
  "responsaveis": ["joao", "ana"],
  "empresa": "12345678000190",
  "empresas": ["12345678000190", "98765432000110"],
  "prioridade": 2,
  "prazo": "2026-11-14",
  "prazo_hora": "14:30",
  "concluida": false,
  "concluida_em": null,
  "criado_em": "2026-11-06 09:12:00",
  "atualizado_em": "2026-11-06 09:12:00",
  "comentarios": [
    {"uid": "…", "autor": "ana", "texto": "Falta o extrato.", "criado_em": "2026-11-06 10:01:02.345"}
  ],
  "itens": [
    {"texto": "Empresas mensais", "concluido": false, "ordem": 0, "pai": "", "empresa": ""},
    {"texto": "", "concluido": true, "ordem": 1, "pai": "Empresas mensais", "empresa": "12345678000190"}
  ]
}
```

Tabelas equivalentes (SQL): `tarefas`, `tarefa_responsaveis`, `tarefa_empresas`, `tarefa_itens`
(com `pai_id`), `tarefa_comentarios`. Ver `backend/motor/db/banco.py`.

Decisões que parecem detalhe mas não são:

- **Prazo e hora em campos separados.** Filtros "Hoje", "Atrasadas" e o calendário comparam a data com
  `=` e `<`. Se a hora fosse junto (`2026-11-14 14:30`), a tarefa sumiria do dia dela.
- **Prioridade 0 a 3** (3 = mais urgente). Ordem da lista: prioridade, depois prazo.
- **Vários responsáveis**, mas `responsavel` guarda o primeiro (compatibilidade e lugares onde só cabe
  um nome). O mesmo para `empresa` × `empresas`.
- **Guarde o login/CNPJ como veio, mesmo se este computador não conhece a pessoa/empresa.** Sem isso,
  a máquina que não conhecia regravava a tarefa sem dono — e ela sumia para todo mundo.
- **Comentários, não "campo de observação".** Um campo único era apagado sem querer por quem editava
  depois. Comentário tem autor e data, entra embaixo e ninguém sobrescreve ninguém.
- **Data do comentário com milésimos**: dois recados no mesmo segundo saíam em ordens diferentes em
  cada computador.
- **Gravação mesclando**: ao gravar a tarefa, mescle com o que um colega gravou nesse meio-tempo
  (comentários por `uid`), em vez de sobrescrever.

### 3.3 Regras de negócio

**Quem vê:** só os envolvidos — responsáveis e quem criou. Admin/sócio veem todas.

| Ação | Quem pode |
|---|---|
| Criar | qualquer um com `pode_editar` |
| Editar textos, empresas, checklist | responsáveis, quem criou, admin/sócio |
| Mudar o prazo ao editar | só admin/sócio (os demais mantêm o prazo original) |
| Concluir / reabrir, marcar item do checklist | **só os responsáveis** |
| Comentar | quem pode editar |
| Apagar comentário | quem escreveu, ou admin/sócio |
| Excluir a tarefa (com checklist e comentários) | quem criou, ou admin/sócio — o responsável **não** some com a tarefa de quem passou para ele |

**Vários responsáveis — dois modos na criação:**

- `compartilhada` (padrão): uma tarefa só, com todos; qualquer um conclui.
- `individual`: uma cópia por pessoa, cada uma com o checklist **inteiro** e a própria conclusão.

**Checklist em dois níveis:**

| Tipo de linha | Como reconhece | Tem caixinha? |
|---|---|---|
| Item comum | sem empresa, sem filhos | sim |
| Empresa | com `empresa` (pendurada num item ou solta) | sim |
| Item-grupo | sem empresa, **com** filhos | não — mostra "2 de 3" e fecha sozinho |

- O item-grupo **não entra** na conta que fecha a tarefa (senão ela nunca fecharia).
- **Fecha sozinha** quando o último item marcável é marcado.
- **Reabre sozinha** quando o checklist *estava* completo, a tarefa *estava* concluída e alguém
  desmarca um item. Leia o estado **anterior** do banco: senão quem concluiu à mão sem marcar item por
  item teria a tarefa reaberta a cada clique.
- `marcar_item` devolve o novo estado da tarefa (`true`/`false`) ou `null` se não mudou — a tela usa
  isso para redesenhar o cartão.
- **Editar não mexe nos tiques**: o que está marcado no banco fica (alguém pode ter marcado enquanto o
  formulário estava aberto). Marcar é só pelo cartão.
- Empresa tirada da tarefa leva junto a linha dela no checklist.

**Situação do prazo** (texto + cor para o cartão): atrasada · vence hoje (e "hoje às 14:30, já passou")
· próximos dias · sem prazo · concluída em dd/mm. Ver `rotulo_prazo` em `core/tarefas.py`.

**Pessoa/empresa inativa** continua aparecendo nas opções do formulário quando já está na tarefa
(marcada como inativa) — senão some da tarefa ao salvar.

### 3.4 API

```
GET    /api/tarefas?filtro=&busca=&dia=&so_minhas=   lista + contagens + pode_criar
GET    /api/tarefas/mes?ano=&mes=&so_minhas=         bolinhas do calendário por dia
GET    /api/tarefas/contador                         nº em aberto (badge do menu)
GET    /api/tarefas/opcoes?pessoas=1,2&empresas=3    pessoas e empresas para escolher
POST   /api/tarefas                                  criar  → { ok, ids: [...] }
GET    /api/tarefas/{id}                             detalhe com checklist em árvore
PUT    /api/tarefas/{id}                             editar
DELETE /api/tarefas/{id}                             excluir
POST   /api/tarefas/{id}/concluir      {concluido}
POST   /api/tarefas/{id}/itens/{item}  {concluido}   → { ok, tarefa_concluida: true|false|null }
GET    /api/tarefas/{id}/comentarios
POST   /api/tarefas/{id}/comentarios   {texto}
DELETE /api/tarefas/{id}/comentarios/{cid}
```

Filtros: `aberto` (padrão), `atrasadas`, `hoje`, `semana` (próximos 7 dias), `sem_prazo`,
`concluidas`, `todas`. Cada cartão já vem com `pode_editar`, `pode_concluir`, `pode_excluir`,
`pode_comentar` — a tela só obedece.

### 3.5 Tela (`frontend/src/rotas/tarefas/TelaTarefas.tsx`)

- **Painel de andamento** à esquerda: cada situação com a sua cor e a contagem é, ao mesmo tempo, o
  filtro da lista (em aberto · atrasadas em vermelho · vencem hoje em âmbar · próximos 7 dias em azul ·
  sem prazo · concluídas em verde · todas). Mais a barra de itens do checklist feitos.
- **Dois modos:** lista e calendário grande (mês, semana começando na segunda). Calendário pequeno ao
  lado da lista: clicar num dia filtra.
- **Interruptor "só as minhas"** e busca (com `useDeferredValue` para não pesquisar a cada tecla).
- **Cartão:** faixa de prioridade, título, prazo com cor da situação, etiquetas de responsáveis e
  empresas, progresso do checklist, nº de comentários e o último comentário. Checklist abre no próprio
  cartão. Links no texto viram botões que abrem no navegador.
- **Formulário** em painel lateral: título, descrição, prioridade, prazo + hora, responsáveis
  (com o modo compartilhada/individual quando há mais de um), empresas, checklist com itens-grupo.
- **Comentários** como conversa (autor, data, apagar o próprio).
- Exclusão sempre com diálogo de confirmação.
- Contador de tarefas em aberto no menu lateral.

---

## 4. Estudos

### 4.1 O que é

Treinamento interno. Um estudo é uma **trilha** (várias aulas) ou uma **aula solta**.

- **Pessoal:** qualquer um cria; só quem criou vê.
- **Compartilhado:** criado por admin, sócio ou líder; vai para a equipe, ou só para pessoas/cargos
  escolhidos.

A aula é a mesma para todos; **o andamento é de cada um.**

### 4.2 Modelo de dados — um dono por arquivo

```
estudos/<uid>.json               o estudo           → só quem criou grava
estudos/_andamento/<login>.json  o que a pessoa fez → só ela grava
estudos/_anexos/<uid>/<arquivo>  arquivos das aulas e dos relatórios
estudos/_arquivo/<uid>.json      arquivados (lidos só quando alguém pede)
```

Por isso não há mesclagem nem conflito. A **entrega** do relatório fica no andamento da pessoa; a
**avaliação** fica no arquivo do estudo (quem grava é quem criou).

```json
// estudos/<uid>.json
{
  "uid": "…", "titulo": "Novo Simples Nacional", "descricao": "…",
  "criador": "maria", "criador_nivel": 5, "criado_em": "2026-10-16 08:00:00",
  "pessoal": false, "para_logins": [], "para_cargos": ["analista"],
  "prazo": "2026-11-30", "relatorio": "aprovar",
  "aulas": [
    {"id": "a1", "titulo": "Aula 1", "prazo": "", "links": ["https://…"], "resumo": "…",
     "anexos": ["apostila.pdf"], "itens": [{"id": "i1", "texto": "Ler o capítulo 2"}]}
  ],
  "avaliacoes": {"joao": {"aprovado": false, "obs": "Faltou o exemplo.", "por": "maria", "em": "…"}},
  "arquivado_em": null
}

// estudos/_andamento/joao.json
{
  "<uid do estudo>": {
    "vistas": ["a1"], "itens": ["i1"],
    "relatorio": {"texto": "…", "anexo": "relatorio - joao - resumo.pdf", "enviado_em": "…"},
    "atualizado_em": "…"
  }
}
```

### 4.3 Regras de negócio

**Para quem vai um compartilhado:**

- Sem pessoas nem cargos escolhidos ("para todos") → vai só para quem está **abaixo** de quem criou
  (sócio cria → líder até estagiário; líder cria → analista até estagiário). Mesmo nível ou acima só
  acompanha.
- Com escolha → quem está em `para_logins` ou tem cargo em `para_cargos`.
- **Quem entrou na equipe depois de o estudo ser criado não recebe** (o novato não herda centenas de
  aulas antigas). Compare a data de criação do estudo com a da conta da pessoa.

**Quem altera:** quem criou; num compartilhado, também admin/sócio. Pessoal × compartilhado não muda
depois de criado. Estudo arquivado não se edita (tira do arquivo antes).

**Relatório de conclusão** — escolhido por estudo:

| `relatorio` | Conclui quando |
|---|---|
| `nao` | viu todas as aulas |
| `anexar` | viu todas as aulas **e** entregou o relatório (texto ou arquivo) |
| `aprovar` | viu todas, entregou, **e quem criou aprovou** (pode devolver com observação; a pessoa reenvia) |

Só entrega relatório depois de concluir todas as aulas. Avaliação mais antiga que o último envio é
ignorada (reenviou depois de devolvido → volta a "aguardando").

**Situação de um estudo para uma pessoa** (`situacao` em `core/estudos.py`): aulas feitas/total,
estado do relatório (`falta`, `entregue`, `aguardando`, `devolvido`, `aprovado`), concluído, atrasado
(prazo do estudo vencido ou prazo de alguma aula não vista vencido).

**Estado do cartão** (ordem de prioridade): arquivado → acompanhando (não é para mim) → concluído →
atrasado → em andamento → a começar.

**Arquivar:** o concluído fica à vista N dias (configurável, padrão 7) e depois vai sozinho para o
arquivo. Se o auto-arquivar falhar, ignore — é só arrumação.

**Anexos:**

- Enviados pela tela em base64; limite de 25 MB.
- **Lista branca de extensões** (PDF, planilha, documento, imagem, vídeo, compactado). Nada de
  `.exe`, `.html`, `.bat`.
- Nunca sobrescreve: `arquivo (2).pdf`, `arquivo (3).pdf`…
- Use só o `basename` do nome recebido (evita `..\..\`).
- Valide o `uid` com regex `^[0-9a-f]{32}$` antes de montar caminho.
- Relatório de colega (`relatorio - <login> - …`) só é visto por quem criou o estudo. Compare sem
  diferenciar maiúsculas (o Windows não diferencia).

**Visão da equipe:** quem criou (e admin/sócio/líder num compartilhado) vê, por pessoa, aulas feitas,
relatório e botões de aprovar/devolver.

### 4.4 API

```
GET    /api/estudos?filtro=&busca=&arquivados=     lista + contagens por filtro
GET    /api/estudos/opcoes                         pessoas, cargos, regras de relatório, novo_uid
POST   /api/estudos                                criar/editar (uid + "novo": true na criação)
DELETE /api/estudos/{uid}
POST   /api/estudos/{uid}/arquivar      {sim}
POST   /api/estudos/{uid}/aulas/{aula}  {sim}      marcar aula vista
POST   /api/estudos/{uid}/itens/{item}  {sim}      marcar item da aula
POST   /api/estudos/{uid}/anexos        {nome, dados(base64)}
GET    /api/estudos/{uid}/anexos/{nome}            baixar
POST   /api/estudos/{uid}/anexos/{nome}/abrir      abrir no programa do Windows
POST   /api/estudos/{uid}/relatorio     {texto, nome, dados}
GET    /api/estudos/{uid}/equipe
POST   /api/estudos/{uid}/avaliar       {login, aprovado, obs}
```

Filtros: pessoa comum → "Para fazer" (padrão) · "Concluídos" · "Todos"; quem compartilha →
"Acompanhando" (padrão) também. O `uid` do estudo novo é gerado antes (vem em `opcoes`) para os anexos
poderem subir antes de salvar.

### 4.5 Tela (`frontend/src/rotas/estudos/`)

- Cartões com barra lateral colorida pelo estado (âmbar a começar · azul em andamento · vermelho
  atrasado · verde concluído · cinza acompanhando/arquivado) e selo com o nome do estado.
- Cartão abre as aulas: links, resumo, anexos, itens com caixinha, botão "marquei como vista".
- Formulário (`FormEstudo.tsx`): título, descrição, prazo, pessoal × compartilhado, para quem
  (pessoas/cargos), regra de relatório, aulas sanfonadas (título, prazo, links, resumo, anexos, itens).
  Aula única sem título herda o título do estudo; com várias aulas, aula com conteúdo e sem título
  bloqueia o salvar.
- Janelas (`Janelas.tsx`): entregar relatório; equipe (andamento de cada um, aprovar/devolver).
- Botão "Arquivados" para ver/restaurar.

---

## 5. E-mail (caixa dentro do sistema)

### 5.1 O que é

Substitui o Outlook: ler, baixar anexos, escrever, responder, encaminhar, rascunhos, pastas, busca.

**Arquitetura:** o próprio computador conversa direto com o provedor por **IMAP** (ler) e **SMTP**
(enviar), com `imaplib` e `smtplib` da biblioteca padrão do Python. **Sem servidor intermediário e sem
copiar mensagens para a base** — como o Outlook em IMAP. Assim sistema, Outlook e webmail enxergam as
mesmas mensagens e podem conviver durante a troca.

Referência: `backend/servicos/email_caixa.py` (≈1.250 linhas), `frontend/src/rotas/email/`,
plano em `docs/PLANO-EMAIL.md`.

Provedor de referência: Locaweb — IMAP `email-ssl.com.br:993` (SSL), SMTP `email-ssl.com.br:465` (SSL).
A tela deixa trocar servidor e porta. Provedores que exigem OAuth (Gmail, Microsoft 365) precisam de
outro fluxo de login — este guia cobre e-mail + senha.

### 5.2 Conta e senha

- **Cada pessoa conecta a SUA caixa**, uma vez, com botão "Testar" antes de salvar e "Desconectar".
- **A senha fica só neste computador, cifrada pelo Windows (DPAPI)**, ligada ao usuário do Windows
  (`CryptProtectData` via `ctypes`). Nunca vai para a base compartilhada, git ou código. Trocou de
  computador → conecta de novo. Em outro sistema operacional: Keychain (macOS) / Secret Service (Linux)
  ou a biblioteca `keyring`.
- Perfil: nome de exibição e assinatura HTML (limite de tamanho; dá para importar as assinaturas do
  Outlook da pasta `%APPDATA%\Microsoft\Signatures`).

### 5.3 Conexões IMAP

- **Uma conexão aberta por pessoa, reaproveitada.** Abrir IMAP a cada clique é lento.
- Antes de usar, `NOOP` com **timeout curto (5 s)**: o provedor derruba conexões paradas, e esperar o
  timeout normal (25 s) travaria a tela. Caiu → abre outra sozinho.
- **Conexão e trava separadas para o trabalho de fundo** (contador de não lidas, pop-up de e-mail
  novo). Senão apagar/enviar esperava na fila atrás deles.
- **Conexão avulsa para buscas demoradas** (no texto, em todas as pastas): o servidor leva ~1 min para
  ler milhares de e-mails, e o resto da caixa continua funcionando.
- **Nome de pasta sempre entre aspas e sem quebra de linha** (`_q`): evita comando IMAP escondido.
- Erros traduzidos: senha recusada → "Confira em «Conta»"; sem rede → "Confira a internet".

### 5.4 Pastas

- Reconhece Entrada, Enviados, Rascunhos, Arquivo, Spam, Lixeira pelas marcas especiais do IMAP
  (`\Sent`, `\Drafts`, `\Trash`…) e, na falta delas, pelo nome ("Sent", "Itens enviados"…). Ordem fixa
  dessas no topo; as criadas pela pessoa embaixo.
- Nomes de pasta no IMAP vêm em **UTF-7 modificado** — converta para mostrar e para gravar.
- Criar, renomear e apagar pasta (as especiais não). Lista de pastas em cache por 2 minutos.

### 5.5 Lista e leitura

- Lista **por UID**, 50 por página, carregando mais ao rolar (`antes_de=<menor uid visto>`).
  Busca só os cabeçalhos (`BODY.PEEK[HEADER.FIELDS …]`, flags, tamanho).
- Filtros: todas · não lidas (`UNSEEN`) · sinalizadas (`FLAGGED`) · com anexo. Busca por remetente/
  assunto (rápida) ou no texto (avulsa).
- Leitura com `BODY.PEEK[]` (não marca como lida sozinho) e depois `STORE +FLAGS (\Seen)` de propósito.
  Guarda as 3 últimas mensagens brutas em memória para reabrir rápido.
- Corpo: prefere HTML, senão texto. **Imagens coladas (`cid:`) viram `data:`** para aparecerem.
- Anexos listados com índice, nome, tipo e tamanho; baixar um, baixar todos em `.zip`, ou abrir.
- **Conversa:** agrupa pelo assunto sem `RE:/ENC:/FW:/TR:` (`assunto_base`).
- Ações em lote: marcar lida/não lida, sinalizar, mover (arrastando para a pasta), excluir (vai para
  a Lixeira; na Lixeira apaga de vez só os escolhidos).

### 5.6 Segurança (o e-mail vem de fora)

- **HTML limpo com DOMPurify** e mostrado num **`<iframe sandbox="allow-same-origin">` com `srcDoc`** —
  sem `allow-scripts`, nada roda. Tags proibidas: `form input button textarea select iframe object
  embed link meta base`; atributo `srcset` proibido; links ganham `rel="noopener noreferrer"`.
- Clique em link dentro da moldura é interceptado: **só `http`/`https`**, aberto no navegador do sistema.
- Abrir anexo direto só para a lista branca de documentos (a mesma dos Estudos); o resto só baixa.
  Anexo aberto vai para uma pasta temporária com a marca "veio da internet" e é limpo depois.
- Recomendado (está no plano): imagens externas só carregam com um clique, porque avisam ao remetente
  que o e-mail foi aberto.
- **Tema escuro:** inverte as cores do e-mail com CSS (`filter: invert(1) hue-rotate(180deg)`) e
  desinverte imagens; botão "ver com as cores originais".

### 5.7 Escrever e enviar

- Modos: novo · responder · responder a todos · encaminhar (leva os anexos originais) · rascunho.
- Editor rico simples (`contentEditable` limpo com DOMPurify), campos Para/Cc/Cco com sugestão de
  contatos (últimos 400 enviados + colegas do escritório + e-mails das empresas, em cache de 10 min), anexos do computador (limite total 25 MB).
- Resposta leva `In-Reply-To` e `References` (para o Outlook agrupar a conversa) e o texto citado.
- **Envio:** porta 465 → `SMTP_SSL`; outra porta → `SMTP` + `STARTTLS`. Trate destinatário recusado
  ("Saiu, mas o servidor não entregou para: …").
- **Depois de sair, em segundo plano:** grava a cópia em "Enviados" (`APPEND` com `\Seen`), marca a
  original como respondida (`\Answered`) e apaga o rascunho. A tela responde "enviado" assim que o
  SMTP aceitar — subir a cópia com anexos demora. Se a cópia falhar, **avise a pessoa** (o e-mail já
  foi, mas não aparece em Enviados).
- **Rascunho** salvo no próprio IMAP (pasta Rascunhos) com um cabeçalho próprio
  (`X-Plus-Rascunho: <id>`) para substituir a versão anterior em vez de acumular cópias.

### 5.8 Avisos de e-mail novo

- Contador de não lidas da Entrada no menu (`STATUS INBOX (UNSEEN)` — um pedido só).
- Pop-up de e-mail novo: a tela guarda o maior UID visto; o servidor devolve até 5 não lidos com UID
  maior. Na primeira chamada só devolve o UID atual (não dispara pop-up de tudo).
- Atualização: Entrada a cada 30 s, outras pastas e lista de pastas a cada 120 s; nada durante busca
  no texto. Sem internet → devolve 0/vazio em silêncio, sem erro na tela.

### 5.9 API

```
GET  /api/email/conta              POST /api/email/conta  {email, senha, servidores, portas}
POST /api/email/conta/desconectar  POST /api/email/perfil {nome_exibicao, assinatura_html}
GET  /api/email/assinaturas-outlook
GET  /api/email/nao-lidas          GET  /api/email/novos?depois_de=
GET  /api/email/pastas             POST /api/email/pastas/criar | renomear | apagar
GET  /api/email/mensagens?pasta=&antes_de=&filtro=&termo=&no_texto=
GET  /api/email/mensagem?pasta=&uid=
GET  /api/email/conversa?pasta=&uid=
GET  /api/email/anexo?pasta=&uid=&indice=      POST /api/email/anexo/abrir
GET  /api/email/anexos-zip?pasta=&uid=
POST /api/email/marcar  {pasta, uids, lida?, sinalizada?}
POST /api/email/mover   {pasta, uids, destino}
POST /api/email/excluir {pasta, uids}
POST /api/email/enviar  {para, cc, cco, assunto, html, anexos, resposta?, rascunho_id?}
POST /api/email/rascunho            POST /api/email/rascunho/descartar
GET  /api/email/contatos            GET  /api/email/buscar?termo=&no_texto=
```

### 5.10 Tela (`frontend/src/rotas/email/TelaEmail.tsx`, `Escrever.tsx`)

- Sem conta → tela "Conectar a caixa" com os campos já preenchidos com o provedor padrão.
- Três colunas: **pastas** (ícone, nome, nº de não lidas, soltar e-mails arrastados) · **lista**
  (remetente, assunto, data relativa, clipe, negrito se não lida, seleção múltipla, menu do botão
  direito) · **leitura** (cabeçalho de/para/cc/data, corpo na moldura, anexos com Baixar/Abrir,
  responder/encaminhar, imprimir/salvar PDF, conversa).
- Escrever em janela própria sobre a caixa; o rascunho é salvo sozinho no servidor 3 s depois da
  última alteração (aparece em "Rascunhos" também no Outlook).

---

## 6. E-mails automáticos (lembretes)

Referência: `backend/motor/core/avisos.py`, `core/email_rotina.py`.

Conta de envio própria do sistema (separada das caixas pessoais), configurada pelo admin.

**Três avisos de tarefa:**

1. **Tarefa nova** — na hora em que alguém cria/repassa uma tarefa para você (enviado em segundo plano,
   sem segurar a tela; ao fechar o programa, espera alguns segundos os envios pendentes).
2. **Atrasada** e 3. **Prazo chegando** (hoje ou nos próximos N dias úteis, contando feriados) —
   **juntos, num e-mail só por pessoa por dia**. Oito atrasadas = um e-mail, não oito.

**Quem dispara, sem servidor:** o primeiro computador que abrir o programa no dia ("carteiro").
Se ninguém abrir, nada sai — para horário garantido, use um agendador.

**Duas travas contra repetição:**

- `dia|<e-mail>` no arquivo do dia (`_avisos/<data>.json`) → no máximo um e-mail por pessoa por dia,
  venha de qual computador vier;
- `<uid>|<e-mail>` num registro permanente → cada tarefa nova é comunicada uma vez só.
- A marca é gravada **depois** de o e-mail sair (antes, fechar no meio perdia o lembrete).

**Nunca derrube a abertura do programa** por causa de lembrete: tudo em `try`, devolve um resumo
(`enviados`, `pulados`, `falhas`).

**E-mail em HTML + texto puro** (`multipart/alternative`), com nome do escritório e rodapé configuráveis.

**Modelos com campos que se preenchem** (e-mail ao cliente da rotina mensal):
`{empresa} {cnpj} {competencia} {documentos} {responsavel} {setor} {escritorio}` — modo manual (a
pessoa revisa e envia) ou automático a partir do N-ésimo dia útil.

**Base de teste:** variável `SISTEMA_CONTABIL_SEM_EMAIL=1` → nenhum e-mail automático sai; com
`SISTEMA_CONTABIL_EMAIL_SIMULADO=1` vira um arquivo `.eml` na pasta de teste (dá para abrir no
Outlook). Testes automáticos trocam o enviador por uma função falsa (`definir_enviador`).
O que a pessoa escreve à mão na caixa de e-mail sai mesmo na base de teste.

---

## 7. Ordem sugerida de construção e testes

1. **Usuários e permissões** (seção 2) — tudo depende disso.
2. **Tarefas:** modelo → criar/listar/editar → concluir → checklist simples → checklist em dois níveis
   com fechar/reabrir → comentários → filtros e contagens → calendário → contador no menu.
3. **Estudos:** estudo pessoal com aulas → andamento por pessoa → compartilhado com destinatários →
   anexos → relatório (`anexar`, depois `aprovar`) → visão da equipe → arquivar.
4. **E-mail:** conectar/testar com senha protegida → pastas → lista paginada → leitura segura → anexos
   → ações (marcar, mover, excluir) → escrever/enviar + cópia em Enviados → rascunhos → busca →
   contador e pop-up.
5. **Lembretes por e-mail** (precisa de Tarefas e de uma conta de envio).

**Testes que valem a pena (pytest):**

- Tarefas: quem vê/edita/conclui/exclui (um teste por linha da tabela 3.3); checklist fecha e reabre
  (incluindo "concluí à mão e desmarquei um item" → não reabre); modo individual cria uma cópia por
  pessoa com o checklist inteiro.
- Estudos: destinatários por nível, por cargo e por login; novato não recebe estudo antigo; situação
  em cada regra de relatório; anexo com extensão proibida, nome com `..\`, arquivo repetido.
- E-mail: **servidor IMAP/SMTP simulado** (classe falsa no lugar de `imaplib.IMAP4_SSL` e
  `smtplib.SMTP_SSL`), sem rede e sem conta de verdade. O teste com conta real é feito pela pessoa, na
  tela, digitando a própria senha.
- Lembretes: um e-mail por pessoa por dia; tarefa comunicada uma vez; base de teste não envia.

**Nunca:** testar na base oficial; colocar senha, token ou dado de cliente no git.
