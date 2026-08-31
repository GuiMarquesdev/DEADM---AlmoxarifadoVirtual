# Almoxarifado Virtual — Guia de Desenvolvimento

Este arquivo documenta como as telas deste app são desenvolvidas via `canvas-authoring` MCP,
para que qualquer sessão (ou pessoa) consiga continuar o trabalho do mesmo jeito.

## O que é este projeto

Um app de canvas do Power Apps para controle de estoque de um almoxarifado (MPBA), com telas
`.pa.yaml` sincronizadas com uma sessão de coautoria viva no Power Apps Studio. Os dados ficam
em listas do SharePoint (Itens, Depositos_1, Setores, Saldos, Movimentacoes, Guias).

## Ferramentas (`mcp__canvas-authoring__*`)

- `connect(environment_id, app_id, login_hint)` — conecta na sessão de coautoria. Precisa ser
  chamado antes de qualquer outra ferramenta, e de novo sempre que a sessão parecer "esquecida"
  (ex.: `list_data_sources` retornando vazio quando não deveria).
- `sync_canvas(directoryPath)` — puxa o estado atual da sessão viva pra uma pasta local.
  **Sobrescreve** qualquer arquivo com o mesmo nome nessa pasta — nunca aponte direto pra pasta
  do projeto, sempre pra uma pasta temporária de scratchpad.
- `compile_canvas(directoryPath)` — valida os `.pa.yaml` da pasta e, se não houver erros, **envia**
  pra sessão viva. Warnings de delegação não bloqueiam o envio; erros bloqueiam.
- `get_data_source_schema(dataSourceName)` — schema (colunas/tipos) de uma fonte de dados.
- `describe_control(controlName)` — propriedades de entrada/saída de um tipo de controle.
- `list_controls()` / `list_data_sources()` / `list_apis()` / `describe_api()` — inventário da sessão.

## Rotina segura de push (sempre, antes de editar)

1. `sync_canvas` pra uma pasta temporária em scratchpad.
2. Comparar (`diff`) cada arquivo sincronizado contra o correspondente no projeto local.
3. Se houver diferença real (não só ruído), **adotar o estado do servidor** copiando o arquivo
   sincronizado por cima do local — nunca sobrescrever silenciosamente uma edição feita direto
   no Studio. Drift comum: reposicionamento manual de controles, cores ajustadas, ícones trocados.
4. Fazer as edições pedidas.
5. `compile_canvas` na pasta real do projeto (não na temporária).
6. Conferir 0 erros (warnings de delegação são normais e esperados — o mesmo conjunto aparece em
   todo compile deste projeto).
7. Apagar a pasta temporária.

## Duas peculiaridades da plataforma (esperadas, não bugs)

- **Reordenação ao criar irmãos novos**: ao inserir um controle novo como irmão de outros já
  existentes (ou uma tela nova na `ScreensOrder`), o primeiro push pode fazer a sessão reordenar
  os irmãos de forma não determinística, ignorando a ordem que foi enviada. Corrigido sincronizando
  de novo, adotando a ordem real que o servidor aplicou (ou reescrevendo a ordem certa) e enviando
  uma segunda vez — a segunda vez já mantém a ordem.
- **Z-order por ordem de lista**: controles depois na lista `Children:` renderizam por cima dos
  anteriores. Uma barra de fundo opaca (ex.: uma `_NavBar`) que acabe posicionada DEPOIS de
  controles interativos no mesmo espaço da tela vai cobri-los visualmente e bloquear cliques.

## Pegadinha de YAML

Qualquer texto de fórmula com `: ` (dois-pontos seguido de espaço) dentro de uma linha
`Propriedade: ="texto: com dois pontos"` quebra o parser (`YamlInvalidSyntax`). Sempre que uma
string de fórmula tiver isso, usar escalar de bloco:

```yaml
Text: |-
  ="Alguma coisa: com dois pontos aqui"
```

## Padrões de Power Fx usados neste app

- **Funções auxiliares nomeadas** em `App.pa.yaml` (`Formulas:`), todas recebendo parâmetros
  explícitos (nunca dependendo de nome de campo ambíguo entre tabelas): `ItemSaldoTotal(sku)`,
  `SaldoNoDeposito(sku, deposito)`, `StatusValidade(dataValidade)`, `ValorContratoItem(sku)`, etc.
- **`ForAll(colecao As Alias, ...)`** pra evitar ambiguidade quando a coleção teria um campo com o
  mesmo nome de um campo usado num `LookUp`/`Filter` aninhado (ex.: `SKU` existe tanto em `Saldos`
  quanto em `Itens`).
- **Coleções normalizadas** (`colItens`, `colDepositos`, `colSetores`, `colSaldos`,
  `colMovimentacoes`) espelham as listas reais do SharePoint com nomes de campo estáveis, porque
  `ShowColumns`/`AddColumns`/`RenameColumns` não são suportadas neste serviço de compilação —
  `ForAll` com registro literal é a alternativa que funciona.
- **Confirmação antes de ações com impacto real**: qualquer ação destrutiva ou que altere dado
  fora do fluxo normal (excluir item, resetar histórico, ajustar saldo manualmente) usa um overlay
  de confirmação explicando o que vai acontecer, nunca executa direto no primeiro clique.

## Telas temporárias de uso único

Para operações em massa (seed inicial de saldo, importação de catálogo, consolidação de estoque
entre depósitos), o padrão é criar uma tela **fora da navegação** com um único botão cuja fórmula
é idempotente (segura pra clicar mais de uma vez — normalmente checando `IsBlank(LookUp(...))`
antes de criar, ou filtrando `Quantidade > 0` antes de mover). O usuário abre a tela direto pelo
seletor de telas do Studio, roda, e a tela é removida depois de confirmado que funcionou.

## Mudanças de schema no SharePoint

Este agente **não tem** como criar listas ou colunas no SharePoint — isso é sempre pedido ao
usuário (nome da coluna, tipo exato a escolher no menu do SharePoint). Depois de criado, o schema
só aparece via `get_data_source_schema` depois que o usuário clicar no ícone de atualizar (⟳) da
fonte de dados no painel **Dados** do Studio — às vezes é preciso pedir pra clicar de novo, ou
reconectar (`connect`) e tentar de novo.

## Se a sessão perder dados (fontes de dados ou conteúdo de telas)

Já aconteceu uma vez: a sessão do Studio perdeu a conexão com todas as fontes de dados e, junto,
o conteúdo (`Children`) de quase todas as telas. Os arquivos locais do projeto são a cópia de
segurança — nunca ficam vazios só porque a sessão viva ficou. Recuperação: confirmar que as fontes
de dados voltaram (`list_data_sources`), depois `compile_canvas` apontando pra pasta real do
projeto local restaura tudo de uma vez. Se aparecer erro em `Property 'Theme'` com um nome não
reconhecido, o app perdeu o recurso de tema; corrigir com `Theme: =Blank()` no `App.pa.yaml`.

## Não usar `PDF()`/`Download()`/`PDFViewer` do Power Apps neste projeto

Já foi tentado e abandonado: gerar um PDF de uma guia de saída usando a função experimental
`PDF()` do Power Fx. Lições, pra não repetir o caminho:

- `PDF(Container)` **não** captura o conteúdo corretamente neste tenant — o resultado é um blob
  que não renderiza em nada (nem erro, nem conteúdo). Testado apontando tanto pra um container
  quanto pra tela inteira, mesmo resultado.
- `Download(PDF(...))` **nunca funcionou** e nunca foi documentado pela Microsoft como uma
  combinação válida — dá erro "A URL passada para a função não é válida". A única forma real de
  "baixar" um blob de `PDF()` seria um controle **Visualizador de PDF (experimental)**, que
  também não renderizou nada neste tenant e cujo toggle de ativação nem aparece na lista de
  recursos experimentais deste ambiente (Configurações → Atualizações → Experimental).
- Telas de canvas são um espaço fixo de pixels — não paginam. Uma lista de itens de tamanho
  variável (uma remessa com muitos itens) não cabe de forma confiável numa "folha" simulada
  dentro de uma tela, e imprimir a tela via `Ctrl+P` do navegador corta as bordas (a tela é mais
  larga que a área imprimível da página).
- **Caminho correto pra isso**: um fluxo do **Power Automate** que recebe os dados da guia (via
  gatilho `Power Apps (V2)`, incluindo os itens como um array serializado com `JSON()`), preenche
  um **modelo do Word** (com um Controle de Conteúdo de Seção Repetida pra tabela de itens —
  paginação de verdade, sem limite de linhas), converte pra PDF e salva numa biblioteca do
  SharePoint. O app só precisa chamar o fluxo (`'NomeDoFluxo'.Run(...)`) e mostrar/abrir o link
  do arquivo devolvido — nada de `PDF()`/`Download()`/`PDFViewer`.
- Isso está fora do que dá pra fazer só com o MCP `canvas-authoring` (que só edita telas do app) —
  exige autoria manual no Word e no Power Automate por fora do Studio.
