# Job Finder Automation — Python + APIs (Remotive, RemoteOK, Adzuna, Jooble)

Automação para buscar, filtrar, priorizar e organizar vagas de **Dados / BI / DS Jr**.  
Consulta **APIs oficiais**, aplica filtros (Brasil / RMSP / remoto), remove duplicidades, calcula **score de relevância** e exporta um **Excel formatado** com links diretos para candidatura.

---

## Stack
- Python 3.10+
- Requests, Pandas
- XlsxWriter (export Excel)
- Dotenv (variáveis de ambiente)
- **APIs**: Remotive, RemoteOK, Adzuna, Jooble

---

## Arquitetura (resumo do pipeline)
APIs → Python (Requests + Pandas) → Filtros (área, local, remoto) →  
De-duplicação + Score → Excel com links prontos

---

## Score de relevância (como prioriza)
- **Cargo** citado (Analista de Dados, BI, Data Analyst, DS Jr)
- **Tecnologias** (SQL, Power BI, Python, etc.)
- **Local** (RMSP) ou **Remoto**
- **Recência** (últimos 7 dias com pesos maiores para vagas mais novas)
- **Fonte** (bônus leve para APIs mais confiáveis)

Resultado: as vagas mais aderentes sobem automaticamente no ranking.

---

## Pré-requisitos
- Python 3.10+ instalado
- Chaves das APIs (opcional, mas recomendado):
  - Adzuna: `ADZUNA_APP_ID`, `ADZUNA_APP_KEY`
  - Jooble: `JOOBLE_KEY`

> Remotive e RemoteOK funcionam sem chave. Habilitar Adzuna/Jooble aumenta bastante o volume no Brasil.

---

## Instalação

1. Clone este repositório e entre na pasta:
   ```bash
   git clone https://github.com/<seu-user>/<seu-repo>.git
   cd <seu-repo>
