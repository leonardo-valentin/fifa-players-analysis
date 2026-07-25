# Limpeza e Análise de Dados — FIFA 21

Projeto focado em limpeza de dados utilizando SQL no DB Browser for SQLite, com um dataset público do Kaggle contendo dados de jogadores do FIFA 21 raspados do sofifa.com.

O valor desse projeto está principalmente no processo de limpeza — os dados vieram com uma variedade grande de problemas reais de formatação que exigiram soluções diferentes para cada caso. Este projeto foi feito antes de eu começar a estudar Power BI, por isso não há dashboard.

---

# Objetivo

Transformar um dataset intencionalmente "sujo" em uma base analisável, tratando cada tipo de problema de formatação com SQL, e usar essa base limpa para responder algumas perguntas simples de scouting (agentes livres, jovens promessas, contratos vencendo).

---

# Dataset

- **Fonte:** Kaggle — [FIFA 21 Messy Raw Dataset for Cleaning/Exploring](https://www.kaggle.com/datasets/yagunnersya/fifa-21-messy-raw-dataset-for-cleaning-exploring/data)
- **Contexto:** dataset intencionalmente sujo, típico de dados raspados da web, onde cada front-end estrutura o HTML de forma diferente, tornando os dados de entrada imprevisíveis.
- **Registros:** 18.979 jogadores
- **Atributos:** 77 colunas por jogador (dados pessoais, clube/contrato e atributos de jogo)
- **Arquivos brutos:** dois CSVs com os mesmos jogadores em formatos diferentes — `fifa21_raw_data.csv` (usado para carregar o banco) e `fifa21 raw data v2.csv` (mesma base, já com parte dos campos padronizados)

---

# Processo

1. Carga de `fifa21_raw_data.csv` em um banco SQLite (`banco_dados/fifa.db`).
2. Exploração da estrutura da tabela com `PRAGMA table_info(fifa21_raw_data)`, identificando colunas numéricas importadas como TEXT e nomes de coluna problemáticos (espaços, caracteres especiais).
3. Renomeação de colunas com nomes problemáticos (ex: `W/O`, `Release Date`) antes da importação.
4. Tratamento campo a campo dos problemas de formatação (detalhado abaixo), reunidos em uma view de limpeza (`fifa_clean_data`).
5. Exploração da base limpa com queries sobre agentes livres, jovens promessas e contratos perto do fim.

---

# Problemas Encontrados e Soluções

### Valores monetários — Value, Wage e ReleaseClause

Os três campos vinham como texto com símbolo de euro e sufixos `M` (milhões) e `K` (milhares) — ex: `€67.5M`. Necessário remover o símbolo, identificar o sufixo e multiplicar pelo valor correspondente.

```sql
CASE
    WHEN Value LIKE '%K%' THEN CAST(REPLACE(REPLACE(Value, 'K', ''), '€', '') AS REAL) * 1000
    WHEN Value LIKE '%M%' THEN CAST(REPLACE(REPLACE(Value, 'M', ''), '€', '') AS REAL) * 1000000
    ELSE CAST(REPLACE(Value, '€', '') AS REAL)
END as Value_convertido
```

O mesmo padrão foi aplicado para `Wage` e `ReleaseClause`.

### Altura — Height

Formato americano de pés e polegadas (ex: `5'9"`). Extraídos pés e polegadas separadamente com `INSTR`/`SUBSTR` e convertidos para centímetros.

```sql
SUBSTR(Height, 1, INSTR(Height, '''') - 1) * 30.48 +
REPLACE(SUBSTR(Height, INSTR(Height, '''') + 1), '"', '') * 2.54 AS Height_convertido
```

### Peso — Weight

Sufixo `lbs` em texto, removido e convertido para quilogramas.

```sql
CAST(REPLACE(Weight, 'lbs', '') AS REAL) * 0.453592 AS Weight_convertido
```

### Campos com estrelas — WF, SM, IR

Três colunas traziam o caractere `★` junto ao valor numérico. Removido com `REPLACE` e convertido para número (mesmo padrão nas três).

```sql
CAST(REPLACE(WF, '★', '') AS REAL) AS WF_convertido
```

### AW e DW

Inspecionados e já estavam limpos — valores categóricos de texto (`High`, `Medium`, `Low`) sem necessidade de tratamento.

### LoanDateEnd

Jogadores sem data de fim de empréstimo tinham `N/A` em vez de nulo. Convertido para `NULL`.

```sql
CASE WHEN LoanDateEnd = 'N/A' THEN NULL ELSE LoanDateEnd END AS LoanDateEnd_convertido
```

### Joined

Data de entrada no clube em texto (`Jul 1, 2004`). Extraído apenas o ano, a partir dos últimos 4 caracteres.

```sql
CAST(SUBSTR(Joined, -4) AS INTEGER) AS Joined_convertido
```

### Team_and_Contract

Nome do time e período de contrato vinham juntos na mesma coluna, separados por um caractere de newline. Separado em duas colunas usando `INSTR` com `CHAR(10)`.

```sql
SUBSTR(Team_and_Contract, 1, INSTR(Team_and_Contract, CHAR(10)) - 1) AS Teams,
SUBSTR(Team_and_Contract, INSTR(Team_and_Contract, CHAR(10)) + 1) AS ContractPeriod
```

Um efeito colateral interessante dessa separação: para agentes livres, o campo de clube na verdade traz a nacionalidade do jogador (o jogo não atribui um clube a quem está sem contrato) — por isso, nas análises abaixo, a coluna `Teams` mostra o país, não um clube, quando `ContractPeriod` é `Free`.

---

# Decisões de Limpeza

`ContractPeriod` não foi dividido em ano de início e fim dentro da view. O campo tem casos especiais como `Free` e `On Loan` que quebrariam a extração de ano — essa divisão é feita diretamente nas queries quando necessário, com `SUBSTR(ContractPeriod, -4)` para o ano final, excluindo os casos de empréstimo com `NOT LIKE '%Loan%'`.

---

# View Final

```sql
CREATE VIEW fifa_clean_data AS
SELECT Name, LongName, Nationality, Age, BP, OVA, POT,
    CASE
        WHEN Value LIKE '%K%' THEN CAST(REPLACE(REPLACE(Value, 'K', ''), '€', '') AS REAL) * 1000
        WHEN Value LIKE '%M%' THEN CAST(REPLACE(REPLACE(Value, 'M', ''), '€', '') AS REAL) * 1000000
        ELSE CAST(REPLACE(Value, '€', '') AS REAL)
    END as Value_convertido,
    CASE
        WHEN Wage LIKE '%K%' THEN CAST(REPLACE(REPLACE(Wage, 'K', ''), '€', '') AS REAL) * 1000
        WHEN Wage LIKE '%M%' THEN CAST(REPLACE(REPLACE(Wage, 'M', ''), '€', '') AS REAL) * 1000000
        ELSE CAST(REPLACE(Wage, '€', '') AS REAL)
    END as Wage_convertido,
    CASE
        WHEN ReleaseClause LIKE '%K%' THEN CAST(REPLACE(REPLACE(ReleaseClause, 'K', ''), '€', '') AS REAL) * 1000
        WHEN ReleaseClause LIKE '%M%' THEN CAST(REPLACE(REPLACE(ReleaseClause, 'M', ''), '€', '') AS REAL) * 1000000
        ELSE CAST(REPLACE(ReleaseClause, '€', '') AS REAL)
    END as ReleaseClause_convertido,
    SUBSTR(Height, 1, INSTR(Height, '''') - 1) * 30.48 + REPLACE(SUBSTR(Height, INSTR(Height, '''') + 1), '''', '') * 2.54 AS Height_convertido,
    CAST(REPLACE(Weight, 'lbs', '') AS REAL) * 0.453592 AS Weight_convertido,
    CAST(REPLACE(WF, '★', '') AS REAL) AS WF_convertido,
    CAST(REPLACE(IR, '★', '') AS REAL) AS IR_convertido,
    CAST(REPLACE(SM, '★', '') AS REAL) AS SM_convertido,
    CASE WHEN LoanDateEnd = 'N/A' THEN NULL ELSE LoanDateEnd END AS LoanDateEnd_convertido,
    CAST(SUBSTR(Joined, -4) AS INTEGER) AS Joined_convertido,
    SUBSTR(Team_and_Contract, 1, INSTR(Team_and_Contract, CHAR(10)) - 1) AS Teams,
    SUBSTR(Team_and_Contract, INSTR(Team_and_Contract, CHAR(10)) + 1) AS ContractPeriod
FROM fifa21_raw_data;
```

---

# Análises

Do total de 18.979 jogadores, a distribuição por situação contratual é:

| Situação contratual | Jogadores |
| -------------------- | --------: |
| Contrato normal       |    17.728 |
| Empréstimo (Loan)     |     1.013 |
| Agente livre (Free)   |       238 |

### 1. Jogadores livres ordenados por habilidade

```sql
SELECT Name, BP, OVA, POT, Teams, ContractPeriod
FROM fifa_clean_data
WHERE ContractPeriod LIKE '%Free%'
ORDER BY OVA DESC
LIMIT 10;
```

Os agentes livres mais bem avaliados são Welington Dano (LB, 81) e Juiano Mestres (CB, 81), seguidos por um grupo avaliado em 80 (S. Mandíquez, J. Sildero, E. Schetino, entre outros).

### 2. Jogadores livres com maior potencial e menos de 24 anos

```sql
SELECT Name, BP, OVA, Age, POT, Teams, ContractPeriod
FROM fifa_clean_data
WHERE ContractPeriod LIKE '%Free%'
AND Age < 24
ORDER BY POT DESC
LIMIT 10;
```

O destaque é o mesmo Welington Dano (20 anos, POT 81), seguido por M. Mohamed (22 anos, POT 80) e K. Despodov (23 anos, POT 79) — apostas de longo prazo sem custo de contratação.

### 3. Top jogadores por posição entre os livres

```sql
WITH RankedPlayers AS (
    SELECT
        LongName,
        BP,
        OVA,
        Teams,
        ROW_NUMBER() OVER(PARTITION BY BP ORDER BY OVA DESC, LongName ASC) AS PosicaoRank
    FROM fifa_clean_data
    WHERE ContractPeriod LIKE '%Free%'
)
SELECT LongName, BP, OVA, Teams
FROM RankedPlayers
WHERE PosicaoRank <= 10
ORDER BY BP, OVA DESC;
```

O ranking cobre praticamente todas as posições do campo entre os agentes livres, incluindo goleiro (ex: Jorge Ezequiel Serendero, GK, 80).

### 4. Jogadores com contrato acabando em 2021

```sql
SELECT Name, BP, OVA, POT, Teams,
SUBSTR(ContractPeriod, -4) AS ContractEnd
FROM fifa_clean_data
WHERE ContractPeriod NOT LIKE '%Loan%'
AND SUBSTR(ContractPeriod, -4) = '2021'
ORDER BY OVA DESC
LIMIT 10;
```

Entre os contratos (não emprestados) terminando em 2021, os mais bem avaliados são L. Messi (FC Barcelona, 93), Sergio Ramos (Real Madrid, 89) e S. Agüero (Manchester City, 89) — nomes que se tornariam alvo de negociação antes do fim do vínculo.

---

# Limitações

O OVA é útil para comparar jogadores de forma geral, mas não conta tudo. Habilidades específicas, características de jogabilidade e fatores que não aparecem nos atributos podem ser determinantes na avaliação real de um jogador. As análises aqui são um ponto de partida, não uma conclusão definitiva sobre quem contratar.

---

# Ferramentas Utilizadas

- SQL (SQLite) — `banco_dados/fifa.db`
- DB Browser for SQLite — `sql/fifa.sqbpro`
- Dataset público do Kaggle — [FIFA 21 Messy Raw Dataset](https://www.kaggle.com/datasets/yagunnersya/fifa-21-messy-raw-dataset-for-cleaning-exploring/data)

---

# Principais Aprendizados

Durante este projeto pratiquei:

- Diagnóstico de estrutura de tabela com `PRAGMA table_info` antes de qualquer limpeza.
- Limpeza de dados com SQL: conversão de texto para número (`CASE`/`REPLACE`/`CAST`) para valores monetários com sufixos K/M.
- Manipulação de strings (`SUBSTR`/`INSTR`) para separar campos combinados e converter unidades (pés/polegadas → cm, lbs → kg).
- Tratamento de valores ausentes representados como texto (`N/A` → `NULL`).
- Criação de views SQL para centralizar a lógica de limpeza e reutilizá-la em múltiplas queries.
- Uso de funções de janela (`ROW_NUMBER() OVER (PARTITION BY ...)`) para ranquear jogadores dentro de cada posição.
- Leitura crítica dos próprios resultados, reconhecendo os limites de uma métrica (OVA) antes de tratá-la como conclusão.

---
