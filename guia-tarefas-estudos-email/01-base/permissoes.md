# Perfis e permissões

← [Padrões de código](padroes-de-codigo.md) · [Índice do guia](../README.md) · [Visual: tema, cores e componentes](visual.md) →

---

📄 `backend/motor/db/banco.py`

```python
CARGOS = [
    ("",           "— não definido —", 0),
    ("estagiario", "Estagiário",       1),
    ("auxiliar",   "Auxiliar",         2),
    ("assistente", "Assistente",       3),
    ("analista",   "Analista",         4),
    ("lider",      "Líder",            5),
    ("socio",      "Sócio",            6),
]   # o administrador conta como 7 quando o nível importa (estudos)

def acesso_total(usuario) -> bool:
    """Vê e altera todas as empresas, rotinas e tarefas: admin e sócio."""
    u = usuario or {}
    return u.get("perfil") == "admin" or (u.get("perfil") == "usuario" and u.get("cargo") == "socio")

def cuida_do_cadastro(usuario) -> bool:
    """Admin, sócio e líder."""
    u = usuario or {}
    return acesso_total(u) or (u.get("perfil") == "usuario" and u.get("cargo") == "lider")
```

| Função | Quem | Usada para |
|---|---|---|
| `acesso_total(u)` | admin ou sócio | ver/alterar todas as tarefas; mudar prazo; apagar qualquer tarefa |
| `cuida_do_cadastro(u)` | admin, sócio ou líder | criar estudo para a equipe; acompanhar a equipe |
| `_pode_editar(u)` | qualquer pessoa logada | criar, marcar, comentar (ponto único para um futuro perfil "só consulta") |

---

← [Padrões de código](padroes-de-codigo.md) · [Índice do guia](../README.md) · [Visual: tema, cores e componentes](visual.md) →
