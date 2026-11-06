# Estudos — API

← [Estudos — regras de negócio](regras.md) · [Índice do guia](../README.md) · [Estudos — lista de conferência](conferencia.md) →

---

## API

```
GET    /api/estudos?filtro=&busca=&arquivados=     lista + contagens por aba
GET    /api/estudos/opcoes                         pessoas, cargos, regras de relatório, novo_uid
POST   /api/estudos                                criar/editar (uid; "novo": true na criação)
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

O `uid` do estudo novo vem de `opcoes` **antes** de salvar, para os anexos poderem subir enquanto o
formulário está aberto. `"novo": true` evita que dois formulários abertos com o mesmo `uid` se
sobrescrevam.

Regras do formulário (no serviço): aula única sem título herda o título do estudo; com várias aulas,
aula com conteúdo e sem título bloqueia ("Dê um título à aula 2 (ou apague a aula).").

---

← [Estudos — regras de negócio](regras.md) · [Índice do guia](../README.md) · [Estudos — lista de conferência](conferencia.md) →
