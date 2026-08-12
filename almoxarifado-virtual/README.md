# Almoxarifado Virtual (MPBA)

App de canvas do **Power Apps** para controle de estoque de um almoxarifado — entradas, saídas,
transferências entre depósitos, catálogo de itens, controle de vencimento/lote, guias de remessa
com assinatura digital e alertas de estoque mínimo.

Todo o app foi desenvolvido em pareamento com o **Claude Code**, conectado ao Power Apps Studio
via um MCP server de autoria de canvas. A seção [Como o Claude Code foi usado](#como-o-claude-code-foi-usado-neste-projeto)
explica esse fluxo em detalhe.

## Sumário

- [O que o app faz](#o-que-o-app-faz)
- [Telas](#telas)
- [Modelo de dados](#modelo-de-dados)
- [Arquitetura e arquivos](#arquitetura-e-arquivos)
- [Como abrir o projeto](#como-abrir-o-projeto)
- [Como o Claude Code foi usado neste projeto](#como-o-claude-code-foi-usado-neste-projeto)
- [Histórico de evolução](#histórico-de-evolução)
- [Peculiaridades conhecidas da plataforma](#peculiaridades-conhecidas-da-plataforma)

## O que o app faz

- **Controle de saldo por depósito e por item**, com estoque mínimo configurável e alerta de
  estoque baixo.
- **Entrada de estoque** (`NovaEntrada`) com múltiplos itens por lançamento, preço de custo,
  fornecedor, depósito de destino e data de validade (mantém a validade mais próxima ao somar
  saldo do mesmo item).
- **Saída de estoque** (`NovaSaida`) com múltiplos itens, motivo, setor solicitante e emissão de
  **guia de remessa com assinatura digital** (captura de assinatura em tela, vinculada a um
  código de guia único e aos dados de quem recebeu).
- **Transferência entre depósitos** (`Transferencia`), preservando lote/validade do saldo movido.
- **Catálogo de itens** (`Catalogo` → `DetalheItem` → `DetalheItemHistorico`) com busca, saldo por
  depósito e histórico de movimentações do item.
- **Gestão de depósitos** (`Depositos` → `DetalheDeposito`), incluindo ajuste manual de saldo
  (sempre com confirmação explicando o impacto) e edição de validade por item/depósito com badge
  de status (OK / vence em breve / vencido).
- **Vencimentos** (`Vencimentos`): lista de saldos vencidos ou vencendo nos próximos 30 dias.
- **Estoque baixo** (`EstoqueBaixo`): itens abaixo do mínimo, com filtro por depósito e atalhos
  diretos para reposição ou transferência.
- **Histórico geral de movimentações** (`Historico`), com opção de reset (mediante confirmação).
- **Cadastros** (`Cadastros`): criação, edição e exclusão de itens, depósitos, setores e
  fornecedores.
- **KPIs na Home**: valor total do estoque (R$), total de itens no catálogo, saldos por status,
  atalhos para as áreas mais usadas.
- **Restrição de acesso** (`AcessoRestrito`): o app é desktop-only — em telas estreitas
  (`App.Width < 900`) mostra uma tela de bloqueio em vez do app.

## Telas

Ordem de navegação real do app (ver `_EditorState.pa.yaml`):

| # | Tela | Arquivo | Função |
|---|------|---------|--------|
| 1 | Acesso Restrito | `AcessoRestrito.pa.yaml` | Bloqueio para telas estreitas (mobile) |
| 2 | Home | `Home.pa.yaml` | Dashboard com KPIs e atalhos |
| 3 | Nova Entrada | `NovaEntrada.pa.yaml` | Registro de entrada de estoque |
| 4 | Nova Saída | `NovaSaida.pa.yaml` | Registro de saída + guia de remessa assinada |
| 5 | Transferência | `Transferencia.pa.yaml` | Transferência entre depósitos |
| 6 | Catálogo | `Catalogo.pa.yaml` | Busca e listagem de itens |
| 7 | Detalhe do Item | `DetalheItem.pa.yaml` | Saldo do item por depósito |
| 8 | Histórico do Item | `DetalheItemHistorico.pa.yaml` | Movimentações de um item específico |
| 9 | Depósitos | `Depositos.pa.yaml` | Listagem de depósitos |
| 10 | Detalhe do Depósito | `DetalheDeposito.pa.yaml` | Saldos, ajuste manual, validade |
| 11 | Estoque Baixo | `EstoqueBaixo.pa.yaml` | Itens abaixo do mínimo |
| 12 | Vencimentos | `Vencimentos.pa.yaml` | Itens vencidos / a vencer |
| 13 | Histórico | `Historico.pa.yaml` | Todas as movimentações |
| 14 | Cadastros | `Cadastros.pa.yaml` | CRUD de itens, depósitos, setores, fornecedores |

## Modelo de dados

O backend são listas do **SharePoint**, referenciadas no app por coleções normalizadas
(`colItens`, `colDepositos`, `colSetores`, `colSaldos`, `colMovimentacoes`, `colFornecedores`)
definidas como fórmulas nomeadas em `App.pa.yaml`:

| Lista SharePoint | Uso no app |
|---|---|
| `Itens` | Catálogo: SKU, nome, categoria, unidade, estoque mínimo, preço de contrato |
| `Depositos_1` | Depósitos físicos e seus responsáveis |
| `Setores` | Setores solicitantes de saída |
| `Fornecedores` | Fornecedores usados em entradas |
| `Saldos` | Saldo por item + depósito (+ lote/validade) |
| `Movimentacoes` | Log de entradas, saídas e transferências |
| `Guias` | Guias de remessa: código, depósito, setor, dados do recebedor e assinatura |

Também há funções Power Fx auxiliares centralizadas em `App.pa.yaml` (ex.: `ItemSaldoTotal`,
`SaldoNoDeposito`, `StatusValidade`, `ValorContratoItem`) para não duplicar lógica de negócio nas
telas.

## Arquitetura e arquivos

```
almoxarifado-virtual/
├── App.pa.yaml                    # Fórmulas globais, paleta de cores, coleções de dados
├── _EditorState.pa.yaml           # Ordem das telas no Studio
├── AcessoRestrito.pa.yaml
├── Home.pa.yaml
├── NovaEntrada.pa.yaml
├── NovaSaida.pa.yaml
├── Transferencia.pa.yaml
├── Catalogo.pa.yaml
├── DetalheItem.pa.yaml
├── DetalheItemHistorico.pa.yaml
├── Depositos.pa.yaml
├── DetalheDeposito.pa.yaml
├── EstoqueBaixo.pa.yaml
├── Vencimentos.pa.yaml
├── Historico.pa.yaml
├── Cadastros.pa.yaml
├── AlmoxarifadoVirtualMPBA.msapp  # Pacote compilado do app (Power Apps)
└── AGENTS.md                      # Guia operacional para agentes de IA continuarem o projeto
```

Os arquivos `.pa.yaml` são o **código-fonte** do app no formato Power Apps YAML (uma tela por
arquivo). O `.msapp` é o pacote binário que o Power Apps Studio importa/exporta. Os dois são
mantidos em sincronia — os `.pa.yaml` são a cópia de segurança legível e versionável do estado do
app.

## Como abrir o projeto

1. Importe `AlmoxarifadoVirtualMPBA.msapp` no [Power Apps Studio](https://make.powerapps.com)
   (**Apps → Importar pacote de canvas**), ou abra a pasta com o [Power Platform CLI](https://learn.microsoft.com/power-platform/developer/cli/introduction)
   (`pac canvas pack`/`unpack`) a partir dos arquivos `.pa.yaml`.
2. As listas do SharePoint referenciadas em `App.pa.yaml` (`Itens`, `Depositos_1`, `Setores`,
   `Fornecedores`, `Saldos`, `Movimentacoes`, `Guias`) precisam existir no seu tenant e estar
   conectadas ao app antes de abrir — o app não cria schema de dados sozinho.

## Como o Claude Code foi usado neste projeto

Este app foi construído inteiramente através de **pareamento com o Claude Code**, sem escrever
YAML ou Power Fx manualmente na maior parte do processo. O fluxo de trabalho:

1. **Sessão viva no Power Apps Studio**: o Claude Code se conecta a uma sessão de coautoria real
   no Studio através de um **MCP server de autoria de canvas** (`mcp__canvas-authoring__*`),
   dedicado a editar apps de canvas do Power Apps. Isso significa que as mudanças pedidas em
   linguagem natural ("adiciona um card de estoque baixo na Home", "cria o fluxo de assinatura na
   guia de saída") eram aplicadas diretamente na sessão que eu via abrir no navegador, tela por
   tela, controle por controle.
2. **Sincronização segura**: antes de qualquer edição, o Claude Code puxa (`sync_canvas`) o estado
   atual da sessão viva para uma pasta temporária e compara com os arquivos locais do projeto —
   para nunca sobrescrever silenciosamente um ajuste manual feito direto no Studio (reposicionar
   um controle, trocar uma cor, etc.).
3. **Edição via `.pa.yaml`**: as telas são editadas como YAML estruturado, o que permite ao Claude
   Code fazer alterações precisas (um controle, uma fórmula, uma cor por vez) e manter tudo
   versionado no Git.
4. **Validação antes do envio**: cada lote de mudanças passa por `compile_canvas`, que valida a
   sintaxe e as fórmulas Power Fx e só então publica de volta na sessão viva do Studio — erros
   bloqueiam o envio, então o que chega na tela sempre compila.
5. **Documentação para continuidade**: o arquivo [`AGENTS.md`](AGENTS.md) registra as
   convenções e armadilhas descobertas ao longo do projeto (ex.: por que `ForAll` com registro
   literal é usado em vez de `ShowColumns`/`AddColumns`, a pegadinha de YAML com `: ` dentro de
   fórmulas, o comportamento de z-order dos controles) — para que qualquer sessão futura do
   Claude Code (ou outra pessoa) consiga continuar o trabalho sem redescobrir as mesmas coisas.
6. **Git como histórico do processo**: cada funcionalidade entregue pelo Claude Code foi commitada
   com `Co-Authored-By: Claude Sonnet 5`, então o [histórico de commits](#histórico-de-evolução)
   do projeto é literalmente o changelog do trabalho feito em conjunto.

Na prática, meu papel foi descrever o que o almoxarifado precisava (telas, regras de negócio,
fluxo de assinatura, KPIs) e revisar o resultado no Studio; o Claude Code cuidou da tradução disso
para Power Fx/YAML, validação e publicação.

## Histórico de evolução

- **Reestruturação inicial** — navegação, cadastros reais, catálogo importado (230 itens),
  correção de bugs de busca, remoção de telas obsoletas (`Dashboard`, `GuiaPedido`), restrição de
  acesso desktop-only.
- **Guias de remessa e controle de vencimento** — tela de Vencimentos, campo de validade em
  Entrada/Depósito com badge de status, fluxo completo de guia de remessa com assinatura digital
  (`PenInput`) gravando na lista `Guias`, combobox pesquisável de item em Entrada/Saída, KPIs
  novos na Home (valor total do estoque, total de itens), filtro por depósito em Estoque Baixo.
- **Consolidação** — sincronização de ajustes feitos direto no Studio de volta para os arquivos
  locais, documentação do fluxo de trabalho em `AGENTS.md`.

Para o detalhe completo de cada mudança, ver `git log -- almoxarifado-virtual/`.

## Peculiaridades conhecidas da plataforma

Documentadas em detalhe em [`AGENTS.md`](AGENTS.md):

- Controles novos inseridos como irmãos de controles existentes podem ser reordenados de forma
  não determinística no primeiro envio.
- Controles mais abaixo na lista `Children:` renderizam por cima dos anteriores (z-order por
  ordem de lista).
- Texto de fórmula com `: ` (dois-pontos + espaço) dentro de uma string quebra o parser YAML —
  exige escalar de bloco (`Text: |-`).
- Este app não usa `ShowColumns`/`AddColumns`/`RenameColumns` (não suportadas pelo serviço de
  compilação usado) — o padrão é `ForAll` com registro literal.
