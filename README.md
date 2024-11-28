# Monitoramento do Asterisk e Khomp EBS através do Zabbix com SNMP

Templates Zabbix para monitoramento SNMP do Asterisk e Khomp EBS  
Mantenedor: Igor Silva  
E-mail: igor@ultrix.pro   
Estado: Finalizado  
Data: 20240819  

(c) Copyright 2024, Ultrix  

# Introdução
O template para monitoramento do Khomp EBS foi criado a partir dos seguintes manuais, fornecidos pela Khomp:

**KQueryServer - Manual do usuario  
Última atualização: 2022-12-21 16:51:47**  

**Códigos de Descrição e Estado da K3L  
Última atualização: 2023-05-31 19:55:41**  

Para fazer o monitoramento via SNMP **Asterisk** e **Khomp** na sua máquina disponibilizamos dois Templates Zabbix (TemplateAsterisk.json, TemplateKhomp.json) , para utiliza-los é necessário:  
  
- Instalação do SNMP  
   - Configuração do SNMP  
   - Configuração do SNMP Asterisk  
- MIBS Khomp - Asterisk  
   - Acesso as MIBS  
- Configuração do Zabbix    
   - Hosts  
   - Templates  

    
## Instalação do SNMP

**Configuração do SNMP**

O processo de instalação do SNMP e suas ferramentas no seu sistema varia de acordo com o sistema operacional que você está usando.  

**SUSE/OpenSUSE**:
instala os pacotes necessários para SNMP e suas ferramentas  

`sudo zypper install net-snmp net-snmp-utils`  

**Configuração SNMP**

Para realizar o processo de configuração, siga os passos abaixo:   

Entre no diretório de configuração do SNMP:  
    
 `cd /etc/snmp`  
  

Abra o arquivo de configuração `snmpd.conf` com o editor `vim`:  

  
 `vim snmpd.conf`  
  

Para iniciar o processo:  
inicia o serviço SNMP  

`sudo systemctl start snmpd`  
                 
configura o serviço SNMP para iniciar automaticamente durante a inicialização do sistema  

`sudo systemctl enable snmpd`  
             

Adicione ou modifique as seguintes linhas no arquivo `snmpd.conf`:  
``` conf  
    com2sec readonly  172.28.17.7 public
                     # define a comunidade e o endereço IP para leitura

    group MyROGroup v1         readonly
    group MyROGroup v2c        readonly
                     # define grupos de acesso para SNMPv1 e SNMPv2c

    view    all           included   .1
                     # inclui todas as MIBs para visualização

    #group     context   sec.model sec.level prefix read  write notif
    access MyROGroup ""        any       noauth    exact  all   none  none
                     # define as permissões de acesso para o grupo MyROGroup

    proxy -v 2c -c khomp localhost:14161 .1.3.6.1.4.1.32624
                     # define o proxy SNMP, escutando na porta 14161 e encaminhando para o OID especificado

    syslocation Ultrix
    syscontact Sysadmin (root@localhost)
                     # define a localização do sistema e o contato

    dontLogTCPWrappersConnects yes
                     # evita logar conexões TCP wrappers

    master agentx
    agentXPerms 0660 0550 nobody nobody
                     # configura as permissões para o protocolo AgentX
```

 
  Após as alterações, reinicie o serviço SNMP para aplicar as mudanças:
 `systemctl restart snmpd.service`


Nesta configuração SNMP, criamos um grupo somente leitura que se conecta a um dispositivo exclusivo que possui um endereço IP 172.28.17.7. A chave public é usada para proteger a conexão com este dispositivo. Os membros da comunidade só podem monitorar o que está acontecendo e não alterar as configurações.

A seguir, definimos um grupo de acesso chamado MyROGroup que usa as versões do protocolo SNMP v1 e v2c. Este grupo traz ferramentas para o grupo de leitura, permitindo-lhes acessar informações mas ler.

Para melhorar a acessibilidade, definimos todas as visualizações, que permitem ver mais informações dentro do dispositivo. A configuração também inclui regras de acesso, onde sabemos que o grupo MyROGroup pode ver tudo, mas não pode modificar ou receber notificações.

O comando proxy `proxy -v 2c -c khomp localhost:14161 .1.3.6.1.4.1.32624` redireciona as solicitações SNMP recebidas da porta 14161 para a porta SNMP (ou seja, 161) Além de ser responsável pelo monitoramento do Khomp.Isso permite que o servidor SNMP local atue como intermediário, enviando consultas SNMP recebidas em portas que estão conectadas às portas SNMP padrão, coletando informações do dispositivo de destino e retornando as informações. 

Também o comando `master agentx agentxPerms 0660 0550 nobody nobody` tem a funcionalidade de monitoramento do Asterisk.

Além disso, esta configuração contém informações administrativas como o local onde o dispositivo está definido como “localhost” e o suporte (Sysadmin (root@localhost)) se necessário.

No final, foram corrigidas opções avançadas, como optar por não registrar algumas conexões (dontLogTCPWrappersConnects sim) e configurar SNMP para o agente mestre (agente mestre x). Isso faz com que o sistema SNMP seja confiável e capaz de gerenciar vários dispositivos de maneira sistemática e segura.



**Configuração do SNMP Asterisk**

Edite o arquivo de configuração Asterisk res_snmp.conf para ativar a integração SNMP.
 `vi /etc/asterisk/res_snmp.conf`

 Adicione as seguintes linhas ao final do arquivo res_snmp.conf.
 ```
 subagent = yes
 enabled = yes
 ```

Reinicie o serviço Asterisk.

`service asterisk restart`

Conecte-se ao console do Asterisk e verifique se o módulo SNMP do Asterisk foi carregado.
```
asterisk -rv
module show like snmp
 ```

Em nosso exemplo, podemos ver que o módulo SNMP do Asterisk foi carregado.
```
Module             Description                    Use Count  Status      Support Level
res_snmp.so        SNMP [Sub]Agent for Asterisk   0          Running     extended
1 modules loaded
```

Para testar sua configuração SNMP do Asterisk, use o seguinte comando.
`snmpwalk -v2c -c public 127.0.0.1 1.3.6.1.4.1.22736.1`

Você deve ver uma saída SNMP:
```
iso.3.6.1.4.1.22736.1.1.1.0 = STRING: "16.5.1"
iso.3.6.1.4.1.22736.1.1.2.0 = Gauge32: 160501
iso.3.6.1.4.1.22736.1.2.1.0 = Timeticks: (51023) 0:08:30.23
iso.3.6.1.4.1.22736.1.2.2.0 = Timeticks: (51023) 0:08:30.23
iso.3.6.1.4.1.22736.1.2.3.0 = INTEGER: 8179
iso.3.6.1.4.1.22736.1.2.4.0 = STRING: "https://d1ny9casiyy5u5.cloudfront.net/var/run/asterisk/asterisk.ctl"
iso.3.6.1.4.1.22736.1.2.5.0 = Gauge32: 0
iso.3.6.1.4.1.22736.1.2.6.0 = Counter32: 0
iso.3.6.1.4.1.22736.1.3.1.0 = INTEGER: 338
iso.3.6.1.4.1.22736.1.4.1.0 = INTEGER: 40
iso.3.6.1.4.1.22736.1.4.2.0 = STRING: "us"
```

## MIBS

 Cada objeto na MIB tem um identificador único chamado OID, que é uma sequência de números separados por pontos (por exemplo, 1.3.6.1.2.1.2.2.1.1). Os OIDs são usados para acessar os dados específicos 
 no dispositivo.

**Acesso as MIBS**

Para acessar a MIB Khomp e Asterisk digite o comando:

`cd /usr/share/snmp/mibs`

Dando o comando `ls` podera vizualizar todas as MIBS disponíveis

Você deve ter uma saída:
```
ASTERISK-MIB 
KHOMP-MIB
```

Para vizualizar a MIB do Khomp

`cd /etc/khomp`

Realizando o comando `ls` podera vizualizar a sua KHOMP-MIB

### Configuração Zabbix

**Hosts**

Passo 1: Acessar a Interface do Zabbix
Abra um navegador web e acesse a interface web do Zabbix.
Faça login com suas credenciais de administrador.

Passo 2: Adicionar um Novo Host
Vá para a seção "Configuration":

No menu superior, clique em "Configuration" e depois em "Hosts".
Clique em "Create host":

Na parte superior direita da página "Hosts", clique no botão "Create host".
Passo 3: Configurar as Informações Básicas do Host
Preencha os Detalhes Básicos:

Host name: Insira o nome do host.
Visible name (opcional): Um nome mais amigável para exibição.
Selecione o Grupo:

Na seção "Groups", escolha um grupo de hosts existente ou crie um novo grupo clicando em "Select" e depois em "Create new group".
Adicione um Interface SNMP:

Na seção "Interfaces", clique em "Add".
Type: Selecione "SNMP".
IP address: Insira o endereço IP do dispositivo que você está monitorando.
Port: Insira "161" (porta padrão para SNMP).
DNS name (opcional): Insira um nome DNS se aplicável.
Connect to: Selecione "Agent" se não estiver visível na lista.
Defina a Comunidade SNMP

Na seção "SNMP Interfaces", você não configurará diretamente a comunidade aqui. Em vez disso, você configurará o Item mais tarde com a comunidade apropriada.

**Templates**

Atribuir um Template SNMP:

Vá para a Seção de Importação de Templates:

No menu superior, clique em "Configuration" e depois em "Templates".
Importar um Template:

Clique no botão "Import" no canto superior direito.
Selecionar o Arquivo de Template:

Na página de importação, clique em "Choose File" e selecione o arquivo JSON do template SNMP que você deseja importar.

Configurar Importação:

Options: Escolha as opções apropriadas para sua importação. As opções padrão geralmente são suficientes.
Templates: Confirme que o template SNMP desejado está selecionado.
Clique em "Import" para adicionar o template ao Zabbix.

Na seção "Templates", clique em "Link new templates".
Na caixa de pesquisa, procure por "Template SNMP" ou "Template App SNMP". Dependendo da sua configuração, você pode ter diferentes templates SNMP pré-configurados.
Selecione um template apropriado (por exemplo, Template App SNMP Generic) Clique em "Add" para adicionar o template ao seu host.

                       
                        




