# REPLIC: Orquestrador Baseado em Aprendizado por Reforço para SFCs Resilientes

Este repositório contém o código-fonte oficial e o ambiente de simulação para o **REPLIC**, um orquestrador baseado em *Deep Reinforcement Learning* (DRL) focado na resiliência e alocação adaptativa de Service Function Chains (SFC) para serviços imersivos sob condições severas de falhas.

O simulador de rede adjacente (MUAR-SFC) integra topologias complexas via **NetworkX**, tráfego urbano realista via **Eclipse SUMO**, e o agente de inteligência artificial através da biblioteca **Stable-Baselines3**.

## 🧠 A Inteligência Artificial (REPLIC)

O principal avanço matemático do REPLIC é a formulação do problema de contingência de rede via Processos de Decisão de Markov (MDP), resolvido utilizando **Maskable Proximal Policy Optimization (MaskablePPO)**:

  * **Redundância Geográfica:** Utilização de *action masking* dinâmico para bloquear o agente de alocar réplicas (backups) no mesmo servidor que a VNF primária, acelerando a convergência.
  * **Confiabilidade Virtual Dinâmica:** O agente é recompensado com base na probabilidade de sobrevivência da cadeia, penalizando a sobrecarga sistêmica em servidores específicos (Tierização de Confiabilidade).
  * **Escudo de Cobrança Dupla (Double Charging Shield):** Um ambiente Gymnasium rigorosamente projetado que gera cópias isoladas do grafo da rede a cada passo (step), impedindo a corrupção do estado físico real durante a fase de exploração e cálculo da IA.

## ✨ Destaques de Engenharia

Diferente de simuladores acadêmicos convencionais, este ecossistema foi refatorado seguindo padrões industriais de software:

  * **Performance Atômica:** Acesso a dicionários e atributos nativos do NetworkX com complexidade $O(1)$, substituindo buscas lineares lentas.
  * **Resiliência Baseada em EAFP:** Tratamento de erros idiomático (*Easier to Ask for Forgiveness than Permission*), capturando exceções (como `AttributeError` e `KeyError`) diretamente, garantindo tolerância robusta a falhas em tempo de execução.
  * **Multiplataforma Nativo:** Gerenciamento de caminhos via `Pathlib` e desativação explícita de aceleração por hardware de forma multiplataforma (suportando Linux, macOS ou Windows sem quebra de compatibilidade).
  * **Gerenciamento Twelve-Factor:** Configuração parametrizada e tipada com suporte nativo a arquivos `.env`.

-----

## 🎓 Para a Turma: Entendendo o DRL, o Treinamento e os Pesos

Se vocês vão utilizar, modificar ou debugar o simulador, aqui está um resumo direto de como o "cérebro" do orquestrador funciona.

### 1. Como o Agente "Pensa" (O Ambiente e as Máscaras)
O agente recebe uma **Observação** a cada passo contendo o estado da rede (CPU, cache, custos de banda, latência e confiabilidade). Com base nisso, ele tenta tomar uma **Ação** (escolher o melhor servidor para hospedar uma função virtual).
Para evitar que o agente perca milhares de horas tentando alocar recursos em servidores lotados ou sem caminho de banda, utilizamos o **Action Masking**. Servidores inválidos recebem um "veto" matemático, forçando a IA a escolher apenas rotas viáveis.

### 2. O Sistema de Recompensas (Treinamento)
O aprendizado acontece por tentativa, erro e recompensa. A cada servidor escolhido, o REPLIC calcula o custo total dessa alocação.
* **Falha Leve (Ex: Falta de Recurso):** O episódio acaba e ele recebe uma penalidade de `-40.0`.
* **Falha Grave (Ex: Sem Rota):** Penalidade de `-50.0`.
* **Sucesso Total:** Se ele conseguir costurar toda a cadeia da SFC, recebe um grande bônus de `+40.0`.

### 3. Como Alterar os Pesos (Weights) para Experimentos
O segredo do REPLIC está no dicionário `pesos_fatores`. O modelo atual valoriza a **Confiabilidade acima de tudo**. Os pesos base são:
`{"cpu": 1, "cache": 1, "band": 3, "rel": 6, "lat": 3, "mobile": 0.0}`

Se vocês quiserem fazer o agente ignorar a confiabilidade e focar apenas em economizar largura de banda, basta alterar o valor de `"band"` para `10` e `"rel"` para `1` no arquivo do ambiente (`env_replic.py`) ou passar via inicialização, e observar como o comportamento da rede muda!

### 4. Determinismo vs Exploração
Durante o *treinamento*, a IA escolhe ações com uma certa aleatoriedade (para descobrir novas rotas). Porém, quando rodamos a **simulação oficial**, o REPLIC ativa a flag `deterministic=True`. Isso significa que ele não "joga dados": ele toma a decisão puramente matemática baseada no que aprendeu, garantindo que os testes de vocês sejam 100% reproduzíveis.

-----

## 🚀 Instalação e Configuração

Para garantir a reprodutibilidade absoluta dos resultados do artigo, utilizamos o gerenciador de pacotes [uv](https://github.com/astral-sh/uv) (escrito em Rust).

**1. Instale o uv:**

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

> **🪟 Para Windows (PowerShell):**
> O comando acima é exclusivo para sistemas Unix. No Windows, abra o PowerShell (como Administrador) e execute:
> ```powershell
> powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
> ```
> *(Lembre-se de fechar e reabrir o terminal após a instalação para atualizar o PATH).*

**2. Clone e prepare o ambiente:**

```bash
git clone https://github.com/DavidGBUF/replic.git
cd replic
uv sync
```

> **🪟 Para Windows:**
> Os comandos do Git funcionam da mesma maneira no Prompt de Comando (CMD) ou PowerShell.

**3. Configuração (.env):**
Crie e edite as variáveis de ambiente locais da simulação:
```bash
cp .env.example .env
```

> **🪟 Para Windows:**
> Utilize o comando `copy` ao invés de `cp`:
> ```cmd
> copy .env.example .env
> ```

-----

## 🖥️ Como Executar

O projeto opera como um pacote Python consolidado.

### Simulando a Rede

Para rodar os cenários de avaliação do artigo, utilize os comandos CLI registrados:

```bash
# Simulação Padrão Única
uv run muar-sim

# Simulação em Lote (Testes de Estresse Paralelos)
uv run muar-sim-paralelo
```

> **🪟 Para Windows:** Você pode rodar os comandos `uv run` de forma idêntica. O pacote lida com a execução multiplataforma automaticamente.

### Treinando o Agente REPLIC

Se você deseja treinar um novo modelo ou continuar o treinamento a partir dos pesos (weights) localizados na pasta `rl_saved_models/`:

```bash
# Treinamento padrão usando o ambiente base do REPLIC e MaskablePPO
uv run python src/muar_sfc/train.py --env replic --timesteps 150000 --episodes 1000

# Treinamento sem Action Masking (apenas PPO) para comparação
uv run python src/muar_sfc/train.py --env replic --no-masking
```

-----

## ⚙️ Configuração da Simulação (`.env`)

A simulação gerencia parâmetros dinamicamente. Para alterar o comportamento, injetar falhas ou testar diferentes arquiteturas, edite o arquivo `.env` na raiz do projeto (use sempre o prefixo `MUAR_`).

### Orquestração e Topologia
  * **`MUAR_ALG`**: Algoritmo orquestrador (ex: `replic`, `greedy`, `ga`, `vegeta`).
  * **`MUAR_TOPOLOGY`**: Nome da rede substrata alvo (ex: `luxembourgv2`).
  * **`MUAR_N_SESSIONS`**: Quantidade total de sessões SFC a serem geradas.
  * **`MUAR_N_PLAYERS`**: Usuários móveis paralelos na simulação.

### Controle de Desastres (Falhas e Resiliência)
  * **`MUAR_BACKUP`**: Ativa a funcionalidade de criação de réplicas de resiliência (`True` ou `False`).
  * **`MUAR_NUMBER_OF_FAILS`**: Quantidade de falhas de servidores físicos a serem injetadas.
  * **`MUAR_CRASH_AT`**: Lista de tempos determinísticos para forçar falhas sistêmicas (ex: `[125, 155, 185]`).

### Tierização e Confiabilidade Virtual
A IA reage à confiabilidade baseada em níveis de servidores (Tiers). Calibre as probabilidades:
  * **Alta Confiabilidade (Tier C):** `MUAR_REL_HIGH` (Padrão: `0.999`) | Estresse: `0.02`
  * **Confiabilidade Normal (Tier B):** `MUAR_REL_NORMAL` (Padrão: `0.98`) | Estresse: `0.08`
  * **Baixa Confiabilidade (Tier A):** `MUAR_REL_LOW` (Padrão: `0.95`) | Estresse: `0.15`

-----

## 🏗️ Estrutura do Repositório

```text
replic/
├── src/
│   └── muar_sfc/           # Código-fonte principal
│       ├── main.py         # Ponto de entrada do simulador
│       ├── train.py        # Pipeline de treinamento RL
│       ├── algorithms/     # REPLIC, Inommus, Greedy, etc.
│       ├── controllers/    # Orquestrador, Gerenciador de Backup
│       ├── core/           # Configurações Pydantic e Grafos
│       └── utils/          # Auxiliares de Rede e Heurísticas
├── tests/                  # Testes automatizados (Pytest)
├── rl_saved_models/        # Pesos da rede neural treinada (.zip)
├── pyproject.toml          # Dependências estritamente fixadas
└── .env                    # Topologia e hiperparâmetros
```

## 🧑‍💻 Qualidade de Código

Para checar os padrões de software antes de subir código novo:

```bash
# Linting e formatação automática
uv run ruff check . --fix

# Checagem Estática de Tipos
uv run pyright
```

-----

## 📝 Como Citar

Se você utilizar o simulador REPLIC ou parte de seus módulos matemáticos em sua pesquisa, por favor cite nosso artigo:

```bibtex
@article{REPLIC2026,
  title={REPLIC: Reinforcement Learning-based Orchestrator for Resilient SFC under Severe Failures},
  author={Leonardo, Hugo and de Nazaré, David Galhego and de Brito, Matheus Morais and Erick},
  journal={Journal or Conference Title (To Be Filled)},
  year={2026}
}
```

## 👥 Autores

Pesquisa desenvolvida no âmbito da **Universidade Federal do Pará (UFPA)**.

  * **Hugo Leonardo** — hugosantos@ufpa.br
  * **David Galhego** — david.galhego@icen.ufpa.br
  * **Matheus Morais de Brito** - matheus.moraes.brito@itec.ufpa.br
  * **Erick Mamede Silva da Costa** - erick.costa@itec.ufpa.br
