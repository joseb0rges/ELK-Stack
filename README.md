![Texto Alternativo](https://images.contentstack.io/v3/assets/bltefdd0b53724fa2ce/blt6e99912c868e1fe8/5cfaecd5c148e2da2c69b812/Enable-security-1.jpg)

### Configuração de TLS para Elasticsearch, Kibana e Logstash

#### Etapa 1: Configurar o arquivo `/etc/hosts`

##### No node1:
```plaintext
127.0.0.1 kibana.local logstash.local
192.168.0.2 node1.elastic.test.com node1
192.168.0.3 node2.elastic.test.com node2
```

##### No node2:
```plaintext
192.168.0.2 node1.elastic.test.com node1
192.168.0.3 node2.elastic.test.com node2
```

#### Etapa 2: Criar certificados SSL e habilitar TLS para Elasticsearch no node1

##### 2.1: Definir variáveis de ambiente
```sh
ES_HOME=/usr/share/elasticsearch
ES_PATH_CONF=/etc/elasticsearch
```

##### 2.2: Criar diretórios temporários
```sh
mkdir -p ~/tmp/cert_blog
```

##### 2.3: Criar arquivo `instance.yml`
```yaml
instances:
  - name: 'node1'
    dns: [ 'node1.elastic.test.com' ]
  - name: 'node2'
    dns: [ 'node2.elastic.test.com' ]
  - name: 'my-kibana'
    dns: [ 'kibana.local' ]
  - name: 'logstash'
    dns: [ 'logstash.local' ]
```

##### 2.4: Gerar certificados CA e de servidor
```sh
cd $ES_HOME
bin/elasticsearch-certutil cert --keep-ca-key --pem --in ~/tmp/cert_blog/instance.yml --out ~/tmp/cert_blog/certs.zip
```

##### 2.5: Descompactar certificados
```sh
cd ~/tmp/cert_blog
unzip certs.zip -d ./certs
```

##### 2.6: Instalar TLS no Elasticsearch

###### 2.6.1: Copiar certificados para a pasta de configuração
```sh
cd $ES_PATH_CONF
mkdir certs
cp ~/tmp/cert_blog/certs/ca/ca* ~/tmp/cert_blog/certs/node1/* certs
```

###### 2.6.2: Configurar `elasticsearch.yml`
```yaml
node.name: node1
network.host: node1.elastic.test.com
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.key: certs/node1.key
xpack.security.http.ssl.certificate: certs/node1.crt
xpack.security.http.ssl.certificate_authorities: certs/ca.crt
xpack.security.transport.ssl.key: certs/node1.key
xpack.security.transport.ssl.certificate: certs/node1.crt
xpack.security.transport.ssl.certificate_authorities: certs/ca.crt
discovery.seed_hosts: [ "node1.elastic.test.com" ]
cluster.initial_master_nodes: [ "node1" ]
```

###### 2.6.3: Iniciar e verificar log do cluster
```sh
systemctl start elasticsearch
grep '\[node1\] started' /var/log/elasticsearch/elasticsearch.log
```

###### 2.6.4: Definir senha de usuários integrados
```sh
cd $ES_HOME
bin/elasticsearch-setup-passwords auto -u "https://node1.elastic.test.com:9200"
```

###### 2.6.5: Acessar API `_cat/nodes` via HTTPS
```sh
curl --cacert ~/tmp/cert_blog/certs/ca/ca.crt -u elastic 'https://node1.elastic.test.com:9200/_cat/nodes?v'
```

#### Etapa 3: Habilitar TLS para Kibana no node1

##### 3.1: Definir variáveis de ambiente
```sh
KIBANA_HOME=/usr/share/kibana
KIBANA_PATH_CONFIG=/etc/kibana
```

##### 3.2: Criar pasta de configuração e copiar certificados
```sh
mkdir -p /etc/kibana/config/certs
cp ~/tmp/cert_blog/certs/ca/ca.crt ~/tmp/cert_blog/certs/my-kibana.crt ~/tmp/cert_blog/certs/my-kibana.key /etc/kibana/config/certs
```

##### 3.3: Configurar `kibana.yml`
```yaml
server.name: "my-kibana"
server.host: "kibana.local"
server.ssl.enabled: true
server.ssl.certificate: /etc/kibana/config/certs/my-kibana.crt
server.ssl.key: /etc/kibana/config/certs/my-kibana.key
elasticsearch.hosts: ["https://node1.elastic.test.com:9200"]
elasticsearch.username: "kibana"
elasticsearch.password: "<kibana_password>"
elasticsearch.ssl.certificateAuthorities: [ "/etc/kibana/config/certs/ca.crt" ]
```

##### 3.4: Iniciar Kibana e testar login
Acesse `https://kibana.local:5601/` em um navegador e faça login com o usuário `elastic` e a senha definida anteriormente.

#### Etapa 4: Habilitar TLS para Elasticsearch no node2

##### 4.1: Definir variáveis de ambiente
```sh
ES_HOME=/usr/share/elasticsearch
ES_PATH_CONF=/etc/elasticsearch
```

##### 4.2: Copiar certificados do node1 para node2
```sh
scp node1:/etc/elasticsearch/certs/ca.crt /etc/elasticsearch/certs/
scp node1:/etc/elasticsearch/certs/node2.* /etc/elasticsearch/certs/
```

##### 4.3: Configurar `elasticsearch.yml`
```yaml
node.name: node2
network.host: node2.elastic.test.com
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.key: certs/node2.key
xpack.security.http.ssl.certificate: certs/node2.crt
xpack.security.http.ssl.certificate_authorities: certs/ca.crt
xpack.security.transport.ssl.key: certs/node2.key
xpack.security.transport.ssl.certificate: certs/node2.crt
xpack.security.transport.ssl.certificate_authorities: certs/ca.crt
discovery.seed_hosts: [ "node1.elastic.test.com" ]
```

##### 4.4: Iniciar e verificar log do cluster
```sh
systemctl start elasticsearch
grep '\[node2\] started' /var/log/elasticsearch/elasticsearch.log
```

##### 4.5: Acessar API `_cat/nodes` via HTTPS
```sh
curl --cacert ~/tmp/cert_blog/certs/ca/ca.crt -u elastic:<password set previously> 'https://node2.elastic.test.com:9200/_cat/nodes?v'
```

#### Etapa 5: Preparar usuários do Logstash no node1

##### 5.1: Criar função `logstash_write_role`
```json
POST /_security/role/logstash_write_role
{
    "cluster": [
      "monitor",
      "manage_index_templates"
    ],
    "indices": [
      {
        "names": [
          "logstash*"
        ],
        "privileges": [
          "write",
          "create_index"
        ],
        "field_security": {
          "grant": [
            "*"
          ]
        }
      }
    ]
}
```

##### 5.2: Criar usuário `logstash_writer`
```json
POST /_security/user/logstash_writer
{
  "username": "logstash_writer",
  "roles": [
    "logstash_write_role"
  ],
  "password": "<logstash_system_password>",
  "enabled": true
}
```

#### Etapa 6: Habilitar TLS para Logstash no node1

##### 6.1: Criar pasta e copiar certificados
```sh
mkdir -p /etc/logstash/config/certs
cp ~/tmp/cert_blog/certs/ca/ca.crt ~/tmp/cert_blog/certs/logstash.crt ~/tmp/cert_blog/certs/logstash.key /etc/logstash/config/certs
```

##### 6.2: Converter `logstash.key` para PKCS#8
```sh
openssl pkcs8 -in /etc/logstash/config/certs/logstash.key -topk8 -nocrypt -out /etc/logstash/config/certs/logstash.pkcs8.key
```

##### 6.3: Configurar `logstash.yml`
```yaml
node.name: logstash.local
path.config: /etc/logstash/conf.d/*.conf
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.username: logstash_system
xpack.monitoring.elasticsearch.password: '<logstash_system_password>'
xpack.monitoring.elasticsearch.hosts: [ 'https://node1.elastic.test.com:9200' ]
xpack.monitoring.elasticsearch.ssl.certificate_authority: /etc/logstash/config/certs/ca.crt
```

##### 6.4: Criar e configurar `conf.d/example.conf`
```plaintext
input {
  beats {
    port => 5044
    ssl => true
    ssl_key => '/etc/logstash/config/certs/logstash.pkcs8.key'
    ssl_certificate => '/etc/logstash/config/certs/logstash.crt'
  }
}
output {
  elasticsearch {
    hosts => ["https://node1.elastic.test.com:920

0","https://node2.elastic.test.com:9200"]
    cacert => '/etc/logstash/config/certs/ca.crt'
    user => 'logstash_writer'
    password => '<logstash_writer_password>'
  }
}
```

##### 6.5: Iniciar Logstash e verificar logs
```sh
systemctl start logstash
systemctl enable logstash
```

#### Etapa 7: Executar Filebeat e configurar TLS no node1

##### 7.1: Criar pasta de configuração e copiar certificados
```sh
mkdir -p /etc/filebeat/config/certs
cp ~/tmp/cert_blog/certs/ca/ca.crt /etc/filebeat/config/certs
```

##### 7.2: Criar novo arquivo `filebeat.yml`
```yaml
filebeat.inputs:
- type: log
  paths:
    - /etc/filebeat/logstash-tutorial-dataset
output.logstash:
  hosts: ["logstash.local:5044"]
  ssl.certificate_authorities:
    - /etc/filebeat/config/certs/ca.crt
```

#### Etapa 8: Usar Filebeat para ingerir dados

##### 8.1: Preparar dados de log de entrada
```sh
mkdir -p /etc/filebeat/logstash-tutorial-dataset
cp /root/logstash-tutorial.log /etc/filebeat/logstash-tutorial-dataset/
```

##### 8.2: Iniciar Filebeat
```sh
systemctl start filebeat
systemctl enable filebeat
```

##### 8.3: Verificar logs
```sh
journalctl -u filebeat
```

##### 8.4: Criar padrão de indexação no Kibana
Acesse Kibana e crie um padrão de índice que corresponda aos dados sendo ingeridos, utilizando o campo `@timestamp` como filtro de tempo.

Com essas etapas, você terá configurado o Elasticsearch, Kibana e Logstash com TLS habilitado e Filebeat ingerindo dados de maneira segura.
