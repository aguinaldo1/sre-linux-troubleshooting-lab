# INC-001 - Saturação de I/O

## Resumo

Foi realizado um incidente controlado para investigar degradação de desempenho relacionada a operações de entrada e saída (I/O) em um ambiente Linux.

O objetivo foi aplicar uma metodologia estruturada de troubleshooting, partindo do sintoma de degradação, levantando hipóteses, coletando evidências e identificando o processo responsável pela carga de escrita durante o incidente.

Durante o teste foram observados aumento de `I/O wait`, maior utilização do dispositivo `/dev/sdd` e atividade intensa de escrita associada ao processo `dd`.

---

## Ambiente

- Linux executado através do WSL2
- Sistema de arquivos raiz associado ao dispositivo `/dev/sdd`
- Projeto executado no sistema de arquivos raiz

Por se tratar de um ambiente virtualizado pelo WSL2, as métricas do dispositivo de bloco foram interpretadas como evidências de pressão de I/O no ambiente observado.

Os dados coletados não são suficientes para afirmar que existe defeito ou problema físico no dispositivo de armazenamento.

---

## Sintoma

Durante uma carga controlada de escrita, o ambiente apresentou aumento de `I/O wait` e maior utilização do dispositivo de armazenamento.

A investigação buscou determinar se a degradação poderia estar relacionada a:

- saturação de CPU;
- pressão de memória;
- falta de espaço em disco;
- alta atividade de I/O;
- processo consumindo excessivamente recursos de armazenamento.

---

## Hipóteses iniciais

As seguintes hipóteses foram consideradas:

1. Saturação de CPU
2. Pressão de memória
3. Falta de espaço em disco
4. Alta atividade de I/O
5. Processo gerando atividade excessiva de escrita

As hipóteses foram investigadas progressivamente com base nas evidências coletadas.

---

## Investigação

### 1. CPU

A análise inicial utilizando `top` apresentou aproximadamente:

- CPU idle: `84,3%`
- I/O wait: `8,9%`

O alto percentual de CPU ociosa enfraqueceu a hipótese de saturação de CPU.

Ao mesmo tempo, o valor de `I/O wait` chamou atenção para uma possível espera relacionada a operações de entrada e saída.

O `I/O wait`, isoladamente, não foi considerado prova de que o armazenamento era a causa raiz. Ele foi utilizado como sinal para aprofundar a investigação.

---

### 2. Capacidade de disco

A capacidade do sistema de arquivos foi analisada utilizando:

```bash
df -h
```

O sistema de arquivos raiz apresentava aproximadamente:

- Capacidade total: `1007 GB`
- Espaço utilizado: `34-35 GB`
- Utilização: `4%`

Com apenas aproximadamente 4% da capacidade utilizada, a hipótese de falta de espaço em disco perdeu força.

Essa análise também demonstrou uma distinção importante:

> Capacidade de armazenamento e desempenho de armazenamento são problemas diferentes.

Um disco pode possuir bastante espaço disponível e ainda apresentar pressão ou degradação de I/O.

---

### 3. Atividade de I/O no dispositivo

A atividade do dispositivo foi analisada utilizando:

```bash
iostat -xz 1 5
```

Durante o incidente controlado foram observados valores como:

- `I/O wait` chegando a `46,15%`
- utilização de `/dev/sdd` chegando a `95,20%`
- aumento da latência de escrita
- aumento da atividade na fila de I/O

Essas métricas demonstraram períodos de pressão significativa de I/O durante a execução da carga controlada.

---

### 4. Identificação do processo

Depois de identificar pressão no dispositivo, a investigação avançou para descobrir qual processo estava produzindo atividade de I/O.

Foi utilizado:

```bash
pidstat -d 1
```

Durante o incidente foi identificado:

- Processo: `dd`
- PID: `33582`
- Atividade de escrita observada: aproximadamente `262144 kB/s`

O processo `dd` havia sido iniciado intencionalmente para gerar uma carga controlada de escrita.

Essa evidência permitiu correlacionar a atividade do processo com a pressão de I/O observada durante o teste.

---

## Diagnóstico

A degradação observada durante o incidente controlado foi correlacionada com uma carga intensa de escrita gerada pelo processo `dd`.

Durante a execução da carga:

- o `I/O wait` aumentou significativamente;
- `/dev/sdd` apresentou alta utilização;
- houve aumento da latência de escrita;
- `pidstat` identificou o processo `dd` realizando atividade intensa de escrita.

A combinação dessas evidências sustenta a conclusão de que a carga controlada gerada pelo `dd` produziu pressão de I/O no ambiente observado.

As evidências não permitem concluir que existe defeito no dispositivo físico de armazenamento.

---

## Causa raiz

No contexto do incidente controlado, a causa da pressão de I/O foi a carga intensiva de escrita gerada intencionalmente pelo processo:

```text
dd
```

A identificação foi sustentada pela correlação entre as métricas do dispositivo e a atividade de escrita observada no processo.

---

## Mitigação

A carga controlada gerada pelo `dd` foi encerrada, removendo a fonte conhecida de escrita intensiva utilizada no incidente.

Nenhum serviço ou processo não relacionado foi reiniciado ou encerrado sem evidências que justificassem essa ação.

---

## Validação

Após o encerramento do processo `dd`, foram realizadas novas verificações.

Ainda foram observados alguns picos posteriores de I/O.

A investigação adicional mostrou:

- nenhum processo realizando atividade relevante de I/O no `pidstat`;
- valores muito baixos de `Dirty` e `Writeback`;
- `/dev/sdd` associado ao sistema de arquivos raiz utilizado pelo WSL2.

Como os picos posteriores não puderam ser diretamente associados ao processo `dd`, eles não foram atribuídos à mesma causa raiz.

Essa distinção é importante para evitar conclusões que ultrapassem as evidências disponíveis.

---

## Principais comandos utilizados

```bash
top
df -h
iostat -xz 1 5
pidstat -d 1
lsblk
```

### `top`

Utilizado para observar CPU, `I/O wait` e processos em execução.

### `df -h`

Utilizado para verificar a capacidade e utilização dos sistemas de arquivos.

### `iostat -xz 1 5`

Utilizado para observar métricas de desempenho dos dispositivos de armazenamento.

### `pidstat -d 1`

Utilizado para identificar processos responsáveis por atividade de I/O.

### `lsblk`

Utilizado para compreender a relação entre dispositivos de bloco e sistemas de arquivos.

---

## Lições aprendidas

- Lentidão da aplicação não significa automaticamente saturação de CPU.
- `I/O wait` deve ser tratado como um sinal de investigação, e não como prova isolada da causa raiz.
- Capacidade de disco e desempenho de I/O são problemas diferentes.
- `iostat` fornece evidências sobre o comportamento dos dispositivos.
- `pidstat` permite correlacionar atividade de I/O com processos específicos.
- Alta utilização de um dispositivo não significa automaticamente falha física do armazenamento.
- Métricas de dispositivos em ambientes virtualizados como WSL2 precisam ser interpretadas com cuidado.
- Uma mitigação deve sempre ser seguida por validação.
- A causa raiz deve ser sustentada pela correlação de múltiplas evidências.
- Não devemos atribuir eventos posteriores a uma causa anterior sem evidências que sustentem essa relação.
