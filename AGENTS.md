# Diretrizes de Desenvolvimento - Power Platform Skills

Este arquivo fornece orientação para Agentes de IA ao trabalhar com código neste repositório.

## O que é este repositório

Um **marketplace de plugins** para desenvolvimento em Power Platform, mantido pela Microsoft. O manifesto do marketplace Open Plugins (`marketplace.json`) referencia plugins individuais em `plugins/`. Cada plugin tem seu próprio `AGENTS.md` com orientações específicas do plugin.

## Estrutura do Repositório

```
power-platform-skills/
├── marketplace.json          # Manifesto do marketplace Open Plugins (lista todos os plugins disponíveis)
├── .claude-plugin/           # Espelhos de manifesto legado para assinaturas existentes
│   └── marketplace.json
├── plugins/                  # Diretório contendo os plugins individuais
│   └── <plugin-name>/        # Plugin individual (ex.: power-pages)
│       ├── .plugin/
│       │   └── plugin.json   # Manifesto do plugin
│       ├── .claude-plugin/
│       │   └── plugin.json   # Espelho de manifesto legado
│       ├── AGENTS.md         # Diretrizes de desenvolvimento específicas do plugin
│       ├── agents/           # Arquivos de persona de agente
│       ├── commands/         # Pontos de entrada de comandos
│       ├── shared/           # Recursos e documentação compartilhados
│       └── skills/           # Fluxos de skills (SKILL.md em subdiretórios)
├── shared/                   # Recursos compartilhados entre plugins
│   └── skills/               # Definições de skills compartilhadas
│       └── <skill-name>/     # SKILL.template.md + arquivos .md de fluxo
├── AGENTS.md                 # Diretrizes genéricas de desenvolvimento (este arquivo)
└── README.md                 # Visão geral do repositório
```

## Desenvolvimento Local

Teste um plugin localmente iniciando seu agente de IA com o caminho do plugin:

```bash
claude --plugin-dir /path/to/plugins/<plugin-name>
```

Não existem comandos de build, lint ou teste no nível raiz. As ferramentas de build/teste vivem dentro de cada plugin.

## Convenções de Plugin

Cada plugin segue esta estrutura:

- `.plugin/plugin.json` — metadados do Open Plugins (nome, versão, palavras-chave)
- `.claude-plugin/plugin.json` — espelho legado de `.plugin/plugin.json` mantido para assinaturas existentes
- `.mcp.json` — configuração de servidor MCP (opcional)
- `agents/` — definições de agentes (arquivos `.md` com front matter YAML)
- `skills/` — definições de skills, cada uma em seu próprio subdiretório com um `SKILL.md`
- `scripts/` — scripts utilitários compartilhados referenciados por skills e agentes
- `references/` — documentos de referência compartilhados usados por múltiplas skills

Skills são definidas em arquivos `SKILL.md` com front matter YAML (name, description, allowed-tools, model, hooks). O campo `allowed-tools` deve usar uma **lista separada por vírgulas** (ex.: `allowed-tools: Read, Write, Edit, Bash, Glob, Grep`) — não sintaxe de array JSON (`["Read", "Write"]`) nem sintaxe de lista YAML. Cada skill pode incluir scripts de validação em um subdiretório `scripts/`, executados como hooks de Stop quando a sessão da skill termina.

## Skills Compartilhadas Entre Plugins

Skills que se aplicam a todos os plugins vivem em `shared/skills/<skill-name>/`. A lógica do fluxo é escrita uma única vez em um arquivo `.md` compartilhado, e cada plugin tem um `skills/<skill-name>/SKILL.md` enxuto que contém apenas o front matter YAML e uma referência ao caminho do fluxo empacotado dentro daquele plugin no momento da instalação.

**Padrão:**
- `shared/skills/<skill-name>/<workflow>.md` — Fluxo completo (fases, instruções, definições de campos)
- `shared/skills/<skill-name>/SKILL.template.md` — Template de SKILL.md (front matter + referência ao fluxo); suporta o placeholder `{{PLUGIN_NAME}}`
- `plugins/<plugin>/skills/<skill-name>/SKILL.md` — Wrapper por plugin gerado a partir do template acima
- `plugins/<plugin>/skills/<skill-name>/<workflow>.md` — Arquivo de fluxo copiado e empacotado com o plugin para que funcione após instalar apenas o diretório daquele plugin

Isso mantém a skill descobrível em cada plugin preservando a portabilidade no momento da instalação. Instalações via marketplace copiam apenas o diretório do plugin, então os wrappers por plugin não podem referenciar caminhos `shared/` da raiz do repositório em tempo de execução. Em vez disso, aponte o wrapper para `${PLUGIN_ROOT}/skills/<skill-name>/<workflow>.md` e mantenha uma cópia física do fluxo compartilhado nesse caminho por plugin. Não use symlinks do Git para conteúdo compartilhado; instalações no Windows e em plugin-hosts podem materializá-los como arquivos de link simples. Ao atualizar uma skill compartilhada, edite o arquivo de fluxo e/ou `SKILL.template.md` em `shared/`, depois atualize os wrappers por plugin (front matter + referência ao fluxo empacotado, com `{{PLUGIN_NAME}}` substituído) e copie o conteúdo do fluxo para cada plugin adotante. Faça commit da fonte compartilhada e das cópias por plugin juntas.

## Telemetria Compartilhada

O código de telemetria 1DS para todos os plugins vive em `shared/telemetry/`. Cada plugin adotante mantém uma cópia física da biblioteca em sua própria árvore, em `plugins/<plugin>/scripts/lib/telemetry/lib`, junto com o `ikey.json` real daquele plugin. Não use symlinks do Git para essa cópia; os plugin-hosts podem não os dereferenciar de forma confiável.

Edite `shared/telemetry/` primeiro, depois atualize a cópia de `scripts/lib/telemetry/lib` de cada plugin adotante na mesma mudança, para que a fonte canônica e o conteúdo empacotado no plugin permaneçam sincronizados.

**Nunca reutilize a chave de instrumentação ou o event stream de outro plugin.** Ao adotar telemetria em um plugin novo, copie apenas a biblioteca agnóstica de roteamento (`shared/telemetry/lib` → `plugins/<plugin>/scripts/lib/telemetry/lib`) — **não** copie o `ikey.json` real de um adotante existente (nem seu `resolver.js`). O `ikey.json` de cada plugin carrega as chaves de instrumentação, o roteamento do coletor e o `event_stream_name` daquele plugin especificamente; comece a partir do `shared/telemetry/ikey.json` placeholder (toda chave de região é `PLACEHOLDER_REPLACE_BEFORE_SHIPPING` e vem com `disabled: true`) e provisione uma chave nova, específica do plugin, antes de lançar. Copiar uma chave já commitada em outro plugin (ex.: pegar o `ikey.json` do `power-pages` inteiro) atribui erroneamente os eventos do novo plugin ao stream Kusto do outro plugin e o polui — a etapa de cópia deve trazer apenas código de biblioteca, nunca o `ikey.json`/`resolver.js` já provisionado de outro plugin.

Esse invariante é reforçado por CI: `node scripts/validate-telemetry-ikeys.js` (integrado ao workflow `validate-repository-metadata`) varre todo `plugins/*/**/ikey.json`, ignora valores placeholder/vazios, e falha se a mesma chave de instrumentação ou `event_stream_name` aparecer em dois plugins diferentes. Um único plugin reutilizando uma chave entre regiões é permitido; apenas a reutilização entre plugins falha. Execute localmente após alterar o `ikey.json` de qualquer plugin.

O roteamento de iKey/coletor por plugin é conectável via um `resolver.js` colocado ao lado do `ikey.json` do plugin (implementando o contrato `resolve`/`isProvisioned`); a biblioteca compartilhada fornece apenas esse contrato mais um fallback de chave estática, não nenhuma lógica de roteamento. Uma variável de ambiente de opt-out por plugin `POWER_PLATFORM_SKILLS_TELEMETRY_<PLUGIN>_OPTOUT` (derivada como o nome do plugin em maiúsculas com caracteres não alfanuméricos colapsados para `_`, com sufixo `_OPTOUT`) desabilita a transmissão para automação quando definida como `1`/`true` (convenção dotnet `*_TELEMETRY_OPTOUT`); ela tem a **precedência mais alta**, sobrepondo tanto a escolha persistida em `config.json` quanto `/<plugin>:telemetry on`.

### CI precisa desabilitar a transmissão de telemetria (opt-out)

O `ikey.json` commitado de um plugin adotante já vem **habilitado** (`disabled: false`) com uma chave de instrumentação de produção real, então qualquer processo que execute um hook ou script emissor de telemetria **sem isolar a emissão** vai enviar (POST) um evento real (mas com conteúdo fictício) para o coletor de produção. Execuções de CI não são uso real, e esses eventos poluem o stream de telemetria de produção.

**Portanto: todo job do GitHub Actions que executa a suíte de testes — ou qualquer etapa que possa executar um hook/script emissor de telemetria de um plugin adotante — DEVE definir a variável de opt-out do plugin no nível do job (ou workflow).** Para `power-pages`:

```yaml
jobs:
    <job-name>:
        runs-on: <runner>
        env:
            POWER_PLATFORM_SKILLS_TELEMETRY_POWER_PAGES_OPTOUT: "1"
        steps: ...
```

Esse opt-out suprime **apenas a transmissão** (o espelho de diagnóstico local ainda é gravado), então é seguro e não tem efeito sobre o que o job realmente testa. Testes que precisam verificar que a emissão *acontece* limpam a variável no ambiente do processo que eles próprios disparam e roteiam o evento para uma sonda local `POWER_PLATFORM_SKILLS_FAKE_HTTPS`, em vez do coletor real — então o opt-out no nível do job nunca os quebra. Referência existente: `.github/workflows/power-pages-script-tests.yml`. Ao adicionar um novo workflow desse tipo (ou uma nova etapa emissora a um já existente), adicione essa variável de ambiente na mesma mudança; trate um job de CI que rode os testes sem ela como um vazamento de telemetria de produção.

Adotantes atuais: `power-pages`. Outros adotam sob demanda.

## Compatibilidade com Marketplace Legado

Mantenha o `.claude-plugin/marketplace.json` da raiz e o `.claude-plugin/plugin.json` de cada plugin
como espelhos JSON de suas contrapartes no Open Plugins. O marketplace compartilhado da raiz deve
permanecer compatível com ambos os formatos, mantendo as entradas por plugin mínimas: cada entrada
de plugin deve incluir apenas os campos obrigatórios `name` e `source` (relativo à raiz do
repositório). Mantenha `owner` e `metadata` no nível do marketplace porque eles descrevem a coleção,
mas armazene os metadados de exibição/atualização por plugin (description, version, license,
keywords, etc.) no `.plugin/plugin.json` de cada plugin, em vez de duplicá-los ou sobrescrevê-los no
índice do marketplace. Assinaturas de marketplace existentes ainda podem resolver os caminhos legados
durante a auto-atualização, então remover ou deixar esses arquivos divergirem pode forçar os usuários
a reinstalar. Como os espelhos são arquivos commitados (não symlinks), atualize a fonte e as cópias
legadas juntas, depois execute `node scripts/validate-legacy-compatibility.js` após mudanças de
metadados.

## Convenções de Código

**DRY (Don't Repeat Yourself):** Nunca duplique lógica entre arquivos. Cada plugin tem utilitários compartilhados (ex.: `scripts/lib/`) e documentos de referência compartilhados (ex.: `references/`). Sempre verifique se já existem helpers reutilizáveis antes de escrever código novo. Ao adicionar lógica compartilhada, coloque-a nos módulos compartilhados do plugin — não em diretórios de skills individuais.

### Comentários de código

A maior parte do código deste repositório é composta de scripts e hooks Node.js que chamam `pac`/`az`, fazem chamadas às APIs do Dataverse e da Power Platform, e fazem parsing de saída de CLI pouco estruturada. O raciocínio por trás de uma linha raramente é óbvio a partir da linha em si, então comentários importam.

* Prefira comentar demais o código quando o raciocínio não for óbvio. Comentários devem explicar o **PORQUÊ** o código foi escrito de uma forma específica; o **PORQUÊ** é a parte mais importante.
* Comente detalhes de implementação não óbvios: riscos de concorrência, restrições de ciclo de vida, requisitos de compatibilidade, peculiaridades de plataforma, contornos (workarounds) do PAC CLI / Dataverse, e desvios intencionais do helper ou API óbvios.
* Ao fazer parsing de strings, logs, saída de CLI, payloads OData ou outros dados pouco estruturados, inclua um comentário com um exemplo do formato bruto sendo processado. Mostre casos extremos, regras de escape, delimitadores, campos opcionais ou entradas malformadas (mas observadas na prática) quando afetarem o parser.
* Quando o código segue um padrão externo, protocolo ou convenção da Power Platform (códigos de status do Dataverse, formatos de erro OData, contratos de campo de telemetria), inclua links válidos para a fonte relevante no Microsoft Learn ou na especificação, para que leitores futuros possam verificar a regra e entender por que o código a segue.
* Quando o código lida com telemetria, tokens de autenticação, ou qualquer coisa sensível à privacidade/segurança, explique o escopo, o comportamento de opt-in/fail-closed, e **por quê** — não apenas o que faz.
* Não adicione comentários que apenas narram código óbvio, como "define o intervalo" logo antes de atribuir um intervalo.
* Mantenha comentários de workaround próximos ao workaround. Inclua um link de issue quando o workaround estiver ligado a um bug upstream, e descreva a condição para removê-lo quando conhecida.

Bons comentários explicam a restrição ou o trade-off:

```javascript
// `pac auth who` cold-starts the .NET runtime (~4s on Windows), so cache the parsed
// result per process — repeated hook invocations must only fork the CLI once.
let cachedAuth;
```

```javascript
// Refresh the bearer token roughly every 60s instead of on every poll. A long solution
// export outlives the token's lifetime, but refreshing each 5s cycle would hammer the
// az CLI for no benefit.
const tokenRefreshEvery = Math.max(1, Math.floor(60000 / intervalMs));
```

```javascript
// Telemetry must never break the hook it runs inside, so this is fail-closed: a missing
// executable, a timeout, or an unparseable banner all resolve to null rather than throw.
return null;
```

```javascript
// Allowlist-only scrubbing: the event spec already restricts payload fields to values
// that cannot carry PII, so this is a documented seam for a future regex pass — not a
// no-op left unfinished by mistake.
function scrub(value) {
  return value;
}
```

Código que segue um padrão ou convenção externa deve linkar a fonte:

```javascript
// Dataverse asyncoperations terminal states: statecode 3 (Completed) with statuscode 30
// (Succeeded) means done; 31 (Failed) and 32 (Canceled) are the failure terminals.
// See: https://learn.microsoft.com/en-us/power-apps/developer/data-platform/reference/entities/asyncoperation
if (statecode === 3 && statuscode === 30) {
  return { status: 'Succeeded' };
}
```

Mantenha comentários de workaround perto do workaround e linke a issue de rastreamento:

```javascript
// Workaround: `pac solution export` can exit 0 while the Dataverse async job is still
// running, so ignore the exit code and poll asyncoperations to a terminal state instead.
// Remove once the CLI blocks on the job result.
// Tracking: https://github.com/microsoft/power-platform-skills/issues/1234 (use the real issue)
const status = await pollAsyncOperation(asyncJobId, envUrl, token);
```

Comentários de parsing devem mostrar o formato bruto e os casos extremos importantes:

```javascript
// Parse the `pac auth who` banner, a label/value block, e.g.:
//   Authority:    https://login.microsoftonline.com/<tenant>
//   Tenant ID:    00000000-0000-0000-0000-000000000000
//   User:         user@contoso.com
// Values can themselves contain ':' (URLs), so match only up to the first colon after
// the label, then trim. The JSON profile files are intentionally NOT parsed — that
// format is internal and varies across PAC CLI versions.
// `label` is a fixed, code-controlled string (e.g. 'Tenant ID'), so it is safe to
// interpolate into the pattern. If a label ever comes from untrusted input, escape it
// first to avoid regex injection.
const re = new RegExp('^\\s*' + label + '\\s*:\\s*(\\S.*?)\\s*$', 'im');
```

```javascript
// Dataverse OData errors arrive as:
//   { "error": { "code": "0x80040217", "message": "..." } }
// but some PAC surfaces capitalize the envelope as "Error", so check both before
// falling back to plain-text pattern matching.
const odataError = parsed.error || parsed.Error;
```

Evite comentários que apenas repetem o código:

```javascript
// Set the interval to five seconds.
const intervalMs = 5000;

// Loop over the findings.
for (const finding of findings) {
  report(finding);
}
```

## Mantendo Este Arquivo

Ao adicionar novos plugins ou alterar a estrutura no nível do repositório, atualize este arquivo. Para mudanças específicas de um plugin, atualize o `AGENTS.md` daquele plugin (ex.: `plugins/power-pages/AGENTS.md`).

## Documentação Externa

- <a href="https://learn.microsoft.com/en-us/power-pages/configure/create-code-sites">Power Pages Code Sites</a>
- <a href="https://learn.microsoft.com/en-us/power-platform/developer/cli/reference/pages">PAC CLI Reference</a>
- <a href="https://learn.microsoft.com/en-us/rest/api/power-platform/powerpages/websites/create-website">Create Website API</a>
