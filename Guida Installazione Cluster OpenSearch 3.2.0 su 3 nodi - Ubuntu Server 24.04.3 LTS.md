Guida Installazione Cluster OpenSearch 3.2.0 su 3 nodi - Ubuntu Server 24.04.3 LTS
****************************************************************************************************************
Data guida: gennaio 2026 by Sandro Sabbioni

Versione: OpenSearch 3.2.0 (rilasciata 19 agosto 2025)

Nodi: opensearch1 → 192.168.0.31    opensearch2 → 192.168.0.32    opensearch3 → 192.168.0.33

Firewall: disabilitato
Utente: utente normale con sudo

Password admin iniziale: Temporanea.123!

Certificati DN: C=IT / ST=Lazio / L=Roma / O=Leonardo / CN=...
Heap JVM: 8 GB (consigliato con 16 GB RAM totali)

ESEGUI SU TUTTI E 3 I NODI (opensearch1, opensearch2, opensearch3):

sudo apt update && sudo apt upgrade -y
sudo apt install -y lsb-release ca-certificates curl gnupg unzip

# Disabilita swap permanentemente
sudo swapoff -a
sudo sed -i '/swap/s/^/#/' /etc/fstab

# Limite memoria per OpenSearch
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Aggiungi repository OpenSearch 3.x (compatibile con 3.2.0)
curl -fsSL https://artifacts.opensearch.org/publickeys/opensearch-release.pgp | sudo gpg --dearmor -o /etc/apt/keyrings/opensearch.gpg

echo "deb [signed-by=/etc/apt/keyrings/opensearch.gpg] https://artifacts.opensearch.org/releases/bundle/opensearch/3.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-3.x.list

sudo apt update

# Installa ESATTAMENTE la versione 3.2.0 con password scelta
sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD='Temporanea.123!' apt install opensearch=3.2.0 -y

# Se la versione non è disponibile via apt, scarica manualmente il .deb (x64 / amd64):

# wget https://artifacts.opensearch.org/releases/bundle/opensearch/3.2.0/opensearch-3.2.0-linux-x64.deb
# sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD='Temporanea.123!' dpkg -i opensearch-3.2.0-linux-x64.deb
# sudo apt install -f   # risolve eventuali dipendenze mancanti

# Imposta heap JVM a 8 GB
sudo sed -i 's/-Xms1g/-Xms8g/' /etc/opensearch/jvm.options
sudo sed -i 's/-Xmx1g/-Xmx8g/' /etc/opensearch/jvm.options

******************************************************************************************************************
SOLO SU OPENSEARCH1 → Generazione certificati self-signed:

mkdir -p ~/opensearch-certs && cd ~/opensearch-certs

# Root CA
openssl genrsa -out root-ca-key.pem 4096
openssl req -new -x509 -sha256 -key root-ca-key.pem -out root-ca.pem -days 3650 -subj "/C=IT/ST=Lazio/L=Roma/O=Leonardo/CN=opensearch-root-ca"

# Admin cert
openssl genrsa -out admin-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem
openssl req -new -key admin-key.pem -out admin.csr -subj "/C=IT/ST=Lazio/L=Roma/O=Leonardo/CN=admin"
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out admin.pem -days 3650
rm admin-key-temp.pem admin.csr

# Nodo 1
openssl genrsa -out node1-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node1-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node1-key.pem
openssl req -new -key node1-key.pem -out node1.csr -subj "/C=IT/ST=Lazio/L=Roma/O=Leonardo/CN=opensearch1"
echo "subjectAltName=IP:192.168.0.31,DNS:opensearch1" > node1.ext
openssl x509 -req -in node1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out node1.pem -days 3650 -extfile node1.ext
rm node1-key-temp.pem node1.csr node1.ext

# Nodo 2
openssl genrsa -out node2-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node2-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node2-key.pem
openssl req -new -key node2-key.pem -out node2.csr -subj "/C=IT/ST=Lazio/L=Roma/O=Leonardo/CN=opensearch2"
echo "subjectAltName=IP:192.168.0.32,DNS:opensearch2" > node2.ext
openssl x509 -req -in node2.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out node2.pem -days 3650 -extfile node2.ext
rm node2-key-temp.pem node2.csr node2.ext

# Nodo 3
openssl genrsa -out node3-key-temp.pem 2048
openssl pkcs8 -inform PEM -outform PEM -in node3-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out node3-key.pem
openssl req -new -key node3-key.pem -out node3.csr -subj "/C=IT/ST=Lazio/L=Roma/O=Leonardo/CN=opensearch3"
echo "subjectAltName=IP:192.168.0.33,DNS:opensearch3" > node3.ext
openssl x509 -req -in node3.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out node3.pem -days 3650 -extfile node3.ext
rm node3-key-temp.pem node3.csr node3.ext

DISTRIBUZIONE CERTIFICATI (da opensearch1 con user 'ldo'):

sudo mkdir -p /etc/opensearch/certs
sudo cp root-ca.pem admin.pem admin-key.pem node*.pem node*-key.pem /etc/opensearch/certs/
sudo chown -R opensearch:opensearch /etc/opensearch/certs
sudo chmod 600 /etc/opensearch/certs/*.pem

# Copia su nodo 2
scp root-ca.pem admin.pem admin-key.pem node2.pem node2-key.pem ldo@192.168.0.32:~/
ssh ldo@192.168.0.32 "sudo mkdir -p /etc/opensearch/certs && sudo mv ~/*.pem /etc/opensearch/certs/ && sudo chown -R opensearch:opensearch /etc/opensearch/certs && sudo chmod 600 /etc/opensearch/certs/*.pem"

# Copia su nodo 3
scp root-ca.pem admin.pem admin-key.pem node3.pem node3-key.pem ldo@192.168.0.33:~/
ssh ldo@192.168.0.33 "sudo mkdir -p /etc/opensearch/certs && sudo mv ~/*.pem /etc/opensearch/certs/ && sudo chown -R opensearch:opensearch /etc/opensearch/certs && sudo chmod 600 /etc/opensearch/certs/*.pem"

******************************************************************************************************************

SU OGNI NODO → Configura /etc/opensearch/opensearch.yml (usa sudo nano /etc/opensearch/opensearch.yml)

...
cluster.name: cluster-leonardo-roma
node.name: opensearch1                    # <-- cambia: opensearch2 / opensearch3
network.host: 192.168.0.31                # <-- cambia: .32 / .33

discovery.seed_hosts: ["192.168.0.31:9300", "192.168.0.32:9300", "192.168.0.33:9300"]
cluster.initial_cluster_manager_nodes: ["opensearch1", "opensearch2", "opensearch3"]

bootstrap.memory_lock: false

plugins.security.ssl.transport.pemcert_filepath: certs/node1.pem
plugins.security.ssl.transport.pemkey_filepath: certs/node1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false

plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/node1.pem
plugins.security.ssl.http.pemkey_filepath: certs/node1-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem

plugins.security.allow_unsafe_democertificates: false
plugins.security.allow_default_init_securityindex: true

plugins.security.authcz.admin_dn:
  - 'CN=admin,O=Leonardo,L=Roma,ST=Lazio,C=IT'

plugins.security.nodes_dn:
  - 'CN=opensearch*,O=Leonardo,L=Roma,ST=Lazio,C=IT'
...

******************************************************************************************************************

SU TUTTI I NODI:

sudo systemctl daemon-reload
sudo systemctl enable opensearch
sudo systemctl start opensearch
sudo systemctl status opensearch

curl -v -k --max-time 30 https://192.168.0.31:9200/_cat/nodes?v
openssl s_client -connect 192.168.0.31:9200 -showcerts

******************************************************************************************************************
SOLO SU OPENSEARCH1 → Applica configurazione di sicurezza:

echo 'export JAVA_HOME=/usr/share/opensearch/jdk' | sudo tee -a /etc/environment
source /etc/environment

cd /usr/share/opensearch/plugins/opensearch-security/tools

sudo -E ./securityadmin.sh \
  -cd /usr/share/opensearch/plugins/opensearch-security/securityconfig/ \
  -icl \
  -nhnv \
  -cacert /etc/opensearch/certs/root-ca.pem \
  -cert /etc/opensearch/certs/admin.pem \
  -key /etc/opensearch/certs/admin-key.pem \
  -h 192.168.0.31
  
******************************************************************************************************************

VERIFICA STATUS NODO:
curl --cacert /etc/opensearch/certs/root-ca.pem https://192.168.0.31:9200 -u 'admin:Temporanea.123!'
curl --cacert /etc/opensearch/certs/root-ca.pem https://192.168.0.32:9200 -u 'admin:Temporanea.123!'
curl --cacert /etc/opensearch/certs/root-ca.pem https://192.168.0.33:9200 -u 'admin:Temporanea.123!'

VERIFICA STATUS CLUSTER:
curl --cacert /etc/opensearch/certs/root-ca.pem "https://192.168.0.31:9200/_cluster/health?pretty" -u 'admin:Temporanea.123!'

VERIFICA NODO MASTER:
curl --cacert /etc/opensearch/certs/root-ca.pem -u 'admin:Temporanea.123!' "https://192.168.0.31:9200/_cat/master?v"

VERIFICA ELENCO NODI:
curl --cacert /etc/opensearch/certs/root-ca.pem -u 'admin:Temporanea.123!' "https://192.168.0.31:9200/_cat/nodes?v"

VERIFICA INDICI:
curl --cacert /etc/opensearch/certs/root-ca.pem -u 'admin:Temporanea.123!' "https://192.168.0.31:9200/_cat/indices?v"

VERIFICA ACCESSO TLS:
curl --cacert /etc/opensearch/certs/root-ca.pem \
  --cert /etc/opensearch/certs/admin.pem \
  --key /etc/opensearch/certs/admin-key.pem \
  "https://192.168.0.31:9200/_cluster/health?pretty"


Log per troubleshooting:

sudo journalctl -u opensearch -f
sudo tail -f /var/log/opensearch/cluster-leonardo-roma.log
