# Web - sites do IntegraCAR

Dois repositórios de páginas web: o site institucional do projeto, ainda
planejado, e uma versão anterior da página do dataset de uso do solo.

| Repositório | Visibilidade | Situação |
| --- | --- | --- |
| [`integracar-web`](#integracar-web) | Público | Planejado. Só tem README |
| [`LANDING-PAGE-INTEGRACAR-ES`](#landing-page-integracar-es) | Privado | Versão anterior da página do IntegraCAR-LULC-10K |

---

## integracar-web

Site institucional do IntegraCAR, previsto para
[integracar.agr.br](https://integracar.agr.br): informações sobre o projeto, os
parceiros e acesso aos demais módulos (sistema de gestão e dashboard).

**Estado atual:** o repositório só tem o README. Não há código ainda.

Desenho previsto:

- servido no **mesmo servidor** (VPS) do `integracar-gestao` e do
  `integracar-dashboard`, com roteamento pelo nginx;
- estrutura prevista:

```
integracar-web/
├── public/
├── src/
└── README.md
```

Existe hoje uma página de apresentação estática dentro do
`integracar-dashboard` (`src/index.html`, "IntegraCAR · IDAF · IFES"), com os
parceiros e o link do Instagram do projeto. Ela pode servir de ponto de partida
para este repositório.

Ao começar o desenvolvimento, documente aqui: stack escolhida, como rodar
localmente, como é feito o deploy na VPS e qual `location` do nginx atende o
site.

---

## LANDING-PAGE-INTEGRACAR-ES

Página estática do dataset **IntegraCAR-LULC-10K**, em inglês (`index.html`) e
português (`index-pt.html`). É uma versão **anterior** da página que hoje está
no repositório público [`IntegraCAR-LULC-10K`](../lulc/README.md#integracar-lulc-10k-landing-page).
Os arquivos são os mesmos (`index.html`, `index-pt.html`, `script.js`,
`style.css`, `imagensSobre/`, logo e favicon), com estas diferenças:

| Ponto | `LANDING-PAGE-INTEGRACAR-ES` | `IntegraCAR-LULC-10K` |
| --- | --- | --- |
| Seção de modelos (código de treino e pesos) | Não tem | Tem |
| Link do artigo | `paper.pdf` local (arquivo não está no repositório) | Registro do artigo no SIBGRAPI 2026 |
| Links para o Hugging Face e para o `integracar-lulc-builder` | Não tem | Tem |
| README | Não tem | Só o título |

Tecnologia: HTML estático com Bootstrap 5, Bootstrap Icons, fonte Inter e
animações AOS, carregados por CDN. Não há build. Para ver localmente, abra o
`index.html` no navegador ou rode `python -m http.server` na pasta.

**Recomendação:** se a versão do `IntegraCAR-LULC-10K` é a oficial, arquivar
este repositório no GitHub (Settings → Archive) evita que alguém edite a cópia
errada.
