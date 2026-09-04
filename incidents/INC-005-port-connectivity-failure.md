# INC-005 - Falha de Conectividade por Porta Incorreta

## Resumo

Foi realizado um incidente controlado para investigar uma falha de conectividade em uma aplicação Java que permanecia ativa no sistema, mas não respondia na porta esperada pelos clientes.

O objetivo foi praticar troubleshooting de conectividade, diferenciando o estado do processo, a porta utilizada pela aplicação e a disponibilidade real do endpoint.

Durante o incidente, a aplicação foi alterada intencionalmente para escutar na porta `8081`, enquanto o cliente continuava tentando acessar a porta `8080`.

Como resultado, o serviço permanecia ativo no `systemd`, porém as conexões realizadas na porta esperada eram recusadas.

---

## Ambiente

- Linux executado através do WSL2
- Aplicação desenvolvida em Java
- Aplicação gerenciada pelo `systemd`
- Serviço: `sre-lab-app.service`
- Endpoint: `/health`
- Porta esperada pelo cliente: `8080`
- Porta utilizada durante o incidente: `8081`

---

## Arquitetura esperada

O funcionamento normal da aplicação era:

```text
Cliente
   ↓
localhost:8080
   ↓
SreLabApp
   ↓
GET /health
   ↓
HTTP 200
```

A aplicação Java utilizava:

```java
HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);
```

Portanto, a porta esperada para o serviço era:

```text
8080
```

---

## Baseline

Antes da geração do incidente, a aplicação estava funcionando normalmente.

O health check foi realizado utilizando:

```bash
curl -i http://localhost:8080/health
```

A aplicação respondeu:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

A porta utilizada pelo processo também foi verificada:

```bash
ss -ltnp | grep :8080
```

A saída confirmou que o processo Java estava escutando na porta:

```text
*:8080
```

Nesse momento, havia evidências de que:

- o processo Java estava em execução;
- a aplicação estava escutando na porta esperada;
- o endpoint `/health` estava acessível;
- o cliente recebia uma resposta HTTP válida.

---

## Sintoma

Após a geração do incidente, uma tentativa de acessar:

```text
http://localhost:8080/health
```

falhou.

O `curl` não conseguiu estabelecer conexão com a aplicação na porta `8080`.

Entretanto, o serviço continuava ativo no `systemd`.

Esse comportamento criou um cenário importante de troubleshooting:

```text
Serviço ativo
     ≠
Aplicação disponível na porta esperada
```

---

## Hipóteses iniciais

Diante de uma aplicação aparentemente ativa, mas inacessível, algumas hipóteses possíveis incluem:

1. serviço parado;
2. aplicação encerrada;
3. porta incorreta;
4. processo não escutando em nenhuma porta;
5. conflito de porta;
6. aplicação escutando somente em uma interface específica;
7. bloqueio de conectividade;
8. erro interno da aplicação.

Nenhuma hipótese foi considerada causa raiz antes da coleta de evidências.

---

## Geração do incidente

A aplicação estava originalmente configurada para utilizar:

```java
new InetSocketAddress(8080)
```

Para gerar uma falha controlada, a porta foi alterada para:

```java
new InetSocketAddress(8081)
```

A aplicação foi recompilada:

```bash
javac app/SreLabApp.java
```

Em seguida, o serviço foi reiniciado:

```bash
sudo systemctl restart sre-lab-app.service
```

A alteração fez com que o processo Java passasse a escutar na porta `8081`.

---

## Investigação

### 1. Estado do serviço

O serviço permaneceu ativo após a alteração.

O `systemd` apresentava o processo Java em execução.

Essa evidência enfraqueceu a hipótese de que a indisponibilidade havia sido causada simplesmente pela interrupção do serviço.

A investigação precisava continuar em outra camada.

---

### 2. Teste da porta esperada

A aplicação foi testada na porta utilizada normalmente pelos clientes:

```bash
curl -i http://localhost:8080/health
```

A conexão foi recusada.

Isso demonstrou que, embora o processo estivesse ativo, a aplicação não estava disponível na porta esperada.

---

### 3. Verificação dos sockets

Foi utilizado:

```bash
ss -ltnp
```

para identificar as portas TCP em estado de listening.

A investigação mostrou que o processo Java estava associado a:

```text
*:8081
```

e não a:

```text
*:8080
```

Essa foi a principal evidência para direcionar o diagnóstico.

---

### 4. Teste da porta 8081

Após identificar que o processo estava escutando na porta `8081`, o endpoint foi testado diretamente:

```bash
curl -i http://localhost:8081/health
```

Resultado:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

Essa evidência demonstrou que:

- o processo Java estava funcionando;
- a aplicação estava saudável;
- o endpoint `/health` estava funcionando;
- o problema estava relacionado à porta utilizada para exposição do serviço.

---

## Evidência adicional - Log inconsistente

Durante a investigação foi observado um comportamento importante.

Mesmo após a aplicação ter sido alterada para escutar na porta `8081`, o log continuava apresentando:

```text
SRE Lab App listening on port 8080
```

Porém, a análise utilizando:

```bash
ss -ltnp
```

demonstrava que o processo estava realmente escutando em:

```text
*:8081
```

A causa da inconsistência era a mensagem de log ter sido escrita de forma fixa no código.

Em outras palavras:

```text
Log informado:     8080
Socket observado:  8081
```

Nesse cenário, a evidência obtida diretamente do estado do sistema foi utilizada para determinar o comportamento real da aplicação.

---

## Diagnóstico

A indisponibilidade na porta `8080` foi causada pela alteração da porta utilizada pela aplicação Java.

A aplicação estava configurada para escutar em:

```text
8081
```

enquanto o cliente continuava tentando acessar:

```text
8080
```

As principais evidências foram:

- serviço `systemd` ativo;
- falha de conexão na porta `8080`;
- processo Java escutando em `*:8081`;
- ausência do listener esperado em `8080`;
- resposta HTTP `200` ao acessar `/health` através da porta `8081`.

A combinação dessas evidências permitiu localizar a falha na configuração de porta da aplicação.

---

## Causa raiz

No contexto do incidente controlado, a causa raiz foi uma configuração incorreta da porta utilizada pela aplicação.

A configuração esperada era:

```java
new InetSocketAddress(8080)
```

Durante o incidente ela foi alterada para:

```java
new InetSocketAddress(8081)
```

Isso criou uma divergência entre:

```text
porta esperada pelo cliente
          ↓
         8080

porta utilizada pela aplicação
          ↓
         8081
```

---

## Mitigação

A configuração da aplicação foi restaurada para:

```java
new InetSocketAddress(8080)
```

A aplicação foi recompilada:

```bash
javac app/SreLabApp.java
```

O serviço foi reiniciado:

```bash
sudo systemctl restart sre-lab-app.service
```

---

## Validação

Após a mitigação, o endpoint foi testado novamente:

```bash
curl -i http://localhost:8080/health
```

Resultado:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

Também foi realizada nova verificação dos sockets:

```bash
ss -ltnp | grep :8080
```

O processo Java voltou a aparecer associado a:

```text
*:8080
```

A recuperação foi confirmada através de duas perspectivas:

```text
Socket
  ↓
Java escutando em 8080

Aplicação
  ↓
GET /health
  ↓
HTTP 200
```

---

## Principais comandos utilizados

```bash
curl -i http://localhost:8080/health
curl -i http://localhost:8081/health
ss -ltnp
javac app/SreLabApp.java
sudo systemctl restart sre-lab-app.service
```

### `curl`

Utilizado para testar a aplicação pela perspectiva do cliente e verificar se o endpoint HTTP estava acessível.

### `ss`

Utilizado para verificar sockets TCP em estado de listening e identificar em qual porta o processo Java estava realmente escutando.

### `javac`

Utilizado para recompilar a aplicação Java após as alterações controladas.

### `systemctl restart`

Utilizado para reiniciar o serviço e carregar a nova versão da aplicação.

---

## Lições aprendidas

- Um processo ativo não significa automaticamente que a aplicação está disponível para os clientes.
- `systemctl` informa o estado do serviço, mas não substitui testes de disponibilidade da aplicação.
- Uma aplicação pode estar saudável e ainda assim estar inacessível na porta esperada.
- `curl` permite testar o comportamento da aplicação pela perspectiva do cliente.
- `ss` permite verificar diretamente quais portas estão em estado de listening.
- A porta esperada pelo cliente deve corresponder à porta utilizada pela aplicação.
- Logs devem ser tratados como uma fonte de evidência, mas podem conter informações incorretas ou desatualizadas.
- Logs hardcoded podem gerar informações inconsistentes durante troubleshooting.
- Evidências de diferentes camadas devem ser correlacionadas antes de determinar a causa raiz.
- A mitigação deve ser validada tanto no nível de rede quanto no nível da aplicação.
- Troubleshooting não deve parar apenas porque o serviço aparece como `active (running)`.
