# TP01 — Ciência de Dados I | Grupo 06

Trabalho prático de pré-processamento e análise de dados da disciplina **Ciência de Dados I** (CEFET-MG Campus VII, Timóteo-MG). O trabalho analisa os resultados de uma prova aplicada à turma a partir de três bases de dados: respostas por aluno, formulário de sondagem pós-prova e metadados das questões.

---

## Estrutura do projeto

```
ds-assignment/
├── data/
│   ├── raw/                              ← Dados originais (somente leitura)
│   │   ├── TP01_-_Bases_de_dados_-_formulario.csv
│   │   ├── TP01_-_Bases_de_dados_-_prova.csv
│   │   └── TP01_-_Bases_de_dados_-_questoes.csv
│   └── processed/
│       ├── base_integrada_Grupo06.csv    ← Base final tratada e integrada
│       └── visualizacoes/               ← Exportações dos 10 gráficos analíticos
│           ├── P1_distribuicao_notas.png
│           ├── P2_acerto_por_questao.png
│           ├── P3_desempenho_por_assunto.png
│           ├── P4_horas_vs_nota.png
│           ├── P5_esperada_vs_real.png
│           ├── P6_perfil_top.png
│           ├── P7_radar_turma.png
│           ├── P8_heatmap_aluno_questao.png
│           ├── P9_simulados_ia_vs_nota.png
│           └── P10_tempo_vs_nota.png
├── docs/
│   └── CDI_-_Trabalho_Prtico_01.pdf     ← Enunciado original do trabalho
├── TP01_Grupo06.ipynb                    ← Notebook principal (entregável)
├── relatorio_TP01_Grupo06.md            ← Relatório detalhado (entregável)
├── requirements.txt                      ← Dependências Python com versões fixadas
└── .venv/                               ← Ambiente virtual (não versionado)
```

---

## Descrição de cada arquivo

### Entregáveis

| Arquivo | Descrição |
|---|---|
| `TP01_Grupo06.ipynb` | Notebook Jupyter principal. Contém todo o pipeline: diagnóstico, pré-processamento, criação de atributos, transformações, agregações e as 10 visualizações analíticas (P1–P10). Executa de cima a baixo sem intervenção manual a partir dos CSVs em `data/raw/`. |
| `relatorio_TP01_Grupo06.md` | Relatório escrito completo. Documenta ponto a ponto o planejamento da análise, cada decisão de pré-processamento com sua justificativa, a escolha de cada tipo de gráfico e os principais insights encontrados. |
| `data/processed/base_integrada_Grupo06.csv` | Base de dados final resultante da integração das três fontes, após limpeza, criação de atributos e transformações. Contém 42 registros e todos os atributos originais + derivados. |

### Dados originais

| Arquivo | Descrição |
|---|---|
| `data/raw/TP01_-_Bases_de_dados_-_prova.csv` | Gabarito da prova por aluno. 42 registros. Colunas: Código, Turma, Término (horário de entrega), Q1–Q15 (1=acerto, 0=erro, vazio=pulou), Erros, Nota Questões, Faltas, %CH, %AU. Separador `\|`, encoding `latin1`. |
| `data/raw/TP01_-_Bases_de_dados_-_formulario.csv` | Sondagem pós-prova respondida pelos alunos. 40 registros (39 após remoção de duplicata). 35 colunas incluindo percepção de dificuldade, horas de estudo (texto livre), uso de simulados e autoavaliação de compreensão por tema (15 tópicos, escala ordinal de 3 níveis). Separador `\|`, encoding `latin1`. |
| `data/raw/TP01_-_Bases_de_dados_-_questoes.csv` | Metadados das 15 questões da prova. Colunas: Questão (Q1–Q15), Assunto, Simulado de origem (1, 2 ou 3) e flag de questão de código (1 ou vazio). Separador `\|`, encoding `latin1`. |

### Visualizações

Todas as imagens são exportadas automaticamente pelo notebook em `data/processed/visualizacoes/`.

| Arquivo | Conteúdo |
|---|---|
| `P1_distribuicao_notas.png` | Histograma + box plot das notas reais, estratificados por turma (T1 e T2). |
| `P2_acerto_por_questao.png` | Barras horizontais com a taxa de acerto de cada questão (Q1–Q15), ordenadas da mais difícil à mais fácil. Cores distinguem questões de código (laranja) de conceituais (azul). Anotações marcam as questões com armadilhas técnicas nas alternativas. |
| `P3_desempenho_por_assunto.png` | Barras duplas com eixo duplo: taxa de acerto real (por assunto) × compreensão média declarada no formulário. Evidencia divergências entre percepção e desempenho. |
| `P4_horas_vs_nota.png` | Scatter plot de horas de estudo vs. nota real, com linha de regressão e coeficiente de Pearson (r = −0,06). |
| `P5_esperada_vs_real.png` | Scatter plot de nota esperada vs. nota real, com diagonal y = x como referência de estimativa perfeita. Evidencia o viés de superconfiança da turma (+1,76 pts em média). |
| `P6_perfil_top.png` | Barras agrupadas comparando o perfil médio do Top 25% (nota ≥ 12,6 pts) com o restante da turma em quatro dimensões: horas de estudo, simulados feitos, compreensão média e acerto em questões de código. |
| `P7_radar_turma.png` | Gráfico de radar (Plotly) comparando T1 e T2 em cinco dimensões normalizadas: taxa de acerto, horas de estudo, compreensão, simulados e nota esperada. |
| `P8_heatmap_aluno_questao.png` | Heatmap 42×15: cada célula representa acerto (verde), erro (vermelho) ou questão pulada (cinza). Alunos ordenados por nota crescente; questões ordenadas por dificuldade crescente. |
| `P9_simulados_ia_vs_nota.png` | Box plots duplos: nota real e erro_esperado por perfil de uso de IA nos simulados (Fez sem IA / Usou IA em parte / Usou IA em todos). |
| `P10_tempo_vs_nota.png` | Scatter plot de tempo de prova (minutos) vs. nota real, com linha de regressão e pontos coloridos por turma (r = 0,02). |

### Documentação e configuração

| Arquivo | Descrição |
|---|---|
| `docs/CDI_-_Trabalho_Prtico_01.pdf` | Enunciado original do trabalho prático, disponibilizado pelo professor. Contém os objetivos de aprendizagem, descrição das bases, 10 perguntas analíticas e critérios de avaliação. |
| `requirements.txt` | Lista de dependências Python com versões fixadas. Inclui pandas, numpy, matplotlib, seaborn, plotly, scipy, jupyter, ipykernel e kaleido. |
| `README.md` | Este arquivo. Descreve a estrutura do projeto e cada arquivo. |

---

## Como reproduzir

```bash
# Ativar o ambiente virtual
source .venv/bin/activate

# Executar o notebook (abre no navegador)
jupyter notebook TP01_Grupo06.ipynb
```

O notebook executa de cima a baixo sem intervenção. Todos os arquivos de saída (`base_integrada_Grupo06.csv` e os 10 PNGs) são regenerados automaticamente em `data/processed/`.

Para instalar as dependências do zero:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m ipykernel install --user --name=ds-assignment --display-name "DS Assignment"
```

---

## Principais decisões técnicas

| Decisão | Escolha | Justificativa resumida |
|---|---|---|
| NaN em Q1–Q15 | Preservar (não → 0) | Pular questão é ação ativa permitida pela prova; tratar como erro introduziria viés |
| Join prova + formulário | LEFT JOIN | Preserva os 42 alunos da prova; os 3 sem formulário ficam com NaN explícito |
| Q7 = 11 (aluno 36116) | Corrigir para 1 | Verificação cruzada: Erros=5 e Nota=11,2 só são consistentes com Q7=1 |
| Término inválido (2 alunos) | `tempo_prova_min` = NaN | Sem base para estimar o valor real; melhor NaN explícito que imputação inventada |
| Discretização de nota | Igual largura com cortes pedagógicos | 11,2 pts = mediana e ponto natural (8 acertos × 1,4 = 11,2) |
| Discretização de horas | Igual frequência (tercis) | Distribuição assimétrica; tercis garantem grupos comparáveis |

---

*CEFET-MG Campus VII — Timóteo-MG | Prof. Thiago Goveia | Grupo 06*
