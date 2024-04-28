# Projeto de segurança de rede com Firewall, WAF e SIEM
## 2º Projeto Prático do bootcamp em Segurança da Informação do Programa Desenvolve Boticário

Este repositório contém o passo a passo referente ao desenvolvimento e implementação de uma solução abrangente de segurança de rede, que integre um firewall, um Web Application Firewall (WAF) e um Sistema de Informações e Eventos de Segurança (SIEM). O objetivo é aplicar na prática os conhecimentos adquiridos na plataforma Alura durante o Bootcamp em Segurança da Informação, configurando políticas de segurança, detectando e respondendo a ameaças, e criando painéis de monitoramento para garantir a integridade e a segurança da rede.	

## Briefing do projeto
Neste projeto, a **intranet** é acessada pelos desenvolvedores e é hospedada em servidores específicos, como o servidor de dados (**Database**) e o **servidor web**. Um **Firewall** atua como intermediário do tráfego, implementando regras de segurança, com sistemas de detecção e prevenção de intrusões (**IDS** e **IPS**) integrados para identificar e impedir possíveis ataques. Adicionalmente, um **WAF** opera como um **proxy reverso**, recebendo as requisições e ocultando o IP privado para a rede pública.

Além disso, o sistema conta com o mecanismo **DMZ** (Demilitarized Zone), onde a rede externa é segregada da interna. A porção externa está voltada para a internet, onde o WAF é localizado, enquanto a porção interna engloba o Database (DMZ Banco) e o servidor web (DMZ Web). Em suma, o sistema é dividido em zonas distintas: a zona externa, onde está o WAF, e a zona interna, onde estão localizados o Database e o servidor web. Esta segmentação em sub-redes é essencial para mitigar os riscos de ataques, impedindo que um eventual ataque tenha acesso irrestrito às informações do sistema.

![esquema de rede](https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/rede-segura.jpg?raw=true)

### Requisitos
- Virtualização de quatro máquinas: WAF, Server, Firewall e SIEM
- Firewall com regras de bloqueio e acesso
- Sistema de detecção e prevenção de intrusões
- Envio de logs do WAF e Firewall para o SIEM
- Criptografia com certificado SSL
- Configuração de acesso remoto
- Filtros de log e dashboards no SIEM

## Passo a passo

### Configure o ambiente
1.	Baixe e instale o [VirtualBox](https://www.virtualbox.org/wiki/Downloads) para simular as máquinas virtuais necessárias.
2.	Faça o download do arquivo .iso do pfSense [aqui](https://www.pfsense.org/download/) (versão 2.7.2, arquitetura AMD64) e do Debian 12 [aqui](https://www.debian.org/download).
3.	Baixe o [pacote de extensões](https://download.virtualbox.org/virtualbox/7.0.16/Oracle_VM_VirtualBox_Extension_Pack-7.0.16.vbox-extpack) do VirtualBox e o adicione.
4.	No VirtualBox, use o atalho `Ctrl + H` e desative o Servidor DHCP para o adaptador VirtualBox Host-Only Ethernet Adapter. Aplique as mudanças.
5.	Crie uma nova máquina virtual chamada **firewall** a partir do .iso do pfSense, seguindo as instruções do [site](https://docs.netgate.com/pfsense/en/latest/install/download-installer-image.html?_gl=1*1lw4mf8*_ga*MTQ2NjEyNTI0MS4xNzEzMzE4ODM5*_ga_TM99KBGXCB*MTcxNDIwODkyNS43LjEuMTcxNDIwODk2My4yMi4wLjA.) para instalação.

### Configure a rede

Acesse as configurações de rede do firewall no VirtualBox e ajuste os adaptadores da seguinte maneira: 

- **Adaptador 1**:

    - Conectado a: `Placa em modo Bridge`

    - Nome: `Sua placa na máquina física que se encontra ligada à rede`

- **Adaptador 2**:
  
    - Conectado a: `Placa de rede exclusiva de hospedeiro (host-only)`
      
    - Nome: `VirtualBox Host-Only Ethernet A Adapter`
 
- **Adaptador 3**:
  
    - Conectado a: `Rede interna`
      
    - Nome: `DMZ`
      
- **Adaptador 4**:
    - Conectado a: `Rede interna`
      
    - Nome: `DMZEXT`

Execute a máquina do firewall, atribua as interfaces e seus respectivos IPs de acordo com os parâmetros: 

INTERNET (wan) &emsp; -> &ensp; em0 &emsp; -> &ensp; v4/DHCP4: atribuição dinâmica de IPV4

INTRANET (lan) &emsp;&nbsp; -> &ensp; em1 &emsp; -> &ensp; v4: 192.168.56.10/24

DMZ (opt1) &emsp;&emsp;&emsp; -> &ensp; em2 &emsp; -> &ensp; v4: 172.100.1.10/24

DMZEXT (opt2) &emsp;&nbsp; -> &ensp; em3 &emsp; -> &ensp; v4: 172.100.2.10/24

As atribuições de interface e de IP podem ser feitas com as opções 1 e 2 do menu do pfSense no terminal, respectivamente. Acesse a interface web do pfSense inserindo o IP da LAN no navegador e faça login com as credenciais padrão: username <ins>admin</ins> e password <ins>pfsense</ins>. É altamente recomendável alterar a senha padrão. Isso pode ser feito acessando **System > User Manager** no menu e clicando no ícone :pencil2:.

No menu, vá para **Interfaces > Interface Assignments** e certifique-se de que todas as interfaces estejam adicionadas e configuradas corretamente. Para este projeto, o único IP que será alterado conforme o usuário é o da interface INTERNET, pois a placa está no modo Bridge.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/interfaces.png?raw=true" alt="interfaces" />
</p>

### Configure o firewall

> [!NOTE]
> Em todas as regras criadas no pfSense, habilite a opção **Log packets that are handled by this rule**.

Seguindo uma boa prática na configuração de firewalls, recomenda-se criar uma regra inicial para bloquear todo o tráfego e, posteriormente, adicionar regras acima para permitir tráfegos específicos. No pfSense, novas regras são adicionadas em **Firewall > Rules**. Em cada uma das interfaces, clique em :arrow_heading_up: **Add**, altere o campo <ins>Action</ins> de `Pass` para `Block` e o campo <ins>Protocol</ins> de `TCP` para `Any`. 

Ao final da página, clique em :floppy_disk: **Save** e confirme clicando em :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/regra-BLOCK_GERAL.png?raw=true" alt="regra de block geral" />
</p>

Neste projeto, para simplificar a criação de regras, foram criados aliases para os IPs da interface DMZ e as portas WEB e SSH. 

Se desejar replicar isso, navegue até o menu **Firewall > Aliases > IP** e clique em :heavy_plus_sign: **Add**. Adicione um nome e uma descrição correspondente nos campos <ins>Name</ins> e <ins>Description</ins>, e não altere o campo <ins>Type</ins>. Adicione os seguintes IPs em **Hosts**: 

<table>
<thead>
  <tr>
    <th colspan="2">Hosts</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>172.100.1.101</td>
    <td>Graylog</td>
  </tr>
  <tr>
    <td>172.100.1.100</td>
    <td>WEB</td>
  </tr>
</tbody>
</table>

Clique em :floppy_disk: **Save** e confirme clicando em :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-aliases-IP.png" alt="aliases de IP" />
</p>

Repita o mesmo processo em **Firewall > Aliases > Ports**, adicionando as portas em **Ports**:

<table>
<thead>
  <tr>
    <th colspan="2">Ports</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>80</td>
    <td>HTTP</td>
  </tr>
  <tr>
    <td>443</td>
    <td>HTTPS</td>
  </tr>
  <tr>
    <td>22</td>
    <td>SSH</td>
  </tr>
</tbody>
</table>

Salve e confirme as mudanças. 

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-aliases-PORTS.png" alt="aliases de PORTS" />
</p>

Voltando para **Firewall > Rules**, crie as regras necessárias para todas as interfaces. 

### Crie regras para permitir o tráfego interno

#### Interface INTERNET

Para que o WAF seja publicado na internet quando configurado no futuro, adicione uma regra na interface INTERNET clicando em :arrow_heading_up: **Add**. 

Navegue até o campo **Destination**, altere <ins>Destination</ins> para `Address or Alias` e insira o IP `172.100.2.100` no placeholder <ins>Destination address</ins>. Logo abaixo, insira as portas WEB `80` e `443` em <ins>Destination Port Range</ins>. Neste projeto, elas estão com o alias `PORT_ADM`. 

Insira uma descrição, clique em :floppy_disk: **Save** e :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-rules-INTERNET.png" alt="regras da INTERNET" />
</p>

#### Interface INTRANET

Por padrão, o pfSense inclui uma regra de Anti-lockout na interface INTRANET. Neste projeto, por motivos de estudo, essa regra foi substituída por uma personalizada, o que não é obrigatório. Para reproduzir isso, simplesmente crie uma nova regra acima da regra de bloqueio. Navegue até **Source**, altere o campo <ins>Source</ins> para `INTRANET subnets`. Navegue até **Destination**, e altere <ins>Destination</ins> para `INTRANET address`, inserindo as portas WEB ou seu alias em <ins>Destination Port Range</ins>. Insira uma descrição e clique em :floppy_disk: **Save**.

Para que o firewall não bloqueie o acesso ao WAF, adicione uma nova regra com :arrow_heading_up: **Add**. Navegue até **Source** e altere <ins>Source</ins> para `INTRANET subnets`. Navegue até o campo **Destination** abaixo, altere <ins>Destination</ins> para `Address or Alias` e insira o IP `172.100.1.101` no placeholder <ins>Destination address</ins>. Logo abaixo, insira a porta `9000` em <ins>Destination Port Range</ins>.

Insira uma descrição, clique em :floppy_disk: **Save** e :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-rules-INTRANET.png" alt="regras da INTRANET" />
</p>

#### Interface DMZ

Para permitir atualizações de pacotes e configurações no Graylog e no Server, adicione uma regra com :arrow_heading_up: **Add**. 

Navegue até **Source**, altere o campo <ins>Source</ins> para `Address or Alias` e insira o IP do Graylog ou do Server no placeholder <ins>Source address</ins>. Caso não tenha sido um criado um alias para os IPs, duas regras terão de ser criadas. Caso contrário, insira o nome do alias. Nesse projeto, o alias é `UPDATE_DMZ`. Navegue até o **Destination** abaixo e insira as portas WEB ou seu alias em <ins>Destination Port Range</ins>. Insira uma descrição e clique em :floppy_disk: **Save**.

Adicione uma nova regra para a consulta DNS clicando em :arrow_heading_up: **Add**. 

Altere o campo <ins>Protocol</ins> de `TCP` para `UDP`. Navegue até **Source**, altere o campo <ins>Source</ins> para `DMZ subnets`. Navegue até **Destination** abaixo e em <ins>From</ins> escolha a opção `DNS (53)`. Insira uma descrição e clique em clique em :floppy_disk: **Save**.

Confirme as mudanças com :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-rules-DMZ.png" alt="regras da DMZ" />
</p>

#### Interface DMZEXT

Repita o processo anterior para permitir atualizações de pacotes e configurações no WAF, adicionando duas regras com :arrow_heading_up: **Add**. 

Navegue até **Source**, altere o campo <ins>Source</ins> para `Address or Alias` e insira o IP `172.100.2.100` em <ins>Source address</ins>. Navegue até o **Destination** abaixo e insira as portas WEB ou seu alias em <ins>Destination Port Range</ins>. Insira uma descrição e clique em :floppy_disk: **Save**.

Crie uma regra para consultas DNS acima e altere o campo <ins>Protocol</ins> de `TCP` para `UDP`. Navegue até **Source**, altere o campo <ins>Source</ins> para `DMZEXT subnets`. Navegue até **Destination** abaixo e em <ins>From</ins> escolha a opção `DNS (53)`. Insira uma descrição e clique em :floppy_disk: **Save**.

Como o Graylog será usado como Syslog e receberá os logs do WAF, é preciso liberar o tráfego nessa direção adicionando uma nova regra acima. Altere o campo <ins>Protocol</ins> de `TCP` para `UDP`, depois, navegue até **Source** e altere o campo <ins>Source</ins> para `Address or Alias`. Insira o IP `172.100.2.100` em <ins>Source address</ins>. Navegue até o campo **Destination** abaixo, altere <ins>Destination</ins> para `Address or Alias` e insira o IP `172.100.1.101` no placeholder <ins>Destination address</ins>. Logo abaixo, insira a porta `1514` em <ins>Destination Port Range</ins>. Insira uma descrição e clique em :floppy_disk: **Save**.

Por fim, para permitir que o WAF envie as requisições ao server, adicione uma nova regra acima. Navegue até **Source** e altere o campo <ins>Source</ins> para `Address or Alias`. Insira o IP `172.100.2.100` em <ins>Source address</ins>. Navegue até o campo **Destination** abaixo, altere <ins>Destination</ins> para `Address or Alias` e insira o IP `172.100.1.100` no placeholder <ins>Destination address</ins>. Logo abaixo, em <ins>From</ins> escolha a opção `HTTP (80)`. Insira uma descrição e clique eem :floppy_disk: **Save**.

Confirme as mudanças com :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-rules-DMZEXT.png" alt="regras da DMZEXT" />
</p>

### Traduza os IPs

Para garantir o reconhecimento dos IPs do Graylog e do WAF na rede, é necessário realizar a tradução dos endereços de IP através do processo de NAT (Network Address Translation). Para isso, siga o caminho **Firewall > NAT > 1:1** e adicione dois novos mapeamentos clicando em :arrow_heading_up: **Add**. 

Começando com o Graylog, altere o campo <ins>Interface</ins> de `INTERNET` para `INTRANET`. Navegue até <ins>External subnet IP</ins> e altere o campo <ins>Address</ins> para `192.168.56.11`. Logo abaixo, em <ins>Internal IP</ins>, altere o campo <ins>Address</ins> para `172.100.1.101`. Adicione uma descrição e clique em :floppy_disk: **Save**.

Para configurar o WAF, é importante escolher um IP que esteja na mesma rede da INTERNET e que não esteja em uso. Para verificar isso, execute o seguinte comando no seu terminal CMD utilizando o IP escolhido. No contexto deste projeto, o IP selecionado é `192.168.0.110`.

```
ping 192.168.0.110
```

Caso não obtenha uma resposta, o IP está disponível para uso. Repita o mesmo processo de adicionar um mapeamento, mantendo o campo <ins>Interface</ins> como `INTERNET`. Navegue até <ins>External subnet IP</ins> e altere o campo <ins>Address</ins> para o IP escolhido. Logo abaixo, em <ins>Internal IP</ins>, altere o campo <ins>Address</ins> para `172.100.2.100`. Adicione uma descrição e clique em :floppy_disk: **Save**.

Confirme as mudanças com :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-NAT.png" alt="mapeamento com NAT" />
</p>

### Crie IPs virtuais

O último passo para acessar as interfaces do Graylog e WAF é adicionar IPs virtuais correspondentes aos IPs traduzidos para ambos. Para isso, siga o caminho **Firewall > Virtual IPs** e adicione dois novos IPs clicando em :heavy_plus_sign: **Add**.

Começando com o Graylog, marque a opção `IP Alias` em <ins>Type</ins> e altere a <ins>Interface</ins> para `INTRANET`. No campo <ins>Address(es)</ins> insira o IP `192.168.56.11/24`. Ao final da página, adicione uma descrição e clique :floppy_disk: **Save**.

Para o WAF, repita o mesmo processo, mantendo a <ins>Interface</ins> como `INTERNET`. No campo <ins>Address(es)</ins> insira o IP escolhido na etapa anterior, no caso do projeto atual, `192.168.0.110/24`. Ao final da página, adicione uma descrição e clique :floppy_disk: **Save**.

Confirme as mudanças com :white_check_mark: **Apply Changes**.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/firewall-virtual-IPs.png" alt="virtual IPs" />
</p>

### Configure os sistemas de detecção e prevenção de intrusões

No menu do pfSense, navegue pelo caminho **System > Package Manager > Available Packages**. Procure pelo pacote `Snort` e instale sua versão mais atualizada.

Para configurar e gerenciar o pacote Snort instalado, navegue em **Services > Snort > Interfaces** e clique em :heavy_plus_sign: **Add**. Escolha a opção `INTERNET (em0)` para <ins>Interface</ins> e adicione uma descrição. Na seção **Alert Settings**, habilite as opções `Send Alerts to System Log` e `Enable Packet Captures` para enviar os logs ao Graylog e ativar a captura de tráfego, respectivamente. Clique em :floppy_disk: **Save** ao final da página. 

Ative o Snort nesta interface clicando no ícone :arrow_forward: abaixo de **Snort Status**. Adicione as configurações globais desejadas em **Services > Snort > Global Settings**.

### Configure o Server

Retorne ao VirtualBox e crie uma nova máquina virtual denominada **server** utilizando o arquivo .iso do Debian que foi previamente baixado.

Após concluir o processo de criação, acesse as configurações de rede do server e ajuste o adaptador da seguinte forma:

- **Adaptador 1**:

    - Conectado a: `Rede interna`
      
    - Nome: `DMZ`

Inicie a máquina e prossiga com o processo de instalação do Debian, desconsiderando as configurações de rede. Finalizada a instalação, execute o seguinte comando para mudar o nome do host: 

```
nano /etc/hostname
```

Substitua o conteúdo do arquivo pelo seguinte:

```
server
```

Salve com `Ctrl + O`. Para configurar o servidor DNS, execute o comando: 

```
nano /etc/resolv.conf
```

Substitua o conteúdo do arquivo pelo seguinte:

```
nameserver 1.1.1.1
nameserver 1.0.0.1
```

Salve com `Ctrl + O`. Para configurar os hosts, execute o comando :

```
nano /etc/hosts
```

Substitua o conteúdo do arquivo pelo seguinte:

````
127.0.0.1       localhost
172.0.1.1       server.lab      server

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
````

Salve com `Ctrl + O`. Por fim, para configurar os IPs, execute o comando: 

```
nano /etc/network/interfaces
```

Substitua o conteúdo do arquivo pelo seguinte:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet static
        address 172.100.1.100/24
        gateway 172.100.1.10
```

Salve com `Ctrl + O`. Execute os comandos para atualizar as configurações:

```
update-grub
reboot
```

Para instalar e ativar o servidor NGINX, execute os comandos: 

```
apt update
apt install nginx
systemctl enable nginx
systemctl restart nginx
```

### Configure o SIEM

Retorne ao VirtualBox e crie uma nova máquina virtual denominada **Graylog** utilizando o arquivo .iso do Debian que foi previamente baixado.

Após concluir o processo de criação, acesse as configurações de rede do Graylog e ajuste o adaptador da seguinte forma:

- **Adaptador 1**:

    - Conectado a: `Rede interna`
      
    - Nome: `DMZ`

Inicie a máquina e prossiga com o processo de instalação do Debian, desconsiderando as configurações de rede. Finalizada a instalação, execute o seguinte comando para mudar o nome do host: 

```
nano /etc/hostname
```

Substitua o conteúdo do arquivo pelo seguinte:

```
graylog
```

Salve com `Ctrl + O`. Para configurar o servidor DNS, execute o comando: 

```
nano /etc/resolv.conf
```

Substitua o conteúdo do arquivo pelo seguinte:

```
nameserver 1.1.1.1
nameserver 1.0.0.1
```

Salve com `Ctrl + O`. Para configurar os hosts, execute o comando :

```
nano /etc/hosts
```

Substitua o conteúdo do arquivo pelo seguinte:

````
127.0.0.1       localhost
172.0.1.1       graylog.lab     graylog

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
````

Salve com `Ctrl + O`. Por fim, para configurar os IPs, execute o comando: 

```
nano /etc/network/interfaces
```

Substitua o conteúdo do arquivo pelo seguinte:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet static
        address 172.100.1.101/24
        gateway 172.100.1.10
```

Salve com `Ctrl + O`. Execute os comandos para atualizar as configurações:

```
update-grub
reboot
```

Instale o e configure o Graylog seguindo as [instruções](https://go2docs.graylog.org/4-3/downloading_and_installing_graylog/debian_installation.html?tocpath=Downloading%20and%20Installing%20Graylog%7CInstalling%20Graylog%7C_____6).

> [!WARNING]
> Devido a questões de compatibilidade, neste projeto a versão do Debian foi alterada de 12 para 11. O MongoDB foi instalado na versão 4.2.21 e o Graylog na versão 4.3.

Acesse a interface do Graylog em `http://192.168.56.11:9000` e faça o login com a senha escolhida. 

Para configurar o Graylog como um Syslog, siga as recomendações da [documentação](https://graylog.org/post/how-to-use-graylog-as-a-syslog-server/). Atente-se para a definição do campo <ins>Port</ins> como `1514`. 

Voltando a interface do pfSense, para habilitar o envio de logs ao Graylog, siga o caminho no menu **Status > System Logs > Settings**. 

Navegue até o final da página, no campo **Remote Logging Options**. Logo abaixo, habilite a opção `Send log messages to remote syslog server`. Defina o <ins>Source address</ins> como `DMZ` e adicione o IP `172.100.1.101:1514` em <ins>Remote log servers</ins>. Marque apenas a opção `Firewall Events` abaixo e clique em :floppy_disk: **Save**.

Ao retornar à página inicial do Graylog e clicar em atualizar, os logs do pfSense devem estar disponíveis.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/graylog-pfsense-logs.png" alt="logs do pfSense no Graylog" />
</p>

### Configure o WAF

Retorne ao VirtualBox e crie uma nova máquina virtual denominada **WAF** utilizando o arquivo .iso do Debian que foi previamente baixado.

Após concluir o processo de criação, acesse as configurações de rede do WAF e ajuste o adaptador da seguinte forma:

- **Adaptador 1**:

    - Conectado a: `Rede interna`
      
    - Nome: `DMZEXT`

Inicie a máquina e prossiga com o processo de instalação do Debian, desconsiderando as configurações de rede. Finalizada a instalação, execute o seguinte comando para mudar o nome do host: 

```
nano /etc/hostname
```

Substitua o conteúdo do arquivo pelo seguinte:

```
server
```

Salve com `Ctrl + O`. Para configurar o servidor DNS, execute o comando: 

```
nano /etc/resolv.conf
```

Substitua o conteúdo do arquivo pelo seguinte:

```
nameserver 1.0.0.1
nameserver 1.1.1.1
```

Salve com `Ctrl + O`. Para configurar os hosts, execute o comando :

```
nano /etc/hosts
```

Substitua o conteúdo do arquivo pelo seguinte:

````
127.0.0.1       localhost
172.100.2.100   waf.lab waf

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
````

Salve com `Ctrl + O`. Por fim, para configurar os IPs, execute o comando: 

```
nano /etc/network/interfaces
```

Substitua o conteúdo do arquivo pelo seguinte:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet static
        address 172.100.2.100/24
        gateway 172.100.2.10
```

Salve com `Ctrl + O`. Execute os comandos para atualizar as configurações:

```
update-grub
reboot
```

Instale o e configure o ModSecurity + NGINX seguindo as [instruções](https://kifarunix.com/install-modsecurity-3-with-nginx-on-debian-12/?expand_article=1).

Para enviar os logs do WAF ao Graylog, execute o comando: 

```
nano /usr/local/nginx/conf/nginx.conf
```

No arquivo, abaixo de `error_log /var/log/nginx/error.log;` adicione o seguinte texto: 

```
access_log syslog:server=172.100.1.101:1514;  
error_log syslog:server=172.100.1.101:1514;    
```

Salve com `Ctrl + O`. Confira se a configuração foi adicionada corretamente executando o comando: 

```
 nginx -t
```

Por fim, recarregue o NGINX com as novas configurações executando o comando: 

```
nginx -s reload
```

Para verificar se o WAF está operacional e se os logs estão sendo enviados corretamente para o Graylog, acesse a interface do WAF utilizando o IP traduzido [nesta](https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec?tab=readme-ov-file#traduza-os-ips) etapa. Teste com algum ataque, como exemplo: 

```
http://192.168.0.110/?exec=/bin/bash
```

Caso tudo esteja funcionado de maneira correta, os logs do WAF devem aparecer na página inicial do Graylog. 

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/graylog-WAF.png" alt="logs do WAF no Graylog" />
</p>

### Adicione um certificado SSL e configure o WAF como um proxy reverso

Crie um certificado SSL auto assinado seguindo as [instruções](https://linuxize.com/post/creating-a-self-signed-ssl-certificate/).

Configure o proxy reverso executando os comandos: 

```
mkdir /usr/local/nginx/sites-available
cd /usr/local/nginx/sites-available
nano site.lab_conf
```

Altere o conteúdo do arquivo para o seguinte: 

```
server {
        listen 443 ssl;

        ssl_certificate /usr/local/nginx/certssl/example.crt;
        ssl_certificate_key /usr/local/nginx/certssl/example.key;

        server_name site.lab;

        location / {
                proxy_pass http://172.100.1.100/;
                proxy_set_header Host $http_host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }

}
```

Salve com `Ctrl+O`. Feito isso, execute os comandos:

```
cd
mkdir /usr/local/nginx/sites-enabled/
ln -s /usr/local/nginx/sites-available/site.lab_conf /usr/local/nginx/sites-enabled/site.lab_conf
```

Execute o comando para editar o arquivo de configurações do NGINX:

```
nano /usr/local/nginx/conf/nginx.conf
```

E adicione o seguinte texto dentro do bloco `http`:

```
include /usr/local/nginx/sites-enabled/*;
```

Salve com `Ctrl+O` e confira se as configurações estão corretas executando: 

```
 nginx -t
```

Por fim, recarregue o NGINX com as novas configurações.

```
nginx -s reload
```

### Facilite a leitura dos logs e crie dashboards no Graylog

Volte ao Graylog e busque pela palavra **Modsecurity**. Clique em algum log e, dentro dele, clique em uma **message** e em `Create extractor`. Selecione a opção `Regular Expression` e clique em `Submit`. 

No campo <ins>Regular Expression</ins>, para cada extractor, insira [expressões regulares](http://turing.com.br/material/regex/introducao.html) para extrair informações importantes do log com facilidade, como exemplo:


- **IP do client**
```
^.*\[client\ (.*)\] ModSecurity.*$
```

- **ID da regra**
```
^.*\[client\ (.*)\] ModSecurity.*$
```

- **Código da ameaça**
```
^.*code (\d+).*$
```

- **Severidade da ameaça**
```
^.*\[severity\ \"(\d*)\"\]\ \[ver.*$
```

Altere a <ins>Condition</ins> para `Only attempt extraction if field contains string` e, no campo <ins>Field contains string</ins> abaixo, insira `ModSecurity:`. Adicione um nome para o campo novo, um título para o extrator e ao final clique em `Create extractor`. 

Para os [logs do pfSense](https://docs.netgate.com/pfsense/en/latest/monitoring/logs/raw-filter-format.html), retorne para a página inicial do Graylog e procure pela palavra **filterlog**. Clique em algum log e, dentro dele, clique em uma **message** e em `Create extractor`. Selecione a opção `GROK pattern` e clique em `Submit`. 

No campo <ins>Regular Expression</ins>, adicione padrões sucessivos para extrair informações importantes do log com facilidade. Como exemplo, o padrão abaixo remove informações desnecessárias e mostra os seguintes dados do log: número da regra, interface, motivo do log, ação do firewall, direção do tráfego, versão do IP, protocolo, IPs de origem e destino, portas de origem e destino.

```
%{BASE10NUM:rule},%{DATA:UNWANTED},%{DATA:UNWANTED},%{DATA:UNWANTED},%{WORD:iface},%{WORD:motive},%{WORD:action},%{WORD:direction},%{BASE10NUM:ip_version},%{DATA:UNWANTED},%{DATA:UNWANTED},%{DATA:UNWANTED},%{DATA:UNWANTED},%{DATA:UNWANTED},%{DATA:UNWANTED},%{DATA:UNWANTED},%{WORD:protocol},%{DATA:UNWANTED},%{IPV4:source_ip},%{IPV4:destination_ip},%{BASE10NUM:source_port},%{BASE10NUM:destination_port}
```

Altere a <ins>Condition</ins> para `Only attempt extraction if field contains string` e, no campo <ins>Field contains string</ins> abaixo, insira `filterlog`. Adicione um nome para o campo novo, um título para o extrator e ao final clique em `Create extractor`. 

Utilizando o Graylog, os extractors criados podem ser utilizados para gerar Dashboards com informações valiosas. 

Para isso, no menu, clique em **Dashboards** e depois em **Create new dashboard**. Localize o ícone de :heavy_plus_sign: o lado esquerdo do painel e clique nele. Ao abrir as opções, selecione `Aggregation`. Edite a agregação conforme necessário para obter as informações desejadas de forma ágil. Por exemplo, a agregação abaixo demonstra a relação entre bloqueios e permissões do pfSense em forma de porcentagem.

<p align="center">
  <img src="https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/graylog-dashboard.png" alt="dashboard no graylog usando extractor" />
</p>

### Configure o acesso remoto

Finalizando as configurações do sistema, habilite o acesso remoto com o SSH. 

Na interface do pfSense, navegue pelo menu **System > Advanced**. Navegue abaixo até **Secure Shell**, e em <ins>Secure Shell Server</ins> habilite a opção `Enable Secure Shell`. Clique em :floppy_disk: **Save** ao final da página. 

Em cada uma das máquinas, execute o comando abaixo e garanta que o SSH está instalado. 

```
apt update
apt install openssh-server
```
## Conclusão

Em suma, a implementação meticulosa de medidas de segurança, como segmentação em zonas e a integração de múltiplas camadas de defesa, incluindo Firewall, IDS/IPS e WAF, demonstrou ser eficaz na proteção contra possíveis ameaças e ataques cibernéticos. A simulação realizada no Virtualbox, com quatro máquinas distintas - WAF, Server, Firewall e SIEM - ilustrou a aplicação prática dessas medidas. Todos os [requisitos](https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/tree/main#requisitos) implementados formam uma defesa sólida e abrangente, essencial para enfrentar os desafios constantes do cenário cibernético atual.
