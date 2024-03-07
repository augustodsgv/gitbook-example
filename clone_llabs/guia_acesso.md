## Obtendo Acesso ao Controlador
1. Certifique que você possui chaves SSH registradas no GitHub ([aqui](https://github.com/settings/keys)).
2. Entre na organização Pesquisa-Cloud-UFSCar.
    - Você já está nela se você consegue ler isso.
3. Peça para o Miguel ou o administrador atual registrar as suas chaves.
    - NEW: Colocamos o comando em um crontab rodando uma vez por hora. Agora basta esperar e ir tomar um chá ou café.
    - NEW NEW: Você pode adicionar usuários fora da organização:
        1. Adicione o username no arquivo `/usr/local/etc/ghacct.yaml`
        2. Rode o comando `sudo ghacct`
4. Desligue a VPN da magalu :^)
5. Acesse o controlador usando `ssh -p 2002 USERNAME@cirrus.dc.ufscar.br`, onde `USERNAME` é o seu nome de usuário no GitHub.
    - Se for o seu primeiro login, você vai precisar definir uma senha.

## Obtendo Acesso à VPN
~~O processo de registro na VPN ainda é bem manual, não tive tempo de tentar montar um script :(.~~

Boas notícias: Eu montei um script :) são 5h42 da manhã e eu não dormi.

1. Faça login por ssh no controlador.
2. Use o comando `sudo wgtemplatefill` para gerar as configurações.
3. Coloque a sua seção de configuração no fim do `/etc/wireguard/wg0.conf`
4. Sincronize a configuração usando `sudo wg syncconf wg0 /etc/wireguard/wg0.conf`
5. Instale a configuração do seu cliente na sua máquina.
    1. Baixe o pacote de ferramentas do WireGuard com `sudo apt install wireguard-tools`
    2. Salve a configuração do cliente em um arquivo, por exemplo, `wg0.conf`
    3. Levante a interface usando `sudo wg-quick up wg0.conf`

## Obtendo Acesso ao Dashboard
1. Faça login por ssh no controlador.
2. Obtenha as credenciais de administração usando `sudo cat /srv/keystone_admin.txt`
3. Faça login no dashboard em `https://10.84.193.8/horizon/`
4. **CRIE UM USUÁRIO SEU PARA USO GERAL:**
    1. **NÓS NÃO ESTAMOS USANDO O PROJETO DE ADMINISTRAÇÃO PARA TAREFAS QUE NÃO SÃO DE ADMINISTRAÇÃO**
    2. Em Identidade > Domínios, selecione o domínio dc como contexto, ou crie outro domínio seu.
    3. Em Identidade > Usuários, crie um novo usuário.
        - Você pode usar o projeto dc como projeto seu, ou criar outro projeto.
5. Faça login com o seu usuário.

## Obtendo Acesso ao `openstack-cli`
1. Faça login por ssh no controlador.
2. Obtenha o certificado de CA do OpenStack usando `cat /srv/openstack.crt`
3. Salve o texto do certificado de CA em uma pasta apropriada (no controlador ou na sua máquina pessoal).
4. Faça login com o seu usuário no Dashboard.
5. Na sua conta, baixe o Arquivo OpenStack RC para uma pasta apropriada (no controlador ou na sua máquina pessoal).
6. Carregue as variáveis de ambiente usando `source caminho/do/arquivo/rc.sh`
7. Especifique o caminho do certificado de CA usando `export OS_CACERT=caminho/do/certificado/openstack.crt`