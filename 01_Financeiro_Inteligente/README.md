# 💰 Módulo 01: Financeiro Inteligente - Automação e ETL

## 📚 Sumário
- [💡 Objetivo do Módulo](#-objetivo-do-módulo)
- [⚙️ Tecnologias e Ferramentas](#️-tecnologias-e-ferramentas)
- [📁 Estrutura do Projeto](#-estrutura-do-projeto)
- [🏗️ Pipeline ETL - Power Query](#️-pipeline-etl---power-query)
- [💻 Automação VBA](#-automação-vba)
- [🚀 Guia de Execução](#-guia-de-execução-quick-start)


## 💡 Objetivo do Módulo
Este módulo demonstra a prova de conceito do ecossistema ZenithFlow, criando um pipeline end-to-end de dados financeiros com:

- Extração automática do GitHub

- Transformação com Power Query (Linguagem M)

- Automação de relatórios via VBA e Power Automate

O objetivo é automatizar o fechamento mensal de múltiplas filiais, consolidando receitas e despesas e gerando saldos e acumulados.

<br>

## ⚙️ Tecnologias e Ferramentas
| Categoria | Ferramenta | Uso no Projeto |
| :--- | :--- | :--- |
| **ETL e Modelagem** | Excel Power Query (Linguagem M) | Extração dos arquivos via links públicos do GitHub, limpeza e modelagem de dados. |
| **Automação** | VBA (Visual Basic for Applications) | Automação do fluxo de trabalho: Atualização das consultas, criação de PDF e distribuição por e-mail. |
| **Orquestração** | Power Automate / Agendador de Tarefas | Possibilita execução automática em horários pré-definidos. |
| **Fonte de Dados** | GitHub (Raw Files) | Repositório remoto para leitura via Web.Contents(), simulando um ambiente de produção com SharePoint ou DataLake. |

---

## 📁 Estrutura do Projeto

### Arquivos de Entrada (RAW Data)
Os arquivos de entrada são fictícios e simulam dados reais, frequentemente **despadronizados** e provenientes de diversas fontes, exigindo o tratamento robusto do Power Query.
* **`Despesas_Filiais`:** Contém registros de custos e despesas operacionais.
* **`Receitas_Filiais`:** Contém registros de vendas e receitas por canal/filial.
* **`Links_Financeiro.xlsx`:** lista de URLs para download automátic

### Saída (Output)
* **`Relatorios/Dashboard_Financeiro.xlsx`:** Contém o Modelo de Dados (Tabela Mestra Consolidada) e o Dashboard de visualização, atualizado pela macro.
* **Relatório PDF:** Arquivo gerado automaticamente com o *snapshot* do Dashboard.

```

01_Financeiro_Inteligente/
│
├── Dados/
│   ├── Despesas_Filiais/
│   ├── Receitas_Filiais/
│   └── Links_Financeiro.xlsx
│
├── Relatorios/
│   ├── 01_Financeiro_Modelo_Dados.xlsx
│   └── Dashboard_Financeiro.pdf
└── README.md

```

---

## 🏗️ Pipeline ETL - Estrutura em Power Query

1.  **Extração (E):** Leitura automática dos links públicos hospedados no GitHub, com tratamento de metadados para evitar bloqueio de firewall (PrivacyLevels).

```m
LinkDoCSV = "https://raw.githubusercontent.com/.../links_financeiro.csv"
CSVUrlSegura = Value.ReplaceMetadata(LinkDoCSV, [IsDataSource=true, PrivacySetting="Public"])
ConteudoCsvLinks = Csv.Document(Web.Contents(CSVUrlSegura), [Delimiter=",", Encoding=65001])


```
>🔹 Todo o código M completo está disponível em CODE_SNIPPETS.md
 ou na seção colapsável abaixo.

<details> <summary>Código completo do Pipeline ETL (Power Query)</summary>

```m

// Funções completas de download, expansão, transformação e agregação
AddTipo = Table.AddColumn(ValidLinks, "Tipo", ... )
ReceitasExp = Table.ExpandTableColumn(...)
DespesasExp = Table.ExpandTableColumn(...)
DataExpanded = Table.Combine({ReceitasExp, DespesasExp})
AddFilial = Table.AddColumn(DataExpanded, "Filial", ...)
AddMes = Table.AddColumn(FinalColumns, "Mes", each Date.MonthName([Data]), type text)
AddAno = Table.AddColumn(AddMes, "Ano", each Date.Year([Data]), type number)
AddSaldo = Table.AddColumn(AddAno, "Saldo", each if [Tipo]="Receita" then [Valor] else -[Valor], type number)
AddSaldoAcumulado = Table.AddColumn(AddSaldo, "SaldoAcumulado_Filial", ...)
AddSaldoAcumuladoOrg = Table.AddColumn(AddSaldoAcumulado, "SaldoAcumulado_Org", ...)
FinanceiroResumo = Table.Group(FinanceiroBase, {"Ano","Mes","Tipo","Filial","Categoria"}, ...)

```
</details>

<br>

2. **Transformação (T):**

    * Extrai Tipo (Receita ou Despesa) e Filial

    * Padroniza colunas e formatos

    * Expande linhas de cada arquivo dinamicamente

3. **Enriquecimento:**

    * Cria colunas de Mês e Ano

    * Calcula Saldo, Saldos Acumulados por Filial e Organização

4. **Carga (L):**

    * Financeiro Base: linha a linha detalhada

    * Financeiro Resumo: agregação por mês, categoria e tipo de lançamento



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




---



