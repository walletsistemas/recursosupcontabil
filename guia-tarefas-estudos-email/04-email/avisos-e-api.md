# E-mail — avisos de e-mail novo e API

← [E-mail — escrever e enviar](escrever-e-enviar.md) · [Índice do guia](../README.md) · [E-mail — lista de conferência](conferencia.md) →

---

## Avisos de e-mail novo

- **Número do menu:** `STATUS INBOX (UNSEEN)` — um pedido só.
- **Pop-up de e-mail novo:** a tela guarda o maior UID já visto (por pessoa); o servidor devolve
  até 5 não lidos com UID maior. Na primeira chamada só devolve o UID atual (não dispara pop-up de
  tudo o que já existia).
- **Frequência:** pop-up a cada 20 s (também com a janela em segundo plano); lista da Entrada a cada
  30 s; outras pastas e a lista de pastas a cada 120 s; nada durante busca no texto.
- **Sem internet:** devolve 0/vazio em silêncio, sem erro na tela.

📄 `frontend/src/componentes/layout/Avisos.tsx` (trecho)

```tsx
const em = useQuery({
  queryKey: ["avisos-email", ultimoEmail],
  queryFn: () => emailApi.novos(ultimoEmail),
  refetchInterval: 20_000,               // e-mail novo aparece em até 20 s
  refetchIntervalInBackground: true,
});
```

## API

```
GET  /api/email/conta                    POST /api/email/conta  {email, senha, imap_*, smtp_*}
POST /api/email/conta/desconectar        POST /api/email/perfil {nome_exibicao, assinatura_html}
GET  /api/email/assinaturas-outlook
GET  /api/email/nao-lidas                GET  /api/email/novos?depois_de=
GET  /api/email/pastas                   POST /api/email/pastas/criar | renomear | apagar
GET  /api/email/mensagens?pasta=&antes_de=&filtro=&termo=&no_texto=
GET  /api/email/mensagem?pasta=&uid=     GET  /api/email/conversa?pasta=&uid=
GET  /api/email/anexo?pasta=&uid=&indice=     POST /api/email/anexo/abrir?pasta=&uid=&indice=
GET  /api/email/anexos-zip?pasta=&uid=
POST /api/email/marcar  {pasta, uids, lida?, sinalizada?}
POST /api/email/mover   {pasta, uids, destino}
POST /api/email/excluir {pasta, uids}           → { ok, lixeira: true|false }
POST /api/email/enviar  {para, cc, cco, assunto, html, anexos, originais?, resposta?, rascunho_id?, substitui?}
POST /api/email/rascunho                 POST /api/email/rascunho/descartar
GET  /api/email/contatos                 GET  /api/email/buscar?termo=&no_texto=
```

Tipos principais — 📄 `frontend/src/api/email.ts`

```ts
export type Pasta = { id: string; nome: string; caminho: string; tipo: string; nao_lidas: number; total: number };
export type Pessoa = { nome: string; email: string };
export type Resumo = { uid: number; de: Pessoa[]; para: Pessoa[]; assunto: string; data: string;
                       lida: boolean; sinalizada: boolean; anexo: boolean; tamanho: number };
export type Anexo = { indice: number; nome: string; tipo: string; tamanho: number };
export type Mensagem = { uid: number; assunto: string; de: Pessoa[]; para: Pessoa[]; cc: Pessoa[]; data: string;
                         html: string; texto: string; anexos: Anexo[]; responder_para: Pessoa[];
                         message_id: string; references: string; rascunho_id: string };
```

---

← [E-mail — escrever e enviar](escrever-e-enviar.md) · [Índice do guia](../README.md) · [E-mail — lista de conferência](conferencia.md) →
