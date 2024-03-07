### Limpando o Ambiente Experimental
1. Se assegure que o ambiente não está em uso.
2. TODO
3. Acesse o controlador experimental usando ssh com `ssh -L 5240:localhost:5240 -p 2002 ubuntu@10.43.0.2` $^1$
<!-- O IP 10.43.0.2 foi fixado. -->

## Importante
1. lembre-se de utilizar `sudo apt update` antes de qualquer instalação

## Ferramentas Importantes para Rede
1. Instale o net-tools com `sudo apt install net-tools` 
2. Instale o net-tools com `sudo apt install nmap` 
3. Instale o net-tools com `sudo apt install ipmitool` 

### Instalando o MAAS
1. Acesse o controlador do ambiente experimental.
2. Instale o MAAS com `sudo snap install maas --channel=3.4/edge`
3. Instale o PostgreSQL com `sudo apt install postgresql-14`
4. Crie um usuário para o MAAS:
    1. Entre na linha interativa do psql usando `sudo -iu postgres psql`
    2. Crie um usuário usando `CREATE USER maas WITH ENCRYPTED PASSWORD 'bRWIbS9ZVKQO0QfSiNi1aA';`\
        2.1 Possível verificar usuários com `\du`
    3. Saia da linha interativa usando `\q`
5. Crie um banco de dados para o MAAS com `sudo -iu postgres createdb -O maas maas`
6. Inicialize o banco de dados com `sudo maas init region+rack --database-uri "postgres://maas:bRWIbS9ZVKQO0QfSiNi1aA@localhost/maas" --maas-url "http://127.0.1.1:5240/MAAS"` \
    (deve-se definir a MAAS URL com o IP localhost da maquina onde estamos http://$hostname -i:5240/MAAS [127.0.1.1 default])
7. Crie um admin no MAAS com `sudo maas createadmin` e preencha os campos que aparecerem (username, password, email).
    1. OBS: Campo SSH pode deixar vazio, vamos colocar usando a WEB UI, ou pode adicionar a chave SSH do computador que irá utilizar para conectar nas máquinas.
8. Verifique se o maas está funcionado usando `sudo maas status` (o campo bind9 e http deve estar rodando)
10. No seu browser acesse `localhost:5240/MAAS` (cheque se você usou `-L 5240:localhost:5240` quando se conectou ao SSH)
11. Faça login com o usuário e senha criado com sudo maas createadmin e faça a configuração inicial (tudo pode ser alterado posteriormente)
    1. dns: 8.8.8.8
    2. Escolher versão do ubuntu, selecionar `amd64` e clicar em `update selection` (*importante*: estamos usando 22.04 Jammy)
    3. OBS: em casos de erros acesse os logs em `/var/snap/maas/common/log/regiond.log` e `/var/snap/maas/common/log/rackd.log`, podemos alterar configurações do controller sem ter que derrubar o maas apenas reiniciando em `/var/snap/maas/XXXX/rackd.conf` e `/var/snap/maas/XXXX/regiond.conf` onde XXXX é o número snapshot que estiver nesse caminho.
12. Configurar o netplan usndo `sudo nano /etc/netplan/*.yaml` (utilizar tab pra completar o nome do *.yaml, pode variar, mas é o único arquivo yaml que vai ter na pasta)
    1. adicionar
    ```yaml
    ethernets:
        eno3:
            addresses: 10.42.0.1/16
    ```
    2. Rodar `sudo netplan apply`
13. Ligar DHCP server:
    1. Na barra lateral vá em Subnets
    2. Na subnet com ip 10.42.0.0/16 clicar em untaggeed na coluna VLAN
        1. Esse IP é da rede eno3 onde estão os nós fisicos colocados usando o switch lá no servidor
    3. Clicar em Configure DHCP
    4. Clicar em MAAS provides DHCP
    5. Clicar em Provide DHCP from rack controller(s)
    6. Colocar o controller que criamos (provavelmente nome controller)
    7. Range definir como quiser
    8. Gateaway ip: 10.42.0.1 (ip do controller, caso queira averiguar clica na subnet e desce até o final)
14. Adicionar nós maquinas no maas caso não tenham aparecido em máquinas: (estão definidas para dar netboot [PXE BOOT])
    1. Verificar no range definido usando nmap se algum BMC foi "fisgado"
        1. Usando o nmap 
            1. Se range `10.42.0.0 - 10.42.2.255` rodar `sudo nmap 10.42.0-2.0-255 -T5`
            2. Os IPs que aparecerem como Host is up são algo fisico, nossas máquinas no caso
        2. Ou listando o DHCP leases
            3. `sudo cat /var/snap/maas/common/maas/dhcp/dhcpd.leases | grep "^lease" | sort | uniq` 
    3. Verificar nos IPs encontrados se existe um servidor IPMI:
        1. Rodar `ipmitool -I lanplus -H {IP Encontrado Colocar Aqui} -U root -P root`
        2. Se aparecer `NO commands provided` e uma serie de comandos então é um servidor IPMI
        3. Verificar se está ligado a máquina, rodar `ipmitool -I lanplus -H {IP Encontrado Colocar Aqui} -U root -P root chassis power status`
        4. Se aparecer off, rodar `ipmitool -I lanplus -H {IP Encontrado Colocar Aqui} -U root -p root chassis power on`
    4. Esperar um pouco. Se as maquinas ligadas não aparecerem após poucos minutos:
        1. Baixe ou clone o repositório do projeto (este que você está lendo agora) para o seu computador
        2. Copie o arquivo _lpxelinux.0_ para o nó controller
            1. Pela UI 
                1. Abra o gerenciador de arquivos do linux e clique em outros locais.
                2. Conecte-se a `sftp://ubuntu@stratus.dc.ufscar.br:2002`
                3. Copie o arquivo `_lpxelinux.0_` que está em `Infra/etc` para alguma pasta no controller
           2. Ou pelo terminal
               1. Rode `sftp -P 2002 ubuntu@stratus.dc.ufscar.br`
               2. No contexto do SFTP, rode `put {Path do seu repositorio local}/MaterialCloud/etc/lpxelinux.0 {path do controller}`
       3. Apague a arquivo atual do _lpxelinux.0_ `sudo rm /var/snap/maas/common/maas/boot-resources/snapshot-X-Y/bootloader/pxe/i386/lpxelinux.0` \
          (onde X e Y são numeros aleatorios de cada instalação, use tab ao colocar `snapshot-` para autocompletar)
       4. Mova o novo arquivo _lpxelinux.0_ para o mesmo path do que você apagou \
        `sudo mv {path arquivo}/lpxelinux.0 /var/snap/maas/common/maas/boot-resources/snapshot-X-Y/bootloader/pxe/i386/`
       5. Reinicie as máquinas encontrados `ipmitools -I lanplus -H {IP Encontrado Colocar Aqui} -U root -P root chassis power reset` (faça em todas as máquinas)
       6. Verifique se as máquinas estão ligadas `ipmitool -I lanplus -H {IP Encontrado Colocar Aqui} -U root -P root chassis power status`
       7. Espere alguns segundos e verifique novamente na UI do MAAS, as máquinas devem estar conectadas ao MAAS
       8. Para melhor organização nomeie os nós
            1. Clicar no nome do nó 
            2. Na tela de configurações que aparecer clique no nome do nó no canto superior esquerdo
            3. De um nome que facilite entendimento (`compute0` e `compute1` por exemplo)
15. Realizar Comission no nó $^2$
    1. Clicar no nó recentemente adicionado (são adicionados com nomes aleatórios), pode trocar o nome para melhor organização, um se chamará compute0 e outro compute1
    2. Ir para aba Configurações do nó
    3. Editar power type
        1. Power type: selecione IPMI
        2. ip address: colocar o ip do servidor ipmi que encontramos
        3. power user: root
        4. power password: root
    4. Voltando agora a aba Machines vamos selecionar o nó configurado
    5. Se ele estiver dando comissioning, clicar comissioning e abort
    6. Clicar em Take Action -> Comission
        1. Selecionar allow ssh access and prevent machine from powering off e clicar em comission
    7. Esperar o processo acabar, em caso de falha do Comission:
        1. Va na barra lateral e clique em Settings
        2. Selecione Proxy nas configurações
        3. Selecione "MAAS provide proxy" $^3$
        4. Salve a alteração.
        5. Tente dar Comission novamente no nó
16. Criar _Bridge_ para o _NOVA_ nos nós que demos comission $^4$ $^5$:
    1. Na aba Machines clicar no nome do nó para ver suas configurações
    2. Clicar na aba Network
    3. Selecionar a interface que aparece (eno1 deve estar aparecendo) clicando no quadrado de seleção a esquerda do nome dela.
    4. Clicar em Create Bridge
    5. Na nova tela vamos configurar a _Bridge_
        1. Bridge Type: selecionar Open vSwitch(ovs)
        2. Bridge Name: colocar um nome fácil de lembrar por exemplo br-ex (*atenção*: o nome não pode ser "br0")
        3. Subnet: selecionar a subnet onde está nosso servidor DHCP: _10.42.0.0/16_
        4. IP Mode: selecionar "Auto assign"
        5. As outras configurações devem estar preenchidas por padrão. Caso não:
            1. Fabric: vá na barra lateral em subnets e veja a fabric da subnet escolhida acima
            2. VLAN: vá na barra lateral em subnets e veja a VLAN da subnet escolhida acima
            3. MAC Address: O mesmo da interface escolhida para criar a _bridge_
17. Criar _Bridge_ para o lxd VM HOST $^6$:
    1. Verificar se a _bridge_ do controller já existe:
        1. Clicar na aba Controllers na barra lateral
        2. Clicar no nome do controller listado (esperado que seja controller.maas)
        3. Ir para a aba Network na nova tela.
        4. Se aparecer uma network chamada br0 do tipo bridge está tudo certo.
        5. Se não existir a _bridge_ vamos criá-la:
            1. No terminal rodar o comando `sudo nano /etc/netplan/50-cloud-init.yaml `
            2. Altere adicionando a nossa _bridge_ da seguinte maneira:
             ```yaml
            network:
                bridges:
                br0:
                    addresses:
                    - 10.42.0.1/16
                    interfaces:
                    - eno3
                    macaddress: 00:26:b9:34:b7:c8
                    mtu: 1500
                    nameservers:
                    addresses:
                    - 10.42.0.1
                    search:
                    - maas
            ```
            3. Ainda no mesmo arquivo, encontre a rede eno3 em _ethernets_ e garanta que ela esteja assim: 
            ```yaml
                eno3:
                    match:
                        macaddress: 00:26:b9:34:b7:c8
                    mtu: 1500
                    set-name: eno3
            ```
            4. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
            5. No terminal aplicar as mudanças na rede, rodar o comando `sudo netplan apply`
            6. Reinicie o maas para encontrar as configurações, rodar o comando `sudo snap restart maas`
18. Instalar o lxd e criar nosso VM host:
    1. Verificar se o lxd já está instalado:
        1. No terminal rodar o comando `lxd`
        2. Se não aparecer `Error: This must be run as root`, instalar lxd:
            1. No terminal rodar `sudo snap install lxd`
    2. Com o lxd instalado, inicializar rodando o comando `sudo lxd init`
    3. Para as perguntas que aparecerem no terminal seguir os seguintes passos (responder como segue):
        - no (maas possui suporte para lxd clustering ja)
        - yes (new storage pool)
        - default
        - dir (where was mainly tested)
        - no (if connected to maas each vm will be added as new device)
        - no (we dont want to create a bridge we want to use br0 that already exists)
        - yes (configure lxd to use existing bridge)
        - br0 (bridge name)
        - yes (lxd server availabe over the network so you can access)
        - default address
        - default port 8443
        - default (stale cached images update automatically)
        - password (14159265)
        - password
        - default
        - default 
    4. Entrar na web UI do MAAS e na barra lateral entrar em LXD
    5. Na nova tela clicar em Add lxd host
    6. Deixar as configurações que vieram como default e adicionar em lxd address: `10.42.0.1` (o endereço da bridge criada)
    7. Selecionar o botão verde da tela de configuração (deve ser Generate New Certificate)
    8. Selecionar Add trust to lxd via cmd line
    9. Copiar o comando na caixa que aparecer até `EOF`
    10. No terminal colar o comando copiado, garantindo que será rodado com `sudo` (caso não tenha adicione antes de colar o comando)
    11. Voltar a configuração na web UI do MAAS e caso não tenha alterado automaticamente para próxima etapa,clicar no botão verde.
    12. Chegou em uma tela com estado de `connected` e uma bola verde indicando.
    13. Selecionar na opção add new project e escrever um nome para o projeto (fácil de lembrar), por exemplo `tst-project`
    14. Selecionar Save e sua VM host deve estar criada.
19. Criar uma VM para instalar o Juju.
    1. Na web UI do MAAS na barra lateral entrar em LXD
    2. Clicar no nome do projeto com o nome criado ao criar a VM host, supostamente `tst-project`
    3. Na nova tela selecionar o botão Add VM
    4. Uma nova tela apareceu para colocar configurações da nova máquina virtual, preencher da seguinte maneira:
        1. VM name: juju-machine (pode ser qualquer nome, essa será a máquina do juju)
        2. Cores: 2 (minimo para rodar o Juju)
        3. RAM: 4096 (4 gb mínimo para rodar o Juju)
        4. Storage configuration Size(GB): 50 (mínimo para rodar o Juju)
        5. Selecionar Compose Machine
    5. Com a máquina criada vamos adicionar uma tag para instalar o Juju (é necessário para o Juju encontrar a máquina onde vai ser instalado)
        1. Selecionar a maquina criada `juju-machine` na lista de VMs
        2. Na tela de configurações selecionar Configuration
        3. Na área onde está escrito Tags selecionar Edit
        4. Adicionar na barra que apareceu o nome `juju`
        5. Caso a tag não exista Selecionar a opção que deve aparecer Create tag: juju
        6. Selecionar na Caixa que apareceu Create and add to tag changes
    6. Aguardar o _compose_ da máquina, até o status dela aparecer como ready
20. Instalar o Juju $^7$:
    1. Verificar se a pasta local do juju existe, rodar o comando `sudo find ~/.local/share/juju`
        1. Se receber como resposta `find: ‘/home/ubuntu/.local/share/juju’: No such file or directory`, criar pasta
            1. Criar diretorio local do juju, rodar comando `sudo mkdir -p ~/.local/share/juju`
    1. No terminal instalar o Juju, rodar o comando `sudo snap install juju --channel 3.1`
    2. Criar um arquivo para adicionar uma nuvem ao juju
        1. No terminal criar um arquivo yaml, rodar o comando `touch juju-cloud.yaml`
        2. Editar o arquivo para configurar a nuvem, rodar o comando `sudo nano juju-cloud.yaml`
        3. Adicionar ao arquivo a seguinte configuração:
        ```yaml 
        clouds:
            maas-one:
                type: maas
                auth-types: [oauth1]
                endpoint: http://10.42.0.1:5240/MAAS
        ```
        4. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
    3. Adicionar a nuvem ao Juju, rodar o comando `juju add-cloud --client -f juju-cloud.yaml maas-one` (--client especifica controlador para operar e -f o arquivo, deve aparecer Success no terminal)
    4. Criar um arquivo para adicionar as credenciais ao Juju:
        1. Salvar a chave da api do MAAS.
            1. Na web UI do MAAS selecionar na barra lateral o seu perfil (é o que possui um simbolo de perfil do lado e seu nome de usuário)
            2. Selecionar a aba API KEYS
            3. Salvar/Copiar a chave que aparece ca coluna KEY
        2. No terminal criar um arquivo yaml, rodar o comando `touch juju-credentials.yaml`
        3. Editar o arquivo para configurar a nuvem, rodar o comando `sudo nano juju-credentials.yaml`
        4. Adicionar ao arquivo a seguinte configuração:
        ```yaml 
        credentials:
            maas-one:
                anyuser:
                auth-type: oauth1
                maas-oauth: {admin-api-key}
        ```
        (no lugar de {admin-api-key} colocar a chave da api do MAAS salva)

        5. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
    5. Adicionar a credencial ao Juju, rodar o comando `juju add-credential --client -f juju-credentials.yaml maas-one` (deve aparecer que o usuario _anyuser_ foi adicionado para a nuvem _maas-one_)
21. Resolver Configurações de pacotes de internet para instalação do Juju:
    1. Permitir encaminhamento de pacote da máquina.
        1. Editar configurações de encaminhamento, rodar o comando `sudo nano /etc/sysctl.conf`
        2. Alterar as linhas que estão comentadas para que fiquem da seguinte maneira (permitindo encaminhamento):
        ```conf
        net.ipv4.ip_forward=1
        net.ipv6.conf.all.forwarding=1
        ```
        3. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
    2. Aplicar alterações do arquivo `sysctl.conf`, rodar comando `sudo sysctl --system`
    3. Usar ntf para definir _nat rules_ $^8$
        1. Criar um arquivo com novas _nat rules_, rodar comando `touch nat.rules`
        2. Editar as configurações, rodar comando `sudo nano nat.rules`
        3. Alterar para que fique da seguinte maneira (definindo comunicações externas com a internet para que não utilize nosso ip privado, ao invés ser visto com o ip do stratus):
        ```
        table ip nat {
            chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                ip saddr 10.42.0.0/16 oif "eno2" snat to 200.18.99.88;
            }
        }
        ```
        4. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
    4. Aplicar novas _nat rules_, rodar comando `sudo nft -f nat.rules`
22. Realizar bootstrap do Juju:
    1. Garantir que execute em segundo plano, rodar o comando `tmux`
    2. Realizar bootstrap, rodar o comando `juju bootstrap --bootstrap-series=jammy --constraints tags=juju maas-one maas-controller` (constraints seleciona um nó do maas nesse caso buscando um com a tag juju, bootstrap-series define a _release serie_ do sistema operacional que vai ser instalado)
    3. Para sair da interface do tmux aperte: `Ctrl+B -> D`, (não apertar tudo junta, primeiro `Ctrl+B` depois de soltar aperta `D`)
23. Ao finalizar o bootstrap (é possível ver na web UI do MAAS na aba Machines se o nó do juju está como deployed) criar o modelo do openstack, rodar o comando `juju add-model --config default-series=jammy openstack`
24. Realizar deploy do bundle do openstack.
    1. Baixe ou clone o repositório do projeto (este que você está lendo agora) para o seu computador
    2. Copie o arquivo _bundle.yaml_ para o nó controller
        1. Pela UI 
            1. Abra o gerenciador de arquivos do linux e clique em outros locais.
            2. Conecte-se a `sftp://ubuntu@stratus.dc.ufscar.br:2002`
            3. Copie os arquivos _bundle.yaml_ que está em `Infra/Config-Files` para alguma pasta no controller 
        2. Ou pelo terminal
            1. Rode `sftp -P 2002 ubuntu@stratus.dc.ufscar.br`
            2. No contexto do SFTP, rode `put {Path do seu repositorio local}/Infra/Config-Files/bundle.yaml {path do controller}`
    3. No terminal realizar deploy, rodar comando `juju deploy bundle.yaml`
    4. Para ver o status do deploy, rodar o comando `juju status --color`
25. Ao terminar deploy do bundle realizar deploy do nova no nó compute.
    1. Criar o arquivo de deploy, rodar comando `touch nova-compute.yaml`
    2. adicionar as configurações necessárias, rodar comando `sudo nano nova-compute.yaml`
    3. Colocar a seguinte definição no arquivo:
    ```yaml
    nova-compute:
        config-flags: default_ephemeral_format=ext4
        enable-live-migration: true
        enable-resize: true
        migration-auth-type: ssh
        virt-type: qemu
    ```
    4. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
    1. Verificar o nó que possui o compute0, rodar `juju status --color`
    2. Verificar na coluna Inst id qual possui `compute0`
    3. Vericar da linha da unidade encontrada qual o valor na coluna `Machine`
    4. Realizar deploy, rodar o comando `juju deploy --to 10 --channel 2023.2/stable --config nova-compute.yaml nova-compute` (Esperado que seja o valor 10 na flag --to, mas colocar o número da máquina encontrada no passo anterior)
26. Ao terminar deploy do nova, realizar deploy do subordinado ovn-chassis.
    1. Criar o arquivo de deploy, rodar comando `touch ovn.yaml`
    2. adicionar as configurações necessárias, rodar comando `sudo nano ovn.yaml`
    3. Colocar a seguinte definição no arquivo:
    ```yaml
    ovn-chassis:
        bridge-interface-mappings: br-ex:enp1s0
        ovn-bridge-mappings: physnet1:br-ex
    ```
    4. Depois disso sair do nano apertando: `Ctrl+X -> Y -> Enter`
    5. Realizar deploy, rodar comando `juju deploy --channel 23.09/stable --config ovn.yaml ovn-chassis`
26. Realizar integrações do nova e do ovn-chassis $^9$:
    1. Realizar integrações do nova rodar a lista de comandos no terminal:
    - `juju integrate ovn-chassis:nova-compute nova-compute:neutron-plugin`
    - `juju integrate rabbitmq-server:amqp nova-compute:amqp`
    - `juju integrate nova-cloud-controller:cloud-compute nova-compute:cloud-compute`
    - `juju integrate glance:image-service nova-compute:image-service`
    2. Realizar integrações do ovn-chassis:
        - `juju integrate ovn-chassis:ovsdb ovn-central:ovsdb`
        - `juju integrate ovn-chassis:certificates vault:certificates`

27. Abrir vault server.
    1. Verificar ip da máquinda onde está o vault
        1. listar as unidades, rodar comando `juju status --color`
        2. Verificar na coluna Unit onde se encontra `vault/0*`
        3. Verificar o ip na coluna Public address da linha onde foi encontrado o vault
    2. Entrar na máquina onde está o vaulr, rodar comando `ssh -i ~/.local/share/juju/ssh/juju_id_rsa {ip_found}`, substituir ip_found pelo ip encontrado no passo anterior
    3. Definir endereço do vault, rodar comando `export VAULT_ADDR=http://127.0.0.1:8200`
    4. Iniciar vault server, rodar comando `vault operator init --key-shares=1 --key-threshold=1`
    5. Salvar a saída do comando em algum lugar que seja de fácil acesso, por exemplo um arquivo `vault.out`
    6. Desbloquear o servidor vault, rodar comando `vault operator unseal`
    7. Colocar a senha que se encontra na linha Unseal Key 1: do arquivo `vault.out`
    8. Sair da máquina do vault executando `Ctrl + D`
    9. Autorizar charm no servidor vault, rodar comando `juju run vault/leader authorize-charm token={token root}`, onde {token root} deve ser substituido pela chave que está na linha Initial Root Token do arquivo `vault.out`
    10. Gerar um root para o vault utilizando o juju, rodar comando `juju run vault/leader generate-root-ca`
28. Esperar todas as unidades estarem em estado `active` e Agent `idle`, rodar o comando `watch -c juju status --color` e esperar.
29. Seu openstack deve estar completamente configurado :).
30. Acesse a web UI do openstack
    1. Descubra o ip de acesso para a web UI, rodar o comando `juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}' | head -1`
    2. Gere a senha de acesso de admin, rodar o comando `juju exec --unit keystone/leader leader-get admin_passwd` 
    3. Copie essa senha para um local de fácil acesso
    4. Abra um novo terminal e crie um acesso ao ip via ssh, rodar o comando `ssh -L 8443:10.42.0.17:80 -p 2002  ubuntu@stratus.dc.ufscar.br` 
    5. Acesse a web UI pelo browser de preferência no link `localhost:8443/horizon`
    6. Faça login usando: 
    ```json
    username: admin 
    password: {salva do passo 2}
    domain: admin_domain
    
    ```




### Links Úteis

1. $\quad$ [Tunneling](https://www.ssh.com/academy/ssh/tunneling-example)
2. $\quad$ [Machine's life-cycle](https://maas.io/docs/machines#heading--the-different-states-of-a-machine)
3. $\quad$ [Proxy Servers](https://discourse.maas.io/t/proxy/763)
4. $\quad$ [Bridge explained](https://study-ccna.com/network-bridge-explained/)
5. $\quad$ [Connecting MAAS Networks](https://maas.io/docs/connecting-maas-networks)
6. $\quad$ [LXD for VMs](https://maas.io/docs/setting-up-lxd-for-vms)
7. $\quad$ [Juju install](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-juju.html)
8. $\quad$ [Nat rules](https://cloud.google.com/nat/docs/nat-rules-overview#:~:text=A%20NAT%20rule%20defines%20a,corresponding%20to%20that%20match%20occurs.)
9. $\quad$ [Openstack deploy](https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html)

TODO: Testar Integrações
TODO: ufw?