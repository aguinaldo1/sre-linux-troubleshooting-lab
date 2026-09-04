# Linux Production Troubleshooting Lab

Laboratório prático de troubleshooting em Linux desenvolvido para simular, investigar e documentar incidentes relacionados ao desempenho e à disponibilidade de uma aplicação.

O projeto utiliza uma aplicação Java controlada como alvo dos experimentos e aplica uma metodologia de investigação baseada em hipóteses, coleta de evidências, diagnóstico, mitigação e validação.

## Problema

Uma empresa possui um servidor Linux executando uma aplicação que começou a apresentar degradação de desempenho e problemas de disponibilidade.

Os usuários relatam sintomas como lentidão, aumento no tempo de resposta e impossibilidade de acessar o serviço.

A causa dos problemas inicialmente é desconhecida e deve ser identificada através de uma investigação estruturada do ambiente.

## Impacto no negócio

Problemas de desempenho e disponibilidade podem impedir que usuários concluam suas operações.

Dependendo do serviço afetado, isso pode resultar em:

* aumento da latência;
* indisponibilidade da aplicação;
* abandono de operações;
* degradação da experiência do usuário;
* aumento da carga operacional das equipes técnicas;
* impacto financeiro para a empresa.

## Objetivos

O objetivo deste laboratório é desenvolver e demonstrar uma metodologia estruturada de troubleshooting em ambientes Linux.

O processo de investigação utilizado nos incidentes segue o fluxo:

```text
Sintoma
   ↓
Hipóteses
   ↓
Coleta de evidências
   ↓
Testes
   ↓
Diagnóstico
   ↓
Causa raiz
   ↓
Mitigação
   ↓
Validação
```

O projeto busca demonstrar capacidade prática para:

* investigar problemas de desempenho;
* analisar utilização de recursos;
* identificar processos responsáveis por degradações;
* investigar falhas de serviços;
* analisar problemas de conectividade;
* correlacionar métricas de sistema e processos;
* determinar causas com base em evidências;
* validar a recuperação após uma mitigação;
* documentar incidentes técnicos.

## Ambiente

O laboratório foi desenvolvido utilizando:

* Linux através do WSL2;
* Java / OpenJDK;
* `systemd`;
* aplicação HTTP Java;
* Git e GitHub.

Ferramentas Linux utilizadas durante as investigações incluem:

* `top`;
* `free`;
* `df`;
* `iostat`;
* `pidstat`;
* `ps`;
* `systemctl`;
* `journalctl`;
* `ss`;
* `curl`;
* `lsblk`.

## Aplicação utilizada no laboratório

Foi criada uma pequena aplicação Java para servir como alvo controlado dos experimentos.

A aplicação disponibiliza o endpoint:

```text
GET /health
```

Quando saudável, a resposta esperada é:

```json
{
  "status": "ok"
}
```

A aplicação é executada como um serviço Linux através do `systemd`.

Fluxo simplificado:

```text
Cliente
   ↓
HTTP :8080
   ↓
SreLabApp
   ↓
/health
   ↓
HTTP 200
```

O serviço utilizado pelo laboratório é:

```text
sre-lab-app.service
```

## Estrutura do projeto

```text
sre-linux-troubleshooting-lab/
├── app/
│   ├── SreLabApp.java
│   └── sre-lab-app.service
│
├── incidents/
│   ├── README.md
│   ├── INC-001-io-saturation.md
│   ├── INC-002-cpu-saturation.md
│   ├── INC-003-memory-pressure.md
│   ├── INC-004-service-failure.md
│   └── INC-005-port-connectivity-failure.md
│
├── .gitignore
└── README.md
```

O arquivo compilado `SreLabApp.class` não é versionado pelo Git.

## Incidentes investigados

| ID      | Incidente                                  | Principal área investigada |
| ------- | ------------------------------------------ | -------------------------- |
| INC-001 | Saturação de I/O                           | Disco / I/O                |
| INC-002 | Saturação de CPU                           | CPU / Processos            |
| INC-003 | Pressão de Memória                         | Memória / Processos        |
| INC-004 | Falha de Serviço                           | systemd / Java / Logs      |
| INC-005 | Falha de Conectividade por Porta Incorreta | Rede / Sockets / Aplicação |

### INC-001 - Saturação de I/O

Investigação de uma carga intensiva de escrita correlacionando métricas do dispositivo com o processo responsável pela atividade.

Principais ferramentas:

```text
top
df
iostat
pidstat
lsblk
```

### INC-002 - Saturação de CPU

Investigação de saturação dos CPUs lógicos disponíveis através de processos CPU-bound controlados.

Principais ferramentas:

```text
top
nproc
yes
```

### INC-003 - Pressão de Memória

Investigação do aumento controlado do consumo de RAM e análise da diferença entre memória livre, memória disponível e utilização de swap.

Principais ferramentas:

```text
free
ps
```

### INC-004 - Falha de Serviço

Investigação de uma aplicação Java que não conseguia iniciar corretamente devido a uma configuração inválida no serviço `systemd`.

Principais ferramentas:

```text
systemctl
journalctl
curl
```

### INC-005 - Falha de Conectividade por Porta Incorreta

Investigação de um cenário no qual o processo permanecia ativo, mas a aplicação estava escutando em uma porta diferente daquela esperada pelo cliente.

Principais ferramentas:

```text
systemctl
ss
curl
```

## Metodologia de troubleshooting

Os incidentes não são solucionados através de tentativa e erro.

A investigação busca seguir quatro princípios:

### 1. Começar pelo sintoma

Primeiro é definido aquilo que está sendo observado pelo usuário ou pelo sistema.

### 2. Trabalhar com hipóteses

Possíveis causas são levantadas antes de executar ações corretivas.

### 3. Coletar evidências

Comandos e métricas são utilizados para fortalecer ou enfraquecer cada hipótese.

### 4. Validar depois da mitigação

Uma correção somente é considerada efetiva depois que novas medições demonstram a recuperação do serviço ou recurso.

## Princípio de investigação

Uma das regras adotadas neste laboratório é:

> Não declarar uma causa raiz sem evidências suficientes para sustentá-la.

Por isso, métricas isoladas são tratadas como sinais de investigação e correlacionadas com outras evidências antes de uma conclusão.

## Competências demonstradas

Este projeto apresenta evidências práticas relacionadas a:

* Linux troubleshooting;
* análise de CPU;
* análise de memória;
* análise de I/O;
* análise de processos;
* gerenciamento de serviços com `systemd`;
* análise de logs com `journalctl`;
* troubleshooting de conectividade;
* investigação de sockets e portas;
* health checks;
* análise de causa raiz;
* mitigação de incidentes;
* validação pós-incidente;
* documentação técnica;
* metodologia de Incident Response.

## Status do projeto

Os primeiros cinco cenários de troubleshooting foram implementados e documentados.

Próximas evoluções do laboratório incluem:

* documentação completa para reprodução do ambiente;
* organização das evidências coletadas;
* diagrama da arquitetura do laboratório;
* revisão final de apresentação para portfólio.

## Documentação dos incidentes

Os relatórios completos estão disponíveis no diretório:

```text
incidents/
```

Cada relatório registra as evidências, diagnóstico, causa raiz, mitigação, validação e principais aprendizados do respectivo incidente.

## Uso de Inteligência Artificial

Ferramentas de Inteligência Artificial foram utilizadas como apoio durante o desenvolvimento deste projeto, principalmente para orientação, revisão de documentação e organização do conteúdo.

Os experimentos, comandos, configurações, simulações de incidentes, coleta de evidências, troubleshooting e validações foram executados e analisados pelo autor.

O objetivo do uso de IA neste projeto foi acelerar o aprendizado e melhorar a qualidade da documentação, mantendo a compreensão e a responsabilidade técnica sobre as implementações apresentadas.

