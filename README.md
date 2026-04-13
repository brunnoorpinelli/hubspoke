# Hub-Spoke Analítico no Google Cloud

Documentação e materiais de apresentação da arquitetura **Hub-Spoke Analítica** — uma evolução do Data Lake centralizado e do Data Mesh distribuído, aplicando o padrão hub-spoke de redes ao mundo analítico no Google Cloud.

## Estrutura

```
├── docs/
│   └── arquitetura-hubspoke.md     # Documento completo com diagramas Mermaid
├── slides/
│   ├── hubspoke-apresentacao.md    # Deck MARP — apresentação completa (SCQA)
│   ├── hubspoke-apresentacao.html  # HTML interativo — abre no browser
│   ├── hubspoke-apresentacao.pptx  # PowerPoint editável
│   ├── hubspoke-zen.md             # Deck Zen — 10 slides, estilo minimalista
│   ├── hubspoke-zen.html           # HTML interativo — versão Zen
│   └── hubspoke-zen.pptx          # PowerPoint editável — versão Zen
└── assets/
    └── gcloud-theme.css            # Tema MARP com identidade visual Google Cloud
```

## A Arquitetura

O modelo Hub-Spoke Analítico resolve o trade-off entre autonomia de domínio e governança centralizada:

```
Centralizar o que é plataforma. Distribuir o que é domínio.
```

**Hub (projeto central):** ingestão padronizada, governança via Dataplex, marketplace via Analytics Hub (Sharing)

**Spokes (projetos de domínio):** transformações com Dataform + BigQuery, publicação e consumo de produtos de dados certificados

### Stack Google Cloud

`BigQuery` · `Dataplex` · `Analytics Hub (Sharing)` · `Dataflow` · `Dataproc Serverless` · `Pub/Sub` · `Dataform` · `Cloud Composer` · `Cloud IAM` · `Cloud Logging`

## Como Apresentar

Abra qualquer `.html` da pasta `slides/` em qualquer browser:

- Clique em **⛶ Apresentar** para entrar no modo apresentação
- Navegue com `→` / `←` ou `Space`
- `Esc` para sair

## Regenerar os HTMLs

```bash
npm install
node -e "
import('@marp-team/marp-core').then(({Marp}) => {
  const fs = require('fs');
  const marp = new Marp({html:true});
  const css = fs.readFileSync('assets/gcloud-theme.css','utf-8');
  marp.themeSet.add(css);
  // render slides/<deck>.md → slides/<deck>.html
});
"
```

Ou use o Marp CLI com Node.js ≤ 22:

```bash
npx @marp-team/marp-cli@latest slides/hubspoke-apresentacao.md --html --theme assets/gcloud-theme.css -o slides/hubspoke-apresentacao.html
```
