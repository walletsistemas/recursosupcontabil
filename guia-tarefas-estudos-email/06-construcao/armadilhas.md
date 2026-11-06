# Armadilhas conhecidas (resumo)

← [Ordem de construção e lista de conferência](ordem-e-conferencia.md) · [Índice do guia](../README.md)

---

| Armadilha | O que acontecia | Como evitar |
|---|---|---|
| Permissão só na tela | Qualquer um alterava pela API | Conferir no serviço; teste por regra |
| `id` local no arquivo compartilhado | Outro computador ligava a tarefa à pessoa errada | `uid`, login, CNPJ |
| Login/CNPJ desconhecido descartado | Tarefa perdia o dono para todo mundo | Guardar como veio e religar depois |
| Prazo e hora no mesmo campo | Tarefa sumia do filtro "Hoje" | Campos separados |
| Atraso contado contra hoje depois de concluir | "Atrasada 12 dias" numa tarefa entregue no prazo | Contar contra o dia da entrega |
| Reabrir sem olhar o estado anterior | Tarefa concluída à mão reabria a cada clique | Ler o antes do banco |
| Editar regravando os tiques | Desfazia o que o colega marcou | Editar não mexe nos tiques |
| Campo único de observação | Texto de um apagava o do outro | Comentários com autor e data |
| Arquivo ilegível tratado como vazio | Trabalho do colega apagado | Separar "não existe" de "não li" |
| Estudo "para todos" sem limite | Sócio recebia estudo do líder; novato herdava tudo | Só cargos abaixo; só quem já estava |
| `EXPUNGE` comum | Apagava o que o Outlook só tinha marcado | `UID EXPUNGE` / só estas |
| Conexão IMAP parada | Tela travava 25 s | `NOOP` com timeout de 5 s |
| Contador na mesma fila das ações | Enviar esperava a contagem | Conexão de fundo separada |
| HTML do e-mail sem limpeza | Script e rastreador rodando | DOMPurify + iframe sem scripts |
| Esperar a cópia em Enviados | "Enviando…" demorava com anexos | Responder após o SMTP; cópia em segundo plano |
| Marca de "lembrete enviado" antes do envio | Programa fechado no meio = lembrete perdido | Marcar depois de enviar |
| Todos os computadores no mesmo arquivo de marcas | A marca de um sumia; o lembrete saía de novo | Um arquivo por computador; a leitura junta |
| E-mail automático na base de teste | Cliente recebia e-mail de teste | Variável `SISTEMA_CONTABIL_SEM_EMAIL=1` |

---

← [Ordem de construção e lista de conferência](ordem-e-conferencia.md) · [Índice do guia](../README.md)
