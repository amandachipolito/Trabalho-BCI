# Projeto BCI — Pipeline Completo de Processamento de EEG

## Visão Geral

Este projeto implementa um pipeline completo de processamento e análise de sinais EEG utilizando o dataset **Covert Shifts of Attention (005-2015)**. O objetivo principal é analisar sinais associados à atenção encoberta (covert attention) no intervalo temporal entre a apresentação do **cue** e do **target**, utilizando técnicas clássicas de processamento de sinais e análise estatística.

O pipeline foi estruturado para ser reprodutível, auditável e colaborativo, permitindo que diferentes desenvolvedores trabalhem simultaneamente em etapas específicas sem comprometer a consistência do fluxo geral.

---

# Objetivo Técnico

O pipeline executa as seguintes operações principais:

- leitura dos arquivos `.mat`
- filtragem dos sinais EEG
- segmentação temporal baseada em eventos
- padronização estatística
- redução dimensional com PCA
- decomposição com ICA
- análise dos componentes independentes
- extração de características
- modelagem e avaliação

---

# Organização dos Dados

Os arquivos do dataset devem ser armazenados localmente na pasta:

Projeto_BCI/

Os arquivos `.mat` **não devem ser versionados**, pois possuem grande volume.

O arquivo `.gitignore` deve conter:

*.mat  
Projeto_BCI/  
.ipynb_checkpoints/  
__pycache__/  
*.pyc  
*.swp  
*.tmp  
*.whl  

---

# Pipeline Completo

## 1 — Preparação do Ambiente

Nesta etapa são instaladas e importadas as bibliotecas necessárias:

- numpy  
- scipy  
- matplotlib  
- sklearn  
- pathlib  
- glob  
- os  

Essas bibliotecas permitem manipulação numérica, leitura de arquivos `.mat`, filtragem digital, visualização gráfica e análise estatística.

---

## 2 — Carregamento dos Dados

Cada arquivo `.mat` contém:

- sinais EEG contínuos  
- frequência de amostragem  
- marcações temporais dos trials  
- labels dos cues  
- informações sobre targets  

Estrutura principal:

data.X → matriz EEG  
data.fs → frequência de amostragem  
data.trial → início dos trials  
data.y → labels dos cues  
mrk.target_location → dados dos targets  

Formato típico:

X.shape = (amostras, canais)

Exemplo:

(579736, 62)

Isso indica 62 canais EEG e centenas de milhares de amostras temporais.

---

## 3 — Verificação da Frequência de Amostragem

Foi confirmado que:

fs = 200 Hz

Isso significa:

200 amostras por segundo.

Essa informação é essencial para converter índices em tempo real.

---

## 4 — Filtragem dos Sinais EEG

Os sinais passam por dois filtros principais:

### Notch Filter — 60 Hz

Remove ruído da rede elétrica.

### Bandpass Filter — 5–40 Hz

Mantém frequências relevantes do EEG:

- Theta  
- Alpha  
- Beta  

Remove:

- drift lento (<5 Hz)  
- ruído muscular (>40 Hz)

O formato da matriz permanece:

(amostras, canais)

---

## 5 — Inspeção Visual Inicial

Alguns canais são plotados para verificar:

- estabilidade do sinal  
- ausência de saturação  
- comportamento oscilatório esperado  

Essa etapa permite detectar erros estruturais precocemente.

---

## 6 — Inspeção dos Trials

Cada trial representa um evento experimental.

Estrutura:

trial → posição temporal do cue  
labels → classe do cue  

Número típico:

600 trials

---

## 7 — Análise do Delay Cue → Target

Foi calculado o intervalo temporal entre:

cue → target

Resultados obtidos:

Delay médio ≈ 1.62 s  
Delay mínimo ≈ 0.645 s  
Delay máximo ≈ 2.17 s  

Esse intervalo representa o período em que o participante mantém atenção encoberta.

---

## 8 — Criação das Epochs

Os sinais são segmentados no intervalo:

cue → target

Cada trial gera uma epoch EEG.

Número total:

600 epochs

Formato inicial:

(amostras_variáveis, canais)

---

## 9 — Padronização do Tamanho das Epochs

Como os delays variam, foi adotada a menor janela segura:

129 amostras ≈ 0.645 s

Formato final:

(600, 129, 62)

Isso garante consistência estrutural entre trials.

---

## 10 — Conversão para Matriz 2D

As epochs são convertidas de:

(600, 129, 62)

para:

(600, 7998)

Cada trial passa a ser representado por um vetor de features.

---

## 11 — Padronização Estatística

Aplicação de:

StandardScaler

Objetivo:

média = 0  
desvio padrão = 1  

Resultado:

X_padronizado.shape = (600, 7998)

---

## 12 — PCA Inicial

Aplicação do PCA completo para avaliar a variância explicada.

Resultado:

X_pca.shape = (600, 600)

---

## 13 — Análise da Variância Explicada

São gerados:

- Scree Plot  
- Variância acumulada  

Objetivo:

Determinar o número ideal de componentes.

---

## 14 — PCA Reduzido

Critério adotado:

90% da variância acumulada

Resultado:

Número de componentes = 184  

X_pca_reduzido.shape = (600, 184)

Redução:

7998 → 184 dimensões

---

## 15 — Verificação Pós-PCA

Foram verificadas:

- dimensionalidade reduzida  
- variância preservada  
- média das componentes  
- variância das componentes  
- condição necessária para ICA  

Condição validada:

n_features < n_amostras  

184 < 600  

Variância preservada:

≈ 90%

---

## 16 — Aplicação de ICA

Foi aplicada:

FastICA

Entrada:

X_pca_reduzido.shape = (600, 184)

Saída esperada:

X_ica.shape = (600, 184)

Objetivo:

Separar componentes estatisticamente independentes.

---

## 17 — Análise dos Componentes Independentes

Etapas previstas:

- inspeção visual dos ICs  
- cálculo de kurtosis  
- análise de distribuição  
- identificação de possíveis artefatos  
- avaliação da independência estatística  

---

## 18 — Extração de Características

Possíveis características:

- energia por banda  
- potência espectral  
- métricas temporais  
- métricas estatísticas  
- características derivadas do ICA  

---

## 19 — Modelagem e Avaliação

Possíveis abordagens:

- classificação supervisionada  
- análise temporal da atenção  
- comparação entre pipelines  

Métricas possíveis:

- accuracy  
- matriz de confusão  
- F1-score  

---

# Reprodutibilidade

Para reproduzir o pipeline:

1. Baixar os arquivos `.mat`  
2. Colocar na pasta:

Projeto_BCI/

3. Executar:

main.ipynb

---


# Boas Práticas de Versionamento

Fluxo recomendado:

git pull origin main  
git add main.ipynb  
git commit -m "Descrição clara da alteração"  
git push origin main  

Evitar:

git add .

quando houver dados grandes no diretório.

Sempre adicionar arquivos específicos quando possível.