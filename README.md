# Limpeza e Análise de Dados — FIFA 21

Projeto focado em limpeza de dados utilizando SQL no DB Browser for SQLite, com um dataset público do Kaggle contendo dados de jogadores do FIFA 21 raspados do sofifa.com.

O valor desse projeto está principalmente no processo de limpeza — os dados vieram com uma variedade grande de problemas reais de formatação que exigiram soluções diferentes para cada caso.

---

## Dataset

- **Fonte:** Kaggle — FIFA 21 Raw Dataset
- **Origem dos dados:** https://www.kaggle.com/datasets/yagunnersya/fifa-21-messy-raw-dataset-for-cleaning-exploring/data
- **Contexto:** Dataset intencionalmente sujo, típico de dados raspados da web onde cada desenvolvedor front-end estrutura o HTML de forma diferente, tornando os dados de entrada imprevisíveis.

---

## Exploração Inicial

Antes de qualquer limpeza, usei `PRAGMA table_info` para mapear todas as colunas e seus tipos — identificando quais estavam como TEXT quando deveriam ser numéricas, e quais tinham nomes com espaços ou caracteres especiais.

```sql
PRAGMA table_info(fifa21_raw_data);
```

Colunas com nomes problemáticos como `W/O` e `Release Date` foram renomeadas antes da importação para remover espaços e caracteres especiais, já que isso causa problemas em praticamente qualquer ferramenta de análise.

---

## Problemas Encontrados e Soluções

### Valores monetários — Value, Wage e ReleaseClause

Os três campos vinham como texto com símbolo de euro e sufixos `M` (milhões) e `K` (milhares). Por exemplo: `€67.5M`. Foi necessário remover o símbolo, identificar o sufixo e multiplicar pelo valor correspondente.

```sql
CASE
    WHEN Value LIKE '%K%' THEN CAST(REPLACE(REPLACE(Value, 'K', ''), '€', '') AS REAL) * 1000
    WHEN Value LIKE '%M%' THEN CAST(REPLACE(REPLACE(Value, 'M', ''), '€', '') AS REAL) * 1000000
    ELSE CAST(REPLACE(Value, '€', '') AS REAL)
END as Value_convertido
```

O mesmo padrão foi aplicado para Wage e ReleaseClause.

---

### Altura — Height

Vinha no formato americano de pés e polegadas, como `5'9"` ou `5'11"`. Foi necessário extrair os pés e as polegadas separadamente usando INSTR para localizar o apóstrofo e SUBSTR para extrair cada parte, convertendo para centímetros.

```sql
SUBSTR(Height, 1, INSTR(Height, '''') - 1) * 30.48 + 
REPLACE(SUBSTR(Height, INSTR(Height, '''') + 1), '"', '') * 2.54 AS Height_convertido
```

---

### Peso — Weight

Vinha com o sufixo `lbs` em texto. Removido o sufixo e convertido para quilogramas.

```sql
CAST(REPLACE(Weight, 'lbs', '') AS REAL) * 0.453592 AS Weight_convertido
```

---

### Campos com estrelas — WF, SM, IR

Três colunas tinham o caractere `★` junto ao valor numérico. Removido com REPLACE e convertido para número.

```sql
CAST(REPLACE(WF, '★', '') AS REAL) AS WF_convertido
```

O mesmo padrão foi aplicado para SM e IR.

---

### Campos AW e DW

Inspecionados e estavam limpos — valores categóricos de texto como High, Medium e Low sem necessidade de tratamento.

---

### LoanDateEnd

Jogadores sem data de fim de empréstimo tinham o valor `N/A` em vez de nulo. Convertido para NULL para não interferir em análises futuras.

```sql
CASE WHEN LoanDateEnd = 'N/A' THEN NULL ELSE LoanDateEnd END AS LoanDateEnd_convertido
```

---

### Joined

Data de entrada no clube vinha como texto no formato `Jul 1, 2004`. Extraído apenas o ano dos últimos 4 caracteres.

```sql
CAST(SUBSTR(Joined, -4) AS INTEGER) AS Joined_convertido
```

---

### Team_and_Contract

Duas informações diferentes estavam na mesma coluna separadas por um caractere de newline — o nome do time e o período do contrato. Separado em duas colunas usando INSTR com `CHAR(10)` para localizar a quebra de linha.

```sql
SUBSTR(Team_and_Contract, 1, INSTR(Team_and_Contract, CHAR(10)) - 1) AS Teams,
SUBSTR(Team_and_Contract, INSTR(Team_and_Contract, CHAR(10)) + 1) AS ContractPeriod
```

---

## Decisões de Limpeza

**ContractPeriod não foi dividido em ano de início e fim na view.** O campo tem casos especiais como `Free` e `On Loan` que quebrariam a extração de ano. A divisão é feita diretamente nas queries quando necessário, usando `SUBSTR(ContractPeriod, -4)` para o ano final e excluindo os casos de Loan com `NOT LIKE '%Loan%'`.

---

## View Final

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

## Análises

### 1. Jogadores livres ordenados por habilidade

Jogadores sem clube ordenados pelo OVA atual — oportunidades imediatas de mercado.

```sql
SELECT Name, BP, OVA, POT, Teams, ContractPeriod
FROM fifa_clean_data
WHERE ContractPeriod LIKE '%Free%'
ORDER BY OVA DESC
LIMIT 10;
```

---

### 2. Jogadores livres com maior potencial e menos de 24 anos

Jovens sem clube com alto potencial de crescimento — apostas de longo prazo com baixo custo de aquisição.

```sql
SELECT Name, BP, OVA, Age, POT, Teams, ContractPeriod
FROM fifa_clean_data
WHERE ContractPeriod LIKE '%Free%'
AND Age < 24
ORDER BY POT DESC
LIMIT 10;
```

---

### 3. Top 10 jogadores por posição entre os livres

Usando window function para ranquear os melhores de cada posição separadamente.

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

---

### 4. Jogadores com contrato acabando

Jogadores com alto OVA e contrato terminando em 2021 — oportunidades de mercado antes da janela de transferências.

```sql
SELECT Name, BP, OVA, POT, Teams,
SUBSTR(ContractPeriod, -4) AS ContractEnd
FROM fifa_clean_data
WHERE ContractPeriod NOT LIKE '%Loan%'
AND SUBSTR(ContractPeriod, -4) = '2021'
ORDER BY OVA DESC
LIMIT 10;
```

---

## Limitações

O OVA é um número útil para comparar jogadores de forma geral, mas não conta tudo. Habilidades específicas, características de jogabilidade e problemas que não aparecem nos atributos podem ser determinantes na hora de avaliar um jogador de verdade. As análises aqui são um ponto de partida, não uma conclusão definitiva sobre quem contratar.

---

## Ferramentas utilizadas

- DB Browser for SQLite
- Dataset público do Kaggle — FIFA 21 Raw Dataset
