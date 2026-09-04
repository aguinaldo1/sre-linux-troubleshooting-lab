# INC-004 - Falha de Serviço

## Resumo

Foi realizado um incidente controlado para investigar a indisponibilidade de uma aplicação Java gerenciada pelo `systemd`.

O objetivo foi praticar troubleshooting de serviços Linux, partindo do sintoma de indisponibilidade até a identificação da causa raiz através do estado do serviço e dos logs registrados pelo sistema.

Durante o incidente, uma configuração incorreta foi introduzida intencionalmente no `ExecStart` do serviço, fazendo com que o Java tentasse executar uma classe inexistente.

Como consequência, a aplicação não conseguiu iniciar e entrou em um ciclo de tentativas automáticas de reinicialização.

---

## Ambiente

- Linux executado através do WSL2
- Aplicação desenvolvida em Java
- OpenJDK 18
- Aplicação gerenciada pelo `systemd`
- Serviço: `sre-lab-app.service`
- Endpoint de health check: `/health`
- Porta esperada: `8080`

A aplicação utilizada no laboratório foi criada especificamente para servir como alvo dos experimentos de troubleshooting.

---

## Arquitetura do serviço

O fluxo básico da aplicação era:

```text
systemd
   ↓
sre-lab-app.service
   ↓
Java
   ↓
SreLabApp
   ↓
porta 8080
   ↓
GET /health
```

O serviço utilizava a seguinte configuração principal:

```ini
[Service]
Type=simple
User=aguin
WorkingDirectory=/home/aguin/sre-portfolio/sre-linux-troubleshooting-lab
ExecStart=/usr/bin/java -cp app SreLabApp
Restart=on-failure
RestartSec=3
```

---

## Baseline

Antes da geração do incidente, o serviço apresentava:

```text
Active: active (running)
```

O processo principal era Java:

```text
/usr/bin/java -cp app SreLabApp
```

O log também indicava:

```text
SRE Lab App listening on port 8080
```

A disponibilidade da aplicação foi validada através do health check:

```bash
curl -i http://localhost:8080/health
```

Resultado:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

Portanto, antes do incidente havia evidências tanto em nível de processo quanto em nível de aplicação demonstrando um estado saudável.

---

## Sintoma

Após uma alteração controlada na configuração do serviço, a aplicação deixou de permanecer em execução normalmente.

O `systemd` passou a apresentar:

```text
Active: activating (auto-restart) (Result: exit-code)
```

O processo Java terminava com:

```text
code=exited, status=1/FAILURE
```

O serviço tentava reiniciar automaticamente, mas não conseguia retornar ao estado saudável.

---

## Hipóteses iniciais

Diante de um serviço que não consegue permanecer ativo, algumas hipóteses possíveis incluem:

- erro na configuração do serviço;
- comando `ExecStart` inválido;
- aplicação encerrando durante a inicialização;
- dependência ausente;
- problema de permissão;
- arquivo ou classe necessária não encontrada;
- conflito de porta;
- erro interno da aplicação.

Neste incidente, nenhuma dessas possibilidades foi considerada causa raiz antes da coleta de evidências.

---

## Geração do incidente

A configuração original era:

```ini
ExecStart=/usr/bin/java -cp app SreLabApp
```

Para gerar uma falha controlada, ela foi alterada para:

```ini
ExecStart=/usr/bin/java -cp app SreLabAppBroken
```

A classe:

```text
SreLabAppBroken
```

não existia.

Após a alteração, o `systemd` foi instruído a reler suas configurações:

```bash
sudo systemctl daemon-reload
```

O serviço foi reiniciado:

```bash
sudo systemctl restart sre-lab-app.service
```

---

## Investigação

### 1. Estado do serviço

O primeiro passo foi analisar o estado do serviço:

```bash
systemctl status sre-lab-app.service --no-pager
```

Foi observado:

```text
Active: activating (auto-restart) (Result: exit-code)
```

e:

```text
Process: ExecStart=/usr/bin/java -cp app SreLabAppBroken
code=exited, status=1/FAILURE
```

Essas informações demonstravam que:

1. o `systemd` conseguia executar o comando configurado;
2. o processo Java era iniciado;
3. o processo terminava com erro;
4. o `systemd` tentava iniciar novamente o serviço.

Nesse momento já havia evidência de falha durante a inicialização da aplicação, mas ainda era necessário descobrir o motivo.

---

### 2. Análise dos logs

Os logs do serviço foram consultados utilizando:

```bash
journalctl -u sre-lab-app.service -n 20 --no-pager
```

Foram encontradas as seguintes mensagens:

```text
Error: Could not find or load main class SreLabAppBroken
Caused by: java.lang.ClassNotFoundException: SreLabAppBroken
```

Também foram observadas sucessivas tentativas de reinicialização:

```text
Scheduled restart job, restart counter is at 56.
```

e posteriormente:

```text
Scheduled restart job, restart counter is at 58.
```

Os logs forneceram a evidência necessária para determinar por que o processo Java estava terminando.

---

## Diagnóstico

A falha do serviço foi causada por uma classe Java inválida configurada na diretiva `ExecStart` do `systemd`.

O serviço estava tentando executar:

```text
SreLabAppBroken
```

Porém, essa classe não existia no classpath da aplicação.

A JVM retornou:

```text
java.lang.ClassNotFoundException: SreLabAppBroken
```

Como consequência, o processo Java terminava com:

```text
status=1/FAILURE
```

O serviço estava configurado com:

```ini
Restart=on-failure
```

Por isso, o `systemd` tentava reiniciar automaticamente a aplicação após cada falha.

Entretanto, como a configuração permanecia incorreta, cada nova tentativa executava novamente o mesmo comando inválido.

---

## Causa raiz

A causa raiz foi uma configuração incorreta na diretiva:

```text
ExecStart
```

do serviço `systemd`.

A configuração inválida era:

```ini
ExecStart=/usr/bin/java -cp app SreLabAppBroken
```

A classe `SreLabAppBroken` não existia, provocando uma `ClassNotFoundException` e impedindo a inicialização da aplicação.

---

## Restart Loop

O incidente também demonstrou um comportamento importante relacionado à recuperação automática.

O serviço possuía:

```ini
Restart=on-failure
RestartSec=3
```

O comportamento observado foi:

```text
systemd inicia o Java
        ↓
Java procura SreLabAppBroken
        ↓
classe não encontrada
        ↓
processo termina com status 1
        ↓
systemd aguarda
        ↓
systemd tenta novamente
        ↓
mesma falha ocorre
```

As evidências mostraram dezenas de tentativas de reinicialização.

Isso demonstrou que uma política de restart pode recuperar determinadas falhas transitórias, mas não corrige automaticamente uma falha determinística de configuração.

---

## Mitigação

A configuração incorreta foi corrigida.

O `ExecStart` foi restaurado para:

```ini
ExecStart=/usr/bin/java -cp app SreLabApp
```

Em seguida, o `systemd` foi instruído a recarregar a configuração:

```bash
sudo systemctl daemon-reload
```

O serviço foi reiniciado:

```bash
sudo systemctl restart sre-lab-app.service
```

---

## Validação

Após a mitigação, o estado do serviço foi novamente verificado.

Resultado:

```text
Active: active (running)
```

O processo Java voltou a executar corretamente:

```text
/usr/bin/java -cp app SreLabApp
```

O log confirmou a inicialização:

```text
SRE Lab App listening on port 8080
```

A recuperação também foi validada pela perspectiva do cliente:

```bash
curl -i http://localhost:8080/health
```

Resultado:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

A validação em dois níveis confirmou a recuperação:

```text
systemctl → processo ativo
curl      → aplicação respondendo
```

---

## Principais comandos utilizados

```bash
systemctl status sre-lab-app.service --no-pager
journalctl -u sre-lab-app.service -n 20 --no-pager
sudo systemctl daemon-reload
sudo systemctl restart sre-lab-app.service
curl -i http://localhost:8080/health
```

### `systemctl status`

Utilizado para verificar o estado do serviço, processo principal, código de saída e comportamento de reinicialização.

### `journalctl`

Utilizado para consultar os logs associados ao serviço e encontrar a mensagem de erro produzida pela JVM.

### `systemctl daemon-reload`

Utilizado para fazer o `systemd` reler os arquivos de configuração após uma alteração.

### `systemctl restart`

Utilizado para reiniciar o serviço após a correção.

### `curl`

Utilizado para validar a aplicação pela perspectiva do cliente.

---

## Lições aprendidas

- Um servidor funcionando não significa que a aplicação esteja disponível.
- O estado `active (running)` deve ser complementado por testes em nível de aplicação.
- `systemctl status` fornece evidências importantes sobre o ciclo de vida de um serviço.
- Um código de saída diferente de zero indica que o processo terminou com erro, mas não necessariamente explica sozinho a causa.
- `journalctl` permite aprofundar a investigação através dos logs do serviço.
- Reiniciar um serviço repetidamente não substitui a investigação da causa raiz.
- `Restart=on-failure` pode ajudar na recuperação de falhas transitórias, mas não corrige configurações inválidas.
- Uma falha determinística tende a se repetir enquanto sua causa permanecer presente.
- A mitigação deve corrigir a causa identificada pelas evidências.
- A recuperação deve ser validada tanto no nível do processo quanto na perspectiva do usuário.
- Troubleshooting estruturado reduz a dependência de tentativa e erro.
