# Evidências de Troubleshooting

Esta pasta contém as evidências técnicas coletadas durante os incidentes simulados no projeto **Linux Production Troubleshooting Lab**.

O objetivo é separar os dados observados durante a investigação da análise técnica documentada nos relatórios de incidentes.

## Estrutura

Cada incidente possui três arquivos principais:

```text
INC-XXX/
├── baseline.txt
├── incident.txt
└── validation.txt
```

### `baseline.txt`

Registra o estado do ambiente antes da falha controlada.

Exemplos:

* utilização de CPU;
* memória disponível;
* estado do serviço;
* portas em escuta;
* resposta HTTP;
* capacidade de armazenamento.

### `incident.txt`

Registra as evidências observadas durante o incidente.

Exemplos:

* processos responsáveis pelo consumo de recursos;
* métricas de CPU, memória e I/O;
* erros de aplicação;
* logs do systemd;
* portas utilizadas pela aplicação;
* comportamento das requisições.

### `validation.txt`

Registra o estado do ambiente depois da mitigação.

O objetivo é verificar, por meio de novas medições, se o comportamento esperado foi restaurado.

## Incidentes documentados

| Incidente | Cenário                                    |
| --------- | ------------------------------------------ |
| INC-001   | Saturação de I/O                           |
| INC-002   | Saturação de CPU                           |
| INC-003   | Pressão de memória                         |
| INC-004   | Falha de serviço                           |
| INC-005   | Falha de conectividade por porta incorreta |

## Princípio de investigação

As evidências são utilizadas seguindo o fluxo:

```text
Sintoma
   ↓
Hipóteses
   ↓
Coleta de evidências
   ↓
Correlação
   ↓
Diagnóstico
   ↓
Mitigação
   ↓
Validação
```

Uma causa raiz não deve ser declarada sem evidências suficientes para sustentá-la.

Os relatórios completos das investigações estão disponíveis no diretório:

```text
incidents/
```

