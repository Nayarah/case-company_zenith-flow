#  Módulo 01: Financeiro Inteligente - Automação e ETL


## 💡 Objetivo do Módulo
Este módulo demonstra a prova de conceito do ecossistema ZenithFlow, criando um pipeline end-to-end de dados financeiros com:

- Extração automática do GitHub

- Transformação com Power Query (Linguagem M)

- Modelagem dimensional e visualização em Power Pivot (com DAX)

- Visualização em Power BI

- Automação de relatórios via VBA e Power Automate

O objetivo é automatizar o fechamento mensal de múltiplas filiais, consolidando receitas e despesas e gerando saldos, KPIs e relatórios dinâmicos — tudo dentro do próprio Excel.

<br>


## ⚙️ Tecnologias e Ferramentas
| Categoria | Ferramenta | Uso no Projeto |
| :--- | :--- | :--- |
| **ETL e Modelagem** | Excel Power Query (Linguagem M) | Extração dos arquivos via links públicos do GitHub, limpeza e modelagem de dados. |
| **Automação** | VBA (Visual Basic for Applications) | Automação do fluxo de trabalho: Atualização das consultas, criação de PDF e distribuição por e-mail. |
| **Orquestração** | Power Automate / Agendador de Tarefas | Possibilita execução automática em horários pré-definidos. |
| **Visualização** | Excel (Tabelas Dinâmicas + Dashboards) | Dashboards interativos criados com base no modelo de dados DAX..
| **Fonte de Dados** | GitHub | Repositório remoto para leitura via Web.Contents(), simulando um ambiente de produção com SharePoint ou DataLake. |

---

## 📁 Estrutura do Projeto

### Arquivos de Entrada (RAW Data)
Os arquivos de entrada são fictícios e simulam um **data lake financeiro**, frequentemente despadronizados e provenientes de diversas fontes, exigindo o tratamento robusto do Power Query.
* **`Dados/Despesas_Filiais/`:** Contém registros de custos operacionais por filial e competência.
* **`Dados/Receitas_Filiais/`:** Contém registros de vendas e receitas de multiplos canais por filial e competência.
* **`Dados/Links_Financeiro.xlsx`:** Um arquivo de metadados que contém as colunas Tipo e Filial. O Power Query utiliza as informações desta tabela para construir dinamicamente os caminhos de acesso aos dados brutos no GitHub (simulando uma tabela de mapeamento)

### Processamento
* **`Dados/01_Financeiro_Mestre_ETL.xlsx`:** Como o nome sugere, um arquivo mestre que contém todo o código M e camadas deste pipeline. O star schema (modelo Fato/Dimensão) é implementado na camada GL dentro deste arquivo.As camadas estão melhor descritos abaixo. Esta centralização do código M visa a otimização da manutenção e auditoria do pipeline sendo uma fonte única do fluxo dos dados.

### Saída (Output)

* **`Relatorios/DashboardExcel.xlsm`:**
Relatório automatizado com:

  -  Dashboards em tabelas dinâmicas conectadas ao modelo Power Pivot.
  -  Cálculos DAX (KPIs, acumulados, time intelligence).
  -  Automação VBA para atualização, validação e envio de  - relatório em PDF por e-mail.
  -  `Relatorios_Gerados/Relatorio_Financeiro_YYYY_MM_DD.pdf`:
Relatório consolidado gerado automaticamente via VBA.

* **`Relatorios/01_Financeiro_Modelo_Dados.pbix`:** Contém a camada Gold conectada via Power Query para visualização do relatório e criação de insights do negócio.

* **Relatório PDF:** Arquivo gerado automaticamente com o *snapshot* do Dashboard.

```

01_Financeiro_Inteligente/
│
├── Dados/
│   ├── Despesas_Filiais/
│   ├── Receitas_Filiais/
│   ├── Financeiro_Mestre_ETL.xlsx  
│   └── Links_Financeiro.xlsx
│
├── Relatorios/
│   ├── DashboardExcel.xlsm
│   ├── 01_Financeiro_Modelo_Dados.xlsx
│   └── Dashboard_Financeiro.pdf
│
├── Relatorios_Gerados/
│   └── Relatorio_Financeiro_2025_11_04.pdf
│
└── README.md

```

---

# 🧩 Estrutura Modular — Pipeline ETL
## Extração (E)  
Nesta etapa temos a camada Bronze na consulta `_BZ_Financeiro_Consolidado` do arquivo mestre, que utiliza as funções abaixo para obtenção de arquivos CSV, faz a combinação destes arquivos, normaliza e cria uma chave match com os caminhos dos arquivos utilizando `Table.NestedJoin`, para enriquecer com as colunas tipo e filial obtendo esta informação de forma confiável da origem dos arquivos.

### 🔧**fnGetFolderContent**
  * A função customizada: fnGetFolderContent foi Criada para possibiliar a obtenção automática de qualquer arquivo inserido na pasta compartilhada do GitHub, staging area, por URL API REST via Web.Contents()


>🔹 Código M da função na seção colapsável abaixo.

<details> <summary>fnGetFolderContent (Power Query)</summary>

```m

// Função customizada: fnGetFolderContent - Criada para possibiliar ler todos os arquivos de uma pasta por URL API REST via Web.Contents()
(caminho as text, BaseApiUrl as text, Branch as text) as table =>
let
    // 1. Constrói a URL da API da pasta, usando os parâmetros de entrada
    FullUrl = BaseApiUrl & caminho & "?ref=" & Branch,  

    // 2. Lê o BINÁRIO da API (Web.Contents)
    Source = Web.Contents(FullUrl),

    // 3. Força o Power Query a tratar esta fonte como "Pública"
    Source_API_Public = Value.ReplaceMetadata(Source, [IsDataSource = true, PrivacySetting = "Public"]),

    // 4. Converte o binário para JSON.
    JsonTable = Json.Document(Source_API_Public),  

    // 5. Transforma a lista de registros JSON em uma tabela
    TableContent = Table.FromList(JsonTable, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    

    // 6. Expande para obter os links de download dos CSVs
    ExpandRecords = Table.ExpandRecordColumn(TableContent, "Column1", {"name", "download_url"}, {"NomeArquivo", "FilePath"}),
    
    // 7. Garante que apenas CSVs sejam processados e adiciona o caminho original
    FilterCSV = Table.SelectRows(ExpandRecords, each Text.EndsWith([NomeArquivo], ".csv")),
    AddCaminhoStaging = Table.AddColumn(FilterCSV, "Origem", each caminho, type text)
in
    AddCaminhoStaging

```
</details>

<br>

### 🔧**fxBZ_ReadCSV**
  * Função Customizada: fxBZ_ReadCSV - fxBZ_ReadCSV: Função de Tratamento de Schema Drift e Tipagem Resiliente.

>🔹 Código M da função na seção colapsável abaixo.

<details><summary>fxBZ_ReadCSV</summary>

```m

/*Função Customizada: fxBZ_ReadCSV - Criada para ler arquivos CSVs em pastas 
sem e apropriar do modelo criado pelo Power Query a partir do primeiro arquivo 
e normalizar as colunas dos arquivos CSV
*/

(filePath as text) as table =>
let
    // 1) Ler CSV local ou remoto
    Fonte =
        if Text.StartsWith(filePath, "http", Comparer.OrdinalIgnoreCase) then
            Csv.Document(
                Web.Contents(filePath),
                [Delimiter = ",", Encoding = 65001, QuoteStyle = QuoteStyle.Csv]
            )
        else
            Csv.Document(
                File.Contents(filePath),
                [Delimiter = ",", Encoding = 65001, QuoteStyle = QuoteStyle.Csv]
            ),

    // 2) Cabeçalhos
    Promoted = Table.PromoteHeaders(Fonte, [PromoteAllScalars = true]),

    // 3) Detectar “receita” vs “despesa” pelos nomes originais
    Cols = Table.ColumnNames(Promoted),
    IsDespesa = List.Contains(Cols, "Tipo de Despesa"),
    IsReceita = List.Contains(Cols, "Receita"),

    // 4) Normalizar: Data, Categoria, Valor, Descrição
    NormalizedRaw =
        if IsDespesa then
            // Despesa: renomeia "Tipo de Despesa" -> "Categoria"
            Table.RenameColumns(Promoted, {{"Tipo de Despesa", "Categoria"}}, MissingField.Ignore)
        else if IsReceita then
            // Receita: Renomeia "Receita" -> "Valor", "Canal de Venda" -> "Descrição"
            Table.RenameColumns(
                Promoted,
                {{"Receita", "Valor"}, {"Canal de Venda", "Descrição"}},
                MissingField.Ignore
            )
        else
            Promoted,

    // 5) Garantir que TODAS as 4 colunas existam (se faltar, cria nula)
    EnsureData = if not List.Contains(Table.ColumnNames(NormalizedRaw), "Data")
                    then Table.AddColumn(NormalizedRaw, "Data", each null, type any) else NormalizedRaw,
    EnsureCategoria = if not List.Contains(Table.ColumnNames(EnsureData), "Categoria")
                    then Table.AddColumn(EnsureData, "Categoria", each null, type text) else EnsureData,
    EnsureValor = if not List.Contains(Table.ColumnNames(EnsureCategoria), "Valor")
                    then Table.AddColumn(EnsureCategoria, "Valor", each null, type number) else EnsureCategoria,
    EnsureDescricao = if not List.Contains(Table.ColumnNames(EnsureValor), "Descrição")
                    then Table.AddColumn(EnsureValor, "Descrição", each null, type text) else EnsureValor,

    // 6) Tratar Data de forma resiliente (tenta converter; se falhar, deixa null)
    DataFixed = Table.TransformColumns(
        EnsureDescricao,
        {{"Data", each try DateTime.FromText(Text.Trim(Text.From(_)), "pt-BR") otherwise null, type datetime}}
    ),

    // 7) Tipa as colunas padronizadas
    Typed = Table.TransformColumnTypes(
        DataFixed,
        {{"Data", type datetime}, {"Categoria", type text}, {"Valor", type number}, {"Descrição", type text}},
        "pt-BR"
    )
in
    Typed
```
</details>

<br>

## **Transformação (T) e Enriquecimento:**
Camada Silver (`SL_Financeiro`) tipa e enriquece os dados com colunas de controle (Saldo, Mês, Ano).
Camada Gold (`GL_Fato_Financeiro`) estrutura o modelo Star Schema, gerando:

- `GL_Fato_Financeiro`

- `DimFilial`

- `DimCategoria`

- `DimTipo`

- `Calendario`


## **Carga (L):**
Carga (L)

O modelo dimensional Gold é carregado no Power Pivot, conectando as Foreign Keys para formar um modelo analítico otimizado.
A partir daí, o DAX entra em ação para criar KPIs e medidas dinâmicas, por exemplo:
```dax
M_LucroLiquido_PA
=IF( 
	HASONEVALUE(Calendario[Date]);
	CALCULATE([M_LucroLiquido]; SAMEPERIODLASTYEAR('Calendario'[Date]));
	BLANK()
)

```

## 📊 Dashboard em Excel com Power Pivot e DAX

A modelagem Star Schema foi aproveitada dentro do próprio Excel, conectando o modelo Power Pivot a tabelas dinâmicas.
Com isso, o Excel se transforma em um ambiente completo de BI corporativo.

🔹 Recursos do Dashboard:

- Modelagem Dimensional (Fato + Dimensões no Power Pivot)

- Cálculos DAX com time intelligence e métricas acumuladas

- Segmentações de Dados interativas e filtros dinâmicos

- Automação VBA de fluxo completo (atualiza, valida, gera PDF e envia por e-mail)

- Interface em múltiplas abas (Dashboard / Filiais / Controle)

### 🧩 Vantagens do Power Pivot + DAX no Excel:

|Vantagem|Descrição|
|:--|:--|
|💡 Integração total|Mesmos cálculos e motor DAX do Power BI.|
|⚡ Performance|O modelo tabular é armazenado em memória e processado via VertiPaq.|
|🔄 Automação|VBA orquestra a atualização, proteção e envio dos relatórios.|
|🧱 Escalabilidade local|Ideal para relatórios internos e financeiros sem dependência do Power BI Service.|

## 💻 Automação VBA — Atualização e Distribuição

O módulo de automação (FluxoCompleto_Orquestrador) executa:

1. Atualização de todas as consultas (ETL Power Query);

2. Validação dos dados;

3. Atualização dos dashboards;

4. Exportação das abas Dashboard e Filiais para PDF;

5. Envio automático do relatório via Outlook.

```vba

Public Sub FluxoCompleto_Orquestrador()
    ThisWorkbook.RefreshAll
    Call ValidarDados_LogErros
    Call AtualizarDashboards
    Call GerarRelatorio_SalvarPDF_Email
End Sub
```



## 🚀 Guia de Execução (*Quick Start*)

Este módulo foi projetado para simular um processo real de fechamento financeiro automatizado, com um clique (ou execução agendada via Power Automate / Task Scheduler).

### Pré-requisitos
* Microsoft Excel (com Power Query e suporte a VBA).
* Conexão com a internet (para leitura dos arquivos hospedados no GitHub).
* Configuração de segurança habilitando:
  - Conteúdo externo (consultas da Web)
  - Execução de Macros (VBA)

### Instruções:
1. Abra o arquivo DashboardExcel.xlsm.

2. Clique no botão Fluxo Completo da aba Controle.

3. Aguarde a atualização e o envio automático do relatório PDF por e-mail.

> (A rotina também pode ser agendada via Power Automate ou Agendador de Tarefas do Windows.)

## ⚖️ Power Pivot vs Power BI — Quando usar cada um
|Critério|Power Pivot (Excel)|Power BI Desktop / Service|
|:---|:---|:---|
|💰 Licenciamento|Incluso no Microsoft 365 (sem custo adicional)|Power BI Pro ou Premium por usuário|
|🧩 Modelagem|Mesmo motor DAX e VertiPaq do Power BI|Idêntico, com recursos adicionais (RLS, aggregations, etc.)|
|📊 Visualização|Tabelas Dinâmicas e gráficos nativos do Excel|Painéis interativos, mapas, drill-downs e custom visuals|
|⚙️ Automação|Controlada via VBA, Power Automate ou Task Scheduler|Atualização e distribuição automática na nuvem|
|🧱 Armazenamento|Local (modelo em cache dentro do Excel)|Cloud-based (Workspaces, Datasets, Gateways)Z
|📤 Distribuição|Manual ou via e-mail automatizado|Compartilhamento e governança via Power BI Service|
|🧮 Escalabilidade|Ideal para relatórios financeiros ou locais|Ideal para dashboards corporativos e colaboração|
|🧰 Manutenção|Total controle pelo analista (VBA + Excel)|Governado por pipelines e Dataflows|
|🚀 Cenário ideal|Pequenas equipes, análises financeiras, protótipos ágeis|Grandes times, governança centralizada e reporting em escala|

💡 Resumo:
Use Power Pivot quando quiser agilidade, autonomia e automação local.
Use Power BI quando precisar de colaboração, governança e escalabilidade em nuvem.


---


### 🪶 Autor

👩‍💻 Nayara Almeida

[📎 LinkedIn](https://www.linkedin.com/in/nayara-falmeida/) | [GitHub](https://github.com/Nayarah)
