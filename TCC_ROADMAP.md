# TCC: Design de Peptídeos Antimicrobianos contra o Complexo ABC de *E. coli* usando RFD4-Proteina

Este documento contém uma avaliação do framework, as opções de infraestrutura na nuvem e um roadmap detalhado de 5 dias para concluir o seu TCC utilizando o RFD4-Proteina.

## 1. Avaliação do Framework (RFD4-Proteina)

O RFD4-Proteina é um modelo fundacional avançado (flow-matching) focado no design biomolecular desenvolvido pelo Baker Lab (RosettaCommons) em parceria com a NVIDIA.

**O framework é o melhor para o seu caso?**
**Sim.** Ele é o estado-da-arte atual (junto com modelos como o RFdiffusion) para desenhar novos peptídeos e proteínas. Para o seu objetivo (peptídeos antimicrobianos que se liguem no complexo ABC de *E. coli*), você fará uso da tarefa de **Protein Binder Design** (design de ligantes). O modelo é capaz de realizar o co-design de estrutura e sequência do peptídeo simultaneamente, permitindo que você condicione a geração a "hotspots" (resíduos específicos de interação no alvo).

## 2. Infraestrutura e Custos (Google Colab e Alternativas)

**Vai funcionar no Google Colab?**
**Sim, perfeitamente.** O repositório oficial já fornece suporte explícito e tutoriais (`docs/examples/colab_quickstart.ipynb`) demonstrando o uso em instâncias gratuitas T4 do Google Colab. O tempo de execução de predições únicas é bem rápido, especialmente ao pular a etapa de compilação `torch.compile` e pular a avaliação das métricas (`~metrics`).

**Opções Gratuitas e de Baixo Custo:**
1. **Google Colab (Gratuito / Colab Pro):**
   - **Custo:** R$ 0. O Colab oferece GPUs T4 suficientes para rodar a etapa de geração (`rfd4-generate`). Se você precisar rodar lotes muito grandes rapidamente, o Colab Pro (~R$ 50/mês) permite acessar GPUs melhores como A100.
2. **Hugging Face Spaces (ZeroGPU):**
   - Existe uma interface oficial hospedada na Hugging Face ([rfd4-proteina-zerogpu](https://huggingface.co/spaces/nvidia/rfd4-proteina-zerogpu)) onde você pode carregar seu arquivo PDB/CIF do alvo e desenhar o peptídeo diretamente pelo navegador, usando a cota gratuita "ZeroGPU", sem precisar programar nada no momento inicial.
3. **Plataformas de API (Tamarind Bio / LatchBio):**
   - **Tamarind Bio e afins:** Estas plataformas são excelentes e oferecem pipelines "no-code" (sem código) baseados na nuvem (pay-per-use, muitas vezes dando créditos grátis iniciais). Embora sejam mais comumente usadas para **validação** (ex: rodar AlphaFold3, ColabFold, ESMFold) de forma fácil, elas costumam ser a opção de menor esforço. No seu fluxo, você pode gerar no Colab e usar essas APIs para validar a estrutura do peptídeo gerado com precisão.

## 3. Tutorial Completo para Rodar no Google Colab

Este tutorial ensina o "caminho feliz" para rodar tudo no Google Colab gastando zero.

1. **Crie o notebook no Colab:** Acesse `colab.research.google.com`, crie um novo notebook, vá em `Runtime > Change runtime type` e selecione **T4 GPU**.
2. **Instalação do Framework:**
   Crie uma célula e rode o código abaixo. Ele usará o instalador ultrarrápido `uv`.
   ```bash
   !pip install -q uv
   # Verifica a versão do CUDA do Colab e instala (geralmente cuda12)
   !uv pip install --system --torch-backend=auto rfd4-proteina[all-cuda12]
   ```
3. **Upload do Alvo e Preparação do Spec YAML:**
   Faça upload do arquivo cif da proteína alvo da *E. coli* (ex: `abc_complex.cif`). Crie uma célula de código Python para escrever o arquivo de configuração de ligante:
   ```python
   yaml_content = """
   abc_binder:
     input: abc_complex.cif
     # /0 significa que é uma nova cadeia (o peptídeo de 15 a 30 resíduos), seguido pela cadeia A do alvo (resíduos 1 a 200)
     contig: "15-30, /0, A1-200"
     conditions:
       C_CRDGRP: {1: [{select: "sel('A/*/*/[N,CA,C,O]')"}]} # Mantém as coordenadas do alvo
       C_SEQ: {true: [{select: "sel('A')"}]} # Mantém a sequência do alvo conhecida
       # HOTSPOTS: Altere para o número dos resíduos no Complexo ABC onde o peptídeo DEVE se ligar
       C_HOT: {true: [{select: "sel('A/*/50/[CG,OH]')"}, {select: "sel('A/*/100/[CG,SD]')"} ]}
   """
   with open("binder.yaml", "w") as f:
       f.write(yaml_content)
   ```
4. **Geração dos Peptídeos (Inference):**
   ```bash
   # num_replicates = 10 vai gerar 10 peptídeos diferentes.
   !rfd4-generate inputs=binder.yaml num_replicates=10 batch_size=1 out_dir=./runs/abc_binder '~metrics' compile_lengths=null
   ```
5. **Download dos Resultados:**
   ```python
   import shutil
   from google.colab import files
   # Compacta a pasta de resultados (onde estão os peptídeos) em um .zip e faz download
   shutil.make_archive("resultados_peptideos", "zip", root_dir="./runs/abc_binder")
   files.download("resultados_peptideos.zip")
   ```

## 4. Roadmap de 5 Dias para Concluir o TCC

Para que você consiga concluir a fase prática e compilar os resultados nos próximos dias, siga esta estrutura:

### Dia 1: Preparação do Alvo (*Target Prep*)
- **Objetivo:** Obter a estrutura exata do alvo.
- **Ações:**
  1. Vá ao [RCSB PDB](https://www.rcsb.org/) ou [AlphaFold DB](https://alphafold.ebi.ac.uk/) e busque pela estrutura do Complexo ABC da *E. coli*. Baixe no formato `.cif`.
  2. Abra a estrutura no software PyMOL ou ChimeraX.
  3. **Identifique a fenda de ligação** onde você quer atacar com o peptídeo antimicrobiano. Anote a Letra da Cadeia (Chain ID) e os números dos resíduos-chave ("hotspots").
  4. Exporte (ou faça um crop) apenas do domínio essencial do complexo ABC (remover outras subunidades desnecessárias economizará bastante memória na GPU).

### Dia 2: Geração de Peptídeos (*Binder Design*)
- **Objetivo:** Obter pelo menos ~30 a 50 esqueletos e sequências de peptídeos.
- **Ações:**
  1. Suba seu alvo `.cif` no Google Colab seguindo o tutorial da seção 3.
  2. Ajuste o arquivo `binder.yaml` de acordo com a Chain e os resíduos anotados no dia 1.
  3. Execute o `rfd4-generate` gerando múltiplas réplicas (no mínimo umas 30).
  4. Baixe o ZIP e passe o olho superficialmente nas sequências geradas.

### Dia 3: Validação In Silico (*Refolding e Binding Metrics*)
- **Objetivo:** Filtrar os peptídeos gerados para encontrar os que de fato dobram e se ligam.
- **Ações:**
  1. Como os designs gerados pelo modelo são teóricos, é necessário uma confirmação ortogonal usando preditores de estrutura.
  2. Use o AlphaFold3, ColabFold ou um serviço como Tamarind Bio (caso eles ofereçam folding de complexos).
  3. Preveja a estrutura (Complexo ABC + Peptídeo gerado) para cada candidato.
  4. Analise duas métricas chave:
     - **pLDDT do peptídeo:** Quão confiante é o dobramento do próprio peptídeo (deve ser > 70-80).
     - **iPAE (Interface pAE):** A confiança do AlphaFold na interação peptídeo-alvo (quanto mais baixo, melhor).

### Dia 4: Análise Estrutural e Filtro Final
- **Objetivo:** Escolher de 3 a 5 "top candidatos" para a dissertação.
- **Ações:**
  1. Abra as estruturas validadas pelo AlphaFold no PyMOL/Chimera.
  2. Use ferramentas do PyMOL (como *find polar contacts*) para ver se o peptídeo está formando Pontes de Hidrogênio ou contatos hidrofóbicos fortes com os hotspots selecionados do Complexo ABC.
  3. Salve imagens e renderizações de alta qualidade da região de interação. Isso será o "brilho" do seu TCC.
  4. (Opcional) Faça estimativa de carga global e hidrofobicidade das sequências dos top 5 (AMPs geralmente são catiônicos e anfipáticos).

### Dia 5: Documentação e Escrita da Monografia
- **Objetivo:** Redigir o documento do TCC.
- **Ações:**
  1. **Metodologia:** Descreva o uso do RFD4-Proteina, as condições impostas (`C_HOT`, tamanho do contig), e a validação posterior via AlphaFold.
  2. **Resultados:** Apresente tabelas listando as sequências dos peptídeos gerados que sobreviveram aos filtros (pLDDT, iPAE) e mostre as figuras 3D do peptídeo ancorado no complexo ABC de *E. coli*.
  3. **Discussão:** Contextualize como essas sequências novas poderiam ser testadas in vitro futuramente (MIC tests) para avaliar o poder antimicrobiano real.
