<center>

# Configuração de Rede em Alta Disponibilidade para ambientes de virtualização Proxmox.

<p align="justify">
Este repositório contém a configuração de rede em alta disponibilidade para um servidor Proxmox, com foco na utilização de bonding (agregação de interfaces de rede) e VLANs (Virtual LANs), com o objetivo de garantir uma rede resiliente, segmentada e com balanceamento de carga. A seguir estão os detalhes de configuração e os requisitos necessários para implementar essa solução em ambientes de produção.

Pré-requisitos
Antes de aplicar a configuração descrita neste documento, é importante garantir que os seguintes requisitos sejam atendidos:

1. Interfaces Físicas
Para a implementação de bonding, você deve possuir ao menos duas interfaces de rede físicas disponíveis. No caso de uma configuração de bonding 802.3ad (LACP), é necessário que ambas as interfaces sejam conectadas a switches que suportem agregação de links (LACP).

Exemplo: No arquivo de configuração, estamos utilizando as interfaces enp6s0f0 e enp6s0f1 para o bonding.
Requisito: Ambas as interfaces precisam ser configuradas manualmente para não obterem IP diretamente e estarem preparadas para agregação.
2. Switches com Suporte a LACP (Link Aggregation Control Protocol)
O uso de bonding 802.3ad exige que os switches aos quais as interfaces físicas estão conectadas ofereçam suporte ao protocolo LACP, que permite a agregação das interfaces. O Proxmox, por meio da configuração do arquivo /etc/network/interfaces, será capaz de combinar as interfaces físicas em um único canal de comunicação.

3. Ambiente de Rede Segmentado (VLANs)
Se você estiver utilizando VLANs para segmentação de rede, é necessário que a infraestrutura de rede (como switches e roteadores) esteja configurada para suportar VLAN tagging.

VLANs no ambiente: As VLANs são usadas para segmentar a rede em diferentes sub-redes lógicas, proporcionando mais segurança, organização e controle de tráfego. A configuração de VLANs no Proxmox requer a criação de interfaces VLAN em cima da interface de bonding, utilizando a sintaxe bond0.XXX, onde XXX é o ID da VLAN.
4. Ambiente Virtualizado (Proxmox)
O Proxmox VE deve estar configurado em um ambiente com máquinas virtuais (VMs) ou containers, pois as bridges configuradas (vmbr0, vmbr200, etc.) são utilizadas para conectar essas VMs ou containers às diferentes redes, conforme as VLANs configuradas.

5. Configuração de IP e Gateway
Para a bridge da rede principal (vmbr0), um IP estático e um gateway devem ser configurados para garantir que o servidor Proxmox tenha acesso à rede externa.

Exemplo: O IP 192.168.18.200/24 é atribuído à bridge vmbr0, e o gateway da rede é definido como 192.168.18.1.
6. Conectividade de Rede
Verifique se a conectividade de rede entre o Proxmox e os switches de agregação de links (para o bonding) está funcionando corretamente. Isso pode ser feito através de comandos de rede como ping e traceroute, além de verificar as configurações nos switches para garantir que a agregação de links (LACP) esteja configurada corretamente.
</p>
Estrutura da Rede
A estrutura de rede do Proxmox foi configurada da seguinte forma:

Interfaces Físicas: Configuração de interfaces de rede físicas.
Bonding: Criação de um bond para agregação de interfaces de rede.
Bridges: Configuração de bridges para diferentes redes, cada uma associada a uma VLAN específica.
VLANs: Configuração de VLANs para segmentação de rede.
Configuração do arquivo /etc/network/interfaces
1. Interfaces Físicas
As interfaces físicas são configuradas manualmente, sem a atribuição de IPs, e com MTU ajustado para 1500.

Exemplo:
bash
Copy
Edit
# Configuração da interface loopback
auto lo
iface lo inet loopback

# Configuração da interface física enp2s0 (não atribui IP, apenas configura manualmente)
iface enp2s0 inet manual
    mtu 1500

# Configuração da interface física enp6s0f0 (não atribui IP, apenas configura manualmente)
auto enp6s0f0
iface enp6s0f0 inet manual
    mtu 1500

# Configuração da interface física enp6s0f1 (não atribui IP, apenas configura manualmente)
auto enp6s0f1
iface enp6s0f1 inet manual
    mtu 1500
2. Bonding (Agregação de Interfaces Físicas)
O bond0 é criado como uma interface agregada, utilizando as interfaces enp6s0f0 e enp6s0f1. O modo de bonding utilizado é o 802.3ad, com monitoramento de link (bond-miimon) e política de transmissão (bond-xmit-hash-policy) baseada em camadas 2 e 3.

Exemplo:
bash
Copy
Edit
# Configuração do bonding (agregação de interfaces físicas enp6s0f0 e enp6s0f1)
auto bond0
iface bond0 inet manual
    bond-slaves enp6s0f0 enp6s0f1
    bond-miimon 100
    bond-mode 802.3ad
    bond-xmit-hash-policy layer2+3
    mtu 1500
3. Bridges de Rede
As bridges são configuradas para diferentes redes, com cada bridge associada a uma VLAN específica. As bridges são configuradas para atuar como interfaces de rede para máquinas virtuais, containers, e outros dispositivos.

Bridge vmbr0 (Rede Principal)
A bridge vmbr0 é configurada para a rede principal (192.168.18.200/24), com o gateway definido como 192.168.18.1. Ela utiliza o bond0 como dispositivo de link.

bash
Copy
Edit
# Configuração do bridge vmbr0, para rede principal
auto vmbr0
iface vmbr0 inet static
    address 192.168.18.200/24
    gateway 192.168.18.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    mtu 1500
Bridge vmbr200 (Rede DMZ - VLAN 200)
A bridge vmbr200 é configurada para a rede DMZ, associada à VLAN 200. A interface de rede é o bond0.200.

bash
Copy
Edit
# Configuração do bridge vmbr200, para rede DMZ (VLAN 200)
auto vmbr200
iface vmbr200 inet manual
    bridge-ports bond0.200
    bridge-stp off
    bridge-fd 0
    mtu 1500
Bridge vmbr300 (Rede de Servidores - VLAN 300)
A bridge vmbr300 é configurada para a rede de servidores, associada à VLAN 300. A interface de rede é o bond0.300.

bash
Copy
Edit
# Configuração do bridge vmbr300, para rede de servidores (VLAN 300)
auto vmbr300
iface vmbr300 inet manual
    bridge-ports bond0.300
    bridge-stp off
    bridge-fd 0
    mtu 1500
Bridge vmbr400 (Rede Administrativa - VLAN 400)
A bridge vmbr400 é configurada para a rede administrativa, associada à VLAN 400. A interface de rede é o bond0.400.

bash
Copy
Edit
# Configuração do bridge vmbr400, para rede administrativa (VLAN 400)
auto vmbr400
iface vmbr400 inet manual
    bridge-ports bond0.400
    bridge-stp off
    bridge-fd 0
    mtu 1500
4. Configuração de VLANs
As VLANs são configuradas sobre o bond0, criando interfaces VLAN específicas (bond0.200, bond0.300, bond0.400). Essas interfaces são configuradas como interfaces "manuais", sem atribuição de IP diretamente.

VLAN 200
bash
Copy
Edit
auto bond0.200
iface bond0.200 inet manual
    vlan-raw-device bond0
VLAN 300
bash
Copy
Edit
auto bond0.300
iface bond0.300 inet manual
    vlan-raw-device bond0
VLAN 400
bash
Copy
Edit
auto bond0.400
iface bond0.400 inet manual
    vlan-raw-device bond0
Considerações Finais
Bonding: O uso de bonding aumenta a redundância e a largura de banda agregada das interfaces de rede físicas.
VLANs: As VLANs permitem a segmentação de rede, proporcionando segurança e organização das redes virtuais.
Bridges: As bridges permitem a interconexão de máquinas virtuais e containers com diferentes segmentos de rede.
Essa configuração proporciona uma rede eficiente e segmentada, adequada para ambientes virtualizados no Proxmox.

Essa versão da documentação agora inclui uma seção de pré-requisitos detalhada, que descreve as necessidades para implementar essa configuração de rede no Proxmox, além de todas as informações relevantes para garantir que a infraestrutura seja configurada corretamente e em alta disponibilidade.