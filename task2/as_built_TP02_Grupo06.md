# Documento Técnico *As-Built* — TP02 Mineração de Dados

**Disciplina:** Ciência de Dados I — CEFET-MG
**Trabalho:** TP02 — Mineração de Dados
**Grupo:** 06
**Tema:** Previsão de efetivação de matrícula na Lista de Espera 2/2023 do SISU (Problema B)
**Data de referência:** 2026-06-24

> Este documento descreve **o que foi efetivamente construído** (as-built): decisões de projeto, pipeline, parâmetros e resultados reais das execuções. Os números aqui refletem a saída dos notebooks com `random_state = 42`.

---

## 1. Sumário executivo

Construímos e comparamos **6 classificadores binários** para responder, sob a ótica da instituição: **"dado que um candidato foi convocado, ele efetivará a matrícula?"** (Problema B). A partir de 148.773 registros do SISU, definiu-se uma população de **44.504 convocados com desfecho resolvido** (38,1% de efetivação) e treinaram-se os modelos com validação **Holdout 60/20/20**, balanceamento por **undersampling** no treino e busca de hiperparâmetros via **`PredefinedSplit`** otimizando **AUC-ROC**.

**Melhor modelo:** Random Forest — **AUC-ROC de teste = 0,797**. Todos os modelos ficaram acima do acaso (0,73–0,80), confirmando sinal preditivo real. Os fatores mais determinantes são **contextuais**: posição na lista (`CLASSIFICACAO`), folga de nota (`MARGEM`) e proximidade geográfica (`MESMA_UF`).

---

## 2. Objetivo e escopo

O enunciado define dois problemas:

| Problema | Ótica | Pergunta | Status |
|---|---|---|---|
| A | Candidato | "Serei aprovado?" | **Descartado** |
| B | Instituição | "Esse convocado efetivará a matrícula?" | **Implementado** |

**Por que o Problema A foi descartado:** a coluna-alvo `APROVADO` é **constante** (`"S"` em 100% das 148.773 linhas), portanto não há classe negativa para treinar um classificador. Decisão **confirmada pelo professor**. Em consequência, o item 20 do enunciado (mesmo modelo nos dois problemas) não se aplica; a comparação foi feita **entre os 6 modelos no Problema B**.

---

## 3. Base de dados

- **Arquivo:** `lista_de_espera_sisu_2023_2.csv`
- **Dimensões:** 148.773 linhas × 56 colunas
- **Origem:** dado público governamental (SISU)

**Peculiaridades de carga (tratadas na leitura):**

| Característica | Valor | Consequência se ignorada |
|---|---|---|
| Separador de campos | `\|` (pipe) | Colunas não seriam separadas |
| Codificação | **Latin-1** | `UnicodeDecodeError` em acentos (`Ç`, `Ã`) |
| Separador decimal | **vírgula** (`622,5`) | Notas lidas como texto → modelos quebram |

```python
df = pd.read_csv("lista_de_espera_sisu_2023_2.csv",
                 sep="|", encoding="latin-1", decimal=",", low_memory=False)
```

---

## 4. Arquitetura da solução

A solução foi dividida em **3 notebooks autossuficientes**, conectados pelas **bases tratadas** exportadas em CSV. Cada notebook executa de ponta a ponta de forma independente.

```
 base bruta (CSV)
      │
      ▼
┌─────────────────────┐     ┌──────────────────────────┐     ┌─────────────────────┐
│ nb_EDA              │     │ nb_PreProcessamento      │     │ nb_Modelagem        │
│ (Etapas 0–1)        │     │ (Etapa 2)                │     │ (Etapas 3–5)        │
│ explora a base      │     │ limpa, feat. eng., split │     │ lê as bases,        │
│ → 6 visualizações   │     │ → EXPORTA 5 CSVs         │ ──► │ treina + avalia     │
└─────────────────────┘     └──────────────────────────┘     └─────────────────────┘
                                       │                              ▲
                                       └──── bases_tratadas/ ─────────┘
```

| Notebook | Etapas | Entrada | Saída |
|---|---|---|---|
| `nb_EDA_TP02_Grupo06.ipynb` | 0–1 | base bruta | 6 gráficos + decisões |
| `nb_PreProcessamento_TP02_Grupo06.ipynb` | 2 | base bruta | **5 bases tratadas** |
| `nb_Modelagem_TP02_Grupo06.ipynb` | 3–5 | 5 bases tratadas | ranking, ROC, importâncias |

---

## 5. Definição da população e da variável-alvo

A pergunta do Problema B é **condicional** ("dado que foi convocado"). O campo `MATRICULA` codifica o desfecho:

| Status | Linhas | Uso |
|---|---|---|
| `NÃO CONVOCADO` | 69.729 | **Excluído** — a pergunta não se aplica |
| `PENDENTE` | 34.540 | **Excluído** — desfecho em aberto (rótulo seria incorreto) |
| `EFETIVADA` | 16.943 | Alvo **positivo (1)** |
| `NÃO COMPARECEU` | 26.085 | Negativo (0) |
| `DOCUMENTACAO REJEITADA` | 1.233 | Negativo (0) |
| `CANCELADA` | 243 | Negativo (0) |

**População final (convocados resolvidos): 44.504 linhas** — 27.561 negativos (61,9%) e 16.943 positivos (**38,1%**). Desbalanceamento moderado.

```python
convocados = df[df["MATRICULA"] != "NÃO CONVOCADO"]
base = convocados[convocados["MATRICULA"] != "PENDENTE"]
base["EFETIVOU"] = (base["MATRICULA"] == "EFETIVADA").astype(int)
```

### 5.1 Anti-vazamento (leakage)

Único vazamento real evitado: a própria **`MATRICULA`** (fonte do alvo) e **identificadores/PII** (`CPF`, nome, inscrição). Campos como `CLASSIFICACAO`, `NOTA_CORTE` e `NOTA_CANDIDATO` **são permitidos no Problema B** — são conhecidos **no momento da convocação**, antes da decisão de matrícula (linha do tempo: ENEM → convocação [classificação/corte] → **previsão** → matrícula). A lista de "features proibidas" do enunciado refere-se ao **Problema A** (ótica do candidato, antes da divulgação dos resultados); **confirmado com o professor** que se aplica somente ao A.

---

## 6. Análise Exploratória (EDA) — principais achados

| # | Visualização | Achado | Decisão derivada |
|---|---|---|---|
| 1 | Balanceamento do alvo | 38,1% positivos | Undersampling + métrica AUC-ROC |
| 2 | Notas por classe (boxplots) | Distribuições quase sobrepostas | Nota bruta discrimina pouco |
| 3 | Mapa de correlação | 5 notas por área redundantes com `NOTA_CANDIDATO` (r≈0,74–0,78) | Descartar notas por área |
| 4 | Dimensão geográfica | `MESMA_UF`: **43,7% vs 18,8%** | Confirma `MESMA_UF` como forte preditor |
| 5 | Categóricas | 1ª opção ~43% vs 2ª ~28%; variação por cota/grau | Manter categóricas de contexto |
| 6 | Outliers (IQR) | Outliers **legítimos** (idade/posição/nota reais) | Não remover; usar RobustScaler |

**Correlação com o alvo** (evidência de que o sinal é contextual): `MARGEM` +0,261, `CLASSIFICACAO` −0,198, contra ~0,09 das notas brutas.

---

## 7. Pré-processamento

### 7.1 Engenharia de atributos

| Feature | Fórmula | Hipótese |
|---|---|---|
| `MESMA_UF` | `UF_CANDIDATO == UF_CAMPUS` → {0,1} | Proximidade reduz desistência |
| `IDADE` | `2023 − DT_NASCIMENTO` (campo é o ano) | Perfil etário |
| `MARGEM` | `NOTA_CANDIDATO − NOTA_CORTE` | Folga sobre o corte |

### 7.2 Conjunto final de features (13 colunas → 32 após encoding)

- **Numéricas (6):** `NOTA_CANDIDATO`, `NOTA_CORTE`, `MARGEM`, `CLASSIFICACAO`, `IDADE`, `QT_VAGAS_CONCORRENCIA`
- **Categóricas nominais (6):** `GRAU`, `TURNO`, `TIPO_MOD_CONCORRENCIA`, `TP_COTA`, `SEXO`, `OPCAO`
- **Binária (1):** `MESMA_UF`
- **Descartadas:** 5 notas por área (redundância), `PERCENTUAL_BONUS` (94,7% ausente), identificadores/PII, colunas constantes.

### 7.3 Tratamento de faltantes

| Coluna | Faltante | Tratamento | Justificativa |
|---|---|---|---|
| `TP_COTA` | 56,3% | `NaN → "AMPLA"` (constante) | NaN = sem cota; constante não causa leakage |
| `NOTA_CORTE`, `MARGEM` | 2.583 (5,8%) | Mediana **dentro do pipeline** | Estatística ajustada só no treino (sem leakage) |
| `PERCENTUAL_BONUS` | 94,7% | Descartada | Ausência excessiva |

### 7.4 Particionamento — Holdout 60/20/20 estratificado

| Conjunto | Linhas | % positivos |
|---|---|---|
| Treino | 26.702 | 38,1% |
| Validação | 8.901 | 38,1% |
| Teste | 8.901 | 38,1% |

Implementado em **dois cortes** de `train_test_split` (teste 20%; depois 25% do restante → validação), `stratify=y`, `random_state=42`. Justificativa para Holdout (e não cross-validation): base grande → estimativa estável a custo muito menor.

### 7.5 Desbalanceamento — Undersampling (apenas no treino)

| Treino | Classe 0 | Classe 1 | Total |
|---|---|---|---|
| Antes | 16.537 | 10.165 | 26.702 |
| Depois | 10.165 | 10.165 | **20.330** |

Aplicado **somente ao treino**; validação e teste mantêm a distribuição real (38,1%). Escolha do undersampling sobre `class_weight`/SMOTE: funciona igual para os 6 modelos (KNN e Naive Bayes não aceitam `class_weight`) → comparação justa, e acelera o SVM.

### 7.6 Pipeline de pré-processamento (`ColumnTransformer`)

| Bloco | Colunas | Transformação |
|---|---|---|
| `num` | 6 numéricas | `SimpleImputer(median)` → `RobustScaler` |
| `cat` | 6 categóricas | `OneHotEncoder(handle_unknown="ignore")` |
| `bin` | `MESMA_UF` | `passthrough` |

**Resultado: 32 features.** O pré-processador é **ajustado (`fit`) apenas no treino balanceado**; validação e teste só passam por `transform` → garantia de não-vazamento. `RobustScaler` (mediana/IQR) é robusto aos outliers legítimos e essencial para KNN/SVM.

### 7.7 Bases tratadas exportadas (`bases_tratadas/`)

| Arquivo | Conteúdo |
|---|---|
| `base_tratada_consolidada_Grupo06.csv` | População do Problema B (features + alvo), antes do split |
| `treino_Grupo06.csv` | Conjunto de treino (60%) |
| `validacao_Grupo06.csv` | Conjunto de validação (20%) |
| `teste_Grupo06.csv` | Conjunto de teste (20%) |
| `treino_balanceado_Grupo06.csv` | Treino após undersampling (usado no `fit`) |

CSV em UTF-8, separador `;`. Salvas **antes** do encoding/escala (dados legíveis); a transformação ocorre no pipeline do `nb_Modelagem`.

---

## 8. Modelagem e tuning

### 8.1 Modelos e grades de hiperparâmetros

| Modelo | Grade buscada |
|---|---|
| Árvore de Decisão | `max_depth ∈ {None,6,10,16}`, `criterion ∈ {gini,entropy}` |
| Random Forest | `n_estimators ∈ {200,400}`, `max_depth ∈ {None,14}`, `max_features ∈ {sqrt,log2}` |
| Gradient Boosting | `n_estimators ∈ {150,300}`, `learning_rate ∈ {0.05,0.1}`, `max_depth ∈ {2,3}` |
| KNN | `n_neighbors ∈ {15,31,51}`, `metric ∈ {euclidean,manhattan}` |
| Naive Bayes (Gaussian) | `var_smoothing ∈ {1e-9,1e-7,1e-5}` |
| SVM | `kernel ∈ {linear,rbf}`, `C ∈ {0.1,1,10}` |

> `GaussianNB` (não `MultinomialNB`): as features são majoritariamente contínuas; `MultinomialNB` exige contagens não-negativas.

### 8.2 Estratégia de tuning em Holdout

Como não há cross-validation, usou-se **`PredefinedSplit`** (treino balanceado = fold −1; validação = fold 0) com `GridSearchCV(scoring="roc_auc", refit=False)`. O melhor conjunto é reajustado (`fit`) no treino balanceado. É a forma correta de tunar em Holdout (atende ao item 13 do enunciado).

### 8.3 Hiperparâmetros vencedores e tempos

Conforme orientação do professor, mediram-se **dois tempos**: busca de hiperparâmetros (`GridSearchCV.fit`) e treino (`fit` final).

| Modelo | Melhores hiperparâmetros | AUC val | Busca HP (s) | Treino (s) |
|---|---|---|---|---|
| Random Forest | `max_depth=14, max_features=sqrt, n_estimators=400` | 0,782 | 16,4 | 5,7 |
| Gradient Boosting | `learning_rate=0.1, max_depth=3, n_estimators=300` | 0,775 | 15,2 | 9,3 |
| SVM | `kernel=rbf, C=10` | 0,756 | **73,3** | 17,4 |
| KNN | `metric=manhattan, n_neighbors=31` | 0,748 | 4,6 | 0,0 |
| Árvore de Decisão | `criterion=gini, max_depth=10` | 0,738 | 2,7 | 0,1 |
| Naive Bayes | `var_smoothing=1e-7` | 0,719 | 0,1 | 0,0 |

> O SVM domina o custo de busca (73 s) pela grade `kernel × C` em RBF sobre 20 mil pontos.

---

## 9. Avaliação e resultados

Avaliação final no **conjunto de teste** (8.901 linhas, distribuição real de 38,1%, nunca usado no tuning). Métrica principal: **AUC-ROC** (robusta ao desbalanceamento); secundárias: F1, acurácia, precisão, recall.

### 9.1 Ranking (conjunto de teste, ordenado por AUC-ROC)

| # | Modelo | AUC-ROC | F1 | Acurácia | Precisão | Recall |
|---|---|---|---|---|---|---|
| 1 | **Random Forest** | **0,797** | 0,668 | 0,703 | 0,582 | 0,783 |
| 2 | Gradient Boosting | 0,791 | 0,657 | 0,701 | 0,583 | 0,752 |
| 3 | SVM | 0,774 | 0,658 | 0,690 | 0,568 | 0,783 |
| 4 | KNN | 0,759 | 0,634 | 0,664 | 0,541 | 0,765 |
| 5 | Árvore de Decisão | 0,751 | 0,633 | 0,673 | 0,553 | 0,741 |
| 6 | Naive Bayes | 0,733 | 0,631 | 0,656 | 0,533 | 0,773 |

Saídas geradas: **curva ROC com os 6 modelos** (`figs/roc_problemaB.png`), **matrizes de confusão** (`figs/matrizes_confusao.png`) e **importância de variáveis** das 3 árvores (`figs/feature_importance.png`).

### 9.2 Interpretação

- **Ensembles lideram** (RF e GB): melhor AUC, custo de treino moderado, capturam interações não-lineares — típico em dados tabulares.
- **Preditores mais importantes** (concordância entre Árvore/RF/GB): `CLASSIFICACAO` (posição na lista), `MARGEM`/`NOTA_CORTE` (folga de nota), `MESMA_UF` (proximidade), `OPCAO`. O desfecho é guiado por **contexto**, não pela nota absoluta.
- **Trade-off recall × precisão:** treino balanceado eleva o recall (~0,78) ao custo de precisão (~0,58). O limiar de 0,5 pode ser recalibrado conforme o custo que a instituição atribui a convocar quem não comparece.
- **Naive Bayes** é o mais fraco (0,733): a suposição de independência entre atributos é violada.

---

## 10. Limitações e trabalhos futuros

- **Desbalanceamento:** comparar undersampling com `class_weight` e SMOTE.
- **Features de alta cardinalidade:** agregar curso/IES via *target/frequency encoding*.
- **Limiar de decisão:** calibrar ao custo de negócio (matrícula não efetivada × vaga ociosa).
- **`IDADE`** deriva apenas do ano de nascimento (granularidade anual).

---

## 11. Reprodutibilidade

- **Ambiente:** Python (`.venv`), scikit-learn 1.9.0, pandas 2.2.2, numpy 1.26.4, matplotlib 3.9.0, seaborn 0.13.2.
- **Semente:** `RANDOM_STATE = 42` em todos os splits, undersampling e modelos.
- **Ordem de execução:** `nb_EDA` → `nb_PreProcessamento` (gera `bases_tratadas/`) → `nb_Modelagem` (consome).

```bash
jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.kernel_name=python3 \
  nb_PreProcessamento_TP02_Grupo06.ipynb
jupyter nbconvert --to notebook --execute --inplace \
  --ExecutePreprocessor.kernel_name=python3 \
  nb_Modelagem_TP02_Grupo06.ipynb
```

---

## 12. Estrutura de entregáveis

```
task2/
├── nb_EDA_TP02_Grupo06.ipynb
├── nb_PreProcessamento_TP02_Grupo06.ipynb
├── nb_Modelagem_TP02_Grupo06.ipynb
├── bases_tratadas/            (5 CSVs tratados → Drive)
├── figs/                      (9 PNGs: 6 EDA + ROC + confusão + importância)
├── dbs_TP02.txt               (links do Drive das bases)
└── as_built_TP02_Grupo06.md   (este documento)
```

**Pacote final (`TP02_Grupo06.zip`):** `slides_TP02_Grupo06.pdf` + os 3 notebooks + `dbs_TP02.txt`.
**Pendências:** subir as bases ao Drive e preencher `dbs_TP02.txt`; produzir os slides (Seção 11).

---

## 13. Matriz de rastreabilidade de decisões

| Decisão | Escolha | Justificativa | Onde |
|---|---|---|---|
| Problema A | Descartado | Alvo `APROVADO` constante | `nb_EDA` 1.2 |
| População/alvo | Excluir `NÃO CONVOCADO` e `PENDENTE` | Pergunta condicional; evitar rótulo ambíguo | `nb_EDA` 1.3 / `nb_PreProc` |
| Anti-leakage | Remover só `MATRICULA` + PII | `CLASSIFICACAO`/`NOTA_CORTE` permitidos no B (confirmado) | `nb_PreProc` 2.1 |
| Redundância | Descartar 5 notas por área | Multicolinearidade (heatmap) | `nb_PreProc` 2.1 |
| Outliers | Manter + RobustScaler | Outliers legítimos | `nb_PreProc` 2.4 |
| Validação | Holdout 60/20/20 estratificado | Base grande; preserva 38% | `nb_PreProc` 2.2 |
| Desbalanceamento | Undersampling (só treino) | Igual p/ 6 modelos; acelera SVM | `nb_PreProc` 2.3 |
| Encoding | One-Hot uniforme | Nominais; comparação justa | `nb_PreProc` 2.4 |
| Tuning | `PredefinedSplit` + AUC-ROC | Holdout sem CV (item 13) | `nb_Modelagem` 3 |
| Métrica | AUC-ROC principal | Robusta ao desbalanceamento | `nb_Modelagem` 4 |
