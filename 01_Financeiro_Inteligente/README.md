#  Módulo 01: Financeiro Inteligente - Automação e ETL


## 💡 Objetivo do Módulo
Este módulo demonstra a prova de conceito do ecossistema ZenithFlow, criando um pipeline end-to-end de dados financeiros com:

- Extração automática do GitHub

- Transformação com Power Query (Linguagem M)

- Visualização em Power BI

- Automação de relatórios via VBA e Power Automate

O objetivo é automatizar o fechamento mensal de múltiplas filiais, consolidando receitas e despesas e gerando saldos e acumulados.

<br>


## ⚙️ Tecnologias e Ferramentas
| Categoria | Ferramenta | Uso no Projeto |
| :--- | :--- | :--- |
| **ETL e Modelagem** | Excel Power Query (Linguagem M) | Extração dos arquivos via links públicos do GitHub, limpeza e modelagem de dados. |
| **Automação** | VBA (Visual Basic for Applications) | Automação do fluxo de trabalho: Atualização das consultas, criação de PDF e distribuição por e-mail. |
| **Orquestração** | Power Automate / Agendador de Tarefas | Possibilita execução automática em horários pré-definidos. |
| **Visualização** | Powe BI | Visualização do relatório e criação de insights do negócio com medidas de time intelligence.
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
│   ├── 01_Financeiro_Modelo_Dados.xlsx
│   └── Dashboard_Financeiro.pdf
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
A consulta `SL_Financeiro` do arquivo mestre é a camada Silver deste projeto onde temos a consulta da camada bronze da etapa anterior e há a tipagem dos dados e o enriquecimento das colunas Saldo, Mês e Ano.

A consulta `GL_Fato_Financeiro`do arquivo mestre é a camada Gold onde temos a tabela fato da etapa Silver resumida e a criação das Foreign Key para as dimensões originadas desta consulta e são elas `DimFilial`, `DimCategoria`, `DimTipo` e `Calendário` também contidas no arquivo mestre.


## **Carga (L):**
Conexão direta do modelo de dados da etapa Gold com a tabela fato, dimensões e calendário no Power BI via Power Query.




##  💻 Automação VBA — Atualização e Distribuição

Macro Run_Update() atualiza todas as consultas, gera PDF do Dashboard e prepara e-mail:
```
Sub Run_Update()
    ThisWorkbook.RefreshAll
    Sheets("Dashboard").ExportAsFixedFormat Type:=xlTypePDF, Filename:=ThisWorkbook.Path & "\Relatorio_Financeiro.pdf"
    ' Abre e-mail com PDF anexado
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
1. **Abrir Arquivo:** Inicie a execução abrindo o arquivo 01_Financeiro_Modelo_Dados.pbix no Power BI Desktop. Este arquivo está configurado para conectar-se ao Financeiro_Mestre_ETL.xlsx, que contém todo o pipeline de dados.

2. **Atualizar:** Clique em Atualizar. O Power BI executará o pipeline completo de forma encadeada:

- BZ: Lê metadados e baixa CSVs.

- SL: Limpa e enriquece.

- GD: Cria as tabelas Fato e Dimensões.

Visualização: O Modelo Dimensional (Schema Estrela) estará pronto para uso.



---


### 🪶 Autor

👩‍💻 Nayara Almeida

[📎 LinkedIn](https://www.linkedin.com/in/nayara-falmeida/) | [GitHub](https://github.com/Nayarah)
