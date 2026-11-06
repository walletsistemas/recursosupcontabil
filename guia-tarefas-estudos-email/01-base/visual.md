# Visual: tema, cores e componentes

← [Perfis e permissões](permissoes.md) · [Índice do guia](../README.md) · [Tarefas](../02-tarefas/README.md) →

---

Objetivo: telas modernas e calmas, com cara de produto Apple — fundo bege claro, cartões brancos com
cantos bem arredondados, sombras leves, uma cor de destaque (verde), fonte Inter, animações curtas.

## Cores como variáveis (claro e escuro)

📄 `frontend/src/index.css` (trecho)

```css
@import "tailwindcss";
@custom-variant dark (&:where(.escuro, .escuro *));   /* tema escuro: classe .escuro no <html> */

:root {
  --fundo: #f4f0ea;      --superficie: #ffffff;
  --texto: #2b2a28;      --texto-2: #5e564e;      --texto-3: #70675d;
  --borda: rgb(43 42 40 / 0.1);                   --campo: rgb(43 42 40 / 0.05);
  --verde: #2f8257;      --verde-texto: #245f42;  --vermelho: #c0432f;
  --primario: #2b2a28;   --primario-texto: #f4f0ea;
  --menu: #ebe4da;
  /* situação: no prazo · com atraso · vencida · a vencer */
  --st-prazo: #2f8257;   --st-atraso: #c98a1b;    --st-vencida: #c0432f;  --st-aberta: #d6cdc1;
  /* tipo, no calendário e no "Seu dia" */
  --tp-rotina: #4f6fa8;  --tp-tarefa: #8a5a83;    --tp-estudo: #2f7f7a;
}
.escuro {
  --fundo: #191816;      --superficie: #22211e;
  --texto: #f2ede6;      --texto-2: #c9c0b5;      --texto-3: #9a9288;
  --borda: rgb(255 255 255 / 0.09);               --campo: rgb(255 255 255 / 0.06);
  --verde: #2e8257;      --verde-texto: #8ed3ae;  --vermelho: #e8735f;
  --primario: #e6ded3;   --primario-texto: #1b1a18;
  --menu: #262420;
}

/* as variáveis viram classes do Tailwind: bg-superficie, text-texto-2, border-borda, bg-verde… */
@theme inline {
  --font-sans: "Inter", -apple-system, "Segoe UI", system-ui, sans-serif;
  --color-fundo: var(--fundo);
  --color-superficie: var(--superficie);
  --color-texto: var(--texto);
  --color-texto-2: var(--texto-2);
  --color-texto-3: var(--texto-3);
  --color-borda: var(--borda);
  --color-campo: var(--campo);
  --color-verde: var(--verde);
  --color-verde-texto: var(--verde-texto);
  --color-vermelho: var(--vermelho);
  --color-primario: var(--primario);
  --color-primario-texto: var(--primario-texto);
}
```

Trocar o tema = `document.documentElement.classList.toggle("escuro", tema === "escuro")` e guardar a
escolha.

## Cores com significado

| Significado | Cor | Onde |
|---|---|---|
| Urgente / atrasado / vencida | vermelho | prioridade 3, prazo vencido, estudo atrasado |
| Alta / vence hoje / a começar | âmbar/laranja | prioridade 2, prazo do dia |
| Próximos dias / em andamento | azul (`#5B7BC0`) | filtro "Próximos 7 dias", estudo em andamento |
| Concluído / no prazo / ação principal | verde | concluídas, botões "Criar", "Enviar" |
| Sem prazo / acompanhando / baixa | cinza | neutros |

## Botão no estilo shadcn

📄 `frontend/src/componentes/ui/botao.tsx`

```tsx
import { Slot } from "@radix-ui/react-slot";
import { cva, type VariantProps } from "class-variance-authority";
import { Loader2 } from "lucide-react";

const variantes = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap font-medium transition-all " +
    "duration-200 active:scale-[0.97] disabled:pointer-events-none disabled:opacity-50 cursor-default " +
    "focus-visible:ring-4 focus-visible:ring-verde/25",
  {
    variants: {
      variante: {
        primario: "bg-primario text-primario-texto shadow-sm hover:brightness-110",
        verde: "bg-verde text-white shadow-sm hover:brightness-105",
        leve: "bg-campo text-texto hover:bg-borda",
        fantasma: "text-texto-2 hover:bg-campo hover:text-texto",
        link: "text-verde-texto hover:underline px-0 h-auto",
      },
      tamanho: {
        p: "h-8 px-3 text-[13px] rounded-lg",
        m: "h-10 px-4 text-sm rounded-xl",
        g: "h-12 px-5 text-[15px] rounded-2xl",
        icone: "size-9 rounded-full",
      },
    },
    defaultVariants: { variante: "primario", tamanho: "m" },
  },
);

export const Botao = React.forwardRef<HTMLButtonElement, Props>(
  ({ className, variante, tamanho, asChild, carregando, children, disabled, ...props }, ref) => {
    const Comp = asChild ? Slot : "button";
    return (
      <Comp ref={ref} className={cn(variantes({ variante, tamanho }), className)}
            disabled={disabled || carregando} {...props}>
        {carregando && <Loader2 className="size-4 animate-spin" />}
        {children}
      </Comp>
    );
  },
);
```

## Padrões que se repetem nas três telas

- **Cartão:** `rounded-2xl border border-borda bg-superficie` com sombra quase invisível; ao passar o
  mouse, a borda fica esverdeada e a sombra cresce. Faixa colorida de 4 px (`w-1`) à esquerda
  (prioridade ou estado). Ações em ícones à direita, apagadas quando a pessoa não pode. Clicar em
  qualquer parte do cartão expande.
- **Etiquetas ("chips")** pequenas e arredondadas: cor de fundo 12% + texto da mesma cor.
- **Diálogos** (Radix Dialog) com fundo desfocado; formulários em diálogo com botões "Cancelar"
  (leve) e ação principal (verde) no rodapé.
- **Animações** com Motion: cartão entra subindo 6 px e aparecendo (200 ms) e sai encolhendo
  (`AnimatePresence`, `layout`); painéis com mola (`type: "spring"`).
- **Avisos** com Sonner: `toast.success("Tarefa concluída.")` e `toast.error(<frase que veio do servidor>)`.
- **Lista vazia** com uma frase que explica e, quando faz sentido, o botão de criar.
- **Ícones** Lucide, traço fino, 16–18 px.

---

← [Perfis e permissões](permissoes.md) · [Índice do guia](../README.md) · [Tarefas](../02-tarefas/README.md) →
