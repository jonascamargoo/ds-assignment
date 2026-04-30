# Trabalho Prático 02 — Ciência de Dados I
**Grupo 06 | CEFET-MG Campus VII, Timóteo-MG**
**Disciplina:** Ciência de Dados I — Prof. Thiago Goveia
**Referência teórica:** Tan et al. (2006), Capítulos 1 e 2

---

## 1. Contexto e Objetivos

Este trabalho analisa os dados de uma prova aplicada à turma de Ciência de Dados I. Três bases de dados foram disponibilizadas: os resultados da prova por aluno, um formulário de sondagem pós-prova e os metadados de cada questão. O objetivo central é demonstrar o pipeline completo de pré-processamento — diagnóstico, integração, limpeza, criação de atributos, transformações e visualizações analíticas — aplicado a dados reais com problemas reais.

---

## 2. Planejamento da Análise

### 2.1 Pipeline adotado

Antes de escrever qualquer código, o problema foi decomposto nas seguintes etapas sequenciais:

```
Entendimento do domínio
        ↓
Diagnóstico das bases (estado bruto)
        ↓
Definição das estratégias de limpeza
        ↓
Integração das três fontes
        ↓
Engenharia de atributos
        ↓
Transformações e normalização
        ↓
Agregações analíticas
        ↓
Visualizações (P1–P10)
        ↓
Reflexão crítica
```

### 2.2 Decisões de engenharia antes de iniciar

**Isolamento do ambiente:** foi criado um ambiente virtual Python (`.venv`) com versões fixadas em `requirements.txt`. Isso garante que qualquer pessoa que clone o repositório reproduza exatamente os mesmos resultados, eliminando problemas de compatibilidade de biblioteca.

**Imutabilidade dos dados brutos:** os arquivos CSV originais foram copiados para `data/raw/` e nunca modificados. Todas as transformações operam sobre cópias (`df.copy()`). Isso segue o princípio de auditabilidade: sempre é possível rastrear o dado original.

**Separação entre dado processado e original:** o arquivo `base_integrada_Grupo06.csv` em `data/processed/` representa o resultado do processamento, não substitui a fonte.

**Carregamento correto dos arquivos:** os CSVs usam separador `|` (pipe) e encoding `latin1`. Descoberto empiricamente ao verificar que a vírgula padrão do `pd.read_csv` produzia apenas uma coluna. O encoding `latin1` foi necessário para preservar caracteres portugueses (ã, ç, é).

---

## 3. Diagnóstico das Bases (Parte 1)

### 3.1 Descrição dos atributos

#### Base Prova (42 registros, 23 colunas)

| Atributo | Tipo (Tan) | Discreto/Contínuo | Observação |
|---|---|---|---|
| Código | Nominal | Discreto | Identificador do aluno |
| Turma | Nominal | Discreto | T1 ou T2 |
| Término | Nominal | Discreto | String HH:MM — requer parse |
| Q1–Q15 | Nominal binário | Discreto | 1=acerto, 0=erro, NaN=pulou |
| Erros | Razão | Discreto | Contagem de erros |
| Nota Questões | Razão | Contínuo | Escala 0–21 pts |
| Faltas | Razão | Discreto | Faltas de aula (não de questão) |
| %CH / %AU | Razão | Contínuo | Percentuais de frequência |

#### Base Formulário (40 registros, 35 colunas)

| Atributo | Tipo (Tan) | Discreto/Contínuo | Observação |
|---|---|---|---|
| ID | Nominal | Discreto | Liga ao Código da prova |
| Turma | Nominal | Discreto | "Turma 1" / "Turma 2" (formato diferente) |
| Tempo suficiente | Nominal | Discreto | Percepção subjetiva |
| Dificuldade geral | Ordinal | Discreto | Escala qualitativa |
| Nota esperada | Razão | Contínuo | Texto livre — requer parse |
| Horas de estudo | Razão | Contínuo | Texto livre — requer parse |
| Simulado 01/02/03 | Nominal | Discreto | Categórico multi-nível |
| Compreensão por tema (×15) | Ordinal | Discreto | 3 níveis: Não entendi / Teoria / Facilidade |

#### Base Questões (15 registros, 4 colunas)

| Atributo | Tipo (Tan) | Discreto/Contínuo | Observação |
|---|---|---|---|
| questao | Nominal | Discreto | Q1–Q15 |
| assunto | Nominal | Discreto | Tema da questão |
| simulado | Ordinal | Discreto | Simulado de origem (1, 2 ou 3) |
| eh_codigo | Nominal binário | Discreto | 1=questão de código, NaN=não é |

### 3.2 Problemas de qualidade identificados

| # | Base | Atributo | Tipo de problema | Registros afetados |
|---|---|---|---|---|
| 1 | Prova | Q1–Q15 | Valores ausentes (questões puladas intencionalmente) | 7–11 por coluna |
| 2 | Prova | Q7 (Código 36116) | Valor inválido: `11` em vez de `0` ou `1` | 1 |
| 3 | Prova | Nota Questões | Separador decimal vírgula em vez de ponto | 42 (todos) |
| 4 | Prova | Término (Código 54436) | Horário 15:51 para T2 (início 16:35) → duração negativa | 1 |
| 5 | Prova | Término (Código 33124) | Duração implausível: ~23 min para T2 | 1 |
| 6 | Prova | Faltas | Células vazias | 4 |
| 7 | Formulário | ID 36612 | Linha duplicada exata | 1 par (2 linhas) |
| 8 | Formulário | horas_estudo | Texto livre heterogêneo ("3h", "fim de semana", "poucas") | 41 |
| 9 | Formulário | nota_esperada | Texto livre; 1 resposta narrativa longa; vírgula/ponto mistos | 41 |
| 10 | Cruzamento | ID / Código | 3 alunos em Prova sem Formulário (32484, 34740, 35260) | 3 |
| 11 | Questões | eh_codigo | NaN ambíguo — ausência de valor significa "não é código" | 8 de 15 |

---

## 4. Pré-processamento (Parte 2)

### 4.1 Decisão fundamental: NaN em Q1–Q15

**Pergunta:** os campos vazios em Q1–Q15 representam valores ausentes ou uma categoria legítima?

**Análise:** o enunciado da prova permitia explicitamente que cada aluno deixasse até 2 questões sem responder, como estratégia. Pular uma questão é uma ação ativa, não uma falta de dado. Isso a diferencia, por exemplo, de um campo de endereço não preenchido.

**Decisão: preservar como NaN.**

**Consequência técnica:**
- Taxas de acerto por questão usam `.mean()`, que ignora NaN automaticamente — o denominador é o número de alunos que **tentaram** a questão, não o total.
- O atributo `taxa_acerto` individual usa como denominador `n_respondidas` (questões efetivamente respondidas), não 15.
- **Se tivéssemos substituído NaN por 0**, estaríamos penalizando artificialmente alunos que usaram a estratégia de skip, e inflando a taxa de erro das questões mais puladas.

Esta foi a decisão de pré-processamento mais difícil e com maior impacto sobre os resultados de P2 e P3.

---

### 4.2 Limpeza: base Prova

**Problema 1 — Separador decimal (Nota Questões):**
O campo armazena `11,2` em vez de `11.2`. Origem provável: exportação de sistema com locale brasileiro. Solução: `.str.replace(',', '.')` antes do cast para `float`. Decisão simples, mas necessária antes de qualquer operação numérica.

**Problema 2 — Valor inválido Q7 do aluno 36116:**
O campo registra `11` em vez de `0` ou `1`. Para determinar o valor correto, usamos verificação cruzada com os outros campos do mesmo registro:
- Erros = 5 (contagem de respostas erradas)
- Nota = 11,2
- A fórmula de pontuação da prova é: `nota = acertos × 1,4` (cada questão vale 21/15 ≈ 1,4 pts)
- Contando acertos com Q7 = 1: Q1=1, Q2=1, Q4=1, Q5=1, Q7=1, Q10=1, Q11=1, Q15=1 → 8 acertos
- `8 × 1,4 = 11,2` ✓ e erros = 5 ✓
- Com Q7 = 0: seria 7 acertos → `7 × 1,4 = 9,8` ≠ 11,2 ✗

**Decisão: substituir 11 por 1.** Trata-se de typo (tecla pressionada duas vezes). A consistência interna do registro confirma.

**Problema 3 — Término inconsistente (alunos 54436 e 33124):**
O aluno 54436 está em T2 (início 16:35) mas registra término às 15:51 — 44 minutos antes da prova começar. O aluno 33124 registra 16:12 — 23 minutos após o início, duração implausível para uma prova de 15 questões.

**Decisão: `tempo_prova_min = NaN` para os dois.** Não é possível corrigir sem conhecer o valor verdadeiro. A remoção do atributo apenas para esses dois registros é preferível a imputar um valor inventado. Os demais 40 alunos têm `tempo_prova_min` válido.

---

### 4.3 Limpeza: base Formulário

**Remoção de duplicata (ID 36612):**
As duas linhas são idênticas em todos os 35 campos. `drop_duplicates()` remove uma. Resultado: 40 → 39 registros.

**Parsing de `horas_estudo`:**
Este é um campo textual de alta variabilidade. Exemplos reais encontrados:
- `"3h"`, `"4 horas"`, `"9"` → extração numérica direta
- `"fim de semana todo, umas 8 horas no mínimo"` → convencionar 8h
- `"durante 1 dia somente"` → convencionar 8h (estimativa de dia de estudo)
- `"poucas"` → convencionar 2h (estimativa conservadora)
- `"considerando somente a véspera, de 10 a 15 horas"` → ponto médio: 12,5h
- `"não sei. acho que mais de 5h por dia durante a última semana"` → extrair primeiro número: 5h

**Estratégia técnica:** função `parse_horas()` com regex `\d+(?:[.,]\d+)?` para capturar números inteiros e decimais, com regras especiais (`if 'poucas'`, `if 'fim de semana'`, `if 'dia'`) para os casos narrativos. Para faixas ("de 10 a 15"), calculamos o ponto médio.

**Limitação reconhecida:** as horas de estudo são todas auto-reportadas e altamente subjetivas. O flag `horas_estimada` não foi incluído no produto final, mas a imprecisão do dado explica em parte a correlação quase nula com a nota (r = −0,06 em P4).

**Parsing de `nota_esperada`:**
Semelhante ao anterior. O único caso especial foi um aluno que escreveu uma resposta narrativa de 3 linhas: `"Entre 8 a 14. Muito por conta de..."` — extraímos o ponto médio (11,0 pts).

**Mapeamento de compreensão por tema:**
Três níveis ordinais convertidos para numérico:
```
"Não entendi bem"        → 1
"Entendi apenas a teoria" → 2
"Entendi com facilidade"  → 3
```
**Justificativa:** o mapeamento 1-2-3 preserva a distância ordinal uniforme entre os níveis, adequada para calcular médias (`media_compreensao`) e usar em visualizações comparativas.

**Binarização de simulados:**
Cada simulado tinha 5 respostas possíveis (não fiz / parcialmente sozinho / parcialmente com IA / tudo sozinho / tudo com IA). Criamos `fez_simulado_0X` (0/1) para simplificar a análise de participação. A distinção "sozinho vs IA" foi preservada nas colunas originais para análises futuras.

**Tratamento dos 3 alunos sem formulário (32484, 34740, 35260):**
A integração usa `merge(..., how='left')`, mantendo todos os 42 alunos da base Prova. Os três ficam com NaN em todas as colunas do formulário. Eles participam das análises baseadas apenas na prova (P1, P2, P8) mas são excluídos automaticamente das análises que requerem dados do formulário (P4, P5, P6) via `.dropna()`.

---

### 4.4 Limpeza: base Questões

Único tratamento necessário: `eh_codigo` com NaN foi preenchido com `0`. A semântica é clara: ausência do marcador significa que a questão **não é de código**.

---

### 4.5 Integração

```
df = df_prova  (42 registros)
         ↓  LEFT JOIN em Código = ID
df_form  (39 registros)  →  3 alunos ficam com NaN nas colunas do formulário

df_questoes  →  usado como lookup table nas agregações (não expandido em colunas)
```

**Por que left join e não inner join?** Um inner join eliminaria os 3 alunos sem formulário. Perderíamos informação real de desempenho (nota, respostas por questão) por causa de um dado de suporte ausente. O left join preserva todos os dados da prova e deixa explícito onde a informação é incompleta.

---

### 4.6 Atributos criados (Parte 2.3)

| Atributo | Fórmula | Tipo resultante | Justificativa |
|---|---|---|---|
| `tempo_prova_min` | `Término − início_turma` em minutos | Razão, contínuo | Proxy de gestão de tempo; início confirmado: T1=14:40, T2=16:35 |
| `taxa_acerto` | `n_certas / n_respondidas` | Razão, contínuo [0,1] | Desempenho normalizado; denominador = questões tentadas, não 15 |
| `media_compreensao` | `mean(comp_KDD ... comp_SimObjHet)` | Razão, contínuo [1,3] | Resume 15 colunas ordinais em um único índice de autoavaliação |
| `simulados_realizados` | `sum(fez_simulado_01..03)` | Razão, discreto [0,3] | Profundidade de preparação prévia |
| `taxa_acerto_codigo` | `n_código_certas / n_código_respondidas` | Razão, contínuo [0,1] | Isola desempenho nas questões que exigem leitura/análise de código |
| `taxa_acerto_nao_codigo` | `n_conceitual_certas / n_conceitual_respondidas` | Razão, contínuo [0,1] | Complemento — permite comparação direta entre os dois tipos |
| `erro_esperado` | `nota_esperada − Nota Questões` | Intervalo, contínuo | Positivo = superestimou; negativo = subestimou |

**Decisão sobre `tempo_prova_min`:** dois alunos (54436 e 33124) geraram durações negativas ou implausíveis. Filtramos valores fora de [20, 180] minutos com `NaN`. A regra de corte inferior (20 min) é justificada: seria impossível ler e responder 13 questões em menos de 20 minutos.

---

### 4.7 Transformações (Parte 2.4)

#### Discretização

**`nota_faixa`** — método de igual largura baseado em conhecimento de domínio:
```
[0,   7)    → Muito Baixa
[7,  11.2)  → Baixa
[11.2, 14)  → Média
[14, 18.2)  → Alta
[18.2, 21]  → Excelente
```
O limite 11,2 foi escolhido porque corresponde à mediana da turma (11,2 pts) e a um acerto exato de 8 questões — ponto natural de corte no sistema de pontuação (8 × 1,4 = 11,2).

**`horas_faixa`** — método de igual frequência (`pd.qcut`, tercis):
Para `horas_estudo`, a distribuição é irregular e com outliers (até 30h). A discretização por frequência coloca igual número de alunos em cada faixa, tornando a comparação entre grupos mais equilibrada.

**Por que métodos diferentes?** Notas têm semântica pedagógica natural (aprovado/reprovado), então a largura fixa com cortes significativos faz sentido. Horas de estudo têm distribuição assimétrica e sem cortes naturais, então tercis garantem tamanhos de grupo comparáveis.

#### Normalização

Min-max aplicada a todos os atributos numéricos utilizados nas visualizações. Fórmula:
```
x_norm = (x − min) / (max − min)
```
**Motivação:** os atributos têm escalas muito diferentes (nota: 0–21; horas: 0–30; %CH: 0–100). Sem normalização, qualquer visualização de radar ou análise multivariada seria dominada pelos atributos de maior amplitude.

#### Binarização

- `turma_T1`: 1 se T1, 0 se T2 — permite uso em modelos e comparações binárias
- `revisou_prova_bin`: 1 se revisou, 0 caso contrário
- `fez_simulado_0X`: já criado na limpeza do formulário

---

### 4.8 Agregações (Parte 2.5)

| Agregação | Agrupamento | Métrica | Uso |
|---|---|---|---|
| `agg_questao` | Por questão (Q1–Q15) | Taxa de acerto, respondentes | P2, P8 |
| `agg_assunto` | Por assunto | Taxa média, n questões | P3 |
| `agg_turma` | Por turma (T1/T2) | Nota média/mediana, horas, simulados | P1, P7 |
| `agg_tipo_questao` | Código vs. conceitual | Taxa média | P2 (anotações) |

---

## 5. Visualizações Analíticas (Parte 3)

### P1 — Distribuição das Notas

**Gráficos:** histograma + box plot, lado a lado, estratificados por turma.

**Por que dois gráficos?** O histograma mostra a *forma* da distribuição (simetria, bimodalidade, caudas). O box plot mostra *comparação de posição* (mediana, IQR, outliers) entre turmas de forma compacta. Um único gráfico não capturaria os dois aspectos com igual clareza.

**Achados:**
- Nota média: T1 = 10,78 pts, T2 = 11,77 pts — T2 ligeiramente superior
- Mediana idêntica nas duas turmas: 11,2 pts (equivalente a 8 acertos exatos)
- T1 estudou mais em média (9,75h vs 6,62h de T2) mas teve nota menor — coerente com a correlação fraca entre horas e nota (P4)
- Distribuição aproximadamente simétrica em torno da mediana; sem outliers extremos

---

### P2 — Desempenho por Questão

**Gráfico:** barras horizontais ordenadas da menor para a maior taxa de acerto, coloridas por tipo (azul = conceitual, laranja = código), com anotações nas questões com armadilhas técnicas.

**Decisão de design:** a orientação horizontal facilita a leitura dos rótulos longos. A codificação dupla (cor por tipo + posição por dificuldade) permite ver simultaneamente onde cada tipo de questão se concentra na escala de dificuldade.

**Achados:**

| Classificação | Questão | Assunto | Taxa | Tipo | Explicação da dificuldade |
|---|---|---|---|---|---|
| Mais difícil | Q12 | Similaridade entre Objetos | 25,8% | Conceitual | Armadilha: opção errada usava dissimilaridade como similaridade para atributo razão |
| 2ª mais difícil | Q6 | Normalização (código) | 31,4% | Código | Armadilha: bins `[0,200,500,1000,∞]` parecem largura igual, mas são por domínio |
| 3ª mais difícil | Q14 | Seleção features (código) | 39,5% | Código | Requer conhecer a limitação do Pearson: apenas relações lineares |
| 4ª mais difícil | Q13 | Correlação Pearson | 43,2% | Conceitual | Y = X + 10 → r = 1,0; muitos escolheram cosseno incorretamente |
| Mais fácil | Q10 | Viés algorítmico | 92,9% | Conceitual | Questão conceitual com alternativas erradas evidentemente incorretas |

**Questões de código vs. conceituais:**
- Código: média 57,0% de acerto
- Conceitual: média 64,2% de acerto
- As questões de código foram ligeiramente mais difíceis, mas Q12 e Q13 (conceituais) estão entre as três mais difíceis — a dificuldade dependeu mais da precisão das alternativas do que do tipo de questão.

---

### P3 — Desempenho por Assunto vs. Percepção de Compreensão

**Gráfico:** barras duplas com eixo Y duplo — eixo esquerdo para taxa de acerto real (azul), eixo direito para compreensão média declarada (laranja, escala 1–3).

**Por que eixo duplo?** As duas métricas têm escalas incompatíveis (0–1 vs 1–3) mas queremos compará-las visualmente para o mesmo assunto. Um eixo duplo, quando bem rotulado, é a solução mais direta.

**Achados:**
- **Normalização** teve dois comportamentos opostos: Q4 (61%) e Q6 (31%). Alunos que declararam entender normalização erraram Q6 — porque a questão testou a *classificação do tipo* de discretização, não o conceito de normalização em si.
- **Similaridade entre Objetos** (Q12, 25,8%) é o candidato mais provável de maior divergência entre compreensão declarada e desempenho — o cálculo exige combinar quatro tipos de atributos com fórmulas distintas sob pressão de prova.
- **KDD** (Q15, 75,6%) e **Viés algorítmico** (Q10, 92,9%) são os assuntos mais alinhados: conceituais, com alta declaração de compreensão e alto acerto.

---

### P4 — Relação entre Horas de Estudo e Desempenho

**Gráfico:** scatter plot com linha de regressão linear, pontos coloridos por turma.

**Achado central: r = −0,06 (correlação praticamente nula)**

Este é o resultado mais surpreendente do trabalho. Algumas interpretações possíveis:
1. **Qualidade vs. quantidade:** estudar mais horas sem método eficaz não garante resultado.
2. **Viés no auto-relato:** as horas foram declaradas retrospectivamente e de forma vaga ("fim de semana todo", "não sei"). A imprecisão do dado pode mascarar uma correlação real.
3. **Efeito de teto/chão:** a prova tem dificuldade homogênea suficiente para que tanto alunos que estudaram pouco quanto os que estudaram muito ficassem em faixas similares.

---

### P5 — Percepção vs. Realidade

**Gráfico:** scatter plot com diagonal y = x como referência de estimativa perfeita.

**Achados:**
- **Correlação moderada:** r = 0,534 — quem foi bem tendeu a esperar notas maiores, mas não com precisão
- **Viés sistemático de +1,76 pts:** o offset positivo não é ruído — é um padrão coletivo. A mediana do erro é +2,0 pts.
- **77% superestimou** a própria nota; apenas 23% subestimou
- **0% acertou exatamente** a própria nota
- **Casos extremos:** um aluno superestimou em +8,4 pts; outro subestimou em −9,2 pts (provável caso de ansiedade de prova ou baixa autoconfiança)

**Interpretação:** a assimetria do erro (77% vs 23%) indica que os alunos não perceberam seus erros durante a prova, ou que a confiança na hora do exame era sistematicamente maior que o desempenho real.

---

### P6 — Perfil dos Alunos com Melhor Desempenho

**Gráfico:** barras agrupadas comparando Top 25% (nota ≥ 12,6 pts) vs. Restante 75%.

**Decisão de corte:** o quartil superior (75%) foi escolhido como critério de "melhor desempenho" por ser um limiar estatístico objetivo, não arbitrário.

**Análise:** o gráfico revela em quais características os top performers diferem do restante — horas de estudo, simulados realizados, compreensão média e taxa de acerto em questões de código.

---

### P7 — Radar: Perfil Médio por Turma

**Gráfico:** radar (Plotly `go.Scatterpolar`) comparando T1 e T2 em cinco dimensões normalizadas: taxa de acerto, horas de estudo, compreensão média, simulados realizados, nota esperada.

**Por que radar?** Cinco dimensões em barras separadas exigiria cinco gráficos distintos, perdendo a visão integrada. O radar permite comparar simultaneamente o "perfil" de cada turma em uma única figura, tornando imediatamente visível em quais dimensões diferem.

**Normalização obrigatória:** as cinco dimensões têm escalas incompatíveis. Antes de plotar, cada atributo foi normalizado min-max para [0,1] dentro da tabela `agg_turma`, garantindo que nenhuma dimensão domine visualmente por ter escala maior.

---

### P8 — Heatmap: Aluno × Questão

**Gráfico:** heatmap 42×15 com verde = acerto, vermelho = erro, cinza = pulou. Alunos ordenados por nota crescente (eixo Y), questões ordenadas por taxa de acerto crescente (eixo X).

**Por que este ordenamento?** Colocar os alunos mais fracos no topo e as questões mais difíceis à esquerda cria um gradiente visual natural: o canto superior esquerdo é o "pior cenário" (alunos fracos nas questões difíceis) e o canto inferior direito é o "melhor cenário". Padrões de cluster ficam imediatamente visíveis.

**O que o gráfico revela que nenhum outro consegue:** em quais questões os alunos de nota alta também erraram (questões objetivamente difíceis para todos) versus em quais questões só os alunos fracos erraram (questões discriminatórias). Também evidencia os padrões de skip — se as questões mais puladas são as mais difíceis ou se há estratégia independente da dificuldade.

---

### P9 — Simulados por Faixa de Nota

**Gráfico:** barras empilhadas a 100%, mostrando a distribuição de `simulados_realizados` (0, 1, 2 ou 3) dentro de cada faixa de nota.

**Decisão técnica:** normalizar por linha (não por coluna) permite ver a *composição de cada faixa de nota*. Se alunos que fizeram todos os simulados se concentram nas faixas superiores, isso aparece como uma barra azul escura dominando o lado direito do gráfico.

---

### P10 — Clustermap de Correlações

**Gráfico:** `seaborn.clustermap` com 11 atributos numéricos, hierarquicamente agrupados por similaridade de correlação.

**Por que clustermap em vez de heatmap simples?** O heatmap comum exibe as correlações na ordem original das colunas. O clustermap usa dendrograma hierárquico (scipy) para reordenar linhas e colunas de forma que atributos correlacionados entre si fiquem adjacentes — revelando grupos naturais (ex.: métricas de desempenho, métricas de preparação, métricas de autoavaliação) sem que o analista precise especificá-los.

---

## 6. Reflexão Crítica (Parte 4)

### 6.1 Decisão de pré-processamento mais difícil

O tratamento dos NaN em Q1–Q15 foi a decisão de maior impacto. A escolha entre "pular = errar (→ 0)" e "pular = categoria própria (→ NaN)" muda materialmente as taxas de acerto por questão e a taxa individual de cada aluno. Optamos por preservar NaN porque a regra da prova torna o skip uma decisão ativa e legítima do aluno. Tratar como erro seria introduzir viés contra a estratégia que o próprio enunciado da prova incentivava.

### 6.2 Insights surpreendentes

**1. Horas de estudo não prevê nota (r = −0,06)**
A correlação quase nula entre horas de estudo e nota contradiz a expectativa natural. Isso pode indicar que a qualidade do estudo importa mais que a quantidade, que o auto-relato de horas é pouco confiável, ou que a dificuldade da prova nivelou os resultados independentemente da preparação.

**2. 77% dos alunos superestimaram a nota com viés sistemático de +1,76 pts**
Ninguém acertou a própria nota exatamente. O erro não é aleatório — é consistentemente positivo, com mediana de +2,0 pts. Isso revela um viés coletivo de superconfiança: os alunos sentiram mais segurança durante a prova do que o desempenho real justificava.

**3. Q12 separou quem memorizou de quem compreendeu**
A questão mais difícil da prova (25,8% de acerto) tinha sua armadilha em uma diferença de uma linha: a alternativa errada mais atraente usava `|0,7 − 0,4| = 0,3` como *similaridade* do atributo razão normalizado — quando 0,3 é a *dissimilaridade*. A resposta correta exigia `sim = 1 − 0,3 = 0,7`. Quem decorou o cálculo errou; quem entendeu o conceito de que similaridade e dissimilaridade são complementares acertou.

### 6.3 Dados adicionais desejados

1. **Horário de início individual** por aluno: usamos o início da turma como proxy, gerando NaN para 2 alunos com `Término` inconsistente. Com o horário exato, `tempo_prova_min` seria preciso para todos.

2. **Histórico acadêmico prévio (CRA ou notas anteriores)**: permitiria controlar a capacidade prévia do aluno e isolar o efeito real do estudo sobre o desempenho, respondendo à pergunta de P4 com mais rigor.

3. **Tempo gasto por questão**: o tempo total não distingue estratégias. Dados granulares por questão revelariam onde os alunos hesitaram mais e se o tempo investido em uma questão aumenta a chance de acerto.

---

## 7. Estrutura do Repositório

```
ds-assignment/
├── data/
│   ├── raw/                              ← CSVs originais (somente leitura)
│   │   ├── TP01_-_Bases_de_dados_-_prova.csv
│   │   ├── TP01_-_Bases_de_dados_-_formulario.csv
│   │   └── TP01_-_Bases_de_dados_-_questoes.csv
│   └── processed/
│       ├── base_integrada_Grupo06.csv    ← base tratada e integrada
│       └── P1–P10_*.png                 ← exportações das visualizações
├── TP01_Grupo06.ipynb                    ← notebook principal (reproduzível)
├── relatorio_TP01_Grupo06.md             ← este documento
├── requirements.txt                      ← dependências fixadas
└── .venv/                               ← ambiente virtual Python
```

**Para reproduzir:**
```bash
source .venv/bin/activate
jupyter notebook TP01_Grupo06.ipynb
```
O notebook executa de cima a baixo sem intervenção manual, a partir dos CSVs em `data/raw/`.

---

## 8. Decisões Técnicas Resumidas

| Decisão | Alternativa considerada | Escolha feita | Razão |
|---|---|---|---|
| NaN em Q1–Q15 | Substituir por 0 | Preservar NaN | Skip é ação ativa permitida pela prova |
| Join prova+formulário | Inner join | Left join | Preservar todos os 42 alunos da prova |
| Q7=11 do aluno 36116 | Tratar como NaN | Corrigir para 1 | Verificação cruzada com Erros e Nota confirma |
| Término inválido (2 alunos) | Imputar por mediana | NaN | Não há base para estimar o valor real |
| Discretização de nota | Igual frequência | Igual largura com cortes pedagógicos | 11,2 pts é ponto natural (8 acertos) |
| Discretização de horas | Igual largura | Igual frequência (tercis) | Distribuição assimétrica; tercis equilibram grupos |
| Normalização geral | Z-score | Min-max | Dados sem distribuição normal definida; min-max é intuitivo |
| P10 – heatmap | Heatmap simples | Clustermap | Agrupamento hierárquico revela clusters naturais sem hipótese prévia |
| Scipy | Não instalar | Instalar como dependência | Necessário para `seaborn.clustermap` |
