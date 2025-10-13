
# ⚡ ZenithFlow: Ecossistema Automatizado de Dados

## 💡 Sobre o Projeto
Este portfólio demonstra a construção de um ecossistema completo de Business Intelligence (BI) e Automação de Dados, modelado sob a empresa fictícia **ZenithFlow**. O objetivo é apresentar soluções de ponta a ponta, desde a ingestão de dados desestruturados até a geração e distribuição automatizada de relatórios gerenciais.

Os dados utilizados são **fictícios** e o repositório é configurado para ser totalmente portátil, utilizando o próprio GitHub como fonte de dados (URLs Raw), ideal para demonstração.

***

### 🌟 Conceito ZenithFlow
O nome combina os seguintes conceitos que guiam o projeto:

| Conceito | Significado | Aplicação no Projeto |
| :--- | :--- | :--- |
| **Zenith** (Zênite) | O ponto mais alto que um astro pode alcançar; sinônimo de auge ou ápice. | Reflete a meta de levar a inteligência de cada departamento (Financeiro, Marketing, Vendas) ao seu **ponto máximo de excelência**. |
| **Flow** (Fluxo) | Fluxo de trabalho ou **automação contínua**. | Representa a metodologia de trabalho utilizada: criação de fluxos de dados automatizados e eficientes para garantir que a informação chegue de forma rápida e precisa ao seu destino. |

---

## 🔗 Sumário Executivo do Ecossistema

Esta é a visão geral dos módulos que compõem o ecossistema ZenithFlow:

| Módulo | Foco Principal | Ferramentas | Status | Detalhes |
| :--- | :--- | :--- | :--- | :--- |
| **01 - Financeiro Inteligente** | ETL, Consolidação Contábil e Distribuição Automatizada. | Power Query (M), VBA, Excel. | **Completo** | [Acessar Módulo](./01_Financeiro_Inteligente/README.md) |
| **02 - Marketing Digital (GA4)** | Extração de dados do Google Analytics (GA4) e *cross-analysis*. | GA4, Power BI / Power Query. | Planejado | [Acessar Pasta](./02_Marketing_Digital_GA4/) |
| **03 - Automação de Vendas** | Integração de CRM e otimização do *pipeline* de vendas. | Python / Power Automate. | Planejado | [Acessar Pasta](./03_Automatizacao_Vendas/) |

---

---
## 🏗️ Estrutura do Repositório
O projeto segue uma arquitetura modular baseada em departamentos:
```
ZenithFlow/
├── 01_Financeiro_Inteligente/   # Contém o código, dados e relatórios do Módulo Financeiro.
│   ├── README.md                # Detalhes técnicos e guia de execução deste módulo.
│   ├── Despesas_Filiais.zip     # Arquivo compactado com dados de despesas (RAW Data).
│   ├── Receitas_Filiais.zip     # Arquivo compactado com dados de receitas (RAW Data).
│   └── Relatorios/              # Contém o arquivo final de dashboard (Ex: .xlsx ou .pbix).
├── 02_Marketing_Digital_GA4/    # Próximo módulo a ser desenvolvido (Futura expansão).
├── 03_Automatizacao_Vendas/     # Módulo futuro.
├── README.md                    # Você está aqui (Visão geral do ecossistema).
└── LICENSE                      # Licença MIT.
```


## 🏭 Considerações de Produção (Ambiente Corporativo)
>**Nota de Escalabilidade:** Este projeto utiliza o GitHub como fonte de dados para fins de demonstração (*Web.Contents*). Em um cenário corporativo real, a arquitetura seria gerenciada via **SharePoint Online** ou Data Lake por razões de segurança e governança de dados. A adaptação da fonte de dados no Power Query seria pontual, garantindo a rápida implementação da solução.


## ⚖️ Licença e Contato

### 👨‍💻 Autor
- **Nome:** Nayara Francisco de Almeida
- **GitHub:** [https://github.com/Nayarah](https://github.com/Nayarah)
- **LinkedIn:** [https://www.linkedin.com/in/nayara-falmeida/](https://www.linkedin.com/in/nayara-falmeida/))
### 🤝 Contribuições
Sinta-se à vontade para abrir **Issues** ou enviar **Pull Requests** se tiver sugestões, melhorias ou encontrar bugs. Todo o feedback é bem-vindo!

### 📄 Licença do Projeto
Este projeto, ZenithFlow, está licenciado sob a **Licença MIT**. Para detalhes completos sobre as permissões e restrições de uso, consulte o arquivo **`LICENSE`** na raiz do repositório.