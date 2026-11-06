# E-mail — segurança

← [E-mail — pastas, lista e leitura](pastas-e-leitura.md) · [Índice do guia](../README.md) · [E-mail — escrever e enviar](escrever-e-enviar.md) →

---

**HTML limpo com DOMPurify e mostrado numa moldura sem scripts** — 📄 `frontend/src/rotas/email/TelaEmail.tsx` (encurtado)

```tsx
import DOMPurify from "dompurify";

function limpar(html: string) {
  DOMPurify.removeAllHooks();
  DOMPurify.addHook("afterSanitizeAttributes", (no) => {
    if (no.tagName === "A") no.setAttribute("rel", "noopener noreferrer");
  });
  const limpo = DOMPurify.sanitize(html, {
    WHOLE_DOCUMENT: true,
    FORBID_TAGS: ["form", "input", "button", "textarea", "select", "iframe", "object", "embed", "link", "meta", "base"],
    FORBID_ATTR: ["srcset"],
  });
  DOMPurify.removeAllHooks();
  return limpo;
}

function Corpo({ m }: { m: Mensagem }) {
  const moldura = useRef<HTMLIFrameElement>(null);
  const limpo = useMemo(() => (m.html ? limpar(m.html) : ""), [m.html]);

  // solta o observador e o clique da mensagem anterior (ao trocar de mensagem ou fechar)
  const soltar = useRef<() => void>(() => {});
  useEffect(() => () => soltar.current(), [limpo]);

  const aoCarregar = () => {
    const doc = moldura.current?.contentDocument;
    if (!doc || !moldura.current) return;
    // a altura acompanha o conteúdo (as imagens chegam depois)
    const ajustar = () => moldura.current && (moldura.current.style.height = `${doc.documentElement.scrollHeight + 8}px`);
    ajustar();
    soltar.current();
    const obs = new ResizeObserver(ajustar);
    obs.observe(doc.documentElement);
    // clique em link: só http/https, aberto no navegador do sistema
    const clique = (e: MouseEvent) => {
      const a = (e.target as HTMLElement).closest("a");
      if (!a) return;
      e.preventDefault();
      const href = a.getAttribute("href") ?? "";
      if (/^https?:\/\//i.test(href)) abrirLink(href);
    };
    doc.addEventListener("click", clique);
    soltar.current = () => {
      obs.disconnect();
      doc.removeEventListener("click", clique);
      soltar.current = () => {};
    };
  };

  if (!m.html) return <pre className="whitespace-pre-wrap">{m.texto || "(mensagem sem texto)"}</pre>;
  return (
    <iframe ref={moldura} title="Conteúdo do e-mail"
      sandbox="allow-same-origin"          // SEM allow-scripts: nada roda dentro
      srcDoc={limpo} onLoad={aoCarregar}
      className="min-h-[200px] w-full rounded-xl border border-borda bg-white" />
  );
}
```

Tema escuro do e-mail (inverte cores e desinverte imagens):

```ts
const CSS_ESCURO = `html{filter:invert(1) hue-rotate(180deg);background:#e6e6e6!important}
img,video,picture,svg,[background],[style*="background-image"]{filter:invert(1) hue-rotate(180deg)}`;
```

**Anexos:**

- Nome limpo (`basename` + troca de `\/:*?"<>|`).
- **Abrir direto só a lista branca** (a mesma dos Estudos); `.exe`, `.html`, `.bat` etc. só baixam
  ("Esse tipo de arquivo não abre direto. Use «Baixar» e confira antes de abrir.").
- Anexo aberto vai para `<pasta do computador>\email\anexos_abertos` com a marca "veio da internet"
  (o Office abre em modo protegido e bloqueia macros). A pasta é limpa ao desconectar ou sair, e a
  limpeza diária apaga o que tiver mais de 1 dia:

  📄 `backend/servicos/email_caixa.py`

  ```python
  def _marcar_internet(caminho):
      """Marca «veio da internet» (Mark of the Web)."""
      if os.name != "nt":
          return
      with open(caminho + ":Zone.Identifier", "w", encoding="ascii") as f:
          f.write("[ZoneTransfer]\r\nZoneId=3\r\n")
  ```

**Imagens externas aparecem sempre** — decisão do escritório (21/10/2026), embora o plano inicial
previsse um clique "Mostrar imagens". Saiba o custo: imagem externa avisa ao remetente que o e-mail
foi aberto. Se o seu cliente preferir privacidade, bloqueie `src` externos no DOMPurify e mostre o
botão.

---

← [E-mail — pastas, lista e leitura](pastas-e-leitura.md) · [Índice do guia](../README.md) · [E-mail — escrever e enviar](escrever-e-enviar.md) →
