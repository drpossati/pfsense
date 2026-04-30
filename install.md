# PfSense in BOX 

-   Estudos e configurações sobre o pfsense

### Imagem

-   AMD64 Memstick USB (Netgate 1537, 1541, 4100, 4200, 5100, 6100, 7100, 8200, All Other Intel/AMD 64-bit)

-   Boot Pen Driver (balenaEtcher)

### Hardware FW-7535D da Lanner Electronics

-   Comunicar o FW-7535D com um computador

*   Identificar a porta serial do PC

    -   `sudo dmesg | grep tty`

    -   Ex.: `ttyUSB0` ou `ttyS0`

    -   No caso de um adaptador USB Serial verificar a necessidade de drivers

-   Acesso via porta serial

    -   Cabo serial RJ45 para DB9

    -   `screen /dev/ttyS0 115200, cs8, -ixon, -ixoff`

        -   Baud Rate: 115200
        -   Data Bits: 8
        -   Parity: None
        -   Stop Bits: 1
        -   Flow Control (Hardware e Software): None

*   Reiniciei o FW-7535D e o terminal do computador se comportará como um monitor mostrando as informações de inicialização do FW-7535D

-   Em caso de falha

    -   Permissões: `ls -l /dev/ttyS*`

    -   Permission denied: `sudo chmod 666 /dev/ttyS0` 

    -   Group dialout: `sudo usermod -aG dialout $USER`


## Instalação do PfSense CE

[Instalação passo a passo da versão 2.8.0](https://ti-videotutoriais.blogspot.com/2025/06/pfsense-280-passo-passo-da-instalacao.html)

-   Opções avançadas
    -   habilitar repositórios CE
    -   habilitar console serial

*   Selecionar interface de rede ativa WAN

    -   A instalação precisa de conexão com a internet

-   Selecionar interface de rede LAN ou Skip

*   Instalação CE

-   Confirmar o disco e a formatação

*   Selecionar a versão (stable preferencial)

    -   Versões da comunidade CE

    -   Versões comerciais

-   **Finalizado a instalação**

    -   reboot
    -   Acesso GUI pelo IP definido
        -   usuário: admin
        -   senha: pfsense