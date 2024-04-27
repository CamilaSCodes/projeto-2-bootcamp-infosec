# Projeto de segurança de rede com Firewall, WAF e SIEM
## 2º Projeto Prático do bootcamp em Segurança da Informação do Programa Desenvolve Boticário

Este repositório contém o passo a passo referente ao desenvolvimento e implementação de uma solução abrangente de segurança de rede, que integre um firewall, um Web Application Firewall (WAF) e um Sistema de Informações e Eventos de Segurança (SIEM). O objetivo é aplicar na prática os conhecimentos adquiridos na plataforma Alura durante o Bootcamp em Segurança da Informação, configurando políticas de segurança, detectando e respondendo a ameaças, e criando painéis de monitoramento para garantir a integridade e a segurança da rede.	

![esquema de rede](https://github.com/CamilaSCodes/projeto-2-bootcamp-infosec/blob/main/imagens_projeto2/rede-segura.jpg?raw=true)

## Briefing do projeto
Neste projeto, a **intranet** é acessada pelos desenvolvedores e é hospedada em servidores específicos, como o servidor de dados (**Database**) e o **servidor web**. Um **Firewall** atua como intermediário do tráfego, implementando regras de segurança, com sistemas de detecção e prevenção de intrusões (**IDS** e **IPS**) integrados para identificar e impedir possíveis ataques. Adicionalmente, um **WAF** opera como um **proxy reverso**, recebendo as requisições e ocultando o IP privado para a rede pública.

Além disso, o sistema conta com o mecanismo **DMZ** (Demilitarized Zone), onde a rede externa é segregada da interna. A porção externa está voltada para a internet, onde o WAF é localizado, enquanto a porção interna engloba o Database (DMZ Banco) e o servidor web (DMZ Web). Em suma, o sistema é dividido em zonas distintas: a zona externa, onde está o WAF, e a zona interna, onde estão localizados o Database e o servidor web. Esta segmentação em sub-redes é essencial para mitigar os riscos de ataques, impedindo que um eventual ataque tenha acesso irrestrito às informações do sistema.

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
5.	Crie uma nova máquina virtual a partir do .iso do pfSense, seguindo as instruções do [site](https://docs.netgate.com/pfsense/en/latest/install/download-installer-image.html?_gl=1*1lw4mf8*_ga*MTQ2NjEyNTI0MS4xNzEzMzE4ODM5*_ga_TM99KBGXCB*MTcxNDIwODkyNS43LjEuMTcxNDIwODk2My4yMi4wLjA.) para instalação.

### Configure a rede

Acesse as configurações de rede do firewall no VirtualBox e ajuste os adaptadores da seguinte maneira: 

- **Adaptador 1**: Intel PRO/1000 MT Desktop (Placa em modo Bridge, Intel(R) Dual Band Wireless-AC 7265)
- **Adaptador 2**: Intel PRO/1000 MT Desktop (Placa de rede exclusiva de hospedeiro (host-only), 'VirtualBox Host-Only Ethernet A Adapter')
- **Adaptador 3**: Intel PRO/1000 MT Desktop (Rede interna, 'DMZ')
- **Adaptador 4**: Intel PRO/1000 MT Desktop (Rede interna, 'DMZEXT')

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

Para permitir a instalação de pacotes e configuração inicial das demais máquinas sem que o firewall bloqueie o tráfego, navegue no menu em **Firewall > Rules**. Em cada uma das interfaces, clique em :arrow_heading_up: **Add**, altere o campo <ins>Protocol</ins> de TCP para Any e clique em :floppy_disk: **Save** ao final da página. Confirme clicando em :white_check_mark: **Apply Changes**.
