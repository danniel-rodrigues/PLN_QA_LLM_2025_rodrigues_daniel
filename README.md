# Avaliação de modelos de LLMs em tarefa de perguntas e respostas utilizando o idioma português brasileiro

Participantes:
```
Allan Cristiano da Silva Santos
Almir Vinícius Bispo do Nascimento
Daniel Santos Rodrigues
José Clenildo Silva Sobrinho
```
O objetivo desta segunda atividade foi avaliar o desempenho de modelos de LLMs gratuitos disponíveis na plataforma
Hugging Face, especificamente na tarefa de perguntas e respostas. Para isso, a equipe selecionou alguns modelos e 
analisou a qualidade das respostas geradas em português brasileiro. Como base para as respostas, foram fornecidos 
dois arquivos de apoio: o primeiro, um caderno de atenção básica do Ministério da Saúde sobre doenças respiratórias
crônicas, em formato PDF; e o segundo, um documento em formato DOCX contendo a relação das principais tabelas do
Departamento de Informática do SUS (DATASUS).

### Cada integrante colaborou de forma descrita abaixo:

**Allan Cristiano** - Criação e elaboração do código utilizado no Google Colab Notebook e pesquisa de modelos na 
plataforma Hugging Face.

**Almir Vinícius** - Busca de modelos na plataforma Hugging Face e colaboração na elaboração de perguntas para
aplicação nos modelos.

**Daniel Santos** - Busca de modelos na platafomra Hugging Face, elaboração das perguntas propostas para o arquivo
da relação das principais tabelas do DATASUS e elaboração do tutorial da atividade.

**josé Clenildo** - Busca de modelos na plataforma Huggind Face, colaboração no código para leitura e estruturação
do arquivo sobre doenças respiratórias crônicas.

### Vídeo da atividade

[Vídeo sobre a atividade proposta](https://drive.google.com/file/d/1RalnLWhka8eRplymGGrGhifa2uqg3Qrx/view?usp=sharing)

## Modelos de LLMs selecionados para a atividade
- [mDeBERTa-v3-base-squad2](https://huggingface.co/timpal0l/mdeberta-v3-base-squad2)
- [Google-bert (Large uncased)](https://huggingface.co/google-bert/bert-large-uncased)
- [RoBERTa-base-squad2](https://huggingface.co/deepset/roberta-base-squad2)

## Configuração do ambiente Google Colab Notebook
1 - Primeiramente é necessário realizar a instalação de todas as ferramentas que serão utilizadas:
```bash
!pip install PyMuPDF python-docx sentence-transformers faiss-cpu "pypdf>=3.0.0" "transformers>=4.0.0" "torch>=2.0.0" requests
```
2 - Após a instalação das ferramentas necessárias, é necessário fazer as devidas importações:
```python
import os
import requests
import numpy as np
import faiss
import time
import fitz  # PyMuPDF
import docx # python-docx
import re    # Para divisão de sentenças

from docx.oxml.table import CT_Tbl
from docx.oxml.text.paragraph import CT_P

from sentence_transformers import SentenceTransformer, CrossEncoder
```
Para o tratamento dos dados de cada arquivo foram criadas duas funções, cada uma com foco em um formato
de arquvio específico. A primeira função abaixo chamada ```processar_pdf_com_sentencas``` lê o arquivo PDF
sobre doenças respiratórias crônicas, extrai todo o texto com a biblioteca PyMuPDF, limpa quebras de linha
desnecessárias e divide o conteúdo em sentenças. Em seguida, essas sentenças são agrupadas em pequenos
blocos (chunks) de quatro frases, com sobreposição de uma sentença entre eles, garantindo continuidade no contexto.
Cada chunk só é armazenado se tiver mais de 10 palavras, sendo salvo junto com metadados básicos. Ao final, a função
retorna uma lista desses chunks processados, prontos para uso na tarefa de perguntas e respostas.
```python
def processar_pdf_com_sentencas(caminho_arquivo):
    """Extrai texto de PDF, dividindo em sentenças e agrupando-as em chunks focados."""
    print("1/4 - Processando PDF com PyMuPDF (chunking por sentenças)...")
    doc = fitz.open(caminho_arquivo)
    texto_completo = "".join([page.get_text("text") for page in doc])
    texto_completo = re.sub(r'\s*\n\s*', ' ', texto_completo)
    sentencas = re.split(r'(?<=[.!?])\s+', texto_completo)

    chunks_com_metadata = []
    sentencas_por_chunk = 4
    overlap = 1

    for i in range(0, len(sentencas), sentencas_por_chunk - overlap):
        grupo_sentencas = sentencas[i : i + sentencas_por_chunk]
        chunk_texto = " ".join(grupo_sentencas).strip()
        if len(chunk_texto.split()) > 10:
            chunks_com_metadata.append({"text": chunk_texto, "metadata": {"page": "N/A"}})

    print(f"✅ PDF processado. {len(chunks_com_metadata)} chunks de sentenças criados.")
    return chunks_com_metadata
```
Logo após foi utilizada a função ```processar_docx_com_contexto_tabela``` para tratar do arquivo em formato DOCX sobre
a relação principal de tabelas do DATASUS. Usando a biblioteca python-docx, percorre suas tabelas e extrai informações
estruturadas delas. Antes de cada tabela, a função tenta identificar um parágrafo que funcione como título ou contexto,
associando-o aos dados extraídos. Para cada linha da tabela (ignorando o cabeçalho), são coletados o nome do campo, sua
descrição e, se houver, os domínios ou valores possíveis. Essas informações são transformadas em frases descritivas e 
armazenadas em chunks com metadados. No fim, a função retorna todos esses chunks, já contextualizados para facilitar na
tarefa de perguntas e respostas.
```python
def processar_docx_com_contexto_tabela(caminho_arquivo):
    """Extrai dados de tabelas de um DOCX, adicionando o nome da tabela como contexto."""
    print("1/4 - Processando DOCX com python-docx (com contexto de tabela)...")
    document = docx.Document(caminho_arquivo)
    chunks_com_metadata = []

    # Itera sobre os elementos do corpo do documento (parágrafos e tabelas)
    for i, block in enumerate(document.element.body):


        if not isinstance(block, CT_Tbl):
            continue


        table = docx.table.Table(block, document)
        contexto_tabela = "Contexto não identificado"

        # Procura por um parágrafo imediatamente antes da tabela para usar como título
        if i > 0 and isinstance(document.element.body[i-1], CT_P):
            paragrafo_anterior = docx.text.paragraph.Paragraph(document.element.body[i-1], document)
            if paragrafo_anterior.text.strip():
                texto_paragrafo = " ".join(paragrafo_anterior.text.strip().split())
                match = re.search(r'LFCES\d+,\s*(\w+)', texto_paragrafo)
                if match:
                    contexto_tabela = match.group(1)
                else:
                    contexto_tabela = texto_paragrafo

        for j, row in enumerate(table.rows):
            if j == 0: continue

            try:
                nome_campo = row.cells[0].text.strip()
                descricao = row.cells[8].text.strip()
                dominios = row.cells[9].text.strip()

                if nome_campo and descricao:
                    sentenca = f"Na tabela '{contexto_tabela}', o campo '{nome_campo}' é descrito como: '{descricao}'."
                    if dominios:
                        sentenca += f" Seus domínios ou valores possíveis são: '{dominios}'."

                    chunks_com_metadata.append({"text": sentenca, "metadata": {"page": "N/A"}})
            except IndexError:
                continue

    print(f"✅ DOCX processado. {len(chunks_com_metadata)} chunks (sentenças com contexto) criados.")
    return chunks_com_metadata
```
Por fim, foi criada a função ```executar_qa``` que foca na realização do fluxo completo de perguntas e respostas (QA)
para cada um dos modelos de LLMs selecionados para essa ativadade. Por ser um código mais extenso, é recomendado que sua
visualização seja feita diretamente no arquivo do Google Colab Notebook presente nesse repositório,
entitulado como ```perguntas_e_respostas_atividade.ipynb```. Para cada pergunta, essa função executa as seguintes etapas:
1) transforma a pergunta em embedding e busca os trechos (chunks) mais relevantes em um índice FAISS,
2) reclassifica esses trechos usando um modelo cross-encoder para selecionar o mais pertinente,
3) envia a pergunta e o contexto selecionado para múltiplos modelos de QA da Hugging Face e
4) imprime as respostas com suas pontuações, registrando também o tempo gasto em cada consulta.

## Conclusão
Devido à estratégia adotada no pré-processamento dos arquivos de contexto, o cálculo de similaridade entre o conteúdo dos arquivos e as perguntas resultava na seleção apenas dos três chunks com maior pontuação, descartando os demais. Entretanto, esses trechos eliminados poderiam conter informações relevantes para auxiliar os modelos na elaboração das respostas.

Esse problema ocorria porque o cálculo de similaridade era realizado de forma pouco criteriosa, considerando apenas a distância textual entre perguntas e trechos, sem uma base semântica consistente. Assim, a análise das respostas fornecidas pelos modelos tornava-se inviável, já que a perda de dados comprometia significativamente a precisão.

Para que o pré-processamento fosse conduzido de maneira adequada, seria necessário converter o conteúdo dos arquivos em embeddings e, a partir deles, realizar o cálculo de similaridade, assegurando que todos os dados fossem preservados, mesmo os aparentemente menos relevantes.
