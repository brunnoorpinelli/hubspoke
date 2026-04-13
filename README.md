# Hub-Spoke Analítico no Google Cloud

Documentação e materiais de apresentação da arquitetura **Hub-Spoke Analítica** — uma evolução do Data Lake centralizado e do Data Mesh distribuído, aplicando o padrão hub-spoke de redes ao mundo analítico no Google Cloud.

## Conteúdo

| Arquivo | Descrição |
|---|---|
| [`arquitetura-hubspoke.md`](arquitetura-hubspoke.md) | Documento completo da arquitetura com diagramas Mermaid |
| [`hubspoke-apresentacao.md`](hubspoke-apresentacao.md) | Deck MARP (fonte da apresentação) |
| [`hubspoke-apresentacao.html`](hubspoke-apresentacao.html) | Apresentação interativa — abre direto no browser |
| [`hubspoke-apresentacao.pptx`](hubspoke-apresentacao.pptx) | Apresentação exportada em PowerPoint editável |

## A Arquitetura

O modelo Hub-Spoke Analítico resolve o trade-off entre autonomia de domínio e governança centralizada:

```
Centralizar o que é plataforma. Distribuir o que é domínio.
```

**Hub (projeto central):** ingestão padronizada, governança via Dataplex, marketplace via Analytics Hub (Sharing)

**Spokes (projetos de domínio):** transformações com Dataform + BigQuery, publicação de produtos de dados certificados

### Stack Google Cloud

`BigQuery` · `Dataplex` · `Analytics Hub (Sharing)` · `Dataflow` · `Dataproc Serverless` · `Pub/Sub` · `Dataform` · `Cloud Composer` · `Cloud IAM` · `Cloud Logging`

## Como Apresentar

Abra `hubspoke-apresentacao.html` em qualquer browser:

- Clique em **⛶ Apresentar** para entrar no modo apresentação
- Navegue com `→` / `←` ou `Space`
- `Esc` para sair

## Regenerar a Apresentação

```bash
npm install
node -e "
const {Marp} = require('@marp-team/marp-core');
const fs = require('fs');
const md = fs.readFileSync('hubspoke-apresentacao.md','utf-8');
const css = fs.readFileSync('gcloud-theme.css','utf-8');
const marp = new Marp({html:true});
marp.themeSet.add(css);
const {html, css:c} = marp.render(md);
// wrap with shell from previous render script
"
```

Ou use o Marp CLI com Node.js ≤ 22:

```bash
npx @marp-team/marp-cli@latest hubspoke-apresentacao.md --html --theme gcloud-theme.css -o hubspoke-apresentacao.html
```
