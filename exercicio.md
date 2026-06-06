# Exercícios de CI

---

## Questão 1: Migração de Pipeline — GitHub Actions para Jenkins

### Contexto

Este projeto já possuía uma pipeline de testes E2E configurada no GitHub Actions (`.github/workflows/01-manual-exec.yaml`). O exercício consistiu em replicar essa mesma pipeline utilizando o Jenkins, executando localmente via Windows.

### O que cada pipeline faz

Ambas executam os mesmos passos:

1. Faz o clone do repositório
2. Instala o Yarn
3. Instala as dependências do projeto (`yarn`)
4. Instala os browsers do Playwright
5. Executa os testes E2E (`yarn run e2e`)

### Comparação: GitHub Actions vs Jenkins

| GitHub Actions (`.github/workflows/01-manual-exec.yaml`) | Jenkins (`Jenkinsfile`) |
|---|---|
| Disparo manual via `workflow_dispatch` | Disparo manual via botão "Build Now" |
| `runs-on: ubuntu-latest` | `agent any` (roda na máquina local) |
| `actions/checkout@v4` | Checkout automático pelo Jenkins |
| `actions/setup-node@v4` | Node já instalado na máquina |
| Comandos com `run:` (shell Linux) | Comandos com `bat` (Windows) |

> **Por que `bat` em vez de `sh`?** O Jenkins estava rodando no Windows. O `sh` é um comando Linux/Mac — no Windows é necessário usar `bat` para executar comandos no terminal.

### Como executar a pipeline no Jenkins

**Pré-requisitos:**
- [Java JDK 21+](https://adoptium.net) instalado
- [Node.js 24+](https://nodejs.org) instalado
- Arquivo `jenkins.war` baixado em [jenkins.io/download](https://www.jenkins.io/download/)

**1. Iniciar o Jenkins**

Abra o terminal na pasta onde está o `jenkins.war` e execute:

```bash
java -jar jenkins.war
```

Aguarde a mensagem `Jenkins is fully up and running` e acesse **http://localhost:8080**.

**2. Configuração inicial (apenas na primeira vez)**

1. Copie a senha exibida no terminal (entre os `*****`)
2. Cole no navegador e clique em **Continue**
3. Clique em **"Install suggested plugins"** e aguarde
4. Crie um usuário admin e finalize a configuração

**3. Criar o job**

1. Na tela inicial, clique em **"New Item"**
2. Digite o nome `pgats-ci`, selecione **Pipeline** e clique em **OK**

**4. Configurar o job**

Na seção **Pipeline** da página de configuração:

- **Definition:** `Pipeline script from SCM`
- **SCM:** `Git`
- **Repository URL:** `https://github.com/deboraeverlyab/pgats-ci.git`
- **Branch Specifier:** `*/master`
- **Script Path:** `Jenkinsfile`

Clique em **Save**.

**5. Executar**

1. Clique em **"Build Now"**
2. Em **"Build History"**, clique no build `#1`
3. Clique em **"Console Output"** para acompanhar a execução

### Resultado esperado

A pipeline executa os 3 testes E2E. O resultado pode variar conforme o estado da aplicação:

- **2 testes passando** — comportamentos de restrição por altura funcionando corretamente
- **1 teste falhando** — bug conhecido na aplicação: após clicar em "Next", a URL não redireciona para a tela de sucesso

O Jenkins reporta `FAILURE` quando qualquer teste falha — esse é o comportamento correto de uma pipeline de CI.

---

## Questão 2: Análise de Falhas com IA via GitHub Models

### Contexto

O GitHub Marketplace disponibiliza actions e integrações que podem agregar valor ao fluxo de CI. O exercício consistiu em escolher uma integração útil e implementá-la na pipeline existente do GitHub Actions.

A escolha foi adicionar **análise automática de falhas com IA** utilizando o **GitHub Models** — serviço gratuito do GitHub que permite usar modelos de linguagem diretamente nas pipelines, sem necessidade de chave de API externa ou créditos adicionais.

### O que foi adicionado

Dois novos passos foram incluídos na pipeline `01-manual-exec.yaml`:

1. **Captura do output dos testes** — o resultado da execução é salvo em `test-output.txt` com `continue-on-error: true`, garantindo que a análise rode mesmo quando há falhas
2. **Análise com IA** — se algum teste falhar, um script Python envia o output para o modelo `meta-llama-3.1-70b-instruct` via GitHub Models API e publica a análise em português na aba **Summary** da pipeline

### Por que GitHub Models e não outra API de IA

O GitHub Models usa o `github.token`, que já existe automaticamente em toda execução de pipeline — sem configuração de secrets, sem cadastro em serviços externos e sem custo adicional.

### Como funciona na prática

```
Testes rodam
    ↓
Algum teste falhou?
    ├── Não → pipeline termina com Success
    └── Sim → IA analisa o output e publica explicação na aba Summary
               → pipeline termina com Failure
```

### Onde ver o resultado

Na aba **Summary** da execução no GitHub Actions, aparece a seção **"Análise de Falhas com IA"** com a explicação em português das falhas encontradas e as possíveis causas.

> A análise só aparece quando há falha — se todos os testes passarem, o passo de IA é ignorado automaticamente.

---

## Questão 3: Self-Hosted Runner no GitHub Actions

### O que é um self-hosted runner

Por padrão, o GitHub Actions executa as pipelines em servidores próprios do GitHub (runners gerenciados), como `ubuntu-latest` ou `windows-latest`. Um **self-hosted runner** é uma máquina própria — local ou em nuvem — registrada no GitHub para executar os jobs no lugar desses servidores.

### Quando faz sentido usar

| Cenário | Por quê usar self-hosted |
|---|---|
| Testes que precisam de hardware específico | GPU, dispositivos físicos, navegadores instalados localmente |
| Ambientes corporativos sem acesso à internet | A máquina própria já está na rede interna |
| Alto volume de execuções | Evita custos com minutos do GitHub Actions |
| Dependências pesadas já instaladas | Evita reinstalar Node, browsers, etc. a cada execução |
| Acesso a serviços internos | Bancos de dados, APIs internas não expostas publicamente |

### Outras plataformas com recurso similar

| Plataforma | Nome do recurso |
|---|---|
| GitHub Actions | Self-hosted runners |
| GitLab CI | Self-hosted runners / GitLab Runner |
| Jenkins | Agents (nós conectados ao servidor Jenkins) |
| Azure DevOps | Self-hosted agents |
| Bitbucket Pipelines | Runners auto-hospedados |
| CircleCI | Self-hosted runners |

> O Jenkins em si já é uma plataforma inteiramente self-hosted — configurado na Questão 1, ele roda na máquina local sem depender de nenhum servidor externo.

### O que foi alterado na pipeline

A única mudança necessária no arquivo `01-manual-exec.yaml` foi trocar o valor de `runs-on`:

```yaml
# antes — servidor gerenciado pelo GitHub
runs-on: ubuntu-latest

# depois — máquina própria registrada como runner
runs-on: self-hosted
```

Também foi adicionado `shell: bash` no passo de testes, pois o runner roda no Windows e o shell padrão nesse sistema é o PowerShell — o `bash` é necessário para os comandos `tee` e `${PIPESTATUS[0]}` funcionarem corretamente via Git Bash.

### Como registrar a máquina como self-hosted runner

**1.** No repositório do GitHub, acesse: **Settings → Actions → Runners → New self-hosted runner**

**2.** Selecione o sistema operacional (**Windows**) e a arquitetura (**x64**)

**3.** Siga os comandos exibidos na tela para baixar e configurar o runner na máquina:

```powershell
# criar pasta e entrar nela
mkdir actions-runner; cd actions-runner

# baixar o agente (versão exibida na tela do GitHub)
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/vX.X.X/actions-runner-win-x64-X.X.X.zip -OutFile actions-runner-win-x64.zip

# extrair
Add-Type -AssemblyName System.IO.Compression.FileSystem
[System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD/actions-runner-win-x64.zip", "$PWD")

# configurar (o token é gerado automaticamente na página do GitHub)
./config.cmd --url https://github.com/deboraeverlyab/pgats-ci --token SEU_TOKEN_AQUI

# iniciar o runner
./run.cmd
```

**4.** Com o runner ativo, execute a pipeline normalmente pelo GitHub Actions — o job será roteado para a máquina local em vez dos servidores do GitHub.
