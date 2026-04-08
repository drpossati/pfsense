# HP ProLiant DL380 G5


### Comandos uteis

-   Ferramenta para a formatação e gravação de DVD-ROM
    ```
    sudo apt update && sudo apt install dvd+rw-tools

    lsblk

    dvd+rw-format -blank /dev/sr0

    growisofs -dvd-compat -Z /dev/sr0=/caminho/para/imagem.iso -speed=4
    ```
    
    -   **lsblk**: Listar todos os dispositivos de armazenamentos conectados

    -   **dvd+rw-format -blank**: Zerar o disco
    
    -   **dvd-compat**: Esse é o nosso trunfo. Ele "tranca" o disco no final da gravação, simulando um DVD-ROM de fábrica.

    -   **Z**: Garante que a ISO seja gravada na primeira trilha (Setor 0), que é o que torna o disco inicializável (bootável).

    -   **/dev/sr0**: Geralmente é o endereço padrão do gravador.

    -   **speed=4**: Força o laser a trabalhar devagar, evitando erros de leitura.


### Atualização dos Firmwares do DL380 G5.

*   Última versão de atualização 2014: SPP2014060.2014_0618.4.iso

-   O Hardware antigo foi desenvolvidos com base em sistemas operacionais como o Windows Server e Linux Red Hat Enterprise da época (2006 e 2007). Pacotes de atualização dos tipos `.exe` e `.rpm`

*   Tentativas de atualizações com _boot_ direto com a ISO SPP (pendrive, iLO2 e DVD-ROM) falharam. Ocorre um erro de leitura que leva a uma reação em cadeia que impede a atualização.

-   LiveCD do CentosOS 5.4, subir um sistema compatível diretamente na memória RAM. Ambientes "Live" rodam em uma partição virtual na RAM (tmpfs) e são "enxutos". Falta espaço temporário para extrair os pacotes gigantescos do SPP e bibliotecas base que o atualizador exigia.

    -   Contornar a limitação de espaço temporário direcionando as pastas temporárias do LiveCD para um pendrive, falha pois a taxa de leitura e gravação das USBs antiga não suporta a extração dos pacotes do SPP.

*   Solução prática e rápida: Instalação do CentosOS 5.11 no DL 380, extração das pastas do SPP.ISO e executar a atualização em `/pasta_do_ISO/hp/swpackages/hpsum`


### Instalação da Proxmox 9.1.

-   Configuração da BIOS para virtualização: 

    -   **Advanced Options $\to$ Processor Options**
        -   Intel(R) Virtualization Technology $\to$ enable
        -   No-Execute Memory Protection $\to$ enable

    -   **Power Management Options $\to$ HP Power Regulator**
        -   HP Power Regulator $\to$ OS Control Mode ou Static High Performance

*   Pendrive de `boot` com o balemaEather e a imagem ISO da Proxmox

-   A placa de vídeo (ATI ES1000) antiga do DL380 não suporta instalação gráfica.

    -   Escolha a opção de instalação via terminal, tecle `e` para edição da linha de `boot`

        -   Na linha do linux: remova o `quiet` e o `splash` e adicione `nomodeset` no lugar. Assim a instalação não esconderá os processos em execução e garantirá o modo de texto puro.

*   Configuração do domínio: `pve.local.host`

-   Partição `LVM-Thin`

    -   Nunca usar partições`ZFS` em discos gerenciados por uma controladora física. 


### Configurações da Proxmox 9.1

-   **Instalação e configuração dos pacotes** `vim` e `sudo` `apt install vim sudo`

    -   Habilitar números de linha no vim: `set number`
    -   Criar novo usuário e habilitar o sudo: `adduser usuario && usermod -aG sudo usuario`
    -   Habilitar o novo usuário Linux (@pam) para painel web da Proxmox: `sudo pveum user add possati@pam`
    -   Poderes de Administrador: `sudo pveum acl modify / -user possati@pam -role Administrator`


*   **Instalação e configuração do** `ssh` `apt install openssh-server`

    -   Alteração da porta: `Port xxxxx`
    -   Bloqueio do usuário root: `PermitRootLogin no`
    -   Limitar usuário: `AllowUsers usuario`
    -   Autorização via chave ssh
        ```
            mkdir -p ~/.ssh
            vim ~/.ssh/authorized_keys (colar a chave pública)
            chmod 700 ~/.ssh        
            chmod 600 ~/.ssh/authorized_keys
        ```
    -   Autorizar o uso de chave: `PubkeyAuthentication yes`
    -   Impedir o uso de senha: `PasswordAuthentication no`

-   **Rede de Alta Disponibilidade (4 placas LAN)**

    -   Configurações: `/etc/network/interfaces`
    
    ```
    auto lo
    iface lo inet loopback

    # Deixa as 4 placas físicas limpas, sob o comando do Bond
    iface PLACA_1 inet manual
    iface PLACA_2 inet manual
    iface PLACA_3 inet manual
    iface PLACA_4 inet manual

    # Cria a interface agregada unindo as 4 placas
    auto bond0
    iface bond0 inet manual
            bond-slaves PLACA_1 PLACA_2 PLACA_3 PLACA_4 #(nome das placas no SO)
            bond-miimon 100 #(monitor checa a placa a cada 100 ms)
            bond-mode active-backup #(modo de backup, placas em redundância)

    # A Ponte principal agora pede o IP via DHCP do pfSense
    auto vmbr0
    iface vmbr0 inet dhcp #( ou static)
            # address SEU_IP_PRINCIPAL
            # gateway SEU_GATEWAY
            bridge-ports bond0
            bridge-stp off
            bridge-fd 0
            hwaddress MAC_DA_PLACA_01 #(MAC do agrupamento das placas)
    ```

    -   `bond0`: Agregação das placas em um único link e IP: 

    -   `vmbr0`: bridge principal, conecta o servidor e as VMs

    -   `ip -br link`: nome e MAC das placas de rede local no SO

    -   `ifreload -a`: aplicar as configurações de rede sem reiniciar
    

*   **Habilitar o repositório livre da Proxmox**

    -   Copiar e renomeia o `pve-enterprise.sources` para `pve-no-subscription`

        ```
        cd /etc/apt/sources.list.d/
        cp pve-enterprise.sources ./pve-no-subscription.sources`
        ```
    -   Edite o novo arquivo e altere o campo `Components` de `pve-enterprise` para `pve-no-subscription`

    -   `/etc/apt/sources.list.d/pve-no-subscription.sources`

        ```
            Types: deb
            URIs: http://download.proxmox.com/debian/pve
            Suites: trixie
            Components: pve-no-subscription
            Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
        ```

    -   Desabilitar os repositórios Enterprise

        -   `mv pve-enterprise.source ./pve-enterprise.bak`
        -   `mv ceph.source ./ceph.bak`

    -   Atualização de todos os pacotes

        `sudo apt update && sudo apt full-upgrade -y`


-   **Desabilitar a mensagem de No-Subscription no Pop-Up**

    -   Alterar a mensagem no arquivo: `/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js`
    
    ```
        sudo sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid subscription'\),)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && sudo systemctl restart pveproxy
    ```
    -   `systemctl restart pveproxy`
