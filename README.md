# CineData Analytics: Agente Text-to-SQL

Agente que responde perguntas em português sobre o catálogo de filmes da CineData, gerando e executando consultas SQL (somente leitura) na camada Gold. A ideia é colocar os dados nas mãos de quem não sabe SQL: a pessoa pergunta "qual produtora teve o maior lucro?" e o agente consulta o banco e responde.

**Stack:** Python 3.12 · PydanticAI · OpenRouter (modelos gratuitos) · Jupyter Notebook · SQLite

## Como funciona

```mermaid
flowchart LR
    U[Pergunta em português] --> A[Agente PydanticAI]
    A -->|prompt com schema e regras| M[Modelo no OpenRouter]
    M -->|SQL| T[Ferramenta executar_sql]
    T -->|só SELECT, máx. 50 linhas| DB[(cinerocket.db<br/>somente leitura)]
    DB -->|resultado em CSV| M
    M --> R[Resposta + SQLs executados]
```

1. O agente envia a pergunta ao modelo junto com o schema do banco e as regras de negócio
2. O modelo escreve um SQL e chama a ferramenta `executar_sql`
3. A ferramenta valida o SQL, executa no banco e devolve o resultado em CSV. Se o SQL der erro, devolve a mensagem para o modelo corrigir e tentar de novo
4. O modelo lê o resultado e escreve a resposta. Junto com ela, o notebook mostra os SQLs executados, quantas requisições foram gastas e qual modelo respondeu

Uma pergunta típica gasta 2 requisições: uma para gerar o SQL e outra para escrever a resposta.

## Como rodar

### Pré-requisitos

- [uv](https://docs.astral.sh/uv/getting-started/installation/) instalado (ele baixa o Python 3.12 sozinho)
- Chave gratuita do [OpenRouter](https://openrouter.ai/keys) (não precisa de cartão)
- Arquivo `cinerocket.db`, disponível no drive da atividade. Ele tem ~580 MB, acima do limite do GitHub, por isso não está no repositório

### Passo a passo

**1. Clonar e instalar as dependências**

```bash
git clone https://github.com/nunnowdc/Atividade-GenAI-CineData-Analytics-Rocket-Lab-2026.git
cd Atividade-GenAI-CineData-Analytics-Rocket-Lab-2026
uv sync
```

O `uv sync` cria a pasta `.venv` com as versões exatas do `uv.lock`.

**2. Configurar a chave do OpenRouter**

```bash
cp .env.example .env           # no Windows (PowerShell): Copy-Item .env.example .env
```

Abra o `.env` e troque `sk-or-v1-sua-chave-aqui` pela sua chave.

**3. Colocar o banco na pasta `data/`**

O arquivo precisa ficar em `data/cinerocket.db` (com esse nome).

**4. Abrir o notebook**

- **VSCode:** abra `notebooks/agente_cinedata.ipynb` e, em *Select Kernel*, escolha o ambiente `.venv` do projeto
- **Jupyter Lab:** `uv run jupyter lab notebooks/agente_cinedata.ipynb`

**5. Rodar**

Use *Run All*.

> ⚠️ **Cota do OpenRouter:** a conta gratuita permite 50 requisições por dia. O *Run All* executa as 14 perguntas do desafio e os testes com o modelo real, gastando cerca de 35 requisições. Para só experimentar, rode até o fim da seção 10 (nada antes disso gasta cota) e faça suas próprias perguntas. A função `verificar_cota()` mostra quantas requisições restam, sem gastar nenhuma. A cota zera às 21h (horário de Brasília).

### Fazendo suas perguntas

```python
await perguntar("Quais os 10 filmes de animação com maior receita?")
await perguntar("E em dólar?", continuar=True)   # continua a conversa anterior
```

## Estrutura do projeto

```
├── data/
│   └── cinerocket.db          # banco SQLite (não versionado)
├── notebooks/
│   └── agente_cinedata.ipynb  # o agente, os testes e as perguntas do desafio
├── .env.example               # modelo do .env (a chave real fica no .env, que é ignorado pelo git)
├── pyproject.toml             # dependências do projeto
├── uv.lock                    # versões exatas das dependências
└── README.md
```

O notebook está dividido em seções:

| Seção | Conteúdo | Gasta cota? |
|---|---|---|
| 1 a 5 | Conexão somente leitura, exploração do banco, qualidade dos dados e respostas calculadas à mão | Não |
| 6 | Configuração: chave, modelos e verificação de cota | Não |
| 7 | Ferramenta `executar_sql` e seus testes | Não |
| 8 | Prompt de sistema: schema gerado do banco, regras de negócio e exemplos | Não |
| 9 | O agente e um teste com modelo falso | Não |
| 10 | Função `perguntar` e testes de fallback e memória (com modelos falsos) | Não |
| 10 (perguntas) | As 14 perguntas do desafio, os testes de guardrails e de memória | **Sim** |

## Decisões técnicas

| Decisão | Por quê |
|---|---|
| **PydanticAI** como framework | Usado nas aulas. Cuida do ciclo do agente (chamar ferramenta, devolver resultado, tentar de novo) e traz fallback entre modelos e modelos falsos para testes |
| **OpenRouter com `nvidia/nemotron-3.5-lightning:free`** | Gratuito, com suporte a ferramentas (tool calling) e recomendado no guia do Rocket Lab para gerar SQL. Fixar um modelo, em vez do `openrouter/free` (que sorteia um modelo a cada chamada), deixa o comportamento previsível na hora de ajustar o prompt |
| **Fallback: nemotron → gemma-4-31b → qwen3.8-27b** | Modelos gratuitos dividem capacidade entre todos os usuários e frequentemente respondem 429 (lotado). Se um falhar, o próximo é tentado. Erros de SQL não trocam de modelo, vão para a autocorreção |
| **Jupyter Notebook** como entregável | Permite mostrar a exploração dos dados, os testes e as respostas do agente no mesmo lugar, com as saídas salvas |
| **uv + Python 3.12** | O `uv.lock` garante que quem clonar instale exatamente as mesmas versões, e o uv baixa o Python 3.12 sem mexer no Python do sistema |
| **pandas 2.x** (e não 3) | O pandas 3 foi bloqueado pelo Smart App Control do Windows 11 durante o desenvolvimento |
| **Uma única ferramenta (`executar_sql`)**, com o schema no prompt | Com ferramentas separadas (listar tabelas, descrever tabela, executar), cada pergunta gastaria 4 a 5 requisições. Com o schema já no prompt, gasta 2. Com o limite de 50 por dia, isso faz muita diferença. O banco tem só 10 tabelas, então o schema cabe tranquilo no prompt |
| **Schema gerado automaticamente** do banco | Sem erro de digitação nem coluna esquecida. Inclui as chaves estrangeiras (para o modelo acertar os JOINs) e os valores possíveis das colunas de categoria (para ele saber, por exemplo, que os gêneros estão em inglês) |
| **Resultado devolvido em CSV** | Tudo que a ferramenta devolve volta para o modelo como texto. Medindo com `tiktoken` uma consulta de 50 filmes: CSV 964 tokens, tabela do pandas 1.176 (+22%), Markdown 1.454 (+51%), JSON 1.902 (+97%) |
| **Máximo de 50 linhas por consulta** | Uma consulta como `SELECT * FROM dim_movies` traria 95 mil linhas e estouraria o contexto do modelo. A ferramenta avisa quando corta ("mostrando 50 de 95.645 linhas") |
| **Autocorreção** (`ModelRetry`, até 2 tentativas) | Se o SQL der erro, a mensagem do SQLite volta para o modelo corrigir, em vez de quebrar a pergunta |
| **Teto de 5 requisições por pergunta** | Protege a cota: um modelo em loop gasta no máximo 5 requisições, não as 50 do dia |
| **SQLs registrados pela própria ferramenta** | Mostra ao usuário os SQLs que realmente rodaram, não os que o modelo diz que rodou, e sem gastar requisição extra |
| **2 exemplos de pergunta e SQL no prompt** (few-shot) | Ensinam os JOINs mais difíceis (gênero e papel da pessoa). Nenhum deles é pergunta do desafio, para não entregar a resposta ao modelo |
| **Memória de conversa opcional** (`continuar=True`) | Com memória sempre ligada, as perguntas independentes virariam uma conversa só, com cada pergunta carregando as tabelas de resultado de todas as anteriores |
| **Testes com modelos falsos** (`FunctionModel`) | A ferramenta, o agente, o fallback e a memória foram testados sem chamar o OpenRouter. A cota foi gasta só testando a qualidade das respostas |
| **Proteção em camadas** | Prompt (recusa assuntos fora do tema) + ferramenta (só SELECT/WITH) + banco em modo somente leitura. As duas últimas não dependem do modelo obedecer |

## Regras de negócio e qualidade dos dados

Antes de construir o agente, os dados foram explorados direto no banco (seções 1 a 5 do notebook). Várias perguntas do desafio dariam respostas erradas sem as regras abaixo, que estão no prompt do agente.

### Regras de negócio

| Regra | Por quê |
|---|---|
| "Receita", "faturamento" e "bilheteria" são a mesma coisa | Definido no enunciado do desafio |
| Valores em **R$** por padrão (dólar só se pedido) | O público é brasileiro e as perguntas do desafio usam R$ |
| **Lucro só com receita e orçamento informados** | A coluna de lucro nunca é nula e engana: sem receita ela vale menos o orçamento, sem orçamento ela vale a própria receita. Quando o usuário pede lucro "considerando filmes com receita informada", o agente mostra as duas versões lado a lado (só receita e receita + orçamento) num único SQL |
| **Margem = lucro ÷ receita**; **retorno (ROI) = lucro ÷ orçamento** | Margem responde "de tudo que faturou, quanto virou lucro"; retorno responde "quanto voltou para cada real investido" |
| **Orçamento mínimo de US$ 10 mil** em margem e retorno | Existem filmes com orçamento de US$ 50 ou US$ 128, claramente erros de cadastro, que dominariam qualquer ranking com margens de 100% e retornos de milhões de %. O corte remove os absurdos sem descartar produções de baixo orçamento reais |
| **Margem de um grupo = soma do lucro ÷ soma da receita** | A média das margens de cada filme explode com receitas minúsculas: um filme com receita de US$ 3 e orçamento de US$ 162 mil tem margem de -5.409.086%, o que deixava todos os gêneros com margem média negativa |
| **Divisões com `* 1.0` antes de dividir** | O SQLite descarta as casas decimais quando os dois valores são inteiros. Como 323 filmes têm valores sem centavos, a margem deles virava 0 e eles sumiam do ranking. A regra mostra o jeito certo (`lucro * 1.0 / receita`) e o errado (`(lucro / receita) * 1.0`), porque só o jeito certo não bastou: o modelo colocou o `* 1.0` depois da divisão |
| **Mínimo de 100 votos (TMDB e IMDb) e 4 avaliações de usuários** em rankings de notas de filmes | Sem isso, filmes com 1 voto dominam (ex.: nota 10 no TMDB com um único voto). Para TMDB/IMDb sobram mais de 4 mil filmes com as duas notas. Nas avaliações de usuários, 93% dos filmes têm só 1 avaliação, então o mínimo é menor. Não se aplica a médias de grupos (por ano, por diretor), em que um filme sozinho pesa pouco |
| Nota TMDB = 0 significa "sem votos" | 36 mil filmes têm nota 0 e quase todos têm 0 votos |
| O papel da pessoa (Ator, Diretor, Roteirista) está em `dim_people` | A tabela `bridge_movie_person` não diz o papel. A mesma pessoa pode aparecer com papéis diferentes (ex.: Tom Hanks como Ator e como Roteirista) |
| Gêneros estão em inglês | "Terror" precisa virar `'Horror'` no SQL |
| "Últimos N anos" a partir da data de hoje | O prompt informa a data atual. O catálogo vai de 2016 a 2029, mas quase tudo é até 2024 |
| Contas e conversões de unidade no SQL | O modelo erra contas de cabeça (escreveu "R$ 0,5 mil" para R$ 451 mil) |
| Títulos sem tradução | O modelo traduziu um título, e o título traduzido não existe no banco |

### Qualidade dos dados

- **Receita é rara:** só 3.373 dos 95.645 filmes (3,5%) têm receita, e só 1.630 têm receita e orçamento
- **Títulos duplicados** com IDs diferentes (ex.: *Die Hart 2: Die Harter* aparece 3 vezes), o que aparece em rankings
- **Popularidade com valores suspeitos:** alguns filmes têm popularidade exatamente igual ao ano (2018, 2019, 2020)
- **Nomes inválidos entre os roteiristas**, como "English" e "United States Of America"
- **`idioma_original` está vazio** em todos os filmes
