# Stack, bibliotecas e frameworks

← [Guia — Tarefas, Estudos e E-mail](../README.md) · [Índice do guia](../README.md) · [Arquitetura](arquitetura.md) →

---

## Como o programa roda

```
┌──────────────── computador do usuário (Windows) ─────────────────┐
│  Janela própria (pywebview + WebView2)                           │
│      └── telas React (arquivos estáticos montados pelo Vite)     │
│                │  fetch + cabeçalho X-Token                      │
│                ▼                                                 │
│  FastAPI + Uvicorn em 127.0.0.1:<porta sorteada>                 │
│      ├── serviços (regras e permissões)                          │
│      ├── motor (dados: SQLite local descartável + JSON)          │
│      ├── IMAP/SMTP direto com o provedor de e-mail  ─────────────┼──► Locaweb (internet)
│      └── lê/grava JSON na pasta compartilhada ───────────────────┼──► Z:\ (rede do escritório)
└──────────────────────────────────────────────────────────────────┘
```

É um **programa desktop**, sem servidor central: cada computador roda o seu Python, e todos
leem e gravam na mesma pasta de rede. O navegador não é usado; a janela é do Windows (WebView2).

## Backend (Python)

| Biblioteca | Versão | Para quê |
|---|---|---|
| Python | 3.14 no instalador (roda em 3.12+) | linguagem |
| **FastAPI** | 0.143 | API (rotas) |
| **Uvicorn** | 0.54 | servidor da API, só em `127.0.0.1` |
| **Pydantic** | 2.14 | validar o corpo dos pedidos |
| **pywebview** | 6.2 | janela própria usando o WebView2 do Windows |
| `imaplib`, `smtplib`, `email` | biblioteca padrão | ler e enviar e-mail (sem dependência extra) |
| `ctypes` + `crypt32` (DPAPI) | biblioteca padrão | guardar a senha do e-mail cifrada pelo Windows |
| `sqlite3` | biblioteca padrão | "rascunho" local para as telas serem rápidas |
| cryptography | 50.0 | assinatura das atualizações (não usado nestes módulos) |
| PyInstaller | 6.21 | gerar o `.exe` |
| Inno Setup | 6 | gerar o instalador |
| **pytest** + httpx | 9.0 / 0.28 | testes (o `TestClient` do FastAPI usa httpx) |

## Frontend (TypeScript)

| Biblioteca | Versão | Para quê, nestes módulos |
|---|---|---|
| **React** | 19.3 | telas |
| **Vite** | 8.3 | montar as telas (`npm run build` → `frontend/dist`) |
| **TypeScript** | 7.0 | tipos |
| **Tailwind CSS** (`@tailwindcss/vite`) | 4.3 | estilo; cores vêm de variáveis CSS ([Visual: tema, cores e componentes](visual.md)) |
| **TanStack Query** | 5.104 | buscar dados, cache, recarregar a cada N segundos, invalidar depois de gravar |
| TanStack Router | 1.170 | troca de telas (histórico em memória, sem endereço) |
| **Radix UI** (dialog, popover, checkbox, tooltip, slot) | 1.x | diálogos, menus suspensos, caixinhas acessíveis |
| shadcn/ui (padrão) + `class-variance-authority`, `clsx`, `tailwind-merge` | — | componentes próprios (`Botao`, `Dialogo`…) no estilo shadcn |
| **Motion** (`motion/react`) | 14.3 | animação de cartões abrindo/fechando (`AnimatePresence`) |
| **Sonner** | 2.0 | avisos rápidos ("Tarefa concluída.") |
| **Lucide** (`lucide-react`) | 1.59 | ícones |
| **DOMPurify** | 3.4.17 (fixa) | limpar o HTML dos e-mails |
| cmdk | 1.1 | busca rápida Ctrl+K (atalhos "Nova tarefa" etc.) |
| Vaul | 1.1 | gaveta lateral (outros módulos) |
| React Hook Form + Zod | 7.90 / 4.6 | formulários maiores (outros módulos; os de tarefa/estudo usam `useState`) |
| TanStack Virtual | 3.14 | listas enormes (outros módulos) |
| Vitest | 5.0 | testes de funções da tela |
| Fonte **Inter** | — | tipografia (com `cv11`, `ss01`) |

**Decisões tomadas** (não reabrir sem motivo): sem Electron, sem Tauri, sem Django, sem pandas.

## Se a sua stack for outra

| Peça daqui | Troque por (exemplos) |
|---|---|
| pywebview + FastAPI local | app web normal com backend no servidor (Node/Express, Django, Laravel…) |
| JSON na pasta de rede | banco de dados de verdade (Postgres, MySQL) — as regras continuam iguais |
| DPAPI (senha do e-mail) | macOS Keychain, Linux Secret Service, biblioteca `keyring`; em servidor web: cofre de segredos + cifragem por usuário |
| `imaplib`/`smtplib` | Node: `imapflow` + `nodemailer`; .NET: MailKit; PHP: `webklex/php-imap` + PHPMailer |
| TanStack Query | SWR, RTK Query ou o que o projeto já usa |
| DOMPurify | **não troque**: é o padrão para limpar HTML de e-mail |

---

← [Guia — Tarefas, Estudos e E-mail](../README.md) · [Índice do guia](../README.md) · [Arquitetura](arquitetura.md) →
