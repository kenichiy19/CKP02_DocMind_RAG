# CKP02 — DocMind RAG: Suporte Técnico de Computadores

**Disciplina:** Prompt Engineering and Artificial Intelligence — FIAP · 2º Semestre 2026
**Professor:** Jorge Luiz Gomes · **Turma:** 1CCPK

## Integrantes

| Nome | RM |
| --- | --- |
| Felipe Pereira Restivo | 570712 |
| Gabriel Rodrigues Zappelloni | 572060 |
| Kenichi Caio Yamamoto | 569815 |
| Maykon de Lima Silva | 574022 |
| Rodger Costa Rios | 571438 |

## Links

- **Notebook no Colab (leitura):** [abrir no Colab](https://colab.research.google.com/github/kenichiy19/CKP02_DocMind_RAG/blob/main/CKP02_DocMind_RAG.ipynb)
- **Repositório:** https://github.com/kenichiy19/CKP02_DocMind_RAG

## Domínio

Mesmo domínio do CKP01: **suporte técnico inicial de computadores pessoais** (hardware, software e conectividade), voltado a usuários sem experiência técnica. No CKP01, o assistente respondia apenas com o conhecimento do modelo. Neste checkpoint, ele passa a responder com base em documentação oficial e a citar a origem de cada informação.

## Pipeline

```
PDFs + catálogo ─► load + limpeza ─► união das páginas ─► split (256 / 512 / 1024)
     ─► embeddings nomic-embed-text ─► ChromaDB (uma coleção por configuração)
     ─► buscar(): busca vetorial (10 candidatos, filtro where= opcional) ─► reranking (3 trechos)
     ─► responder(): gemma4:cloud com os trechos numerados ─► resposta com fontes
```

| Etapa | Implementação |
| --- | --- |
| Load | `PyMuPDFLoader` (PDF) ou leitura direta (TXT/MD), metadados vindos de `docs/catalogo.csv` |
| Limpeza | Remoção de menus, cabeçalhos e rodapés dos artigos salvos do site da Microsoft; remoção de sumário e rodapé do manual da Dell |
| Split | `RecursiveCharacterTextSplitter`, separadores `["\n\n", "\n", ". ", " ", ""]`, overlap de 12,5% |
| Embed | `OllamaEmbeddings(model="nomic-embed-text")`, com os prefixos `search_document:` / `search_query:` do modelo |
| Store | `chromadb.PersistentClient`, coleções `suporte_tecnico_pc_256`, `_512` e `_1024`, distância de cosseno |
| Retrieve | `buscar(consulta, k, categoria, chunk_size, usar_rerank)` |
| Rerank | `HuggingFaceCrossEncoder("cross-encoder/mmarco-mMiniLMv2-L12-H384-v1")` + `CrossEncoderReranker` |
| Generate | `ChatOllama(model="gemma4:cloud", temperature=0)` com prompt que exige citação `[n]` e recusa quando a informação não está na base |
| Avaliação | RAGAS (`faithfulness` e `answer_relevancy`) com 8 perguntas de teste |
| Interface | Gradio (`gr.Blocks`) com filtro de categoria e fontes clicáveis |

## Base de conhecimento

| Documento | Fonte | Categoria |
| --- | --- | --- |
| [Corrigir problemas de conexão Wi-Fi no Windows](https://support.microsoft.com/pt-br/windows/experience/connectivity-networking/fix-wi-fi-connection-issues-in-windows) | Microsoft Support | rede |
| [Solução de problemas de reinicializações inesperadas do Windows e erros de código de parada](https://support.microsoft.com/pt-br/windows/experience/performance-optimization/troubleshooting-windows-unexpected-restarts-and-stop-code-errors) | Microsoft Support | software |
| [Dicas para melhorar o desempenho do PC no Windows](https://support.microsoft.com/pt-br/windows/experience/performance-optimization/tips-to-improve-pc-performance-in-windows) | Microsoft Support | software |
| [Solucionar problemas de atualização do Windows](https://support.microsoft.com/pt-br/windows/deployment/updates-lifecycle/troubleshoot-problems-updating-windows) | Microsoft Support | software |
| [Dicas de economia de bateria para Windows](https://support.microsoft.com/pt-br/windows/experience/power-battery/battery-saving-tips-for-windows) | Microsoft Support | hardware |
| [XPS 13 2 em 1 (9310 2n1) — Manual de serviço, Rev. A02](https://dl.dell.com/topicspdf/xps-13-9310-2-in-1-laptop_Service-Manual_pt-br.pdf) | Dell | hardware |

Os artigos da Microsoft foram acessados em 10/10/2026 e salvos em PDF pelo navegador. O manual da Dell é o PDF original do fabricante.

## Como executar

1. Abra o notebook pelo link do Colab acima.
2. Altere o ambiente para GPU: *Ambiente de execução → Alterar o tipo de ambiente de execução → T4 GPU*.
3. Cadastre a chave da Ollama Cloud como Secret: ícone de chave na barra lateral → *Add new secret* → nome `OLLAMA_API_KEY` → ative o acesso do notebook.
4. Execute *Ambiente de execução → Executar tudo*.

A primeira célula instala as dependências (versões fixadas em `requirements.txt`) e clona este repositório, de onde o notebook lê os documentos. A segunda instala um servidor Ollama local para o modelo de embedding (ver [Decisões técnicas](#decisões-técnicas)).

## Como adicionar novos documentos à base

1. Coloque o arquivo na pasta `docs/`. Formatos aceitos: **PDF**, **TXT** ou **Markdown (.md)**.
2. Acrescente uma linha em `docs/catalogo.csv`:

   | Coluna | Conteúdo | Exemplo |
   | --- | --- | --- |
   | `arquivo` | Nome do arquivo em `docs/` | `teclado_notebook.pdf` |
   | `titulo` | Título do documento, usado na citação | `Solucionar problemas de teclado` |
   | `fonte` | Organização que publicou | `Microsoft Support` |
   | `url` | Link de origem | `https://support.microsoft.com/...` |
   | `categoria` | `hardware`, `rede` ou `software` | `hardware` |
   | `tipo` | Tipo do documento | `artigo_suporte` |
   | `data_acesso` | Data em que o documento foi obtido | `2026-10-10` |

3. Envie o arquivo e o catálogo atualizado para o repositório e execute o notebook novamente (*Executar tudo*). As coleções do ChromaDB são recriadas a cada execução, então o novo documento é indexado automaticamente nas três configurações.

Observações:

- Para artigos do site da Microsoft salvos pelo navegador (*Ctrl+P → Salvar como PDF*), use `Microsoft Support` na coluna `fonte`, o que ativa a remoção dos menus e rodapés do site. Os demais PDFs passam pela limpeza de manuais (sumário e rodapé com número de página).
- Uma categoria nova (por exemplo, `perifericos`) passa a aparecer automaticamente no filtro da interface.
- Documentos sem fonte identificável não devem ser adicionados: a citação de origem faz parte do critério de qualidade da base.

## Decisões técnicas

**Embedding em servidor Ollama local.** A chave da Ollama Cloud funciona para o chat, mas o endpoint de embeddings da nuvem a recusa. O teste executado no Colab, com a mesma chave, retornou:

```
CHAT gemma4:cloud  -> OK: ok
EMBED nomic-embed-text -> ERRO: unauthorized (status code: 401)
```

Para manter o modelo exigido (`nomic-embed-text`) e a mesma classe (`OllamaEmbeddings`), o notebook instala o Ollama no próprio Colab e executa o modelo de embedding localmente. O chat continua exclusivamente no `gemma4:cloud`, via Ollama Cloud. Para usar outro servidor de embeddings, basta alterar `OLLAMA_EMBED_HOST` na seção de configuração.

**Prefixos do nomic-embed-text.** O modelo foi treinado com prefixos que distinguem texto indexado (`search_document:`) de pergunta (`search_query:`). A classe `NomicEmbeddings` estende o `OllamaEmbeddings` apenas para adicioná-los.

**Limpeza dos documentos.** Os artigos salvos do site trazem menus, sugestões do Copilot, cabeçalho e rodapé de impressão em toda página e o rodapé institucional. Sem a limpeza, esse texto seria indexado e competiria com o conteúdo na busca.

**União das páginas antes do split.** Na primeira avaliação, o texto era dividido página por página. Listas que começavam no fim de uma página e terminavam na seguinte ficavam separadas, e o assistente respondia com parte dos passos. Cada documento passou a ser unido em um texto contínuo antes da divisão, e cada chunk mantém a página onde começa, usada na citação.

**Reranker multilíngue.** O `ms-marco-MiniLM-L-6-v2` usado na Aula 07 foi treinado apenas em inglês. A base e as perguntas estão em português, por isso foi usado o `mmarco-mMiniLMv2-L12-H384-v1`, da mesma família, treinado na versão multilíngue do MS MARCO.

**Ajustes no RAGAS.**

- O `ragas 0.4.3` importa um módulo removido da `langchain-community 0.4`. Como a classe só é usada em uma checagem de tipo, um substituto vazio permite carregar a biblioteca.
- Os prompts do RAGAS são em inglês; foi acrescentada a instrução de gerar os textos em português, para que o `answer_relevancy` não seja reduzido pela diferença de idioma.
- O juiz é o próprio `gemma4:cloud`, em instância separada da usada pelo pipeline.
- Avaliações inválidas (JSON malformado do juiz) são refeitas até duas vezes.

**Critério de escolha da configuração.** O `faithfulness` médio ≥ 0,7 é requisito mínimo. Entre as configurações aprovadas, vence a de maior média entre `faithfulness` e `answer_relevancy`. O `faithfulness` sozinho favorece configurações que respondem "não encontrei", pois essas respostas não contêm afirmações falsas; o `answer_relevancy` sozinho não detecta afirmações sem suporte nos trechos.

## Resultados

<!-- RESULTADOS: preenchido após a execução final -->

## Reuso no CKP03

A função `buscar(consulta, k=3, categoria=None, chunk_size=None, usar_rerank=True)` concentra a recuperação e devolve uma lista de `Document` com os metadados de origem (`titulo`, `arquivo`, `pagina`, `url`, `categoria`). No CKP03, ela será exposta ao agente como ferramenta (`@tool`), com a categoria como parâmetro opcional.

## Estrutura do repositório

```
CKP02_DocMind_RAG/
├── README.md
├── CKP02_DocMind_RAG.ipynb      # notebook exportado após "Executar tudo"
├── requirements.txt
├── docs/                        # base de conhecimento
│   ├── catalogo.csv             # metadados e fontes de cada documento
│   └── *.pdf
└── resultados/                  # tabelas do RAGAS geradas pelo notebook
```
