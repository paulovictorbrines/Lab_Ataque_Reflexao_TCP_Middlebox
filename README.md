# Ataque de Reflexão Amplificada Explorando Middlebox – Código e Ambiente de Testes

Este README descreve a configuração do ambiente experimental utilizado para a análise dos ataques de reflexão amplificada sobre TCP explorando middleboxes, abrangendo tanto a estrutura do código-fonte quanto os elementos virtuais e de rede do laboratório. Inicialmente, apresenta-se a arquitetura e o funcionamento do código de ataque, incluindo orientações de execução e informações complementares obtidas por meio de comunicação direta com seu autor. Em seguida, descreve-se o ambiente de testes construído no VMware Workstation Pro, composto por máquinas virtuais representando o atacante, o alvo e os dispositivos middlebox (firewalls pfSense e FortiGate), com suas respectivas configurações.

Essa estrutura possibilitou a simulação controlada do ataque e a análise detalhada de seu comportamento frente a diferentes dispositivos intermediários, permitindo compreender de forma prática os mecanismos de reflexão amplificada e suas implicações de segurança.

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

## Descrição do Código do Ataque

Esta seção documenta o uso e a configuração do código disponível em:
https://github.com/moloch54/Ddos-TCP-Middlebox-Reflection-Attack


<img width="850" alt="tipos_ataque_page" src="https://github.com/user-attachments/assets/895bf494-9bcb-4f47-acd6-c1d6d3c32e39" />
<br>
<br>


O código foi muito útil para a condução dos experimentos em laboratório, permitindo a implementação prática dos conceitos do artigo científico "Weaponizing Middleboxes for TCP Reflected Amplification" de Bock et al. Com ele, foi possível replicar a técnica de Middlebox Reflection (b), facilitando a compreensão e validação dos mecanismos de reflexão amplificada usando middleboxes sobre o protocolo TCP. Nessa técnica, o endereço IP de origem é falsificado como sendo o da vítima, fazendo com que as respostas das middleboxes atinjam diretamente o alvo.

Para entender o ataque, é preciso revisar o processo de estabelecimento de conexão TCP, o Three-Way Handshake, que tem três etapas:

1.  O cliente (SRC) envia um pacote SYN para iniciar a conexão;
2.  O servidor (DEST) responde com um pacote SYN, ACK, reconhecendo a solicitação;
3.  O cliente envia um pacote ACK, finalizando a conexão.

O ataque explora o fato de que, em redes com dispositivos intermediários (como firewalls ou proxies), os pacotes podem seguir caminhos diferentes. O pacote SYN é falsificado (com `SRC=Vítima` e `DST=Servidor`), mas a resposta SYN, ACK do servidor real pode seguir por outra rota e não ser vista pela middlebox. Isso cria uma inconsistência de estado.

O atacante, após o SYN falsificado, envia um segundo pacote ACK (também com IP de origem falsificado). A middlebox, por não ter visto o SYN, ACK, interpreta esse ACK como inválido e pode gerar respostas automáticas, como pacotes RST ou múltiplas mensagens de bloqueio/advertência, amplificando o tráfego contra a vítima.

O ataque pode ser refinado com pacotes contendo dados. Após o SYN, o atacante pode enviar um pacote ACK+PSH com uma carga útil, como uma requisição HTTP. Por exemplo:

  * Envio de um pacote SYN com endereço IP de origem falsificado (da vítima) e destino a domínios com alto potencial de bloqueio por middleboxes: `SRC=Vítima`, `DST=youporn.com (66.254.114.79), facebook.com (157.240.13.35), pornhub.com (66.254.114.41), bittorrent.com (98.143.146.7)`;
  * Envio subsequente de um pacote ACK+PSH com carga útil: requisição HTTP GET, também com `SRC=Vítima`.

Esses pacotes, ao serem processados por middleboxes de filtragem de conteúdo, podem disparar múltiplas respostas, reforçando o efeito de amplificação.

A escolha por esse código público se deu por sua estrutura clara e alinhamento com a metodologia de Bock et al., o que garantiu a conformidade da implementação e permitiu focar nos aspectos experimentais e analíticos, evitando o desenvolvimento do zero.

-----

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

## Execução do Código

Para a execução do código de ataque de reflexão amplificada sobre TCP, é preciso seguir os passos descritos no arquivo README.md do repositório. Primeiro, as dependências devem ser instaladas e configuradas. As ferramentas necessárias são:

  * tcpreplay: para reproduzir pacotes em uma interface de rede.
  * mergecap: para mesclar pacotes de múltiplas threads em um único arquivo .pcap.
  * scapy: biblioteca Python para construção e manipulação de pacotes TCP.

Para instalar as dependências em sistemas Debian/Ubuntu, use o seguinte comando:

```bash
sudo apt-get install tcpreplay mergecap python3-scapy
```

Após a instalação, o código pode ser executado diretamente pelo terminal. O comando para iniciar o ataque é:

```bash
sudo python3 mra.py <tempo_em_segundos> <IP_alvo>
```

O `<tempo_em_segundos>` define a duração do ataque, e o `<IP_alvo>` é o endereço IP que se deseja sobrecarregar.

O script gera pacotes TCP falsificados, enviando um pacote SYN para um site de destino (filtrado por uma middlebox) e, em seguida, um pacote ACK + PSH com um *payload* HTTP. Isso faz com que a middlebox responda com um RST ou uma página de erro. O tráfego gerado é salvo em arquivos .pcap e reproduzido indefinidamente usando o `tcpreplay`, enviando os pacotes para a interface de rede especificada.

Um exemplo de execução seria:

```bash
sudo python3 mra.py 300 123.4.5.6
```


<img width="550" alt="execucao_mra" src="https://github.com/user-attachments/assets/930e1f6b-b577-4971-82d8-1ff4d028afc6" />
<br>
<br>


Isso executará o ataque por 300 segundos contra o IP 123.4.5.6. É importante ressaltar que o uso deste código para ataques não autorizados é ilegal e antiético. A intenção é apenas educacional e para testes em ambientes controlados.

-----

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

## Virtualizador VMware Workstation Pro

Para a virtualização das máquinas virtuais (alvo, atacante e firewalls pfSense e FortiGate), foi usado o VMware Workstation Pro 17 for Personal Use (versão 17.6.3-24583834), disponível gratuitamente. Ele pode ser baixado no site oficial da VMware, mas é necessário se registrar no portal da Broadcom.

O VMware Workstation é um hipervisor de desktop para Windows e Linux, que permite criar e executar VMs com diversos sistemas operacionais sem precisar reiniciar o computador. É uma plataforma robusta e versátil, ideal para desenvolvimento, testes e simulações de software em ambientes virtuais isolados.

-----

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

## Configuração do Laboratório

O ambiente do laboratório foi dividido em três cenários, todos com a mesma topologia de rede e endereços IP da imagem abaixo. Cada cenário tinha três elementos: um alvo, um atacante e uma middlebox (representada por um firewall).


<img width="815" alt="topologia_lab" src="https://github.com/user-attachments/assets/3f935515-db36-42cb-bffc-7888d9db5589" />
<br>
<br>


  * Cenário 1: Firewall com pfSense e o complemento pfBlockerNG.
  * Cenário 2: Firewall com pfSense e os softwares Squid e SquidGuard.
  * Cenário 3: Firewall FortiGate.

Os firewalls foram configurados para simular comportamentos típicos e também configurações incorretas, a fim de mostrar como certas escolhas podem tornar esses dispositivos vulneráveis a ataques de reflexão amplificada sobre TCP.

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

### Máquina Alvo

A máquina alvo foi uma instância do sistema operacional Ubuntu Linux 22.04.5 LTS (Jammy), com as seguintes especificações:

  * Memória RAM: 4096 MB
  * Processador: 2 vCPUs
  * Armazenamento: 25 GB
  * Adaptadores de rede: 1000 Mb/s (Intel Gigabit Internet 82545EM)

A VM possui duas interfaces de rede. A primeira está conectada ao segmento LAN (`ens34`), simulando uma rede local protegida atrás do firewall. A segunda (`ens37`) está em modo NAT para acesso à internet.

A configuração de rede é:

  * Hostname: `Ubuntu`
  * Endereço IP LAN: `192.168.24.61/24`
  * Endereço IP INTERNET: IP via DHCP
  * Gateway padrão: `192.168.24.100`
  * Servidor DNS: `192.168.24.100`

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

### Máquina Atacante

A máquina atacante foi baseada no sistema operacional Kali Linux, versão 2025.1. Suas especificações são:

  * Memória RAM: 8096 MB
  * Processador: 6 vCPUs
  * Armazenamento: 32 GB
  * Adaptadores de rede: 1000 Mb/s (Intel Gigabit Internet 82545EM)

A VM também tem duas interfaces de rede. A primeira está conectada ao segmento WAN (`eth0`), simulando um agente externo na internet. A segunda (`eth1`) está em modo NAT para acesso à internet.

A configuração de rede é:

  * Hostname: `kali`
  * Endereço IP WAN: `10.0.0.2/24`
  * Endereço IP INTERNET: IP via DHCP
  * Gateway padrão: `10.0.0.1`

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

### Firewall pfSense

O primeiro firewall utilizado nos experimentos foi o pfSense Community Edition (CE), na versão 2.7.2-RELEASE. Trata-se de uma distribuição de software de firewall open source, baseada no FreeBSD, que pode ser instalada em um computador físico ou em uma máquina virtual para compor um firewall dedicado em uma rede.

A seguir, são apresentadas as especificações de hardware virtual atribuídas à máquina:

- Memória RAM: 2048 MB  
- Processador: 2 vCPUs  
- Armazenamento: 20 GB  
- Adaptadores de rede: 1000 Mb/s (Intel Gigabit Internet 82545EM)  

A VM do pfSense foi configurada com três interfaces de rede:

- em0: Conectada ao segmento LAN, identificado no VMware como WAN, simulando a saída das máquinas internas em direção à internet;  
- em1: Conectada a outro segmento LAN, identificado como LAN, representando a interface de acesso interno ao firewall;  
- em3: Configurada em modo NAT e utilizada exclusivamente para permitir o acesso à internet via conexão do computador hospedeiro, possibilitando o download de atualizações, pacotes e demais comunicações externas necessárias para a preparação do ambiente de ataque.  

A configuração de rede da máquina virtual do firewall foi definida da seguinte forma:

- Hostname: `pfSense.home.arpa`  
- Endereço IP WAN (em0): `10.0.0.1/24`  
- Endereço IP LAN (em1): `192.168.24.100/24`  
- Endereço IP INTERNET (em2): atribuído via DHCP  

Foi configurado o NAT Outbound para redirecionar o tráfego da rede LAN para a interface WAN, simulando a saída da rede interna para a externa por meio do firewall, conforme imagem a seguir.  


<img width="815" height="184" alt="rules_nat" src="https://github.com/user-attachments/assets/49ffa326-0630-4991-97a9-315a1b267930" />
<br>
<br>


As regras de firewall configuradas para a interface WAN incluem:  

- Uma regra que permite o recebimento de pacotes ICMP (ping) originados de máquinas externas para máquinas internas da LAN;  
- Uma regra que permite `ping` de máquinas externas diretamente ao endereço IP da interface WAN do pfSense;  
- Uma regra que permite o acesso externo ao servidor Web do alvo pela porta 80/TCP, simulando sua exposição pública;  
- Uma regra que permite que máquinas da rede interna acessem livremente todos os sites, sendo posteriormente restringidas pelas soluções de filtragem do pfBlockerNG e squidGuard, responsáveis por bloquear domínios como `youporn.com`, `facebook.com`, `pornhub.com` e `bittorrent.com`, processadas antes dessa regra.  

<img width="815" height="333" alt="rules_wan" src="https://github.com/user-attachments/assets/6903a8bd-f9bd-4b4f-9c5c-5aeb7c7e8e5d" />
<br>
<br>


Para a interface LAN, as regras estão configuradas conforme a seguir, contendo por padrão:  

- A `anti-lockout rule`, que evita o bloqueio do acesso à interface web do pfSense;  
- A regra `default allow LAN to any`, permitindo tráfego de saída irrestrito da LAN;  
- A regra `default allow LAN IPv6 to any`, com comportamento equivalente para tráfego IPv6.  

<img width="815" height="242" alt="rules_lan" src="https://github.com/user-attachments/assets/d028f47a-7e66-4e3e-9e52-760c1bd751a2" />
<br>
<br>


<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->
#### Firewall pfSense + pfBlockerNG

O pfBlockerNG é um pacote adicional do pfSense para bloquear domínios e IPs maliciosos. Ele usa DNSBL (DNS Blackhole List) e listas de IPs.

Para bloquear sites, foi adicionada uma regra explícita de `DROP` para pacotes da rede interna com destino a IPs de sites proibidos (`youporn.com`, `facebook.com`, `pornhub.com` e `bittorrent.com`), obtidos de uma lista criada no pfBlockerNG.

O pacote também exibe uma página de bloqueio local, mas não realiza o redirecionamento real da requisição. O bloqueio ocorre de forma local e não interage com o destino final do tráfego.

<img width="815" alt="rule_ips_proibidos" src="https://github.com/user-attachments/assets/b0cebe52-fbb3-4343-a828-120b87dd0bef" />
<br>
<br>


Página de bloqueio do pacote pfBlockerNG no pfSense:

<img width="815" alt="bloqueio_pfblockerng" src="https://github.com/user-attachments/assets/3a5106be-4c42-41f2-9d47-06a63efcfaa4" />
<br>
<br>


<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

#### Firewall pfSense + Squid + SquidGuard

O Squid é um proxy de cache para a web, e o SquidGuard é um redirecionador de URLs que complementa o Squid. Juntos, eles permitem o bloqueio e redirecionamento de usuários para uma página de bloqueio personalizada.

O SquidGuard permite o redirecionamento real da requisição, alterando a URL e enviando a página de bloqueio para o usuário. Para isso, foi criada uma lista de `target categories` no SquidGuard para os sites proibidos.

<img width="815" alt="sites_proibidos" src="https://github.com/user-attachments/assets/0edf68a8-8c05-42c0-a4c4-b563f7638112" />
<br>
<br>


Página de bloqueio do pacote SquidGuard no pfSense:

<img width="815" alt="bloqueio_squid" src="https://github.com/user-attachments/assets/11a85cd5-acad-4508-8b9c-e31fa68097c7" />
<br>
<br>


<!-- ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ -->

### Firewall FortiGate

O segundo firewall utilizado nos experimentos foi o NGFW FortiGate-VM64, que também apresenta uma página de bloqueio enviada ao usuário ao acessar algum domínio proibido, versão 7.2.0 (build 1157, 220331 - GA.F). A escolha dessa versão se deu pelo fato de ela constar na lista de versões vulneráveis, conforme identificado pela vulnerabilidade CVE-2022-27491.

As especificações do ambiente virtual utilizado nos testes são apresentadas a seguir:

- Memória RAM: 2048 MB  
- Processador: 1 vCPU  
- Armazenamento: Não especificado  
- Adaptador de rede: 10000 Mb/s (Intel Gigabit Internet 82545EM)  

Adotado como substituto do pfSense, o firewall FortiGate foi configurado com três interfaces de rede, replicando a topologia e a função atribuídas na configuração anterior. A configuração de rede foi definida da seguinte forma:

- Hostname: `FortiGate-VM64-KVM`
- Endereço IP WAN: `10.0.0.1/24` (port1)  
- Endereço IP LAN: `192.168.24.100/24` (port2)  
- Endereço IP INTERNET: atribuído via DHCP (port3)  

O FortiGate foi escolhido por oferecer um mecanismo de filtragem de conteúdo mais integrado ao fluxo de rede, com capacidade real de interceptar conexões e responder diretamente com páginas de bloqueio personalizadas, que são efetivamente enviadas ao cliente como respostas HTTP completas. Ao contrário das soluções anteriores, o FortiGate não depende de manipulação de DNS ou de proxies explícitos para exibir mensagens de bloqueio, ele atua diretamente no tráfego, com inspeção profunda de pacotes (DPI) e resposta imediata, o que aumenta a chance de a página ser refletida até o destino forjado em ataques de amplificação.

Além disso, o FortiGate permite um controle mais granular das políticas de segurança e respostas, com ferramentas específicas para personalização de mensagens de bloqueio, análise de sessões e tratamento de conexões baseadas em comportamento. Esses recursos tornam a solução mais adequada para testes de segurança avançados e para a análise de como middleboxes interagem com tráfego forjado.

Portanto, o uso do FortiGate representou uma evolução na metodologia experimental, oferecendo maior controle e visibilidade sobre o tráfego, além de um potencial mais elevado de gerar respostas refletidas úteis para o estudo de ataques de amplificação TCP baseados em middleboxes.

O FortiGate foi configurado com regras de firewall semelhantes às criadas no pfSense, conforme as seguintes políticas:

- Uma regra que permite o recebimento de pacotes ICMP (ping) originados de máquinas externas (rede WAN) para máquinas internas da rede LAN;  

<img width="415" alt="firewall_policy_ping_wan_lan" src="https://github.com/user-attachments/assets/8ac08b55-9c57-4d6a-8880-24aa29e2ffa3" />
<br>
<br>


- Uma regra que permite o tráfego de saída de máquinas internas (rede LAN) para a rede externa (rede WAN) por meio do FortiGate.  

<img width="415" alt="firewall_policy_lan_wan" src="https://github.com/user-attachments/assets/8afec0d7-e85d-4bc5-90d7-39353ee42454" />
<br>
<br>


Para o bloqueio de sites proibidos, foi criado um perfil de Web Filter, que foi posteriormente aplicado às regras de acesso da LAN, conforme mostrado na imagem a seguir.  

<img width="415" alt="sites_proibidos_fortigate" src="https://github.com/user-attachments/assets/1699794d-8796-47ec-bcf8-128ba2b44732" />
<br>
<br>


Página de bloqueio do FortiGate:

<img width="815" alt="bloqueio_fortigate" src="https://github.com/user-attachments/assets/5d3aaa28-f90c-42c9-a15f-4c211affe61e" />
<br>
<br>
