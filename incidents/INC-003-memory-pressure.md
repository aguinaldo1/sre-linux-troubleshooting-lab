# INC-003 - Pressão de Memória

## Resumo

Foi realizado um incidente controlado para investigar pressão de memória em um ambiente Linux, identificar os processos responsáveis pelo aumento do consumo de RAM e validar a recuperação do recurso após a mitigação.

O objetivo foi aplicar uma metodologia estruturada de troubleshooting, estabelecendo um baseline, aumentando o consumo de memória de forma controlada, coletando evidências em nível de sistema e processo e comparando o estado do ambiente antes, durante e depois do incidente.

Durante o teste, dois processos Python alocaram aproximadamente `1,5 GiB` de memória, reduzindo significativamente a quantidade de memória disponível no ambiente.

---

## Ambiente

- Linux executado através do WSL2
- Memória total: `3,8 GiB`
- Swap total: `1,0 GiB`
- Docker ativo
- Workloads Kubernetes ativos

Por já existirem outros workloads executando no ambiente, a geração do incidente foi realizada de forma progressiva para evitar esgotamento desnecessário da memória ou acionamento do OOM Killer.

---

## Sintoma

Durante uma carga controlada, o ambiente apresentou redução significativa da quantidade de memória disponível.

A investigação buscou determinar:

- quanto de memória estava realmente disponível;
- quais processos estavam consumindo memória;
- se havia utilização significativa de swap;
- se o ambiente estava apenas sob pressão de memória ou próximo de esgotamento;
- se ocorreu alguma condição de Out Of Memory (OOM).

---

## Baseline

Antes do incidente, a memória foi analisada utilizando:

```bash
free -h
```

Foram observados aproximadamente:

- Memória total: `3,8 GiB`
- Memória utilizada: `920 MiB`
- Memória livre: `1,6 GiB`
- Memória disponível: `2,7 GiB`
- Swap utilizado: `20 MiB`

O baseline não demonstrava pressão significativa de memória.

A análise dos processos também mostrou que um dos maiores consumidores de memória naquele momento era:

```text
kube-apiserver
```

com aproximadamente:

```text
258340 KiB RSS
```

---

## Geração do incidente

O consumo de memória foi aumentado intencionalmente utilizando processos Python com alocação através de `bytearray`.

A primeira carga foi criada utilizando aproximadamente `768 MiB`:

```bash
python3 -c "a=bytearray(768*1024*1024); input('768 MiB allocated. Press ENTER to release memory...')"
```

O processo permaneceu ativo para manter a memória alocada durante a investigação.

Posteriormente, um segundo processo realizou outra alocação de aproximadamente `768 MiB`.

Com os dois processos ativos, aproximadamente:

```text
1,5 GiB
```

de memória estava sendo utilizada pela carga controlada.

---

## Investigação

### 1. Primeira alocação

Após a primeira alocação de aproximadamente `768 MiB`, foram observados:

- Memória utilizada: `1,6 GiB`
- Memória livre: `826 MiB`
- Memória disponível: `1,9 GiB`
- Swap utilizado: `20 MiB`

A memória disponível caiu de:

```text
2,7 GiB → 1,9 GiB
```

demonstrando o impacto da carga controlada.

---

### 2. Identificação do processo

A investigação em nível de processo identificou:

- Processo: `python3`
- PID: `57557`
- Utilização de memória: `20,0%`
- RSS: `794880 KiB`

O valor de RSS observado era compatível com a alocação controlada de aproximadamente `768 MiB`.

Essa evidência permitiu correlacionar o aumento do consumo de memória com o processo Python utilizado no teste.

---

### 3. Aumento da pressão de memória

Uma segunda alocação de aproximadamente `768 MiB` foi iniciada.

Com aproximadamente `1,5 GiB` de memória alocada pelos processos controlados, foram observados:

- Memória utilizada: `2,4 GiB`
- Memória livre: `121 MiB`
- Memória disponível: `1,2 GiB`
- Swap utilizado: `27 MiB`

A memória livre caiu significativamente:

```text
1,6 GiB → 121 MiB
```

Porém, a memória disponível ainda era aproximadamente:

```text
1,2 GiB
```

Essa diferença foi importante para evitar uma conclusão incorreta de que o ambiente estava sem memória.

---

### 4. Análise de Swap

O uso de swap antes do incidente era aproximadamente:

```text
20 MiB
```

Durante o período de maior pressão foi observado:

```text
27 MiB
```

A diferença foi de apenas aproximadamente:

```text
7 MiB
```

Não foram coletadas evidências de utilização intensa de swap durante o incidente.

Portanto, o cenário não foi classificado como severe swapping.

---

## Diagnóstico

A carga controlada produziu pressão de memória no ambiente Linux.

A memória disponível apresentou redução de:

```text
2,7 GiB → 1,2 GiB
```

Essa redução foi correlacionada com dois processos Python que alocaram aproximadamente `1,5 GiB` de RAM.

A investigação em nível de processo identificou diretamente o workload Python como consumidor significativo de memória.

Apesar da forte redução da memória livre, as evidências mostraram que:

- ainda havia aproximadamente `1,2 GiB` de memória disponível;
- o aumento de swap foi pequeno;
- não houve evidência de swapping severo;
- não houve evidência de Out Of Memory;
- não houve evidência de atuação do OOM Killer.

Portanto, o incidente foi classificado como **pressão de memória**, e não como esgotamento completo de memória.

---

## Causa raiz

No contexto do incidente controlado, a pressão de memória foi causada pela execução simultânea de processos Python realizando alocações através de `bytearray`.

Aproximadamente:

```text
1,5 GiB
```

de RAM foi alocada intencionalmente durante o teste.

A causa foi sustentada pela correlação entre:

```text
redução de memória disponível
        +
RSS do processo
        +
alocação controlada
        +
recuperação após encerramento
```

---

## Mitigação

Os processos Python responsáveis pelas alocações controladas foram encerrados.

Como os processos estavam aguardando entrada através de:

```python
input(...)
```

a execução foi finalizada, permitindo que a memória alocada fosse liberada.

Nenhum processo não relacionado foi encerrado para liberar memória.

---

## Validação

Após a mitigação, as métricas foram coletadas novamente.

Foram observados aproximadamente:

- Memória utilizada: `840 MiB`
- Memória livre: `1,7 GiB`
- Memória disponível: `2,7 GiB`
- Swap utilizado: `27 MiB`

A memória disponível apresentou recuperação de:

```text
1,2 GiB → 2,7 GiB
```

O valor retornou ao mesmo nível observado no baseline:

```text
Baseline:       2,7 GiB available
Incidente:      1,2 GiB available
Pós-mitigação:  2,7 GiB available
```

Essa recuperação após a remoção da carga controlada reforçou a correlação entre os processos Python e a pressão de memória observada.

O swap permaneceu em aproximadamente `27 MiB`.

O fato de o swap não retornar imediatamente ao valor anterior não foi interpretado como falha na mitigação, pois a recuperação da capacidade de RAM foi confirmada pela memória disponível.

---

## Principais comandos utilizados

```bash
free -h
ps aux --sort=-%mem | head
```

### `free -h`

Utilizado para analisar:

- memória total;
- memória utilizada;
- memória livre;
- memória disponível;
- utilização de swap.

### `ps aux --sort=-%mem`

Utilizado para ordenar os processos pelo percentual de utilização de memória e identificar os maiores consumidores.

---

## Conceitos importantes

### `free`

Representa memória completamente não utilizada naquele momento.

Um valor baixo de `free` não significa automaticamente que o sistema está ficando sem memória.

### `available`

Representa uma estimativa da quantidade de memória que ainda pode ser utilizada por novas aplicações sem necessidade significativa de swap.

Durante troubleshooting, esse valor fornece contexto importante para interpretar a situação real da memória.

### RSS

`Resident Set Size` representa aproximadamente a quantidade de memória física atualmente residente utilizada por um processo.

No incidente, o RSS ajudou a correlacionar o processo Python com a alocação controlada.

### Swap

Área de armazenamento utilizada pelo sistema como suporte ao gerenciamento de memória.

A existência de algum uso de swap não significa, isoladamente, que existe pressão severa de memória.

---

## Lições aprendidas

- Um valor baixo de memória `free` não significa automaticamente esgotamento de RAM.
- A memória `available` fornece contexto importante durante troubleshooting de memória no Linux.
- Linux utiliza memória disponível para cache e buffers quando isso é útil.
- RSS ajuda a identificar o consumo de memória física associado aos processos.
- O consumo de memória deve ser correlacionado com evidências em nível de processo.
- Uso de swap deve ser medido, e não presumido.
- Pressão de memória não significa necessariamente ocorrência de Out Of Memory.
- Não devemos afirmar que ocorreu OOM sem evidências que sustentem essa conclusão.
- Experimentos controlados devem evitar esgotar desnecessariamente ambientes que executam outros workloads.
- Baseline, incidente e pós-mitigação permitem demonstrar claramente o impacto e a recuperação.
- Uma mitigação deve sempre ser seguida por novas medições para confirmar a recuperação do recurso.
