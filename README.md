# Análise Integrada de Acidentes Rodoviários Federais (2023 a 2025) com Python

Análise completa dos **213.497 acidentes** registrados pela Polícia Rodoviária Federal
em **2023, 2024 e 2025**, feita em **pandas + Plotly**. O pipeline tem **7 etapas**:
da leitura dos CSVs oficiais até uma síntese em três achados, cada um com hipótese e
limitação explícitas.

O repositório é **autocontido** — dados de origem, bases tratadas e gráficos estão
versionados, então o projeto roda do zero depois do clone, sem nenhuma etapa de download.

---

## Resumo em números

| Métrica | Valor |
|---|---|
| Acidentes registrados | **213.497** (67.766 + 73.202 + 72.529) |
| Acidentes fatais | **15.293** |
| Mortes | **17.833** |
| Taxa global de letalidade | **7,1631%** — 1 acidente fatal a cada 14 registrados |
| Mortes por acidente fatal | 1,17 |
| Acidentes críticos (3+ mortos) | 437 — 0,205% da base, mas **9,36% das mortes** |
| Base analítica | 213.497 × 32 |
| Base modelável | 213.497 × 27 |
| Notebooks | 10, todos executados sem erro |

---

## Etapa 1 — Ingestão e integração

**Notebook:** `notebooks/01_parte1_ingestao.ipynb`

### O que foi feito

Leitura dos três CSVs oficiais da PRF, padronização dos nomes de coluna, conferência de
schema entre os anos e união numa base única — com prova de que nenhuma linha foi
perdida no caminho.

### Como foi feito

- **Leitura com fallback de encoding.** O arquivo é lido tentando `utf-8`, `latin1`,
  `utf-8-sig` e `cp1252`, nessa ordem; só há erro se **todos** falharem. O laço **continua**
  para o próximo encoding em vez de interromper. Na prática `utf-8` falha nos três arquivos
  e `latin1` funciona — o fallback é realmente exercitado, não é código morto.
- **Normalização dos nomes de coluna.** Cada cabeçalho passa por `strip` + `lower`,
  remoção de acentos via normalização Unicode NFKD e substituição de espaço, hífen e
  barra por `_`. O resultado é o mesmo nos três anos, o que permite o `concat` sem
  renomear nada.
- **`id` lido como texto.** O arquivo de 2024 tem uma linha com `id = "6e+05"`
  (notação científica gerada por planilha). Lido como número, o pandas converteria a
  coluna inteira para `float` e a linha perderia o identificador. Com `dtype={"id": "string"}`
  a anomalia fica **visível** em vez de silenciosa — e a linha é preservada.
- **`arquivo_origem`.** Cada linha registra de qual arquivo veio. A redundância com o ano
  extraído da data é proposital: se divergirem, há registro no arquivo errado.
- **Conferência de schema entre os anos.** `assert` de que a interseção dos conjuntos de
  colunas é igual à união — nenhuma coluna existe em um ano e falta em outro.
- **Prova de integridade.** `assert` de que o tamanho do `concat` é exatamente a soma das
  três leituras, verificado **antes** de qualquer remoção de duplicata.

### Dados relevantes

| Item | Valor |
|---|---|
| Linhas por ano | 67.766 (2023) · 73.202 (2024) · 72.529 (2025) |
| Total consolidado | **213.497** |
| Colunas após padronização | 30 (+ `arquivo_origem` = 31) |
| Acidentes fatais | 4.858 + 5.225 + 5.210 = **15.293** |

### Características dos arquivos de origem

| Característica | Valor |
|---|---|
| Separador | `;` |
| Encoding | **latin1** (sem BOM, quebra de linha CRLF) |
| Decimal | vírgula (ex.: `km = "113,1"`) |
| `data_inversa` | **ISO `yyyy-mm-dd`** — atenção: não é `dd/mm/yyyy` |
| Grão | 1 linha = 1 ocorrência de acidente |

**Origem:** [Dados Abertos da PRF](https://www.gov.br/prf/pt-br/acesso-a-informacao/dados-abertos).
MD5 no repositório: `0a7c200814c6bef67f05c86adc73a882` (2023),
`b55ee95f37b24dc2489ce201e1b11ae6` (2024), `49b9b890080023274870a68d326a9e54` (2025).

---

## Etapa 2 — Diagnóstico e limpeza

**Notebook:** `notebooks/02_parte2_limpeza.ipynb`

### O que foi feito

Diagnóstico completo da base consolidada, conversão de tipos, padronização de textos,
validação contra o domínio esperado de cada coluna categórica e remoção das colunas de
localidade fina.

### Como foi feito

- **Diagnóstico.** `info()` com contagem de nulos, contagem de duplicatas exatas e a
  remoção de duplicatas (executada mesmo com zero duplicadas — o notebook reporta
  "0 removidas" explicitamente, para que a ausência de duplicatas fique registrada
  em vez de presumida).
- **Conversão de tipos.** As 12 colunas numéricas passam por substituição de vírgula por
  ponto e `to_numeric`; `data_inversa` é convertida com `format="%Y-%m-%d"`. Um `assert`
  garante que **nenhum valor virou nulo** na conversão.
- **Padronização de textos.** Em todas as colunas de texto: remoção de caracteres
  invisíveis (NBSP, zero-width), `strip` das pontas e compactação de espaços duplos.
  O `upper()` é aplicado apenas aos **identificadores** (`uf`, `br`, `municipio`), que já
  vêm em caixa alta da fonte — a guarda de dtype evita tocar em `br`, que é numérico.
  As colunas **descritivas** (`tipo_acidente`, `causa_acidente`, `fase_dia`,
  `classificacao_acidente`, `dia_semana`, `tipo_pista`, `condicao_metereologica`) mantêm a
  caixa de título publicada pela PRF, que é a forma usada nos rótulos de todo o relatório.
- **Validação de domínio.** Cada coluna categórica é comparada com o domínio esperado
  (`DOMINIOS`). Linha com valor **nulo** ou **fora do domínio** vira `IGNORADO`. O notebook
  registra quantas linhas foram corrigidas em cada coluna e **reconfere depois** que nada
  sobrou, aceitando `IGNORADO` como valor válido.
- **Contagens.** As 8 colunas de contagem de pessoas e veículos são preenchidas com 0 e
  convertidas para `int64`, com `assert` de zero nulos.
- **Remoção de colunas.** Saem `latitude`, `longitude`, `regional`, `delegacia` e `uop` —
  localidade fina, redundante com `uf`/`br`/`municipio` e de baixa cardinalidade. Ficam
  `uf`, `br` e `municipio`. A remoção é checada para não alterar o número de linhas.

### Dados relevantes

| Item | Resultado |
|---|---|
| Nulos na entrada | `uop` 155 · `delegacia` 91 · `regional` 19 · `classificacao_acidente` 3 |
| Duplicatas exatas | **0** (remoção executada, 0 removidas) |
| Espaços duplos corrigidos | 32 (`municipio`) |
| Linhas nulas/inválidas → `IGNORADO` | 3 (`classificacao_acidente`) |
| Não-convertidos em numéricas/data | **0** |
| Colunas após limpeza | 26 |
| Linhas preservadas | **213.497** |
| Mortos | 17.833 |

---

## Etapa 3 — Variáveis derivadas

**Notebook:** `notebooks/03_parte3_variaveis.ipynb`

### O que foi feito

Criação de seis variáveis derivadas que abrem a base para a análise temporal, de
perfil e de risco — todas com validação por `assert`.

### Como foi feito — as fórmulas

| Variável | Fórmula | Validação |
|---|---|---|
| `acidente_fatal` | `1 se mortos >= 1, senão 0` | `assert` de **zero violações** da regra |
| `ano_acidente` | `data_inversa.dt.year` | `assert` de que coincide com `arquivo_origem` |
| `mes_acidente` | `data_inversa.dt.month` | tabela completa ano × mês |
| `fim_de_semana` | `1 se dt.day_of_week ∈ {5, 6} (sábado, domingo)` | `assert` de que coincide com `dia_semana` |
| `data_comemorativa` | `np.select([fim_de_ano, carnaval], [...], default="Normal")` | — |
| `faixa_horaria` | `inicio = (hora // 3) * 3` → `"HHh-HHh"` | `assert` de zero horário inválido |

Detalhes de duas regras:

- **Faixa horária:** blocos de 3 horas, **fechados no início e abertos no fim** — a hora 3
  cai em `03h-06h`, não em `00h-03h`.
- **Data comemorativa:** `Fim de Ano` cobre 20/12 a 02/01; `Carnaval` usa janelas
  específicas por ano (18–22/02/2023, 10–14/02/2024, 01–05/03/2025), porque o feriado
  é mobile e não cai na mesma data a cada ano.

### Dados relevantes

| Janela | Período | Dias | Acidentes |
|---|---|---|---|
| Fim de Ano | 20/12 a 02/01 | 42 | 8.863 |
| Carnaval | janelas por ano | 15 | 2.897 |
| Normal | restante | 1.039 | 201.737 |

| Faixa horária | Acidentes | | Faixa horária | Acidentes |
|---|---|---|---|---|
| 00h-03h | 11.741 | | 12h-15h | 28.421 |
| 03h-06h | 14.792 | | 15h-18h | 36.959 |
| 06h-09h | 33.733 | | 18h-21h | **39.758** |
| 09h-12h | 27.142 | | 21h-24h | 20.951 |

`df_enriquecido`: **213.497 linhas × 32 colunas** · 15.293 fatais (7,16%) · zero violações
da regra do alvo.

---

## Etapa 4 — Duas bases

**Notebook:** `notebooks/04_parte4_bases.ipynb`

### O que foi feito

Geração das duas bases tratadas que alimentam todas as análises seguintes: uma base
analítica completa e uma base modelável sem vazamento de desfecho.

### Como foi feito

- **`base_analitica`** — cópia completa de `df_enriquecido` (213.497 × 32). Serve para
  EDA, gráficos e consumo em BI.
- **`base_modelavel`** — a mesma base **sem** as cinco colunas que só são conhecidas
  **depois** do acidente: `mortos`, `feridos`, `feridos_leves`, `feridos_graves` e
  `classificacao_acidente`. Fica com 213.497 × 27.
- **Verificação de vazamento.** Uma função checa se alguma variável proibida sobreviveu
  na base modelável e **levanta erro** se encontrar.
- **Exportação e reabertura.** As bases são gravadas com `;` e `utf-8-sig`, **reabertas**
  e conferidas com `assert` contra o original: mesmo número de linhas, mesmas colunas
  na mesma ordem e mesmo alvo.

### Dados relevantes

| Base | Linhas | Colunas | Tamanho | Uso |
|---|---|---|---|---|
| `base_analitica_prf_2023_2025.csv` | 213.497 | 32 | 51,5 MB | EDA, gráficos, Power BI |
| `base_modelavel_prf_2023_2025.csv` | 213.497 | 27 | 45,6 MB | modelagem, sem vazamento |

Alvo em ambas: **15.293 fatais (7,16%)**. Nulos na base modelável: **0**.

As bases são **versionadas** no repositório para que as Etapas 5 a 7 rodem sem reexecutar
as Etapas 1 a 4. O notebook as reescreve a cada execução — o conteúdo é idêntico, então
não há divergência entre o que está no Git e o que o código produz.

---

## Etapa 5 — Análises

Quatro níveis, doze perguntas, sobre a base analítica. Todas as tabelas seguem a mesma
regra de leitura:

- **taxa de letalidade** = % de acidentes fatais do grupo, sempre comparada à taxa global
  de **7,1631%**;
- **lift** = taxa do grupo ÷ taxa global (1,00 = média da base);
- **volume mínimo declarado** em cada filtro, com o número de categorias que ficaram de
  fora;
- interpretação de 2–3 linhas por resposta, em linguagem **associativa** — o dado mostra
  concentração, não causa.

| Nível | Foco | Questões | Notebook |
|---|---|---|---|
| 1 | visão geral e temporal | Q1–Q4 | `05_parte5_nivel1.ipynb` |
| 2 | fatores de risco | Q5–Q7 | `06_parte5_nivel2.ipynb` |
| 3 | cruzamentos | Q8–Q10 | `07_parte5_nivel3.ipynb` |
| 4 | casos críticos | Q11–Q12 | `08_parte5_nivel4.ipynb` |

### Nível 1 — visão geral e temporal

**Q1 · Por ano: total de acidentes, total de mortos e taxa de letalidade.**
*Método:* agregação por `ano_acidente` com três somatórios (`size`, `sum` de fatais, `sum`
de mortos); a variação entre os extremos é a diferença entre a última e a primeira taxa.
*Volume mínimo:* não se aplica — todos os anos da base entram (67.766 a 73.202 acidentes).

| Ano | Acidentes | Fatais | Mortos | Taxa | Lift |
|---|---|---|---|---|---|
| 2023 | 67.766 | 4.858 | 5.627 | 7,17% | 1,00 |
| 2024 | 73.202 | 5.225 | 6.163 | 7,14% | 1,00 |
| 2025 | 72.529 | 5.210 | 6.043 | 7,18% | 1,00 |

> **Achado.** A proporção de fatais é **estável**: 7,17% → 7,14% → 7,18%, variação de
> **+0,015 p.p.** entre os extremos, contra a global de 7,16%. Acidentes (67,8–73,2 mil)
> e mortos (5,6–6,2 mil) mantêm a mesma ordem de grandeza.
> *Limitação:* com três observações anuais não é possível afirmar tendência — a série é
> curta demais. O que os dados sustentam é **estabilidade** no período.

**Q2 · Por mês (três anos juntos): qual mês tem a maior taxa de letalidade?**
*Método:* agregação por `mes_acidente` sobre a base inteira; a amplitude é a diferença
entre a maior e a menor taxa.
*Volume mínimo:* nenhum — todos os meses têm mais de 15 mil acidentes.

| Mês | Acidentes | Taxa | Lift | | Mês | Acidentes | Taxa | Lift |
|---|---|---|---|---|---|---|---|---|
| Maio | 17.922 | **7,71%** | 1,08 | | Fevereiro | 15.318 | **6,57%** | 0,92 |
| Junho | 18.083 | 7,68% | 1,07 | | Abril | 17.224 | 6,84% | 0,95 |
| Julho | 18.609 | 7,49% | 1,05 | | Março | 17.515 | 6,85% | 0,96 |
| Agosto | 18.027 | 7,48% | 1,04 | | Outubro | 18.404 | 6,94% | 0,97 |

> **Achado.** Maio tem a maior taxa (**7,71%**) e fevereiro a menor (**6,57%**) —
> amplitude de **1,13 p.p.**, com maio a agosto acima da global (lift 1,04 a 1,08).
> A coincidência com as férias escolares é **parcial** (só julho, em 3º lugar), e o dado
> não permite atribuir a diferença a nenhuma causa específica.

**Q3 · Por fase do dia: como a proporção de fatais se compara — é maior à noite, mesmo
com menos acidentes que de dia?**
*Método:* agregação por `fase_dia`; a razão compara as taxas de Plena Noite e Pleno dia.

| Fase | Acidentes | Fatais | Mortos | Taxa | Lift |
|---|---|---|---|---|---|
| Amanhecer | 10.254 | 1.165 | 1.398 | 11,36% | 1,59 |
| Plena Noite | 73.603 | 7.425 | 8.535 | 10,09% | 1,41 |
| Anoitecer | 11.628 | 778 | 895 | 6,69% | 0,93 |
| Pleno dia | 118.012 | 5.925 | 7.005 | 5,02% | 0,70 |

> **Achado.** Sim, e com folga: `Plena Noite` tem **10,09%** de fatais contra **5,02%** do
> `Pleno dia` — o dobro — e **37,6% menos acidentes** (73.603 contra 118.012), ou seja,
> **menos acidente e mais morte**. `Amanhecer` lidera a taxa (11,36%); as duas fases
> abaixo da global são `Anoitecer` (6,69%, lift 0,93) e `Pleno dia` (5,02%, lift 0,70).

**Q4 · Fim de semana × dias úteis: qual é o risco relativo da taxa de letalidade?**
*Método:* binário `fim_de_semana`; risco relativo = taxa do fim de semana ÷ taxa dos dias
úteis.
*Volume mínimo:* nenhum.

| Grupo | Acidentes | Fatais | Mortos | Taxa | Lift |
|---|---|---|---|---|---|
| Fim de semana | 68.788 | 5.798 | 6.838 | **8,43%** | 1,18 |
| Dias úteis | 144.709 | 9.495 | 10.995 | 6,56% | 0,92 |

> **Achado.** O fim de semana tem **8,43%** de fatais, acima da global de **7,16%**, contra
> **6,56%** dos dias úteis — **risco relativo 1,285** (+28,5%). A diferença está
> **associada a** um padrão que se repete a cada fim de semana, e o dado não separa o
> efeito das características dos acidentes do efeito da exposição (mais quilômetros rodados).

### Nível 2 — fatores de risco

**Q5 · Lift por tipo de acidente: qual eleva mais a chance de morte?**
*Método:* agregação por `tipo_acidente` com filtro de volume; lift = taxa do tipo ÷ global.
*Volume mínimo:* **100 acidentes** — 16 dos 17 tipos são exibidos; fica de fora apenas
`Sinistro pessoal de trânsito`.

| Tipo de acidente | Acidentes | Taxa | Lift |
|---|---|---|---|
| Colisão frontal | 14.233 | **29,81%** | **4,16** |
| Atropelamento de Pedestre | 9.361 | **28,96%** | **4,04** |
| Colisão lateral sentido oposto | 6.200 | 8,73% | 1,22 |
| Eventos atípicos | 871 | 8,15% | 1,14 |
| Atropelamento de Animal | 3.549 | 6,54% | 0,91 |
| … (demais 10 tipos) | — | — | — |
| Incêndio | 4.949 | 0,08% | 0,01 |

> **Achado.** `Colisão frontal` (**29,81%**, lift **4,16**) e `atropelamento de pedestre`
> (**28,96%**, lift **4,04**) têm a taxa mais alta — cerca de **quatro vezes** a global —
> e somam **6.954 dos 15.293** fatais (45%). A queda para o terceiro colocado é abrupta:
> `colisão lateral sentido oposto` fica em 8,73% (lift 1,22). A letalidade **não acompanha
> a frequência**: `colisão traseira` é o tipo mais comum (41.237 casos) e tem lift 0,58.

**Q6 · Top 5 causas por lift. Álcool ou ultrapassagem indevida entram?**
*Método:* agregação por `causa_acidente` com filtro de volume; a coluna de posição lê o
ranking **das 57 causas que passam no filtro** — o recorte que a análise efetivamente
compara. A posição no vocabulário inteiro (75 causas), que inclui as de volume baixo, é
declarada no texto onde for diferente.
*Volume mínimo:* **100 acidentes** — 57 das 75 causas do vocabulário são exibidas.

| Causa | Acidentes | Taxa | Lift | Posição (57 exibidas) |
|---|---|---|---|---|
| Suicídio (presumido) | 438 | 50,91% | 7,11 | 1º |
| Pedestre andava na pista | 1.899 | 41,44% | 5,79 | 2º |
| Entrada inopinada do pedestre | 2.034 | 29,50% | 4,12 | 3º |
| Transitar na contramão | 7.301 | 28,95% | 4,04 | 4º |
| Pedestre cruzava a pista fora da faixa | 1.692 | 25,77% | 3,60 | 5º |
| Ultrapassagem Indevida | 5.093 | 16,59% | 2,32 | 7º |
| Pedestre – ingestão de álcool | 323 | 16,10% | 2,25 | 8º |
| Ingestão de álcool e/ou substâncias psicoativas (pedestre) | 217 | 9,22% | 1,29 | 16º |
| Ingestão de álcool pelo condutor | 11.139 | 4,53% | **0,63** | 38º |

No vocabulário inteiro, sem o filtro de volume, as quatro posições sobem um degrau cada —
`Ultrapassagem` 8º, `Pedestre – ingestão de álcool` 9º, `álcool (pedestre, 217 ac)` 22º e
`álcool (condutor)` 49º —, porque `Participar de racha` (52 acidentes) e outras causas de
volume baixo entram à frente. A conclusão não muda: nenhuma delas está no top 5.

> **Achado.** Álcool e ultrapassagem indevida **não entram no top 5**, mas ambas ficam
> **acima** da média global — o que responde à pergunta de forma mais útil que um
> sim/não. O contraste **dentro** do tema álcool é o achado real: as duas categorias de
> álcool **do pedestre** ficam acima da média (lift 2,25 e 1,29), enquanto a de álcool
> **do condutor** cai para 0,63 — abaixo da média. São categorias distintas do vocabulário
> da base, **associadas a** letalidades diferentes, e o dado não isola o papel da substância
> em nenhuma delas.
> *Ressalva:* `suicídio (presumido)` lidera com folga, mas é uma categoria de **438
> registros** e de natureza distinta — não descreve uma falha de condução. Tratá-la como
> "fator de risco de trânsito" seria erro de interpretação.

**Q7 · Traçado da via: reta ou curva tem taxa pior?**
*Método:* `tracado_via` é uma lista separada por `;`. O tratamento escolhido é
**explode** — cada traçado conta em **todos** os registros que o possuem.
*Volume mínimo:* **mais de 500 acidentes** (≥ 501) — 11 dos 12 traçados são exibidos; fica
de fora apenas `Túnel` (125 acidentes).

| Traçado | Acidentes | Taxa | Lift |
|---|---|---|---|
| Ponte | 2.401 | **10,75%** | 1,50 |
| Declive | 23.349 | 9,85% | 1,38 |
| Aclive | 17.613 | 8,65% | 1,21 |
| Curva | 39.363 | **8,19%** | 1,14 |
| Reta | 154.920 | 7,35% | 1,03 |
| Em Obras | 5.050 | 6,38% | 0,89 |
| Desvio Temporário | 1.002 | 5,49% | 0,77 |
| Viaduto | 3.647 | 3,89% | 0,54 |
| Interseção de Vias | 15.244 | 3,46% | 0,48 |
| Retorno Regulamentado | 5.482 | 3,30% | 0,46 |
| Rotatória | 5.546 | 2,06% | 0,29 |

> **Achado.** **Curva tem taxa pior que reta**, mas a diferença é modesta: **8,19% contra
> 7,35%** (+0,84 p.p.). A resposta mais relevante é que **nem curva nem reta são os piores
> traçados**: `Ponte` lidera com **10,75%** e `Declive` vem em seguida com 9,85%. Os
> traçados de maior taxa são **associados** a rampas, obras e desvios.
> *Consequência declarada:* **53.751 registros (25,2%) têm mais de um traçado**, chegando
> a 8 no mesmo acidente. Com o explode as linhas somam 273.742 contra 213.497 —
> os grupos **não formam partição**, as porcentagens **não somam 100%** e cada taxa deve
> ser lida isoladamente.

### Nível 3 — cruzamentos

**Q8 · Tipo de pista × condição meteorológica: onde está a pior combinação?**
*Método:* `pivot_table` em duas camadas — `size` para a contagem e `mean` para a taxa —
com máscara de volume mínimo aplicada célula a célula; `Ignorado` fica fora da
comparação por não ser uma condição meteorológica.
*Volume mínimo:* **50 por célula** — 21 das 27 combinações são válidas.

| | Chuva | Céu Claro | Garoa | Nevoeiro | Nublado | Sol | Vento |
|---|---|---|---|---|---|---|---|
| **Simples** | 9,60% | 10,15% | 7,50% | **13,95%** | 9,28% | 8,34% | 13,91% |
| **Dupla** | 3,22% | 4,98% | 3,75% | 8,77% | 5,00% | 3,55% | 3,95% |
| **Múltipla** | 3,51% | 4,13% | 3,84% | 7,61% | 4,63% | 3,46% | 4,92% |

As **6 combinações sem taxa** recebem rótulo, e não um `NaN` silencioso, porque são duas
situações distintas:

| Situação | Células | Significado |
|---|---|---|
| **sem casos registrados** | 1 — `Dupla` + `Neve` | a combinação **não ocorre** na base; a taxa seria 0/0, indefinida |
| **sem quantidade mínima de casos** | 5 — `Granizo` e `Neve` nas três pistas | a combinação **existe**, mas tem de 1 a 3 acidentes, abaixo do corte de 50 |

Nas cinco células do segundo grupo há **zero fatais**, então a taxa bruta seria 0,00% —
número que, lido sem contexto, sugeriria "não morre ninguém nessa combinação" quando na
verdade significa "não há amostra suficiente para dizer".

| Margem | Acidentes | Taxa | Lift |
|---|---|---|---|
| Pista Simples | 101.843 | 9,83% | 1,37 |
| Pista Dupla | 88.537 | 4,69% | 0,65 |
| Pista Múltipla | 20.330 | 4,13% | 0,58 |
| Nevoeiro/Neblina | 1.794 | 11,82% | 1,65 |
| Chuva | 20.562 | 6,16% | **0,86** |
| Garoa/Chuvisco | 7.544 | 5,45% | 0,76 |

> **A base destas margens.** As margens de `tipo_pista` acima saem dos **210.710
> acidentes com condição meteorológica conhecida** — os **2.787** com `Ignorado` ficam
> de fora porque `Ignorado` não é uma condição meteorológica, e é justamente o
> cruzamento que exige uma condição real. Na **base completa** (213.497 acidentes) a
> taxa da pista `Simples` é **9,89%** (lift 1,38) e a da `Dupla` é **4,71%** (lift 0,66) —
> são os números da Etapa 7 (adiante), que compara a geometria da via com
> todos os acidentes. Os dois pares são corretos; muda a **população**, e por isso cada
> um é publicado junto da sua base.

> **Achado.** O **tipo de pista domina o resultado**; o clima é secundário. `Simples` tem
> **9,83%** de letalidade contra **4,69%** na `Dupla`, maior nas **7 de 7** condições
> comparáveis. A pior combinação é **Simples + Nevoeiro: 13,95%**. Na margem isolada
> aparece o resultado contraintuitivo: **chuva (6,16%) e garoa (5,45%) têm letalidade
> MENOR que tempo bom** (lift 0,86 e 0,76). A leitura é de **associação**: a base não traz
> velocidade nem exposição.
> *Ressalva:* `Ignorado` ficou fora (2.787 acidentes, 292 fatais, taxa 10,48%, lift 1,46);
> `Granizo` (6 registros) e `Neve` (3) são volumes residuais fora do filtro.

**Q9 · Top 10 BRs em plena noite por número de mortos.**
*Método:* subconjunto `fase_dia == "Plena Noite"`, agregação por `br`, ordenação por
`mortos` decrescente; em paralelo, o mesmo recorte é ordenado por **taxa** para mostrar
que os dois rankings respondem a perguntas diferentes.
`br == 0` — rodovia não informada no boletim — fica fora do ranking: 0 não é uma BR, é a
ausência dela, e como categoria não pode ser priorizada por socorro nem por fiscalização.
*Volume mínimo:* **50 acidentes noturnos** na rodovia — 82 das 118 BRs com registro noturno
passam.

| BR | Acidentes | Mortos | Taxa | Mortos/100 ac. |
|---|---|---|---|---|
| BR-116 | 11.724 | **1.190** | 8,81% | 10,2 |
| BR-101 | 12.533 | **1.076** | 7,75% | 8,6 |
| **BR-316** | 1.524 | 342 | **20,47%** | **22,4** |
| BR-153 | 2.770 | 332 | 10,14% | 12,0 |
| BR-381 | 3.160 | 285 | 8,23% | 9,0 |
| BR-163 | 2.544 | 281 | 8,65% | 11,0 |
| BR-040 | 3.240 | 265 | 7,41% | 8,2 |
| BR-230 | 1.764 | 248 | 12,93% | 14,1 |
| BR-364 | 2.161 | 229 | 9,30% | 10,6 |
| BR-277 | 2.269 | 221 | 8,64% | 9,7 |

> **Achado.** O top 10 por mortos é dominado por **BR-116** e **BR-101**, que juntas
> respondem por **2.266 dos 8.535** mortos noturnos (26,5%); o top 10 completo concentra
> **52,4%** das mortes à noite. As **dez** rodovias da lista têm taxa acima da global de
> 7,16% — a mais próxima é a **BR-040** (7,41%, lift 1,03) —, e a **BR-316** é o caso
> extremo: 20,47%, lift 2,86. Não é um achado, é aritmética: com mais acidentes, a contagem
> absoluta de mortos cresce mesmo quando a taxa fica perto da média — o ranking mede **onde
> as mortes se acumulam**, não onde cada acidente mata mais; as duas listas quase não se
> cruzam (só a BR-316 aparece em ambas), e é essa diferença que dá sentido a publicar as duas.

**Q10 · Fim de Ano, Carnaval e dias normais: a taxa sobe nos feriados?**
*Método:* agregação por `data_comemorativa`; cada período é normalizado por **dias
distintos** com data registrada, o que dá acidentes/dia e mortos/dia; a comparação é
feita contra o grupo `Normal`.
*Volume mínimo:* não se aplica — o volume de cada período já é o total observado.

| Período | Dias | Acidentes | Fatais | Taxa | Lift | Acidentes/dia | Mortos/dia |
|---|---|---|---|---|---|---|---|
| Fim de Ano | 42 | 8.863 | 656 | 7,40% | 1,03 | **211,0** (+8,7%) | **20,21** (+25,2%) |
| Normal | 1.039 | 201.737 | 14.445 | 7,16% | 1,00 | 194,2 | 16,15 |
| Carnaval | 15 | 2.897 | 192 | 6,63% | 0,93 | 193,1 (−0,5%) | 13,67 (−15,4%) |

> **Achado.** Nos três períodos a letalidade é **praticamente a mesma** — Fim de Ano 7,40%
> (lift 1,03), Normal 7,16% e Carnaval 6,63% (lift 0,93), amplitude de apenas 0,77 p.p.
> Não há evidência de que o feriado por si só torne as viagens mais letais. O que muda de
> forma clara é o **volume diário**: o Fim de Ano concentra mais acidentes por dia e,
> sobretudo, **+25,2% de mortos por dia**. O Carnaval tem acidentes por dia praticamente
> iguais aos de um dia normal, mas **15,4% menos mortos por dia**.
> *Ressalvas:* o Carnaval tem só **15 dias** em três anos, o que deixa a taxa sensível a
> poucos casos; e o Fim de Ano inclui 20/12 a 02/01, intervalo em que a exposição viária
> cresce por motivos alheios ao feriado.

### Nível 4 — casos críticos

**Q11 · Acidentes críticos: 3 ou mais mortos.**
*Método:* filtro `mortos >= 3`. O perfil de cada categoria é medido por **razão de
sobre-representação** = % dos críticos ÷ % de todos os fatais; razão acima de 1 significa
categoria sobre-representada entre os críticos.
*Volume mínimo:* não se aplica — o filtro *é* o critério (os 437 acidentes atendem).

| Item | Valor |
|---|---|
| Acidentes com 3+ mortos | **437** — 0,205% da base, 2,86% dos fatais |
| Mortes concentradas | **1.669 — 9,36% de todas as mortes** |
| Média de mortos por crítico | 3,82 (mediana 3) · demais fatais: 1,09 |
| Distribuição | 266 com 3 · 104 com 4 · 67 com 5 ou mais |
| UF que mais concentra | **MG** — 75 acidentes (17,2%), 303 mortos (18,2%) |
| UFs com acidente crítico | 27 |
| Tipo sobre-representado | `Colisão frontal` — 62,93% dos críticos vs 27,74% dos fatais → **razão 2,27** |
| Causa mais sobre-representada | `Ultrapassagem Indevida` → **razão 2,73**; contramão = 27,0% dos críticos |
| Tipo de pista | Simples concentra 88,6% dos críticos (vs 66,9% dos fatais) |
| Fases sobrerrepresentadas | Amanhecer **razão 1,47** · Pleno dia **1,13** · Plena Noite 0,84 |
| Concentração por rodovia | 69 BRs registraram críticos; o top 10 concentra **49,2%** das mortes críticas |
| Dispersão geográfica | 334 municípios |

> **Achado.** Os acidentes críticos são **raros e concentrados**: 437 ocorrências, 100%
> fatais por construção, que respondem por 2,86% dos acidentes fatais mas por **9,36% de
> todas as mortes**. **Minas Gerais** lidera, e a principal causa é **transitar na
> contramão**. A concentração crítica está **associada a** rodovia, não a localidade — os
> 437 acidentes se distribuem por 334 municípios.
> *Ressalva:* o maior evento do período — colisão lateral sentido oposto em TEOFILO OTONI
> (MG), BR-116, com 54 pessoas envolvidas e carga excessiva — levou **37 mortos**. O
> acidente crítico tem ainda perfil de **massa**: mediana de 7 pessoas envolvidas
> (3 nos demais fatais, 2 na base toda).

**Q12 · Top 5 municípios de Pernambuco em 2024 e 2025 por acidentes fatais.**
*Método:* filtro `uf == "PE"` e `ano_acidente ∈ {2024, 2025}`; agregação por `municipio`;
ordenação por **acidentes fatais**, com desempate por mortos e depois por volume, para a
ordem ser determinística. O mesmo recorte é ordenado por mortos como contraponto.
*Volume mínimo:* **50 acidentes** no município — 64 dos 90 municípios são excluídos.

| Município | Acidentes | Fatais | Mortos | Taxa | Mortos/100 ac. |
|---|---|---|---|---|---|
| RECIFE | 1.276 | **45** | 47 | 3,53% | 3,7 |
| GARANHUNS | 186 | **24** | 27 | **12,90%** | 14,5 |
| JABOATAO DOS GUARARAPES | 465 | **24** | 25 | 5,16% | 5,4 |
| CARUARU | 409 | **24** | 25 | 5,87% | 6,1 |
| PETROLINA | 559 | **23** | 25 | 4,11% | 4,5 |

Em PE no biênio: **6.245 acidentes**, 590 fatais e **649 mortos** — taxa de **9,45%**,
acima da global de 7,16%. O top 5 soma **140 fatais (23,7% dos de PE)**, 149 mortos e
2.895 acidentes (46,4% do total).

> **Achado.** O top 5 por **acidentes fatais** é Recife, Garanhuns, Jaboatão, Caruaru e
> Petrolina; **o mesmo conjunto sai pelo ranking por mortos** (149 mortes, 23,0% das mortes
> de PE), então a resposta não muda. O que separa o grupo é a **intensidade**, não o
> volume: o resultado está **associado a** Garanhuns (**12,90%** de taxa), enquanto Recife
> fica em 3,53%.
> *Ressalva metodológica:* o filtro de volume exclui **SALOA**, que tem 13 acidentes e
> **20 mortos** — quase todos em um único evento de 17/10/2025 (16 mortos, BR-423). Ela não
> entraria no top 5 por fatais, mas entraria em 6º pelo ranking por mortos. É a fragilidade
> do ranking por mortes em séries curtas: **um evento pode mover um município inteiro**.

---

## Etapa 6 — Visualizações

**Notebook:** `notebooks/09_parte6_graficos.ipynb`

Três gráficos Plotly em `resultados/`. Cada um responde a uma pergunta escrita na célula
de texto imediatamente acima, tem a **mensagem principal no título** e uma **legenda de
2 linhas** com o que está sendo medido e como ler. Os caminhos são relativos (`Path`), e
nada é escrito fora de `resultados/`.

| Gráfico | Pergunta | Mensagem principal | Arquivo |
|---|---|---|---|
| 1 | Que tipos de acidente elevam a chance de morte? | Frontal (29,81%) e atropelamento de pedestre (28,96%) têm taxa **4× a global**; volume mínimo de 100 acidentes | `g1_taxa_por_tipo_acidente.html` |
| 2 | A taxa de letalidade varia ao longo dos 36 meses? | Oscila só **1,92 p.p.** (6,36% a 8,28%) e **sem tendência** — 12 primeiros e 12 últimos meses com a mesma média | `g2_serie_mensal.html` |
| 3 | Que horas do dia concentram os acidentes fatais? | **03h–06h** tem a taxa mais alta (12,26%), 2,5× a mais segura, com apenas 6,9% dos acidentes | `g3_faixa_horaria.html` |

**Como foram construídos:**

- **Gráfico 1** — barras horizontais de taxa de letalidade por tipo, coloridas por posição
  em relação à global (acima em vermelho, abaixo em cinza), com linha tracejada marcando
  a taxa global. Volume mínimo de 100 acidentes: 16 dos 17 tipos são exibidos.
- **Gráfico 2** — **volume em barras** (eixo à direita) e **taxa em linha** (eixo à
  esquerda), com a taxa global marcada por tracejado. A amplitude é a diferença entre
  máximo e mínimo mensal; a ausência de tendência é testada comparando a média dos 12
  primeiros meses com a dos 12 últimos (7,16% contra 7,17%), com o volume variando de
  4.751 a 6.788 acidentes por mês.
- **Gráfico 3** — barras de volume por faixa de 3h e linha da taxa, com a razão
  pior/melhor calculada (2,50×) e o pico de volume (18h–21h, 39.758 acidentes, 18,6% da
  base, taxa 8,80%) explicitado no subtítulo.

A separação entre **volume** e **gravidade** é justamente o que esses gráficos tornam
visível: a faixa com mais acidentes não é a mais letal.

**Ordem de desenho (G2 e G3).** Volume em barra e taxa em linha dividem o mesmo par de
eixos, e o Plotly desenha o `barlayer` **depois** do `scatterlayer` — com os padrões da
biblioteca a linha some atrás das barras. Por isso a barra recebe `zorder=0` e a linha
`zorder=2` (atributo disponível desde o Plotly **5.21**). A linha da taxa global é uma
*shape*: ela nasce na `shapelayer`, que também fica atrás das barras, e só aparece por
cima com `layer="above"` — aplicado também nos gráficos 1 e 4.

Os HTMLs são **autocontidos** (abrem offline) e estão versionados em `resultados/`. O
PNG é opcional — depende de `kaleido` e do Chrome; sem eles o notebook salva só o HTML e
avisa no output, sem falhar.

---

## Etapa 7 — Síntese

**Notebook:** `notebooks/10_parte7_sintese.ipynb`

Três achados, cada um com observação, onde aparece, comparação com a taxa global,
hipótese e limitação. **Todo número é recalculado no notebook**, a partir da base
analítica, para que ele seja autossuficiente.

| # | Achado | Observação | Onde | Lift | Limitação |
|---|---|---|---|---|---|
| 1 | Colisão frontal e atropelamento de pedestre concentram a letalidade | 29,81% e 28,96%: 11,05% dos acidentes, **45,47% dos fatais** | Gráfico 1 + Q5/Q6 | **4,16** e **4,04** | classificação do BO, subnotificação, sem velocidade |
| 2 | A madrugada tem a taxa mais alta e o volume mais baixo | 03h–06h: 6,9% dos acidentes e **11,9% dos fatais**; 2,50× a faixa mais segura | Gráfico 3 + Q3 | **1,71** (mais segura: 0,69) | horário é o do registro, sem exposição por km |
| 3 | O problema é de geometria de via, não de sazonalidade | Pista Simples 9,89% contra Dupla 4,71%; contramão 6,1× mais frequente; 36 meses sem tendência | Gráfico 2 + Q8/Q11 | **1,38** (Dupla: 0,66) | `tipo_pista` é proxy, confundido com a BR |

> **Base do Achado 3.** O contraste de pista aqui usa a **base completa** (213.497
> acidentes): `Simples` 9,89% (lift 1,38) contra `Dupla` 4,71% (lift 0,66). É um número
> diferente do **9,83%** da margem da Q8 (acima), que exclui os 2.787 acidentes com clima
> `Ignorado` — lá o clima é condição de entrada do cruzamento. Os dois são corretos; a
> diferença é a **população**, declarada nos dois pontos.

**As hipóteses, em uma linha cada:**

1. Nos dois tipos o choque é frontal e **sem espaço de escape** — na frontal os veículos
   se encaram sem para onde recuar; no atropelamento a vítima não tem proteção alguma.
   Energia alta somada a ausência de proteção produz lift equivalente por dois mecanismos
   diferentes, o que reforça que o determinante é a *severidade do impacto*.
2. Três componentes se somam na madrugada: **visibilidade** ruim, **fadiga ou sono** ao
   volante e **tráfego mais escasso** — que remove o "freio social" sem reduzir a
   velocidade. Em volume menor, um excesso de velocidade tem menos tempo de ser corrigido
   por quem vem atrás.
3. A severidade se concentra onde a **separação entre fluxos é menor**: a pista simples
   concentra ultrapassagem em contramão e travessia de pedestre sem faixa, e a contramão
   é justamente o mecanismo de alta letalidade da base. O nevoeiro agrava o mesmo cenário
   ao encurtar a distância de reação.

**O que o filtro de volume mínimo descartou.** Todo filtro tem um custo, e o ponto é
**declará-lo** — assim a ausência pode ser lida como informação e não como esquecimento.

| Categoria | Acidentes | Taxa | Lift | Onde ficaria |
|---|---|---|---|---|
| `Participar de racha` | 52 | 25,00% | 3,49 | 6º por lift, logo atrás do top 5 (lift 3,60) |
| `Sinistro pessoal de trânsito` | 20 | 15,00% | 2,09 | 3º entre os tipos de acidente |
| `Túnel` | 125 | — | — | fora da Q7 (exige > 500) |

Además: 36 das 118 BRs com registro noturno ficam abaixo do corte de volume da Q9; 64 dos
90 municípios de PE saem do top 5 municipal; e 6 das 27 células de pista × clima ficam fora
do cruzamento — **1 sem nenhum registro** (`Dupla` + `Neve`, taxa 0/0 indefinida) e **5
abaixo do volume mínimo de 50** (`Granizo` e `Neve`, de 1 a 3 acidentes). As duas situações
recebem rótulos diferentes, `sem casos registrados` e `sem quantidade mínima de casos`, em
vez de um `NaN` que sugeriria "zero de letalidade" em vez de "sem amostra para dizer".

**O gráfico de síntese** (`resultados/g4_sintese_achados.html`) junta os três achados em
uma escala só — o lift de cada grupo contra a linha da taxa global, com o maior lift no
topo. São **sete grupos**: o primeiro colocado de cada achado (`Colisão frontal`, 4,16;
`Faixa 03h–06h`, 1,71; `Pista Simples`, 1,38), mais quatro casos que mostram os dois lados
da média — `Atropelamento de pedestre` (4,04), 2º do Achado 1 e sustentação da leitura de
severidade do impacto; `Contramão` (4,04, lift da causa `Transitar na contramão`), reforço
do Achado 1 por outro recorte; e os dois contrapontos abaixo de 1,00, `Pista Dupla` (0,66) e
`Faixa 09h–12h` (0,69). O que os três achados têm em comum é a **separação entre acima e
abaixo da taxa global** — nenhum grupo fica perto dela por acaso.

Nenhum dos três achados é uma explicação causal: são leituras de **concentração**, com
hipótese e limitação explicitadas. É o máximo que uma base de acidentes registrados, sem
velocidade, sem geometria e sem exposição, permite afirmar.

---

## Fórmulas e métricas

Todas as medidas usadas no projeto, em um lugar só.

| Métrica | Fórmula | Referência |
|---|---|---|
| Acidente fatal | `1 se mortos >= 1, senão 0` | 15.293 de 213.497 |
| **Taxa de letalidade** | `fatais / acidentes × 100` | global **7,1631%** |
| **Lift** | `taxa do grupo / taxa global` | 1,00 = média da base |
| Risco relativo | `taxa do grupo A / taxa do grupo B` | FDS ÷ úteis = **1,285** |
| Razão de sobre-representação | `% dos críticos / % de todos os fatais` | frontal = **2,27** |
| Mortos por 100 acidentes | `mortos / acidentes × 100` | BR-316 = **22,4** |
| Acidentes por dia | `acidentes / dias distintos com registro` | Fim de Ano = **211,0** |
| Mortos por dia | `mortos / dias distintos com registro` | Fim de Ano = **20,21** |
| Amplitude | `máx(taxa) - mín(taxa)` | mensal = **1,92 p.p.** |
| Variação no período | `taxa do último - taxa do primeiro` | 2023→2025 = **+0,015 p.p.** |
| Média de mortos por acidente | `soma(mortos) / n` | crítico = **3,82** |
| Mortes por acidente fatal | `total de mortos / fatais` | **1,17** |
| Taxa global | `fatais totais / acidentes totais` | 15.293 / 213.497 = **7,1631%** |

Convenções de leitura:

- **Taxa de letalidade** é sempre o **percentual de acidentes fatais do grupo** — nunca
  a divisão de mortos pelo total de mortos. Isso muda a leitura: `Colisão frontal` mata
  em 29,81% dos acidentes que registra, e não em 10,81% das pessoas envolvidas nesses
  acidentes.
- **Lift** de 1,00 significa "igual à média da base"; acima de 1,00, acima da média.
- **Volume mínimo** é declarado em toda tabela e nunca aplicado em silêncio: a tabela
  mostra também o que ficou de fora.
- **Acerto de linguagem**: as leituras são **associativas**. Onde o texto diz que um fator
  está "associado a" uma taxa maior, é porque a base não traz velocidade, geometria nem
  exposição — só o registro do acidente.

---

## Estrutura do repositório

```
.
├── dados_brutos/          # CSVs originais da PRF (versionados)
│   ├── datatran2023.csv
│   ├── datatran2024.csv
│   └── datatran2025.csv
├── dados_tratados/        # bases geradas pelos notebooks (versionadas)
│   ├── base_analitica_prf_2023_2025.csv
│   └── base_modelavel_prf_2023_2025.csv
├── notebooks/
│   ├── 01_parte1_ingestao.ipynb
│   ├── 02_parte2_limpeza.ipynb
│   ├── 03_parte3_variaveis.ipynb
│   ├── 04_parte4_bases.ipynb
│   ├── 05_parte5_nivel1.ipynb
│   ├── 06_parte5_nivel2.ipynb
│   ├── 07_parte5_nivel3.ipynb
│   ├── 08_parte5_nivel4.ipynb
│   ├── 09_parte6_graficos.ipynb
│   └── 10_parte7_sintese.ipynb
├── resultados/            # gráficos gerados pelo notebook (versionados)
│   ├── g1_taxa_por_tipo_acidente.html
│   ├── g2_serie_mensal.html
│   ├── g3_faixa_horaria.html
│   └── g4_sintese_achados.html
└── README.md
```

Os notebooks são **autocontidos**: cada um carrega o seu próprio setup (imports, localização
da raiz por `Path`, leitura da base e helpers de análise) e roda de forma independente,
sem depender de nenhum outro arquivo do repositório.

As bases e os gráficos são **regeneráveis** — o notebook da Etapa 4 reescreve as bases e o
da Etapa 6 reescreve os gráficos a cada execução, com o mesmo conteúdo —, mas estão no
repositório para que a primeira execução seja imediata e para que os artefatos possam ser
inspecionados sem rodar nada.

---

## Como executar

**Não é preciso baixar nada:** os dados, as bases tratadas e os gráficos já estão no
repositório. Após o clone, basta instalar as dependências e executar o notebook da etapa
que quiser.

```bash
pip install pandas numpy plotly jupyter
jupyter notebook notebooks/09_parte6_graficos.ipynb
```

Cada etapa é um notebook próprio: abra o da etapa que quer rodar e use **Restart & Run All**.
O notebook resolve a raiz do projeto por `Path`, então roda tanto aberto da raiz do
repositório quanto de dentro de `notebooks/`. Nenhum CSV original é sobrescrito.

**Ordem de execução:** as Etapas 1 a 4 precisam de `dados_brutos/`; a Etapa 4 gera as bases
em `dados_tratados/`; as Etapas 5 a 7 leem essas bases. Para o **PNG** dos gráficos é
preciso `pip install kaleido` **e** o Chrome instalado — sem eles o notebook da Etapa 6
salva só o HTML e avisa no output, sem falhar.

---

## Verificação

O trabalho foi conferido em **onze frentes**, todas sobre a base reconstruída a partir
dos CSVs oficiais da PRF:

| Frente | O que foi verificado | Resultado |
|---|---|---|
| **Conformidade literal com o enunciado** | Cada requisito das Partes 1 a 4, 6 e 7 e das regras de avaliação do PDF, conferido contra o código: `ler_csv_prf` com fallback de encoding, `pd.concat` em `acidentes_prf_historico`, `info()`/nulos/duplicidades, vírgula decimal, `IGNORADO` e `0`, as 5 colunas removidas, as 5 variáveis novas, `verificar_data_leakage`, exportação `;`/`utf-8-sig`, os 3 gráficos e os 3 achados | 50/50 |
| **As 12 questões** | Célula de texto com a pergunta **antes** da resposta, enunciado cobrindo o que o PDF pede, **volume mínimo declarado** e comparação com a taxa global | 12/12 |
| **Recálculo numérico** | Todos os números das 12 análises recalculados **do zero**, a partir da base tratada, e comparados com o que está escrito nos outputs | 62/62, 0 divergências |
| **Coerência do README** | Cada número do README reconferido contra o recálculo independente | 25/25 |
| **Tabelas em markdown** | Cada número das 14 tabelas do README e dos notebooks reconferido contra recálculo independente | 109/109, 0 divergências |
| **Saída dos gráficos** | Arrays binários do payload decodificados, cada valor plotado comparado com o recálculo; legenda de 2 linhas e título com mensagem nos 4 HTMLs | 22/22 e 32/32 |
| **Qualidade da cadeia** | Fonte bruta → base tratada → HTMLs: nulos, strings vazias, duplicidades exatas, negativos, coerência de tipos, coerência lógica do alvo, ausência de vazamento, marcadores semânticos, regras de `data_comemorativa`/`faixa_horaria`/`fim_de_semana`, payload dos gráficos feito *parse* como JSON | 52/52 |
| **As 12 interpretações** | Presentes, com 2–3 frases, sem linguagem causal e com a ressalva de que é associação | 12/12 |
| **Saídas dos notebooks** | Varredura das saídas dos 10 notebooks em busca de `NaN`, `null`, `None` e `undefined` | 10/10 limpos |
| **Camadas e ordem de desenho** | Reordenação de `barlayer`/`scatterlayer`/`shapelayer` no SVG real de cada gráfico, conferência de `zorder`/`layer` no payload e posição de cada barra e ponto renderizados (16 + 36 + 8 + 7 elementos) | 6/6 e 17/17 |
| **Reprodutibilidade** | Os 10 notebooks reexecutados de ponta a ponta sobre a base reconstruída | 10/10 sem erro |

**Total: 233 verificações, 0 falhas** nas oito frentes originais; as três frentes
acrescentadas na revisão — tabelas em markdown, saída dos gráficos e camadas/ordem de
desenho — somam **186 verificações adicionais, também sem falha**.
A tabela acima tem **419 verificações**, o mesmo que 233 + 186.

A revisão final recalculou tudo o que o texto publica — os 24 números-chave do README, as
33 afirmações da Q11, as 23 da Q12, as 20 da Q7/Q10/Q11 e os 18 valores dos quatro
gráficos — e corrigiu **quatro divergências** que as frentes anteriores não pegavam:
o lift de 2025 na Q1 (publicado 1,02, correto **1,00**), a contagem de tipos não nomeados
na Q5 (11 → **10**), a coluna de posição da Q6 (que misturava o ranking filtrado com o
vocabulário inteiro) e um arredondamento de 26,6% para **26,5%** na Q9. Os notebooks 06 e
07 foram reexecutados; os outros oito não tiveram alteração de conteúdo.

Uma **quinta passada**, sobre ângulos que as anteriores não cobriam — a aritmética interna
de cada tabela (taxa, lift, mortos/100 e somas recalculados a partir das contagens
impressas, **109/109**), as afirmações de filtro declarado ("N de M" em toda tabela) e a
partição da própria tabela acima — encontrou mais **dois defeitos**: a tabela da Q7
publicava **8 dos 11** traçados que passam o corte de volume, sem declarar os outros três
(`Desvio Temporário`, `Viaduto`, `Interseção de Vias`), e o total de verificações desta
seção somava **416** contra os **393** declarados. Ambos corrigidos; nenhum número
publicado mudou de valor.

Além disso, um **clone limpo** do repositório foi testado do zero: os 10 notebooks
rodam sem erro, as duas bases regeneram com **MD5 idêntico** ao das bases commitadas e
os CSVs originais permanecem intactos (MD5 conferido contra os publicados pela PRF).

---

## Decisões técnicas

Registradas porque desviam do comportamento ingênuo e precisam de rastreabilidade.

**1. `ler_csv_prf` com fallback que realmente funciona.** A versão ingênua fazia `raise`
dentro do `except`, então a primeira falha de encoding interrompia a função e o fallback
nunca era tentado. Aqui o laço continua para o próximo encoding e o erro só é levantado
se todos falharem. Nos três arquivos `utf-8` falha e `latin1` funciona — o fallback é
realmente exercitado.

**2. `data_inversa` é ISO, não `dd/mm/yyyy`.** O parse `format="%d/%m/%Y"` transformaria
as 213.497 datas em `NaT`. O parse correto é `%Y-%m-%d`.

**3. `id` lido como texto.** O arquivo de 2024 tem uma linha com `id = "6e+05"` (a
37.162ª linha de dados, notação científica gerada por planilha). Lido como número, o
pandas converteria a coluna inteira para `float` e a linha perderia o identificador. Com
`dtype={"id": "string"}` a anomalia fica visível em vez de silenciosa. **A linha é
preservada**, não descartada.

**4. `arquivo_origem` + `ano_acidente`.** As duas colunas existem: a primeira registra de
qual arquivo a linha veio, a segunda é derivada de `data_inversa`. A redundância é
proposital — se divergirem, há registro no arquivo errado.

**5. Duplicidades exatas.** Os três arquivos têm **zero** duplicatas. A remoção da Etapa 2
é executada mesmo assim, e o notebook reporta "0 removidas" explicitamente.

**6. Verificação de linhas nulas ou inválidas.** Cada coluna categórica é comparada com o
domínio esperado da PRF (`DOMINIOS`) e substituída por `IGNORADO` toda linha cujo valor
seja nulo ou esteja fora desse domínio. O notebook registra quantas linhas foram corrigidas
em cada coluna e reconfere depois que nada sobrou (`assert`). `IGNORADO` é aceito como
valor válido na reconferência.

**7. `strip` + `upper` só nos identificadores.** `uf`, `br` e `municipio` já vêm **100% em
caixa alta** da fonte, então o `upper()` é aplicado de forma explícita e é comprovadamente
inócuo — as duas bases geradas têm MD5 idêntico antes e depois da mudança. As colunas
**descritivas** vêm em **caixa de título**, que é a forma publicada pela PRF e a usada nos
rótulos de todo o relatório; maiusculizá-las exigiria reescrever todos os conjuntos
`DOMINIOS` e todos os títulos das 12 análises sem acrescentar informação. O `upper()` é
guardado por dtype, para não tocar em `br` (que é numérico).

**8. Nomenclatura das análises.** As células e as variáveis do código (`q8`, `q9`, `q10`)
foram renomeadas e reordenadas para casar com a pergunta que servem, de modo que o nome
do bloco, o da variável e o da análise apontem para a mesma coisa.

**9. Critério de ordenação do ranking municipal.** O top 5 de PE 2024–2025 é ordenado por
**número de acidentes fatais**, com desempate por mortos e por volume para a ordem ser
determinística. O ranking por **mortos** é mantido no notebook como contraponto. O
**conjunto** dos cinco é o mesmo nos dois critérios, então a resposta não muda — muda a
ordem dentro do grupo.

**10. `tracado_via` explode, e o não-partição é declarado.** O tratamento escolhido é
**explode** — cada traçado conta em todos os registros que o possuem — porque a pergunta é
sobre gravidade *associada a* um traçado, e um acidente em `Reta;Declive` pertence de fato
às duas categorias. A consequência é declarada no texto: os grupos não formam partição e
as porcentagens não somam 100%.

**11. Posição do título nos gráficos.** O título de cada gráfico tem **três linhas** — a
mensagem principal em negrito e duas linhas de legenda em `<sup>` —, e por isso ocupa mais
altura que o padrão do Plotly, que reserva pouco espaço no topo. Ancorá-lo em
`y=0.99, yanchor="top"` dentro da área de plotagem fazia a primeira linha ser **cortada**
no topo, tanto no HTML quanto na saída do notebook. A correção é `y=0.95` com
`automargin=True`, que deixa a margem superior (`t=120`) absorver as três linhas sem
encostar na borda.

---

## Licença e fonte

Dados públicos da Polícia Rodoviária Federal (Dados Abertos).
