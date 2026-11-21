# Serverless (Rota Vital)

Infraestrutura de Software | CESAR School | 2026.2
| | |
| **Aluna** | [Emily Raquel Marques] |
| **Vídeo** | [https://www.youtube.com/watch?v=8efWkaLxqAs&authuser=0] |
| **Caminho escolhido** | A: AWS SAM CLI rodando localmente (sem conta na AWS e sem custo) |

## 1. O que a função faz

Função HTTP que recebe uma requisição do Rota Vital (`POST /guias`) e devolve a **guia de transporte**.

**Entrada** (corpo JSON):

```json
{"requisicaoId": "R-001", "hospital": "H-Recife", "bolsas": ["O-NEG-0142"]}
```

**Saída** (`201`):

```json
{"guia": "G-R-001", "hospital": "H-Recife", "bolsas": ["O-NEG-0142"], "invocacaoNesteAmbiente": 1}
```

- Sem `requisicaoId`, a função responde `400`.
- Com `"demorar": true`, ela dorme 5 segundos (usado para estourar o limite de tempo).
- `invocacaoNesteAmbiente` vem de uma variável global (`contador`) e mostra se o ambiente foi reaproveitado entre as chamadas.

## 2. Caminho escolhido

**Caminho A: AWS SAM CLI local.** A função (AWS Lambda, Python 3.12) roda no emulador do SAM dentro do Docker. Nada foi publicado na nuvem (a publicação era opcional), então não há conta, cartão nem custo.

## 3. Ambiente usado

| Item | Valor |
|---|---|
| Sistema | Windows (terminais: Git Bash e Prompt de Comando) |
| AWS SAM CLI | 1.174.0 |
| Docker | Docker Desktop, versão 29.8.0 |
| Python no computador | 3.13.7 |
| Runtime da função | python3.12 (x86_64), memória de 128 MB |

O Python do computador (3.13) é diferente do runtime da função (3.12). Por isso o build é feito dentro de um container (`sam build --use-container`).

## 4. Como rodar

### Pré-requisitos
- Docker Desktop **aberto** e com "Engine running" (confira com `docker ps`).
- AWS SAM CLI instalado (`sam --version`).
- Git.
- Python 3.12

### Passo a passo

```bash
# 1. Clonar e entrar na pasta
git clone https://github.com/emilyraquel/rota-vital-serverless.git
cd rota-vital-serverless

# 2. Build (em container, por causa da versão do Python)
sam build --use-container

# 3. Invocar a função com um evento (rode duas vezes e compare os tempos)
sam local invoke -e events/pedido.json
sam local invoke -e events/pedido.json
```

**Gatilho HTTP** (dois terminais, ambos na pasta do projeto):

```bash
# Terminal 1: subir o servidor
sam local start-api

# Terminal 2: chamar a função (rode 3 vezes e observe invocacaoNesteAmbiente)
curl -X POST http://127.0.0.1:3000/guias \
  -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
```

**Com containers mantidos** (compare com o teste anterior): pare o servidor (Ctrl+C) e suba com:

```bash
sam local start-api --warm-containers LAZY
```

**Limite de tempo:**

```bash
# Com Timeout: 3 no template.yaml, este pedido estoura o limite (sleep de 5 s)
curl -X POST http://127.0.0.1:3000/guias \
  -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
```

Para corrigir, mude `Timeout: 3` para `Timeout: 10` no `template.yaml`, rode de novo `sam build --use-container`, reinicie o `sam local start-api` e repita o `curl`.

### Observações para Windows (Git Bash)
- Se o Git Bash não achar o `sam` (`command not found`), crie um alias: `echo "alias sam='sam.cmd'" >> ~/.bashrc` e `echo 'source ~/.bashrc' >> ~/.bash_profile`.
- Os comandos `sam` devem ser rodados **dentro da pasta do projeto**, porque `events/pedido.json` é um caminho relativo.
- Ao parar o servidor com Ctrl+C, o SAM no Windows mostra uma pilha de erro longa e pergunta "Deseja finalizar o arquivo em lotes (S/N)?". Responda `S`. Isso é do encerramento do SAM e não tem relação com a função.
- Cada comando deve ser colado em uma linha só (ou com a barra invertida no fim, como acima), sem ponto no final.

## 5. Resultados medidos

Valores lidos na linha `REPORT` do log do SAM (todos em milissegundos). O traço (—) indica que o campo `Init Duration` não apareceu naquela execução. A memória foi de 128 MB em todas.

### 5.1 `sam local invoke` (um container novo por execução)

| Execução | invocacaoNesteAmbiente | Init Duration | Duration | Billed Duration | Status |
|---|---|---|---|---|---|
| 1ª | 1 | 0,06 ms | 134,70 ms | 135 ms | 201 |
| 2ª | 1 | 0,07 ms | 75,98 ms | 76 ms | 201 |

### 5.2 `sam local start-api` sem a flag

| Chamada | invocacaoNesteAmbiente | Init Duration | Duration | Billed Duration | Status |
|---|---|---|---|---|---|
| 1ª | 1 | 1,32 ms | 193,11 ms | 194 ms | 201 |
| 2ª | 2 | — | 57,17 ms | 58 ms | 201 |
| 3ª | 3 | — | 22,86 ms | 23 ms | 201 |

O log mostrou `Containers created` **na inicialização** do servidor, antes da primeira requisição.

### 5.3 `sam local start-api --warm-containers LAZY`

| Chamada | invocacaoNesteAmbiente | Init Duration | Duration | Billed Duration | Status |
|---|---|---|---|---|---|
| 1ª | 1 | 0,53 ms | 260,28 ms | 261 ms | 201 |
| 2ª | 2 | — | 22,74 ms | 23 ms | 201 |
| 3ª | 3 | — | 20,94 ms | 21 ms | 201 |

Neste modo, o servidor subiu **sem criar container**. Ele só foi criado na primeira requisição.

### 5.4 Limite de tempo (`demorar = true`)

| Cenário | Timeout | Duration | Billed Duration | Resultado |
|---|---|---|---|---|
| Timeout 3 | 3 s | 3000,00 ms | 3000 ms | `timed out after 3 seconds`, HTTP 502; o cliente recebeu `{"message":"Internal server error"}` |
| Timeout 10 | 10 s | 5221,51 ms | 5222 ms | HTTP 201, guia `G-R-003`, Init Duration de 0,46 ms |

Com o Timeout de 10 s, a mesma chamada levou 2221,51 ms a mais do que o limite de 3 s anterior. São os 5000 ms do `sleep` mais cerca de 221,5 ms de execução restante, numa chamada que foi a primeira depois de reiniciar o servidor.

### 5.5 Comparações

**Primeira chamada x chamadas seguintes** (mesma sessão, mesmo servidor):

| Cenário | 1ª chamada | 2ª chamada | 3ª chamada | Diferença 1ª − 2ª | Redução 1ª → 2ª | 1ª ÷ 3ª |
|---|---|---|---|---|---|---|
| `invoke` (container novo a cada vez) | 134,70 ms | 75,98 ms | — | 58,72 ms | 43,6% | — |
| `start-api` sem a flag | 193,11 ms | 57,17 ms | 22,86 ms | 135,94 ms | 70,4% | 8,4 vezes |
| `start-api` com LAZY | 260,28 ms | 22,74 ms | 20,94 ms | 237,54 ms | 91,3% | 12,4 vezes |

**Sem a flag x com `--warm-containers LAZY`:**

| Chamada | Sem a flag | Com LAZY | Diferença (LAZY − sem a flag) |
|---|---|---|---|
| 1ª | 193,11 ms | 260,28 ms | +67,17 ms (+34,8%) |
| 2ª | 57,17 ms | 22,74 ms | −34,43 ms (−60,2%) |
| 3ª | 22,86 ms | 20,94 ms | −1,92 ms (−8,4%) |
| Média da 2ª e 3ª | 40,02 ms | 21,84 ms | −18,18 ms (−45,4%) |

**O que os números mostram:**
- Em todos os cenários, a **primeira chamada é a mais lenta** (cold start), e as seguintes ficam bem mais rápidas porque o ambiente já existe.
- Com LAZY, a primeira chamada ficou **67,17 ms mais lenta** (o container é criado nela), mas as seguintes ficaram mais rápidas em média. Sem a flag, esse custo foi pago antes, na inicialização do servidor.
- Os valores são de uma única rodada cada, num emulador local, então pequenas diferenças (como a 3ª chamada, de 22,86 ms contra 20,94 ms) são variação normal e não devem ser lidas como efeito da flag.
- O `Init Duration` local é muito pequeno (menos de 2 ms), porque o SAM só emula o ambiente. Na nuvem, o cold start costuma ser bem maior.

## 6. Respostas às perguntas

### 1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?

Na primeira invocação, o ambiente de execução ainda precisa ser preparado (cold start). Na fase de Init, o ambiente é criado, o código da função é carregado, o runtime Python é iniciado e o código que fica fora do handler é executado (aqui, os `import` e a linha `contador = 0`). Isso acontece antes de o handler rodar pela primeira vez. Quando o ambiente é reaproveitado, essa fase não se repete e só o handler é executado.

Nos meus testes, a primeira chamada foi sempre a mais lenta: 134,70 ms contra 75,98 ms no `sam local invoke`; 193,11 ms contra 57,17 ms e 22,86 ms no `start-api`; e 260,28 ms contra 22,74 ms e 20,94 ms com LAZY (tabela 5.5). Só a primeira chamada de cada sessão do servidor mostrou `Init Duration`. No `sam local invoke`, cada comando cria um container novo (o `SAM_CONTAINER_ID` muda), por isso as duas execuções têm `Init Duration`.

### 2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global?

Nas duas rodadas do `start-api`, o campo mostrou 1, 2 e 3 enquanto o servidor ficou ligado, e voltou a 1 quando o servidor foi reiniciado. No `sam local invoke` foi sempre 1, porque cada execução usa um container novo.

A diferença que observei entre os modos foi **quando o container é criado**: sem a flag, na inicialização do servidor; com `LAZY`, só na primeira requisição, que por isso foi 67,17 ms mais lenta (tabela 5.5).

Isso mostra que uma variável global só mantém o valor enquanto o mesmo ambiente for reaproveitado, e não existe garantia disso: se o ambiente for descartado, ou se outro ambiente atender a chamada, o valor se perde ou diverge. Por isso, dado que importa não deve ficar em variável global da função. Deve ficar em um armazenamento externo, como um banco de dados ou um cache.

### 3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda e o que fazer com um trabalho mais longo?

Com `demorar = true`, a função executa `time.sleep(5)`, mas o Timeout do template era 3 segundos. A execução foi interrompida aos 3 s: o log mostrou `timed out after 3 seconds`, `Duration: 3000,00 ms` (que também é cobrado: `Billed Duration` de 3000 ms) e `Lambda response must be valid json`. O cliente recebeu `{"message":"Internal server error"}` e o servidor registrou HTTP 502. Depois de aumentar o Timeout para 10 s e refazer o build, a mesma chamada respondeu 201 em 5221,51 ms.

O tempo máximo de uma execução no AWS Lambda é de **15 minutos (900 segundos)**, conforme a página de cotas do Lambda indicada no enunciado. Para um trabalho mais longo, é preciso dividi-lo em etapas menores e encadeá-las (por exemplo, com fila e orquestração, como SQS e Step Functions) ou usar um serviço próprio para processos longos, como contêineres.

### 4. Custo de um milhão de execuções com 128 MB e 100 ms (us-east-1)

Preços usados: US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s.

| Parte | Conta | Resultado |
|---|---|---|
| Requisições | 1.000.000 × US$ 0,20 por milhão | US$ 0,20 |
| GB-s por execução | 0,125 GB (128 MB) × 0,1 s (100 ms) | 0,0125 GB-s |
| GB-s totais | 0,0125 × 1.000.000 | 12.500 GB-s |
| Computação | 12.500 × US$ 0,0000166667 | ≈ US$ 0,2083 |
| **Total** | US$ 0,20 + US$ 0,2083 | **≈ US$ 0,41** |

Valor calculado sem considerar a faixa gratuita (1 milhão de requisições e 400 mil GB-s por mês).

### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

Faz sentido na **geração da guia de transporte** quando chega uma requisição, como nesta atividade: é uma tarefa curta, pontual e sem estado, com uso que varia ao longo do tempo, e só se paga pelo que é executado. O mesmo vale para notificações e validações disparadas por evento.

Não faria sentido onde for preciso manter **estado em memória ou conexão aberta** (por exemplo, acompanhar em tempo real o deslocamento de uma bolsa), em **processos longos**, que passariam do limite de tempo, ou em **cargas constantes e muito altas**, em que um serviço sempre ligado pode compensar mais.

## 7. Problemas encontrados e como foram resolvidos

| Problema | Causa | Solução |
|---|---|---|
| `sam: command not found` no Git Bash | O Git Bash não resolve `sam.cmd` | Alias `sam='sam.cmd'` no `~/.bashrc` e `~/.bash_profile` |
| Docker: "Virtualization support not detected" | WSL/virtualização não estavam ativos | `wsl --install` e reinício do computador |
| `git push` rejeitado | O repositório remoto já tinha um README | `git pull --allow-unrelated-histories`; conflito do README resolvido mantendo a versão local |
| `sam build`: "Binary validation failed for python" | Python local 3.13 e runtime 3.12 | `sam build --use-container` |
| `Runtime.UnmarshalError` | `events/pedido.json` vazio | Evento gerado de novo no Prompt de Comando |
| `JSONDecodeError` | Campo `body` do evento inválido | `body` corrigido para JSON válido |
| `curl`: "Failed to connect to port 3000" | Servidor derrubado por um Ctrl+C | `start-api` iniciado de novo |
| 502 e `timed out after 3 seconds` | `sleep(5)` maior que o Timeout de 3 s (era o esperado) | Timeout de 10 s, novo build e novo teste |

## 8. Evidências

Os prints ficam na pasta [`evidencias/`](evidencias/).

| Arquivo | O que mostra |
|---|---|
| `00-ambiente` | Versões do SAM CLI, Docker e Python, e o `docker ps` sem erro |
| `01-build-succeeded` | `sam build --use-container` concluído (Build Succeeded) |
| `02-sam-init` | Criação do projeto (`Name`, `Runtime: python3.12`, template Hello World) |
| `03-estrutura-projeto` | Arquivos e pastas do projeto |
| `04-evento-gerado` | Campos de `events/pedido.json` (POST, `/guias`, `body` com R-001) |
| `05-invoke-duas-vezes` | Duas invocações com evento, com o `Duration` de cada uma |
| `06-start-api-iniciado` | Servidor HTTP no ar (`Mounting HelloWorldFunction ... /guias [POST]`) |
| `07-start-api-sem-warm` | Três chamadas por HTTP sem a flag (contador 1, 2, 3) e logs do servidor |
| `08-start-api-warm-lazy` | Três chamadas com `--warm-containers LAZY` e logs do servidor |
| `09-timeout-erro` | Erro de limite de tempo com Timeout 3 (502 e `timed out`) |
| `10-timeout-resolvido` | Resposta 201 com Timeout 10 (cerca de 5 s) |

## 9. Vídeo

Gravação de tela mostrando a invocação com evento, a chamada por HTTP e o erro de tempo limite, seguido do ajuste do Timeout para 10 e do novo teste.

Link: [https://www.youtube.com/watch?v=8efWkaLxqAs&authuser=0]

## 10. Uso de IA

- **Ferramenta:** Claude (Anthropic).
- **Finalidade:** Apoio na resolução de erros (SAM CLI no Git Bash, Docker/WSL, Git, versão do Python, arquivo de evento), apoio na criação do README.
- **Como validei:** Executei os comandos no meu computador, conferi as saídas nos logs, guardei prints como evidência e usei os valores reais obtidos nos testes.

## 11. Estrutura do repositório

```
rota-vital-serverless/
├── README.md              este arquivo
├── template.yaml          definição da função e da rota POST /guias
├── samconfig.toml         configuração do SAM
├── hello_world/
│   ├── app.py             código da função (fornecido pelo professor)
│   └── requirements.txt
├── events/
│   └── pedido.json        evento de teste (POST /guias)
├── tests/                 testes gerados pelo template do SAM
└── evidencias/            prints dos testes
```
