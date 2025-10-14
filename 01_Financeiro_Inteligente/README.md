# 💰 Módulo 01: Financeiro Inteligente - Automação e ETL

## 📚 Sumário
- [💡 Objetivo do Módulo](#-objetivo-do-módulo)
- [⚙️ Tecnologias e Ferramentas](#️-tecnologias-e-ferramentas)
- [📁 Estrutura do Projeto](#-estrutura-do-projeto)
- [🏗️ Pipeline ETL - Power Query](#️-pipeline-etl---power-query)
- [💻 Automação VBA](#-automação-vba)
- [🚀 Guia de Execução](#-guia-de-execução-quick-start)


## 💡 Objetivo do Módulo
Este módulo é a primeira prova de conceito do ecossistema **ZenithFlow**.
Ele demonstra como criar um pipeline end-to-end de dados financeiros, com extração automática do GitHub, transformação com Power Query (Linguagem M), e automação de relatórios via VBA e Power Automate.

O foco é automatizar o **fechamento mensal de múltiplas filiais** — consolidando receitas e despesas, gerando saldos e acumulados automaticamente, e entregando relatórios prontos para envio.

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

### Saída (Output)
* **`Relatorios/Dashboard_Financeiro.xlsx`:** Contém o Modelo de Dados (Tabela Mestra Consolidada) e o Dashboard de visualização, atualizado pela macro.
* **Relatório PDF:** Arquivo gerado automaticamente com o *snapshot* do Dashboard.

```

01_Financeiro_Inteligente/
│
├── Dados/
│   ├── Despesas_Filiais/
│   │   ├── MG_despesas.xlsx
│   │   ├── SP_despesas.xlsx
│   │   └── RJ_despesas.xlsx
│   ├── Receitas_Filiais/
│   │   ├── filial_MG.xlsx
│   │   ├── filial_SP.xlsx
│   │   └── filial_RJ.xlsx
│   └── Links_Financeiro.xlsx
│
│── Relatorios/
│    ├── 01_Financeiro_Modelo_Dados.xlsx
│    └── Dashboard_Financeiro.pdf
├── README.md 

```

---

## 🏗️ Pipeline ETL - Estrutura em Power Query

1.  **Extração (E):** Leitura automática dos links públicos hospedados no GitHub, com tratamento de metadados para evitar bloqueio de firewall (PrivacyLevels).

```
LinkDoCSV = "https://raw.githubusercontent.com/Nayarah/case-company_zenith-flow/feat/financeiro-inteligente/01_Financeiro_Inteligente/Relatorios/links_fonanceiro.csv",
CSVUrlSegura = Value.ReplaceMetadata(LinkDoCSV, [IsDataSource = true, PrivacySetting = "Public"]),
ConteudoBinario = Web.Contents(CSVUrlSegura),
ConteudoExcel = Excel.Workbook(ConteudoBinario, null, true)

```

2.  **Transformação (T):**
 Criação do pipeline consolidado com extração dinâmica da Filial e Tipo de Lançamento a partir do nome e caminho dos arquivos.

```
// Extração da Filial e Tipo a partir do nome e caminho
AddTipo = Table.AddColumn(DataExpanded,"Tipo", 
    each if Text.Contains([Link], "/Receitas_Filiais/") then "Receita" 
         else if Text.Contains([Link], "/Despesas_Filiais/") then "Despesa" 
         else "Outros", 
    type text),

AddFilial = Table.AddColumn(AddTipo,"Filial",
    each let
        NomeArquivoReal = Text.AfterDelimiter([Link], "/", {0, RelativePosition.FromEnd}),
        Filial = if [Tipo] = "Receita" then
                    Text.Middle(
                        NomeArquivoReal, 
                        Text.PositionOf(NomeArquivoReal, "Filial_") + Text.Length("Filial_"), 
                        Text.PositionOf(NomeArquivoReal, ".xlsx") - (Text.PositionOf(NomeArquivoReal, "Filial_") + Text.Length("Filial_"))
                    )
                 else Text.BeforeDelimiter(NomeArquivoReal, "_")
    in Filial,
    type text)

```
3. **Enriquecimento**Criação de colunas derivadas para granularidade temporal e indicadores financeiros.
```
AddMes = Table.AddColumn(FinalColumns, "Mes", each Date.MonthName([Data]), type text),
AddAno = Table.AddColumn(AddMes, "Ano", each Date.Year([Data]), type number),
AddSaldo = Table.AddColumn(AddAno, "Saldo", each if [Tipo] = "Receita" then [Valor] else -[Valor], type number),
AddSaldoAcumulado = Table.AddColumn(AddSaldo, "SaldoAcumulado_Filial",
    each let
        FilialAtual = [Filial],
        DataAtual = [Data],
        RegistrosFilial = Table.SelectRows(AddSaldo, each [Filial] = FilialAtual and [Data] <= DataAtual),
        Soma = List.Sum(RegistrosFilial[Saldo])
    in Soma, type number),
AddSaldoAcumuladoOrg = Table.AddColumn(AddSaldoAcumulado, "SaldoAcumulado_Org",
    each let
        DataAtual = [Data],
        RegistrosTotais = Table.SelectRows(AddSaldoAcumulado, each [Data] <= DataAtual),
        SomaTotal = List.Sum(RegistrosTotais[Saldo])
    in SomaTotal, type number)

```

4.  **Carga (L): Modelagem Final**

📊 Financeiro Base

Contém todos os registros detalhados (linha a linha) de receitas e despesas com:

- Filial

- Categoria

- Mês / Ano

- Saldo

- Saldos acumulados por filial e totais da organização

📈 Financeiro Resumo

Resumo agregado por mês, categoria e tipo de lançamento, ideal para dashboards e análises gerenciais.

```
FinanceiroResumo = 
    Table.Group(
        FinanceiroBase,
        {"Ano", "Mes", "Tipo", "Filial", "Categoria"},
        {
            {"Total_Receita", each List.Sum(List.Select([Saldo], each _ > 0)), type number},
            {"Total_Despesa", each List.Sum(List.Select([Saldo], each _ < 0)), type number},
            {"Saldo_Liquido", each List.Sum([Saldo]), type number}
        }
    )

```
🧠 Fluxograma do Pipeline


```
GitHub (Raw XLSX)
       │
       ▼
Power Query (Excel)
  ├─ Extrair links (CSV)
  ├─ Baixar planilhas
  ├─ Tratar e padronizar colunas
  ├─ Identificar Tipo e Filial
  ├─ Criar colunas de Mês e Ano
  ├─ Calcular Saldo e Acumulados
  ├─ Gerar tabela Financeiro_Base
  └─ Agregar em Financeiro_Resumo
  ```

  💻 Automação VBA — Atualização e Distribuição

O módulo VBA Módulo_Automacao.bas executa o processo completo de atualização:
```
Sub Run_Update()
    Application.StatusBar = "Atualizando consultas..."
    ThisWorkbook.RefreshAll
    
    Application.StatusBar = "Gerando relatório PDF..."
    Dim PathPDF As String
    PathPDF = ThisWorkbook.Path & "\Relatorio_Financeiro.pdf"
    Sheets("Dashboard").ExportAsFixedFormat Type:=xlTypePDF, Filename:=PathPDF
    
    Application.StatusBar = "Preparando e-mail..."
    Dim OutlookApp As Object, Mail As Object
    Set OutlookApp = CreateObject("Outlook.Application")
    Set Mail = OutlookApp.CreateItem(0)
    Mail.To = "diretoria@zenithflow.com"
    Mail.Subject = "Fechamento Financeiro Mensal"
    Mail.Body = "Segue o relatório financeiro consolidado."
    Mail.Attachments.Add PathPDF
    Mail.Display
    
    Application.StatusBar = False
End Sub



```

---

## 💻 Guia de Execução (*Quick Start*)

Este módulo foi projetado para simular um processo real de fechamento financeiro automatizado, com um clique (ou execução agendada via Power Automate / Task Scheduler).

### Pré-requisitos
* Microsoft Excel (com Power Query e suporte a VBA).
* Conexão com a internet (para leitura dos arquivos hospedados no GitHub).
* Configuração de segurança habilitando:
  - Conteúdo externo (consultas da Web)
  - Execução de Macros (VBA)

### Instruções
1.  **Clonar o Repositório:** Baixe ou clone o projeto completo do GitHub:
`https://github.com/Nayarah/case-company_zenith-flow`
2.  **Abrir o Arquivo:** Abra o arquivo `01_Financeiro_Inteligente/Relatorios/Dashboard_Financeiro.xlsx`.
3.  **Habilitar o Conteúdo e Macros:** 
    * Ao abrir o arquivo, clique em “Habilitar Edição” e “Habilitar Conteúdo”.
    * Certifique-se de que as macros estão permitidas em:
Arquivo > Opções > Central de Confiabilidade > Configurações de Macro
4. **Atualizar as Consultas (ETL)**
    * Acesse a guia “Dados” > “Atualizar Tudo”.
    * O Power Query executará automaticamente a função fnDownloadExcel e construirá:
      * a tabela Financeiro_Base (dados detalhados);
      * e a tabela Financeiro_Resumo (dados consolidados).

5. **Executar a Automação VBA**
    * Vá até a guia Desenvolvedor.
    * Clique no botão [Run_Update], ou execute manualmente a macro:
Módulo_Automacao.Run_Update

6. **Fluxo de Execução da Macro:**
    1. Atualiza todas as consultas Power Query (ETL).
    2. Atualiza o dashboard e as tabelas dinâmicas.
    3. Gera automaticamente o PDF do relatório consolidado.
    4. Abre o e-mail pré-preenchido no Outlook com o PDF anexado.

🧠 Dica Profissional

* Se quiser agendar a execução diária ou semanal:
  * Use o Power Automate Desktop (fluxo “Executar macro no Excel”).
    * Ou o Agendador de Tarefas do Windows com o comando:
    ```
    excel.exe "C:\Caminho\01_Financeiro_Modelo_Dados.xlsx" /mRun_Update

    ```

---



