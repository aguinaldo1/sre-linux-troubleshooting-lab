# Setup do Linux Production Troubleshooting Lab

Este documento descreve como preparar e executar o laboratório em um ambiente Linux.

O objetivo é permitir que outra pessoa consiga reproduzir a aplicação utilizada nos incidentes e validar o funcionamento do endpoint de health check.

## Pré-requisitos

O ambiente utilizado durante o desenvolvimento foi:

* Linux através do WSL2
* Java / OpenJDK 18
* `javac`
* `systemd`
* `curl`
* Git

Verifique a instalação do Java:

```bash
java -version
```

Verifique o compilador Java:

```bash
javac -version
```

Verifique o caminho do Java instalado:

```bash
which java
```

No ambiente original do laboratório, o Java estava disponível em:

```text
/usr/bin/java
```

## Estrutura principal

Os arquivos utilizados para executar a aplicação estão em:

```text
app/
├── SreLabApp.java
└── sre-lab-app.service
```

A aplicação disponibiliza:

```text
GET /health
```

na porta:

```text
8080
```

## 1. Clonar o projeto

Clone o repositório e acesse o diretório do projeto.

Exemplo:

```bash
git clone <URL_DO_REPOSITORIO>
cd sre-linux-troubleshooting-lab
```

Confirme que está na raiz do projeto:

```bash
pwd
```

## 2. Compilar a aplicação

Execute:

```bash
javac app/SreLabApp.java
```

Esse comando gera o arquivo compilado:

```text
app/SreLabApp.class
```

O arquivo `.class` não é versionado pelo Git.

## 3. Executar manualmente

Antes de iniciar a aplicação manualmente, verifique se a porta `8080` já está sendo utilizada:

```bash
sudo ss -ltnp | grep :8080
```

Se não houver saída, a porta não possui um listener identificado por esse comando.

Execute a aplicação:

```bash
java -cp app SreLabApp
```

A saída esperada é:

```text
SRE Lab App listening on port 8080
```

Mantenha esse terminal aberto.

## 4. Validar o health check

Em outro terminal, execute:

```bash
curl -i http://localhost:8080/health
```

Resultado esperado:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

Isso confirma que a aplicação está acessível pela porta esperada e que o endpoint `/health` está respondendo.

Para encerrar a execução manual, volte ao terminal da aplicação e pressione:

```text
Ctrl + C
```

## 5. Troubleshooting de porta ocupada

Caso a aplicação apresente:

```text
java.net.BindException: Address already in use
```

isso indica que a porta utilizada pela aplicação já está ocupada.

Verifique qual processo está escutando na porta `8080`:

```bash
sudo ss -ltnp | grep :8080
```

Não encerre processos aleatoriamente.

Primeiro identifique qual aplicação ou serviço está utilizando a porta.

Se o processo estiver sendo gerenciado pelo `systemd`, verifique o serviço correspondente antes de tomar qualquer ação.

No laboratório, por exemplo:

```bash
systemctl status sre-lab-app.service --no-pager
```

Se for necessário parar esse serviço:

```bash
sudo systemctl stop sre-lab-app.service
```

Depois confirme novamente:

```bash
sudo ss -ltnp | grep :8080
```

## 6. Preparar o serviço systemd

O repositório contém o template:

```text
app/sre-lab-app.service
```

Antes de utilizá-lo, descubra o usuário Linux atual:

```bash
whoami
```

Descubra o caminho absoluto do projeto:

```bash
pwd
```

Descubra o caminho do Java:

```bash
which java
```

O template contém:

```ini
User=YOUR_USER
WorkingDirectory=/path/to/sre-linux-troubleshooting-lab
ExecStart=/usr/bin/java -cp app SreLabApp
```

Substitua `YOUR_USER` pelo seu usuário Linux.

Exemplo:

```ini
User=meuusuario
```

Substitua:

```text
/path/to/sre-linux-troubleshooting-lab
```

pelo caminho absoluto do repositório.

Exemplo:

```ini
WorkingDirectory=/home/meuusuario/sre-linux-troubleshooting-lab
```

Se o comando:

```bash
which java
```

retornar um caminho diferente de `/usr/bin/java`, ajuste também o `ExecStart`.

## 7. Instalar o serviço

Depois de adaptar o template ao ambiente local, copie o arquivo para o diretório do `systemd`:

```bash
sudo cp app/sre-lab-app.service /etc/systemd/system/sre-lab-app.service
```

Recarregue as configurações do `systemd`:

```bash
sudo systemctl daemon-reload
```

## 8. Iniciar o serviço

Execute:

```bash
sudo systemctl start sre-lab-app.service
```

Verifique o estado:

```bash
systemctl status sre-lab-app.service --no-pager
```

O estado esperado é:

```text
Active: active (running)
```

## 9. Validar a aplicação gerenciada pelo systemd

Execute:

```bash
curl -i http://localhost:8080/health
```

Resultado esperado:

```text
HTTP/1.1 200 OK

{"status":"ok"}
```

Também é possível confirmar o listener:

```bash
sudo ss -ltnp | grep :8080
```

## 10. Consultar logs

Para visualizar os logs recentes do serviço:

```bash
journalctl -u sre-lab-app.service -n 20 --no-pager
```

Durante a inicialização normal, é esperado encontrar:

```text
SRE Lab App listening on port 8080
```

## 11. Parar o serviço

Execute:

```bash
sudo systemctl stop sre-lab-app.service
```

Depois confirme o estado:

```bash
systemctl status sre-lab-app.service --no-pager
```

E verifique se a porta foi liberada:

```bash
sudo ss -ltnp | grep :8080
```

## Fluxo de validação

O processo completo pode ser representado como:

```text
Código Java
   ↓
javac
   ↓
SreLabApp.class
   ↓
Java
   ↓
systemd
   ↓
porta 8080
   ↓
GET /health
   ↓
HTTP 200
```

## Observação

Os valores de usuário, caminho do projeto e caminho do Java podem variar entre ambientes.

Por esse motivo, o arquivo `app/sre-lab-app.service` versionado no repositório funciona como um template e deve ser adaptado antes da instalação no `systemd`.

