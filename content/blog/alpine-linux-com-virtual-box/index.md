---
title: Alpine Linux com VirtualBox
date: "2020-06-25T19:55:30.284Z"
description: Ambiente de Desenvolvimento com VirtualBox
---

Sou usuário Windows há cerca de 20 anos. Tentei me acostumar com Linux (Ubuntu, Kurumin, RedHat, Debian) para
desktop por um tempo, mas nunca achei agradável. Lembro que quando comecei a trabalhar com PHP no meu
primeiro emprego, ainda usava WampServer. Depois de uma certa experiência, aprendi que executar o código
utilizando um ambiente Linux, que por sua vez se iguala ao ambiente de produção, era mais conveniente para as
minhas tarefas diárias. Há vários anos utilizo VirtualBox como meu ambiente de desenvolvimento e aprimorei
bastante no decorrer do tempo. Também experimentei _Docker for Windows_, mas logo no primeiro dia percebi
que _Docker for Windows_ simplesmente instala uma VM com Ubuntu para rodar a _engine_ do Docker e ainda
por cima utiliza _Volume Share_ do VirtualBox para compartilhar arquivos entre Windows e Linux. O desempenho
do sistema de arquivos nessa modalidade deixa a desejar e atrasa o trabalho no dia a dia.

Gostei muito da experiência de utilizar [imagens Alpine no Docker](https://artigos.deleu.dev/alpine-e-docker-multi-build-para-laravel/)
e por isso tive a ideia de substituir minha máquina virtual (de centOS) e instalar o Alpine. A ideia é que a máquina
virtual contenha apenas Docker, Git e Samba instalado. Qualquer outro processo é executado dentro de um container
que posteriormente pode ser descartado. Dessa forma, consigo facilmente trabalhar em projetos utilizando PHP 5.6,
7.1, 7.4, NodeJS e entre outros. Não preciso instalar composer, npm ou yarn ou nenhuma outra dependência na minha
máquina, que se mantém simples e eficiente.

#### Alpine Linux

O Alpine Linux pode ser baixado em https://alpinelinux.org/downloads/ e possui uma imagem dedicada para virtualização
chamada de _Virtual_. Sua ISO possui aproximadamente 40MB e pode ser inserida em um disco Virtual no VirtualBox.
Antes de ligar a máquina, certifique-se de que há uma placa de rede do tipo _NAT_ e outra do tipo _Bridge_. A placa
_NAT_ nos permitirá acessar a internet facilmente fazendo traduções de endereços enquanto que a placa de rede _Bridge_
oferecerá uma rede local entre a máquina física e a virtual.

Ao ligar a máquina, o sistema operacional será inicializado através do CD. Antes de executar qualquer comando, gosto
de usar o `vi` que vem instalado na imagem para editar o arquivo `/etc/resolv.conf` e incluir um _Nameserver_ para
não enfrentar problemas de conectividade com a internet. O conteúdo do arquivo pode ser encontrado abaixo.

```text
nameserver 8.8.8.8
nameserver 8.8.4.4
```

O sistema DHCP do Alpine pode sobrescrever esse arquivo e causar perda de 
conectividade com a internet. A forma como resolvo esse problema é criando
um arquivo em `/etc/udhcpc/udhcpc.conf` com o conteúdo `RESOLV_CONF=no` da 
seguinte forma: 

```bash
mkdir /etc/udhcpc
echo 'RESOLV_CONF=no' > /etc/udhcpc/udhcpc.conf
```  

Depois de salvar e fechar o arquivo, podemos executar o script de instalação do sistema operacional chamado 
`setup-alpine`. O processo é interativo e pedirá configurações como layout do teclado, timezone, configurações
das placas de rede, senha de acesso _root_, disco de instalação e etc. O Layout do teclado é pouco relevante
para mim porque utilizo a máquina através de SSH e Samba. Também não utilizo senha no usuário _root_ pois trata-se
de uma máquina virtual que não está disponível na internet. A maioria das perguntas deixo a configuração _default_
selecionada, as opções que altero são: 

- Timezone: Europe/Amsterdam
- NTP Client: busybox
- disco de instalação: sda
- formato de arquivos: sys
- Formatar disco? y

Assim que a instalação finaliza, é solicitado a reinicialização da máquina. Nessa etapa é muito importante remover
o disco (ISO) da máquina virtual antes de ligá-la novamente, senão o sistema operacional será inicializado a partir
do CD novamente e estaremos de volta à estaca zero.

#### Ajustes iniciais

Depois de ligar a máquina pela primeira vez sem o disco do Alpine, certifique-se de verificar o
arquivo `/etc/resolv.conf` para não correr o risco de perder conectividade com a internet por falta de resolução
de domínio. Se o arquivo não estiver lá, talvez você esqueceu 
de remover o disco da máquina antes de ligá-la novamente. 
É possível verificar a conectividade com a internet usando 
`ping google.com.br`. Se obtiver resultado, então a conexão está estabelecida. 
Use `CTRL + C` para parar a execução do `ping`.

```text
localhost:~# ping google.com
PING google.com (172.217.17.110): 56 data bytes
64 bytes from 172.217.17.110: seq=0 ttl=117 time=16.227 ms
64 bytes from 172.217.17.110: seq=1 ttl=117 time=20.230 ms
``` 

A primeira configuração que faço numa máquina nova é editar o `/etc/ssh/sshd_conf` e modificar as diretrizes 
`PermitRootLogin` e `PermitEmptyPasswords`. Ambas devem ter o valor `yes`. Utilizo o `vi` para executar estes ajustes.

```text
...

PermitRootLogin yes

...

PermitEmptyPasswords yes

...
```

Certifique-se também de remover `#` se essas diretrizes estiverem desabilitadas. `#` significa código comentado.
Depois de alterado o arquivo, precisamos reiniciar o serviço. Podemos fazer
isso executando `service sshd restart`. Com a autorização do 
acesso `root` via ssh, já posso utilizar o [Git Bash](https://git-scm.com/download/win)
para acessar a máquina usando `ssh root@192.168.56.100`. Se quiser confirmar seu endereço ip, use `ifconfig`.
Como mencionei anteriormente, configurei 2 placas de rede (_NAT_ e _Bridge_). A _NAT_ geralmente tem um endereço
IP na classe C no formato 192.168.56.x. Já a placa _Bridge_ tem um endereço IP na classe A no formato `10.0.12.x`.
O IP da placa de rede _Bridge_ não é relevante. Ao obter acesso à máquina através da conexão SSH, já podemos
parar de ligar a máquina virtual com interface visual do VirtualBox e utilizar apenas o modo _Headless_ do VirtualBox.

Outra coisa que faço também é editar o arquivo `/etc/motd` e remover todo o seu conteúdo para que a mensagem inicial
de logon desapareça.

#### Instalações necessárias

Como mencionei, pretendo instalar algumas aplicações básicas nessa máquina virtual. 
Antes de iniciar as instalações, é necessário habilitar o _Community Repository_ que é
onde se encontra o Docker e o Docker Compose. Para isso, vamos editar o arquivo
`/etc/apk/repositories` e remover o `#` da linha `http://dl-cdn.alpinelinux.org/alpine/v3.12/community`
que demarca comentário. Agora podemos executar os seguintes
comandos:

```bash
apk update
apk add bash vim git samba docker docker-compose
rc-update add docker boot
rc-update add samba boot
service docker start
service samba start
```

Agora que instalamos o bash, podemos modificar novamente o arquivo
`/etc/ssh/sshd_conf` e adicionar o seguinte bloco:

```
Match User root
  ForceCommand /bin/bash
```

Depois de finalizar as instalações, fica faltando só configurar 
o Samba para compartilhar os arquivos entre o Alpine e o Windows.

#### Samba

Esse mecanismo de compartilhamento de arquivo é mais eficiente
que o VirtualBox Shared. A grande vantagem é que os arquivos
ficam em um sistema de arquivos Linux. Isso é uma vantagem
porque operações como `composer install` ou `npm install`
são executadas dentro da máquina virtual e costumam
manipular centenas de milhares de arquivos pequenos.
Tais operações requerem bastante IO e consomem muita banda
do disco. Se os arquivos ficarem em um sistema NTFS, o processo
se torna incrivelmente lento porque o Linux precisa aguardar
o reconhecimento de modificação dos arquivos efetivados pelo
Windows. O ponto negativo é que os arquivos serão acessados
pelo Windows através de um 'Disco de Rede' e IDEs como
o PHPStorm não funcionam de forma eficiente. Infelizmente é
um ponto negativo que tive que me acostumar, mas a funcionalidade
de _Reload file from disk_ no PHPStorm ajuda muito.

O arquivo de configuração do Samba está localizado em `/etc/samba/samba.conf`.
É possível identificar a primeira sessão do arquivo através do 
separador `[global]`. Ao final dele, configuraremos algumas
diretrizes. O resultado final está disponível a seguir.
Perceba que inclui algumas linhas comentadas no início.
Elas podem ajudar a identificar a parte do arquivo em que
colocaremos as configurações da sessão `[global]`, ou seja,
logo ao final da sessão.

```text
# These scripts are used on a domain controller or stand-alone 
# machine to add or delete corresponding unix accounts
;  add user script = /usr/sbin/useradd %u
;  add group script = /usr/sbin/groupadd %g
;  add machine script = /usr/sbin/adduser -n -g machines -c Machine -d /dev/null -s /bin/false %u
;  delete user script = /usr/sbin/userdel %u
;  delete user from group script = /usr/sbin/deluser %u %g
;  delete group script = /usr/sbin/groupdel %g

  security = user
  map to guest = Bad Password
  passdb backend = tdbsam
  obey pam restrictions = yes
  unix password sync = yes
  passwd program = /usr/bin/passwd %u
  passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:*        %n\n *password\supdated\ssuccessfully* .
  pam password change = yes
  usershare allow guests = yes
```

A segunda etapa é criar um compartilhamento acessível publicamente.
Nesse passo criaremos uma sessão nova dentro do arquivo, na parte final.

```text
[workspace]
   comment = Alpine Linux Workspace
   path = /home/workspace
   read only = no
   writable = yes
   guest ok = yes
   browseable = yes
   public = yes
   force user = root
   available = yes
```

Certifique-se de ajustar o atributo `path` para apontar para a pasta que
você deseja compartilhar entre o Alpine e o Windows. Certifique-se também
de criar a pasta. Para reiniciar o serviço Samba, podemos executar
`service samba restart`.

#### git-completion.bash

Para usar o `git-completion.bash`, é necessário instalar o `ncursus` para que
o `tput` funcione. 

```
apk add ncursus
```

#### Conclusão

Essa máquina virtual é bem minimalista, leve e eficiente. Possui poucos programas
instalados exatamente com o intuito de evitar conflito entre projetos ou
poluição desnecessária do sistema de arquivos. Podemos utilizar o `git` para
versionar o código fonte e containers Docker para interagirmos com os projetos.
Já fazem 3 anos que trabalho com uma configuração bem similar, mas somente
nos últimos 10 meses que migrei do CentOS para Alpine. Estou muito satisfeito
com essa mudança, principalmente pela simplicidade e facilidade de reconstruir
a máquina virtual do zero se necessário. No próximo artigo pretendo fazer um
complemento para explicar como configuro meu `git` com chave de acesso e
auto-complete.

Se gostou desse artigo me segue no [Twitter](https://twitter.com/deleugyn)!
Se não gostou desse artigo me segue no [Twitter](https://twitter.com/deleugyn)
e me diga o porquê. 

Até a próxima!
