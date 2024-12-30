# Monitoramento do Asterisk e Khomp EBS através do Zabbix com SNMP

Templates Zabbix para monitoramento SNMP do Asterisk e Khomp EBS  

Estado: Em produção   
Data: 2021227  

(c) Copyright 2024, Ultrix  

# Introdução
O template para monitoramento do Khomp EBS foi criado a partir dos manuais 
abaixo, fornecidos pela Khomp. Para ter acesso a essa documentação solicite
através do www.khomp.com.br.

**KQueryServer - Manual do usuario**  
Última atualização: 2022-12-21 16:51:47  

**Códigos de Descrição e Estado da K3L**  
Última atualização: 2023-05-31 19:55:41  

Para fazer o monitoramento via SNMP Asterisk e Khomp disponibilizamos aqui dois 
Templates Zabbix:  

- Zabbix_Template_Asterisk.json
- Zabbix_Template_Khomp.json

Para utilizá-los é necessário:  

- [Instalar o SNMP](#instalar-o-snmp)  
   - [Configurar o SNMP](#configurar-o-snmp)  
   - [Configurar o SNMP do Asterisk](#configurar-o-snmp-do-asterisk)
- [Instalar as MIBS Khomp e Asterisk](#instalar-as-mibs-khomp-e-asterisk)
- [Configurar o Zabbix](#configurar-o-zabbix)
   - [Importar os templates](#importar-os-templates)
    
## Instalar o SNMP

O processo de instalação do SNMP e de suas ferramentas no seu sistema varia 
conforme o sistema operacional utilizado.  

**SUSE/OpenSUSE**:
instala os pacotes necessários ao SNMP:  

`sudo zypper install net-snmp net-snmp-utils`  

### Configurar o SNMP

Para realizar o processo de configuração, siga os passos abaixo:   

Edite o arquivo `/etc/snmp/snmpd.conf` com o editor de texto de sua preferência.

Exemplo de configuração:

``` conf
# Define a SNMP community, IPs permitidos de origem e demais configurações 
# de grupos de acesso. Em geral não é uma boa ideia usar "public" como SNMP 
# community, nem permitir acesso a partir de qualquer IP.
com2sec readonly 192.168.1.1 763190907afc1b132
com2sec readonly 10.15.1.1 3a66818a03d84dbc

group MyROGroup v1         readonly
group MyROGroup v2c        readonly

# inclui todas as MIBs
view    all           included   .1

# Permissões de acesso para o grupo MyROGroup
access MyROGroup ""      any       noauth    exact  all    none   none

# Proxy SNMP, escutando na porta 14161 e encaminhando para o OID especificado
# Essa configuração é necessária para monitoramento do Khomp
proxy -v 2c -c khomp localhost:14161 .1.3.6.1.4.1.32624

syslocation Quixeramobim/CE
syscontact suporte@ultrix.srv.br 

# Desabilita log de conexões aceitas
dontLogTCPWrappersConnects yes

# O agentx é necessário para utilização do SNMP do Asterisk
master agentx
agentXPerms 0660 0550 nobody nobody

# Processos monitorados
proc asterisk
proc mysqld
proc tincd
proc sshd

# Extensão do SNMP com scripts personalizados
extend teste1_extensao /bin/echo "Saida extensao 1"
extend teste2_extensao /bin/echo "Saida extensao 2"
```

Para habilitar e iniciar o serviço SNMP:
```
sudo systemctl start snmpd 

sudo systemctl enable snmpd
```           

Nesta configuração SNMP criamos um grupo somente-leitura que permite a
conexão dispositivos cujos endereços IP estejam listados. A SNMP community é 
usada para proteger a conexão.

A seguir, definimos um grupo de acesso chamado MyROGroup que usa as versões do 
protocolo SNMP v1 e v2c. Este grupo traz ferramentas para o grupo de leitura, 
permitindo-lhes acessar as informações.

A entrada "proxy" redireciona as solicitações SNMP recebidas da porta 14161 
utilizada pelo driver da Khomp para a porta SNMP 161 padrão. Isso permite que o 
servidor SNMP local atue como intermediário, enviando consultas SNMP recebidas 
em portas que estão conectadas às portas SNMP padrão, coletando informações do 
dispositivo de destino e retornando as informações. 

O agentx é necessário ao monitoramento SNMP do Asterisk.

O parâmetro `extend` permite estender as funcionalidades do SNMP padrão através 
da execução de scripts personalizados. Para ter acesso às informações geradas 
pelas extensões criadas é necessário consultar as tabelas
`NET-SNMP-EXTEND-MIB::nsExtendOutput1Table` e 
`NET-SNMP-EXTEND-MIB::nsExtendOutput2Table`:

```
$ snmpwalk -v2c -c 763190907afc1b132 192.168.1.1 NET-SNMP-EXTEND-MIB::nsExtendOutput1Table

NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."teste1_extensao" = STRING: Saida extensao 1
NET-SNMP-EXTEND-MIB::nsExtendOutput1Line."teste2_extensao" = STRING: Saida extensao 2
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."teste1_extensao" = STRING: Saida extensao 1
NET-SNMP-EXTEND-MIB::nsExtendOutputFull."teste2_extensao" = STRING: Saida extensao 2
NET-SNMP-EXTEND-MIB::nsExtendOutNumLines."teste1_extensao" = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendOutNumLines."teste2_extensao" = INTEGER: 1
NET-SNMP-EXTEND-MIB::nsExtendResult."teste1_extensao" = INTEGER: 0
NET-SNMP-EXTEND-MIB::nsExtendResult."teste2_extensao" = INTEGER: 0

$ snmpwalk -v2c -c 763190907afc1b132 192.168.1.1 NET-SNMP-EXTEND-MIB::nsExtendOutput2Table

NET-SNMP-EXTEND-MIB::nsExtendOutLine."teste1_extensao".1 = STRING: Saida extensao 1
NET-SNMP-EXTEND-MIB::nsExtendOutLine."teste2_extensao".1 = STRING: Saida extensao 2
```

### Configurar o SNMP do Asterisk

Para que o SNMP do Asterisk funcione é necessário que ele tenha sido compilado 
com essa fucionalidade habilitada.

Edite o arquivo `/etc/asterisk/res_snmp.conf` e adicione as seguintes linhas ao final do arquivo:
```
subagent = yes
enabled = yes
```

Reinicie o serviço Asterisk:
```
service asterisk restart
```

Verifique se o módulo SNMP do Asterisk foi carregado:
```
$ sudo asterisk -rx "module show like snmp"

Module             Description                    Use Count  Status      Support Level
res_snmp.so        SNMP [Sub]Agent for Asterisk   0          Running     extended
1 modules loaded
```

Para testar use:
```
$ snmpwalk -v2c -c community host 1.3.6.1.4.1.22736.1
```
O resultado de ser parecido com:
```
iso.3.6.1.4.1.22736.1.1.1.0 = STRING: "16.5.1"
iso.3.6.1.4.1.22736.1.1.2.0 = Gauge32: 160501
iso.3.6.1.4.1.22736.1.2.1.0 = Timeticks: (51023) 0:08:30.23
iso.3.6.1.4.1.22736.1.2.2.0 = Timeticks: (51023) 0:08:30.23
iso.3.6.1.4.1.22736.1.2.3.0 = INTEGER: 8179
iso.3.6.1.4.1.22736.1.2.4.0 = STRING: "https://d1ny9casiyy5u5.cloudfront.net/var/run/asterisk/asterisk.ctl"
iso.3.6.1.4.1.22736.1.2.5.0 = Gauge32: 0
```

## Instalar as MIBS Khomp e Asterisk

Cada objeto na MIB tem um identificador único chamado OID, que é uma sequência 
de números separados por pontos (por exemplo 1.3.6.1.2.1.2.2.1.1). Os OIDs são 
usados para acessar os dados específicos no dispositivo.

**Acesso às MIBS**

As MIBs são normalmente armazenadas no diretório `/usr/share/snmp/mibs`.

Os arquivos de MIB `ASTERISK-MIB` e `KHOMP-MIB` devem ser copiados para o 
diretório `/usr/share/snmp/mibs` do servidor de monitoramento.

Normalmente o arquivo `KHOMP-MIB` fica em `/etc/khomp` após a instalação do 
driver.

## Configurar o Zabbix

**Hosts**

Ao adicionar um host configure o acesso via SNMP:

Na seção "Interfaces":
- Type: Selecione "SNMP".
- IP address: Insira o endereço IP do dispositivo a ser monitorado
- DNS name: insira um nome DNS, se houver
- Connect to: selecione IP ou DNS, de acordo com o que foi preenchido acima
- Port: Insira "161" (porta padrão para SNMP)
- SNMP version: SNMPv2
- SNMP community: insira a comunidade SNMP configurada no arquivo `snmpd.conf` 
do host a ser monitorado
- Use bulk requests: deve ficar **desabilitado**, caso contrário havará 
problemas na consulta do SNMP da Khomp

Em geral é conveniente configurar a comunidade como uma macro de abrangência 
global do Zabbix.

### Importar os templates

Dentro do Zabbix clique no botão "Import" no canto superior direito.

Selecione o arquivo JSON do template a ser importado. Neste projeto estão 
disponíveis os templates do Asterisk e da Khomp.

Configurar Importação:

- Options: Escolha as opções apropriadas para sua importação. As opções padrão 
geralmente são suficientes.
- Templates: Confirme que o template SNMP desejado esteja selecionado
- Clique em "Import" para adicionar o template ao Zabbix

O template Asterisk será importado com o nome `TPL SNMP Asterisk`.

O template Khomp será importado com o nome `TPL SNMP Khomp`.

Após a importação faça a associação dos templates aos hosts criados. Alguns 
minutos depois os itens começarão a ser coletados.


                       
                        




