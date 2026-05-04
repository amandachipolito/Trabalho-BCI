# Projeto BCI — Covert Shifts of Attention

Este repositório contém o pipeline de análise para o dataset **Covert Shifts of Attention (BNCI 005-2015)**, cujo objetivo principal é analisar sinais associados à atenção encoberta (covert attention) no intervalo temporal entre a apresentação do **cue** e do **target**, utilizando técnicas clássicas de processamento de sinais e análise estatística. Neste projeto o foco da análise foi a detecção de modulações da banda alfa no EEG associadas à atenção visual encoberta.

O pipeline foi estruturado para ser reprodutível, auditável e colaborativo, permitindo que diferentes desenvolvedores trabalhem simultaneamente em etapas específicas sem comprometer a consistência do fluxo geral.

## Pergunta Operacional do Projeto

> É possível estimar a direção da atenção visual encoberta a partir de modulações da banda alfa no EEG?

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


## Organização do Notebook main e Abordagens Principais

O notebook `main.ipynb` explora duas abordagens distintas para responder à pergunta do projeto:

1.  **Baseline Espectral:** Utiliza a Densidade Espectral de Potência (PSD) via método de Welch, extraindo a potência na banda alfa e classificando com um Support Vector Machine (SVM).
2.  **Modelo Espacial:** Emprega o Common Spatial Patterns (CSP) para extrair características espaciais que maximizam a separação entre classes, seguido por um SVM para classificação.

A comparação entre essas abordagens é crucial, pois a expectativa é que a informação espacial do EEG, capturada pelo CSP, melhore a discriminação das direções de atenção visual.

OBS: em todas as aplicações etapas de inspeção visual dos dados via plotagem de alguns canais são implementadas para desectar erros estruturais precocemente, buscando verificar:
- estabilidade do sinal  
- ausência de saturação  
- comportamento oscilatório esperado
  
## Instalação e Configuração

Para replicar os resultados, é necessário instalar as seguintes bibliotecas Python. O script de instalação detecta automaticamente se o ambiente é Google Colab ou local.

```bash
# Em ambiente Google Colab, a maioria é pré-instalada. O comando abaixo garante as dependências.
# Para ambientes locais, certifique-se de instalar as listadas.

pip install -q numpy scipy gdown matplotlib scikit-learn seaborn mne
```

### Importação das Bibliotecas

```python
import glob
import os
import sys
from pathlib import Path

import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

from scipy.io import loadmat
from scipy.signal import butter, filtfilt, iirnotch, sosfiltfilt, welch

from sklearn.preprocessing import StandardScaler
from sklearn.svm import SVC
from sklearn.model_selection import StratifiedKFold, GridSearchCV, cross_val_predict, cross_val_score
from sklearn.metrics import confusion_matrix

import mne
from mne.preprocessing import ICA
from mne.decoding import CSP
```

## Carregamento dos Dados

Os dados são arquivos `.mat` armazenados no Google Drive. O notebook baixa automaticamente a pasta se ela não estiver presente no diretório de trabalho.

```python
IN_COLAB = "google.colab" in sys.modules
DATA_DIR = Path("/content/Projeto_BCI" if IN_COLAB else "./Projeto_BCI").resolve()
GDRIVE_FOLDER_URL = "https://drive.google.com/drive/folders/1l2wfRKe3_xGU0otL7aIvcwXn578fGs7a?usp=sharing"

# O código no notebook verifica a existência da pasta e realiza o download se necessário.
# gdown.download_folder(GDRIVE_FOLDER_URL, output=str(DATA_DIR), quiet=False, use_cookies=False)
```

O dataset é composto por 8 arquivos `.mat`, cada um correspondendo a um participante, com sinais EEG amostrados a 200 Hz e 62 canais.

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

## Pré-processamento do EEG

O pipeline de pré-processamento inclui:

*   **Filtro Notch:** Remoção de ruído de linha de 60 Hz.
*   **Filtro Passa-banda:** Filtragem entre 8 e 14 Hz (banda alfa).
*   **Normalização (Z-score):** Aplicada canal a canal para padronizar as amplitudes.

## Eventos, Labels e Criação de Épocas

Cada trial representa um evento experimental (número típico: 600 trials), cuja estrutura contém:
- trial → posição temporal do cue  
- labels → classe do cue  

O delay é o intervalo temporal entre o 'cue' (indicação da direção da atenção) e o 'target'(estímulo visual), que representa o período em que o participante mantém atenção encoberta, sendo:
- Delay médio ≈ 1.62 s
- Delay mínimo ≈ 0.645 s
- Delay máximo ≈ 2.17 s

As épocas são padronizadas pelo menor `delay` encontrado entre `cue` e `target` para garantir uniformidade dos vetores para processamento posterior.

## Limpeza por ICA

Independent Component Analysis (ICA) é utilizada para remover artefatos. Componentes são identificados visualmente (e removidos, por exemplo, os componentes 0 e 1, que geralmente correspondem a piscadas oculares e movimentos) e as épocas são reconstruídas sem eles. Para tal, foi adotado o método "fastICA", que permite separar componentes estatisticamente independentes.


## Modelo Baseline: Welch + SVM

### Extração de Features

Para cada época, a Densidade Espectral de Potência (PSD) é calculada nos canais posteriores usando o método de Welch. A potência média na banda alfa (8-14 Hz) é extraída como feature para o classificador. As features são então transformadas em `log10` e padronizadas via z-score.

### Classificação e Avaliação

Um SVM é treinado e avaliado usando validação cruzada estratificada (5 folds). São realizadas otimizações separadas para classificação binária (e.g., Direção 0 vs Direção 1) e multiclasse (todas as 6 direções).

**Resultados:** O modelo Baseline (Welch + SVM) apresenta desempenho próximo ao nível de chance para a classificação multiclasse (~16.66% de acurácia, com algumas direções atingindo ~25% e outras abaixo de 20%), indicando que a potência alfa isolada por canal não é altamente discriminativa.

## Modelo Espacial: CSP + SVM

### Extração de Features

O Common Spatial Patterns (CSP) é aplicado às épocas limpas pela ICA para extrair componentes espaciais que maximizam a variância entre as classes (direções de atenção). São extraídos 10 componentes, e suas variâncias log-normalizadas servem como features para o SVM.

### Classificação e Avaliação

Similarmente ao modelo baseline, um SVM é treinado com as features do CSP e avaliado com validação cruzada. Otimização de hiperparâmetros é realizada para encontrar a melhor configuração do SVM.

**Resultados:** O modelo CSP + SVM demonstra uma melhora substancial em relação ao baseline, atingindo acurácias multiclasse em torno de 30-35%, com picos acima de 40% para algumas direções (e.g., Direção 0). Este resultado sugere que a informação espacial das modulações alfa é crucial para a discriminação das direções de atenção.

## Interpretação Técnica dos Resultados

*   **Resultado Principal:** O pipeline **CSP + SVM** supera significativamente o **Welch + SVM**, sugerindo que a informação discriminativa não reside apenas na potência alfa isolada de cada canal, mas na distribuição espacial e nos padrões de conectividade entre os eletrodos posteriores.
*   **Implicação:** A atenção visual encoberta produz padrões espaciais sutis no EEG que o CSP é capaz de realçar, atuando como um filtro espacial supervisionado que torna evidentes as diferenças entre as classes de atenção.

## Ponto Metodológico Importante

É fundamental notar que este dataset não é um SSVEP clássico com estímulos piscando em frequências fixas. Portanto, o trabalho deve ser descrito como uma análise de **modulações oscilatórias em banda alfa associadas à atenção visual encoberta**, utilizando uma lógica de processamento inspirada em BCIs visuais baseadas em frequência, mas sem a premissa de um SSVEP tradicional.

## Reprodutibilidade

Para reproduzir o pipeline:

1. Baixar os arquivos `.mat`  
2. Colocar na pasta:

Projeto_BCI/

3. Executar:

main.ipynb


## Licença

Esse projeto foi desenvolvido no âmbito da disciplina Interface Cérebro Máquina da graduação em Neurociência da UFABC e é de licença aberta. 

## Boas Práticas de Versionamento

Fluxo recomendado:

git pull origin main  
git add main.ipynb  
git commit -m "Descrição clara da alteração"  
git push origin main  

Evitar:

git add .

quando houver dados grandes no diretório.

Sempre adicionar arquivos específicos quando possível.
```
