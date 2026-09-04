# INC-002 - Saturação de CPU

## Resumo

Foi realizado um incidente controlado para investigar saturação de CPU em um ambiente Linux e identificar os processos responsáveis pelo consumo excessivo de processamento.

O objetivo foi aplicar uma metodologia estruturada de troubleshooting, estabelecendo um baseline do ambiente, gerando uma carga controlada de CPU, coletando evidências em nível de sistema e processo e validando a recuperação após a mitigação.

Durante o incidente, os quatro CPUs lógicos disponíveis foram submetidos a processos CPU-bound, resultando em `0,0%` de CPU idle e aumento do `load average`.

---

## Ambiente

- Linux executado através do WSL2
- CPUs lógicos disponíveis: `4`
- Carga controlada gerada com processos `yes`

A quantidade de CPUs lógicos foi confirmada utilizando:

```bash
nproc
```

Resultado:

```text
4
```

Essa informação foi importante para interpretar corretamente o `load average` e determinar o nível de pressão sobre a capacidade de processamento disponível.

---

## Sintoma

Durante uma carga controlada, o ambiente apresentou esgotamento da capacidade disponível de CPU.

O objetivo da investigação foi determinar:

- se realmente existia saturação de CPU;
- qual era a capacidade disponível do ambiente;
- quais processos estavam consumindo CPU;
- se esses processos poderiam ser correlacionados com a degradação observada.

---

## Baseline

Antes de gerar o incidente, o estado da CPU foi analisado.

Foram observados aproximadamente:

- CPU idle: `82,8%`
- CPU user: `2,2%`
- CPU system: `5,4%`
- Load average: `0.59, 0.76, 1.02`

O baseline demonstrava que existia capacidade de processamento disponível.

Com `82,8%` de CPU idle, não havia evidência de saturação de CPU naquele momento.

---

## Geração do incidente

Inicialmente, um processo CPU-bound foi criado utilizando:

```bash
yes > /dev/null
```

O comando `yes` produz continuamente dados na saída padrão.

O redirecionamento:

```text
> /dev/null
```

descarta essa saída, permitindo que o processo continue executando e consumindo CPU sem produzir grandes quantidades de dados no terminal ou em arquivos.

Durante o primeiro teste, um único processo `yes` apresentou aproximadamente:

```text
94,7% CPU
```

Porém, o ambiente possuía quatro CPUs lógicos.

Um único processo CPU-bound não era suficiente para demonstrar saturação de toda a capacidade disponível.

Por esse motivo, foram executados quatro processos CPU-bound simultaneamente para gerar contenção entre os quatro CPUs lógicos disponíveis.

---

## Investigação

### 1. Utilização da CPU

Durante o incidente controlado, foram observados:

- CPU idle: `0,0%`
- CPU user: `13,5%`
- CPU system: `51,0%`
- CPU nice: `35,4%`

O indicador mais importante para o diagnóstico foi:

```text
CPU idle: 0,0%
```

Isso demonstrou que, naquele momento da coleta, não havia capacidade ociosa de CPU disponível.

---

### 2. Load Average

Durante o incidente foi observado:

```text
load average: 5.12, 4.63, 3.05
```

O ambiente possuía:

```text
4 CPUs lógicos
```

O `load average` de `5.12` no intervalo de um minuto indicava uma quantidade de trabalho superior ao número de CPUs lógicos disponíveis naquele período.

Essa métrica foi analisada em conjunto com a CPU idle e com os processos em execução.

O `load average`, isoladamente, não foi utilizado como única evidência de saturação.

---

### 3. Identificação dos processos

Durante o incidente, quatro processos `yes` apareceram entre os principais consumidores de CPU:

| PID | Utilização de CPU |
|---|---:|
| `49852` | `94,7%` |
| `50611` | `90,8%` |
| `50610` | `90,1%` |
| `50609` | `88,2%` |

Os quatro processos estavam executando simultaneamente e apresentavam consumo elevado de CPU.

Como esses processos haviam sido iniciados intencionalmente para gerar uma carga CPU-bound, foi possível correlacionar diretamente sua execução com a pressão observada sobre a CPU.

---

## Diagnóstico

Durante o incidente controlado foi identificada saturação da capacidade de CPU disponível no ambiente.

As principais evidências foram:

- `4` CPUs lógicos disponíveis;
- CPU idle chegando a `0,0%`;
- load average de um minuto chegando a `5.12`;
- quatro processos `yes` executando simultaneamente;
- cada processo consumindo aproximadamente entre `88%` e `95%` de CPU.

A combinação dessas evidências demonstrou que os processos CPU-bound estavam utilizando a capacidade de processamento disponível e produzindo saturação de CPU durante o teste.

---

## Causa raiz

No contexto do incidente controlado, a causa da saturação foi a execução simultânea de quatro processos CPU-bound:

```text
yes
```

Os processos foram iniciados intencionalmente para consumir capacidade de processamento e simular um cenário de pressão de CPU.

A causa foi identificada pela correlação entre:

```text
CPU idle
        +
load average
        +
quantidade de CPUs
        +
consumo dos processos
```

---

## Mitigação

A carga controlada foi encerrada utilizando:

```bash
pkill yes
```

O comando foi utilizado para encerrar os processos `yes` responsáveis pela geração da carga.

A ausência dos processos após a execução do comando confirmou a remoção da carga controlada.

---

## Validação

Após a mitigação, as métricas de CPU foram coletadas novamente.

Foram observados aproximadamente:

- CPU idle: `80,0%`
- CPU user: `1,5%`
- CPU system: `3,1%`
- Load average: `0.99, 0.99, 2.42`

A CPU idle apresentou recuperação de:

```text
0,0% → 80,0%
```

O load average de um minuto apresentou redução de:

```text
5.12 → 0.99
```

A recuperação da CPU idle e a redução do load average após a remoção dos processos demonstraram que a capacidade de processamento foi restaurada.

Durante a validação também foi observado aumento de `I/O wait`.

Esse comportamento não foi atribuído à causa raiz do INC-002 porque não havia evidências suficientes para relacioná-lo à carga CPU-bound investigada neste incidente.

---

## Principais comandos utilizados

```bash
nproc
top
yes > /dev/null
pkill yes
```

### `nproc`

Utilizado para identificar a quantidade de CPUs lógicos disponíveis no ambiente.

### `top`

Utilizado para observar:

- utilização da CPU;
- CPU idle;
- load average;
- processos com maior consumo de CPU.

### `yes > /dev/null`

Utilizado para gerar uma carga CPU-bound controlada.

### `pkill yes`

Utilizado para encerrar os processos responsáveis pela carga controlada.

---

## Lições aprendidas

- Lentidão de uma aplicação não deve ser automaticamente atribuída à CPU.
- Saturação de CPU deve ser comprovada através de métricas.
- Um único processo CPU-bound não necessariamente satura um ambiente com múltiplos CPUs.
- CPU idle ajuda a identificar a capacidade de processamento ainda disponível.
- `load average` deve ser interpretado considerando a quantidade de CPUs disponíveis.
- `load average` não deve ser utilizado isoladamente para determinar saturação.
- A análise dos processos permite identificar quais workloads estão consumindo CPU.
- A correlação entre métricas do sistema e processos fornece evidências mais fortes para determinar a causa raiz.
- A mitigação deve ser seguida por novas medições para confirmar a recuperação.
- Comparar baseline, incidente e pós-mitigação produz evidências mais confiáveis do que analisar uma métrica isoladamente.
