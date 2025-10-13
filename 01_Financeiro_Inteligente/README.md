# 💰 Módulo 01: Financeiro Inteligente - Automação e ETL

## 💡 Objetivo do Módulo
Este módulo é a primeira prova de conceito do ecossistema **ZenithFlow**. Ele demonstra a construção de um pipeline de ETL (Extração, Transformação e Carga) robusto para consolidar e analisar dados de despesas e receitas de múltiplas filiais.

O foco principal é a **automação do fechamento mensal**, eliminando tarefas manuais de consolidação e garantindo a distribuição rápida e precisa de relatórios gerenciais via VBA.

## ⚙️ Tecnologias e Ferramentas
| Categoria | Ferramenta | Uso no Projeto |
| :--- | :--- | :--- |
| **ETL e Modelagem** | Excel (Power Query) | Ingestão, tratamento, enriquecimento e modelagem dos dados financeiros (Linguagem M). |
| **Automação** | VBA (Visual Basic for Applications) | Automação do fluxo de trabalho: Atualização das consultas, criação de PDF e distribuição por e-mail. |
| **Orquestração** | Power Automate Desktop / Task Scheduler | Sugestões para agendamento da execução do arquivo (próximos passos). |
| **Fonte de Dados** | GitHub (Raw Files) | Repositório de dados brutos (*Web.Contents*) para portabilidade e demonstração. |

---

## 📁 Arquivos de Entrada e Saída

### Arquivos de Entrada (RAW Data)
Os arquivos de entrada são fictícios e simulam dados reais, frequentemente **despadronizados** e provenientes de diversas fontes, exigindo o tratamento robusto do Power Query.
* **`Despesas_Filiais.zip`:** Contém registros de custos e despesas operacionais.
* **`Receitas_Filiais.zip`:** Contém registros de vendas e receitas por canal/filial.

### Saída (Output)
* **`Relatorios/Dashboard_Financeiro.xlsx`:** Contém o Modelo de Dados (Tabela Mestra Consolidada) e o Dashboard de visualização, atualizado pela macro.
* **Relatório PDF:** Arquivo gerado automaticamente com o *snapshot* do Dashboard.

---

## 🏗️ Pipeline ETL (Passo a Passo)

O fluxo de processamento de dados é executado inteiramente via Power Query (Linguagem M) dentro do arquivo `Relatorios/Dashboard_Financeiro.xlsx`, seguindo estas etapas:

1.  **Extração (E):** Conexão simultânea aos links Raw do GitHub para `Despesas_Filiais.zip` e `Receitas_Filiais.zip`.
2.  **Transformação (T):**
    * **Limpeza:** Padronização de nomes de colunas e remoção de linhas em branco.
    * **Enriquecimento:** Criação de colunas de ano/mês para granularidade temporal.
    * **Fusão (Append):** As tabelas de Despesas e Receitas são consolidadas em uma única tabela mestra de **Lançamentos Contábeis**.
3.  **Carga (L):** A tabela mestra consolidada é carregada de volta para o Modelo de Dados do Excel, alimentando a Tabela Dinâmica e a automação VBA.

---

## 💻 Guia de Execução (*Quick Start*)

O projeto foi configurado para ser executado com um clique, simulando a experiência do usuário final.

### Pré-requisitos
* Microsoft Excel (versão 2016 ou superior).
* Configuração de Segurança do Excel deve permitir a execução de Macros (VBA).

### Instruções
1.  **Clonar o Repositório:** Baixe o repositório completo do ZenithFlow para sua máquina local.
2.  **Abrir o Arquivo:** Abra o arquivo `01_Financeiro_Inteligente/Relatorios/Dashboard_Financeiro.xlsx`.
3.  **Habilitar Conteúdo:** Ao abrir, **habilite o conteúdo** e **habilite as macros** (se solicitado).
4.  **Executar Macro:**
    * Vá para a guia "Desenvolvedor" (ou onde você inseriu o botão).
    * Clique no botão **`[Run_Update]`** (ou execute a macro `Módulo_Automacao.Run_Update` via VBA).

A macro executará em sequência:
1.  Atualização de todas as consultas Power Query (ETL).
2.  Atualização da Tabela Dinâmica.
3.  Geração de um PDF do Dashboard.
4.  Abertura da janela de e-mail com o PDF anexado, pronto para envio.

---

## 🛠️ Detalhes Técnicos (Power Query M e VBA)

### 1. Conexão e Segurança (Linguagem M)
A conexão com o GitHub Raw é configurada para garantir a portabilidade no portfólio.

*Trecho de Código M (Exemplo de Conexão):*
```m
let
    Link_Raw = "[https://raw.githubusercontent.com/Nayarah/case-company_zenith-flow/main/01_Financeiro_Inteligente/Despesas_Filiais.zip](https://raw.githubusercontent.com/Nayarah/case-company_zenith-flow/main/01_Financeiro_Inteligente/Despesas_Filiais.zip)",
    Fonte = Web.Contents(Link_Raw),
    DadosDespesas = Tabela.FromBinary(Fonte)
in
    DadosDespesas

```

