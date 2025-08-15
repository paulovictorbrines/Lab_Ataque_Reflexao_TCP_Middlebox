Aqui está o texto LaTeX convertido para o formato Markdown.

-----

# Descrição do Código do Ataque

O código usado para realizar os ataques foi obtido de um repositório público no GitHub, do usuário **moloch54**. O projeto, chamado "Ddos-TCP-Middlebox-Reflection-Attack", contém o arquivo `mra.py` para a execução do ataque. Após uma investigação, foi identificado que o autor do código é Sébastien Meniere, residente em Nancy, França, conforme informações em seu perfil no LinkedIn.

O código foi muito útil para a condução dos experimentos em laboratório, permitindo a implementação prática dos conceitos do artigo científico "Weaponizing Middleboxes for TCP Reflected Amplification" de Bock et al. Com ele, foi possível replicar a técnica de **Middlebox Reflection (b)**, facilitando a compreensão e validação dos mecanismos de reflexão amplificada usando *middleboxes* sobre o protocolo TCP. Nessa técnica, o endereço IP de origem é falsificado como sendo o da vítima, fazendo com que as respostas das *middleboxes* atinjam diretamente o alvo.

Para entender o ataque, é preciso revisar o processo de estabelecimento de conexão TCP, o **Three-Way Handshake**, que tem três etapas:

1.  O cliente (SRC) envia um pacote **SYN** para iniciar a conexão;
2.  O servidor (DEST) responde com um pacote **SYN, ACK**, reconhecendo a solicitação;
3.  O cliente envia um pacote **ACK**, finalizando a conexão.

O ataque explora o fato de que, em redes com dispositivos intermediários (como *firewalls* ou *middleboxes*), os pacotes podem seguir caminhos diferentes. O pacote SYN é falsificado (com `SRC=Vítima` e `DST=Servidor`), mas a resposta SYN, ACK do servidor real pode seguir por outra rota e não ser vista pela *middlebox*. Isso cria uma inconsistência de estado.

O atacante, após o SYN falsificado, envia um segundo pacote **ACK** (também com IP de origem falsificado). A *middlebox*, por não ter visto o SYN, ACK, interpreta esse ACK como inválido e pode gerar respostas automáticas, como pacotes **RST** ou múltiplas mensagens de bloqueio/advertência, amplificando o tráfego contra a vítima.

O ataque pode ser refinado com pacotes contendo dados. Após o SYN, o atacante pode enviar um pacote **ACK+PSH** com uma carga útil, como uma requisição HTTP. Por exemplo:

  * Envio de um pacote **SYN** com endereço IP de origem falsificado (da vítima) e destino a domínios com alto potencial de bloqueio por *middleboxes*: `SRC=Vítima`, `DST=Pornhub|Youporn|Bittorrent...`;
  * Envio subsequente de um pacote **ACK+PSH** com carga útil: requisição HTTP GET, também com `SRC=Vítima`.

Esses pacotes, ao serem processados por *middleboxes* de filtragem de conteúdo, podem disparar múltiplas respostas, reforçando o efeito de amplificação.

A escolha por esse código público se deu por sua estrutura clara e alinhamento com a metodologia de Bock et al., o que garantiu a conformidade da implementação e permitiu focar nos aspectos experimentais e analíticos, evitando o desenvolvimento do zero.

-----

# Execução do Código

Para a execução do código de ataque de reflexão amplificada sobre TCP, é preciso seguir os passos descritos no arquivo README.md do repositório. Primeiro, as dependências devem ser instaladas e configuradas. As ferramentas necessárias são:

  * **tcpreplay**: para reproduzir pacotes em uma interface de rede.
  * **mergecap**: para mesclar pacotes de múltiplas threads em um único arquivo .pcap.
  * **scapy**: biblioteca Python para construção e manipulação de pacotes TCP.

Para instalar as dependências em sistemas Debian/Ubuntu, use o seguinte comando:

```bash
sudo apt-get install tcpreplay mergecap python3-scapy
```

Após a instalação, o código pode ser executado diretamente pelo terminal. O comando para iniciar o ataque é:

```bash
sudo python3 mra.py <tempo_em_segundos> <IP_alvo>
```

Aqui, o `<tempo_em_segundos>` define a duração do ataque, e o `<IP_alvo>` é o endereço IP que se deseja sobrecarregar.

O script gera pacotes TCP falsificados, enviando um pacote **SYN** para um site de destino (filtrado por uma *middlebox*) e, em seguida, um pacote **ACK + PSH** com um *payload* HTTP. Isso faz com que a *middlebox* responda com um **RST** ou uma página de erro. O tráfego gerado é salvo em arquivos .pcap e reproduzido indefinidamente usando o `tcpreplay`, enviando os pacotes para a interface de rede especificada.

Um exemplo de execução seria:

```bash
sudo python3 mra.py 300 123.4.5.6
```

Isso executará o ataque por 300 segundos contra o IP 123.4.5.6. É importante ressaltar que o uso deste código para ataques não autorizados é ilegal e antiético. A intenção é apenas educacional e para testes em ambientes controlados.

-----

# Virtualizador VMware Workstation Pro

Para a virtualização das máquinas virtuais (alvo, atacante e *firewalls* pfSense e FortiGate), foi usado o **VMware Workstation Pro 17 for Personal Use** (versão 17.6.3-24583834), disponível gratuitamente. Ele pode ser baixado no site oficial da VMware, mas é necessário se registrar no portal da Broadcom.

O VMware Workstation é um hipervisor de desktop para Windows e Linux, que permite criar e executar VMs com diversos sistemas operacionais sem precisar reiniciar o computador. É uma plataforma robusta e versátil, ideal para desenvolvimento, testes e simulações de software em ambientes virtuais isolados.

-----

# Configuração do Laboratório

O ambiente do laboratório foi dividido em três cenários, todos com a mesma topologia de rede e endereços IP. Cada cenário tinha três elementos: **um alvo, um atacante e uma *middlebox*** (representada por um *firewall*).

  * **Cenário 1:** Firewall com **pfSense** e o complemento **pfBlockerNG**.
  * **Cenário 2:** Firewall com **pfSense** e os softwares **Squid** e **SquidGuard**.
  * **Cenário 3:** Firewall **FortiGate**.

Os *firewalls* foram configurados para simular comportamentos típicos e também configurações incorretas, a fim de mostrar como certas escolhas podem tornar esses dispositivos vulneráveis a ataques de reflexão amplificada sobre TCP.

### Máquina Alvo

A máquina alvo foi uma instância do sistema operacional **Ubuntu Linux 22.04.5 LTS (Jammy)**, com as seguintes especificações:

  * **Memória RAM:** 4096 MB
  * **Processador:** 2 vCPUs
  * **Armazenamento:** 25 GB
  * **Adaptadores de rede:** 1000 Mb/s (Intel Gigabit Internet 82545EM)

A VM possui duas interfaces de rede. A primeira está conectada ao segmento LAN (`ens34`), simulando uma rede local protegida atrás do *firewall*. A segunda (`ens37`) está em modo NAT para acesso à internet.

A configuração de rede é:

  * **Hostname:** `Ubuntu`
  * **Endereço IP LAN:** `192.168.24.61/24`
  * **Endereço IP INTERNET:** IP via DHCP
  * **Gateway padrão:** `192.168.24.100`
  * **Servidor DNS:** `192.168.24.100`

### Máquina Atacante

A máquina atacante foi baseada no sistema operacional **Kali Linux**, versão 2025.1. Suas especificações são:

  * **Memória RAM:** 8096 MB
  * **Processador:** 6 vCPUs
  * **Armazenamento:** 32 GB
  * **Adaptadores de rede:** 1000 Mb/s (Intel Gigabit Internet 82545EM)

A VM também tem duas interfaces de rede. A primeira está conectada ao segmento WAN (`eth0`), simulando um agente externo na internet. A segunda (`eth1`) está em modo NAT para acesso à internet.

A configuração de rede é:

  * **Hostname:** `kali`
  * **Endereço IP WAN:** `10.0.0.2/24`
  * **Endereço IP INTERNET:** IP via DHCP
  * **Gateway padrão:** `10.0.0.1`

### Firewall pfSense

O primeiro *firewall* usado foi o **pfSense Community Edition (CE)**, versão `2.7.2-RELEASE`. É uma distribuição de software de *firewall* de código aberto baseada no FreeBSD.

Suas especificações de hardware virtual são:

  * **Memória RAM:** 2048 MB
  * **Processador:** 2 vCPUs
  * **Armazenamento:** 20 GB
  * **Adaptadores de rede:** 1000 Mb/s (Intel Gigabit Internet 82545EM)

A VM do pfSense foi configurada com três interfaces de rede:

  * **em0:** Conectada ao segmento WAN (`10.0.0.1/24`).
  * **em1:** Conectada ao segmento LAN (`192.168.24.100/24`).
  * **em3:** Em modo NAT para acesso à internet.

O **NAT Outbound** foi configurado para redirecionar o tráfego da LAN para a WAN. As regras de *firewall* da interface WAN incluíam permissões para **ICMP** (ping) de máquinas externas, acesso externo ao servidor Web do alvo e acesso irrestrito a sites, que seriam posteriormente filtrados. As regras da interface LAN continham as regras padrão de `anti-lockout` e `default allow LAN to any`.

#### Firewall pfSense + pfBlockerNG

O **pfBlockerNG** é um pacote adicional do pfSense para bloquear domínios e IPs maliciosos. Ele usa **DNSBL** (DNS Blackhole List) e listas de IPs.

Para bloquear sites, foi adicionada uma regra explícita de `DROP` para pacotes da rede interna com destino a IPs de sites proibidos (como `youporn.com`, `pornhub.com`, etc.), obtidos de listas no pfBlockerNG.

O pacote também exibe uma página de bloqueio local, mas não realiza o redirecionamento real da requisição. Como o pfBlockerNG não reflete o conteúdo para um alvo externo, ele **não foi útil para o ataque de amplificação**. O bloqueio ocorre de forma local e não interage com o destino final do tráfego.

#### Firewall pfSense + Squid + SquidGuard

O **Squid** é um proxy de cache para a web, e o **SquidGuard** é um redirecionador de URLs que complementa o Squid. Juntos, eles permitem o bloqueio e redirecionamento de usuários para uma página de bloqueio personalizada.

O SquidGuard foi o componente principal deste experimento porque permite o **redirecionamento real da requisição**, alterando a URL e enviando a página de bloqueio para o usuário. Para isso, foi criada uma lista de `target categories` no SquidGuard para os sites proibidos.

### Firewall FortiGate

O segundo *firewall* foi o **NGFW FortiGate-VM64**, versão 7.2.0, escolhido por ser vulnerável à **CVE-2022-27491**. É uma solução comercial amplamente adotada, que oferece proteção avançada e alta performance.

O FortiGate também exibe uma página de bloqueio quando um site proibido é acessado, como visto em ambientes de grandes instituições. Suas especificações de hardware virtual são:

  * **Memória RAM:** 2048 MB
  * **Processador:** 1 vCPU
  * **Armazenamento:** Não especificado
  * **Adaptador de rede:** 10000 Mb/s (Intel Gigabit Internet 82545EM)

O FortiGate foi configurado com três interfaces de rede, replicando a topologia do pfSense:

  * **Endereço IP WAN:** `10.0.0.1/24` (port1)
  * **Endereço IP LAN:** `192.168.24.100/24` (port2)
  * **Endereço IP INTERNET:** IP via DHCP (port3)

A opção pelo FortiGate foi motivada por ele ser mais eficaz em interceptar e responder com **páginas de bloqueio que são efetivamente enviadas ao cliente**. Diferente das soluções anteriores, ele usa **inspeção profunda de pacotes (DPI)** e responde de forma imediata, aumentando a chance de a página ser refletida para o destino falsificado nos ataques. Isso proporcionou maior controle e visibilidade sobre o tráfego, tornando-o mais adequado para os testes de segurança.

As regras de *firewall* do FortiGate foram configuradas de forma similar às do pfSense, permitindo tráfego **ICMP** da WAN para a LAN e o tráfego de saída da LAN para a WAN. Um perfil de **Web Filter** foi criado para bloquear os sites proibidos e aplicado às regras de acesso da LAN.
