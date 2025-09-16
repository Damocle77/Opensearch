Guida completa per installare un cluster **OpenSearch 2.x** su 3 nodi Fedora/RHEL Server con certificati autogenerati:


### **Guida all'Installazione di OpenSearch 2.x in Cluster con TLS**
**Prerequisiti:**
- 3 server Fedora/RHEL Server aggiornati (min 2vCPU 4GB Ram)
- Accesso root/sudo
- Connettività di rete tra i nodi
- Nomi host configurati: `opensearch1`, `opensearch2`, `opensearch3`
	per es: (/etc/hosts)
	192.168.40.26 opensearch1
	192.168.40.27 opensearch2
	192.168.40.28 opensearch3

---

### **Passo 1: Configurazione Base dei Nodi**

**Esegui su tutti i nodi:**
```bash
# Aggiorna il sistema
sudo dnf update -y

# Installa Java 11
sudo dnf install java-11-openjdk -y
sudo dnf install java-17-openjdk -y
java --version

#per selezionare una jvm di un elenco
sudo update-alternatives --config java
#per impostare il path java nelle variabili di ambiente
readlink -f $(which java)
#output del tipo: /usr/lib/jvm/java-11-openjdk-amd64/bin/java
echo $JAVA_HOME
sudo vim /etc/environment
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
source /etc/environment

# Configura i limiti di sistema
echo "opensearch - nofile 65535" | sudo tee -a /etc/security/limits.conf
echo "opensearch - memlock unlimited" | sudo tee -a /etc/security/limits.conf

# Configura kernel parameters
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### **Passo 2: Installazione di OpenSearch**

**Su tutti i nodi:**
```bash
# Scarica il corretto RPM (per es. 2.19.1)
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.17.1/opensearch-2.17.1-linux-x64.rpm
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.19.1/opensearch-2.19.1-linux-x64.rpm
wget https://artifacts.opensearch.org/releases/bundle/opensearch/2.18.0/opensearch-2.18.0-linux-x64.rpm

# Installa il pacchetto
sudo dnf install -y ./opensearch-2.17.1-linux-x64.rpm
sudo dnf install -y ./opensearch-2.18.0-linux-x64.rpm
sudo dnf install -y ./opensearch-2.19.1-linux-x64.rpm

# Verifica l'installazione
rpm -qi opensearch
```

---

### **Passo 3: Generazione Certificati TLS**

**Solo sul nodo `opensearch1`:**

```bash
# Installa OpenSSL
sudo dnf install openssl -y

# Crea directory per i certificati
sudo mkdir ~/opensearch-certs && cd ~/opensearch-certs

# Genera CA root
openssl genrsa -out root-ca-key.pem 2048
openssl req -new -x509 -sha256 -key root-ca-key.pem -subj "/CN=opensearch-CA" -out root-ca.pem -days 1825

# Genera certificati per ogni nodo
for node in opensearch1 opensearch2 opensearch3; do
  openssl genrsa -out $node-key.pem 2048
  openssl req -new -key $node-key.pem -subj "/CN=$node" -out $node.csr
  echo "subjectAltName=DNS:$node" > $node.ext
  openssl x509 -req -in $node.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out $node.pem -days 1825 -extfile $node.ext
done

# Certificato admin
openssl genrsa -out admin-key.pem 2048
openssl req -new -key admin-key.pem -subj "/CN=admin" -out admin.csr
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -sha256 -out admin.pem -days 1825
```

**Distribuisci i certificati agli altri nodi:**   

sposta i certificati creati per gli altri nodi sui nodi secondo corrispondeza usando SCP, MobaXterm o altro metodo
---

### **Passo 4: Configurazione del Cluster**

**Su tutti i nodi:**

1. Crea la directory per i certificati:
```bash
sudo mkdir -p /etc/opensearch/certs

sudo cp ~/opensearch-certs/root-ca.pem /etc/opensearch/certs/

sudo cp ~/opensearch-certs/$(hostname).pem /etc/opensearch/certs/
sudo cp ~/opensearch-certs/$(hostname)-key.pem /etc/opensearch/certs/

sudo cp ~/opensearch-certs/admin.pem /etc/opensearch/certs/
sudo cp ~/opensearch-certs/admin-key.pem /etc/opensearch/certs/

sudo chown -R opensearch:opensearch /etc/opensearch/certs
sudo chmod 640 /etc/opensearch/certs/*.pem
sudo chmod 600 /etc/opensearch/certs/*-key.pem

```

# Su ogni nodo, controlla CN e SAN
openssl x509 -in /etc/opensearch/certs/$(hostname).pem -noout -text | grep -E "(Subject:|DNS:)"

# Output atteso per opensearch1:
# Subject: CN = opensearch1
# DNS:opensearch1

#Test di connettività TLS tra nodi:
# Da opensearch1 verso opensearch2
openssl s_client -connect opensearch2:9300 -servername opensearch2 -verify_return_error -CAfile /etc/opensearch/certs/root-ca.pem


2. Modifica `/etc/opensearch/opensearch.yml`:
```yaml

cluster.name: opensearch-cluster
node.name: ${HOSTNAME}
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
network.host: 0.0.0.0
discovery.seed_hosts: ["opensearch1", "opensearch2", "opensearch3"]
cluster.initial_master_nodes: ["opensearch1", "opensearch2", "opensearch3"]

# Configurazione TLS
plugins.security.ssl.transport.pemcert_filepath: certs/${HOSTNAME}.pem
plugins.security.ssl.transport.pemkey_filepath: certs/${HOSTNAME}-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/${HOSTNAME}.pem
plugins.security.ssl.http.pemkey_filepath: certs/${HOSTNAME}-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem

# Sicurezza
plugins.security.nodes_dn:
  - "CN=opensearch1"
  - "CN=opensearch2"
  - "CN=opensearch3"
 
plugins.security.authcz.admin_dn:
  - "CN=admin"
plugins.security.allow_default_init_securityindex: true

```
---

### **Passo 5: Avvio del Cluster**

**Su tutti i nodi:**
```bash
sudo systemctl daemon-reload
sudo systemctl start opensearch
sudo systemctl enable opensearch
```
---

### **Passo 6: Inizializzazione della Sicurezza**

**Solo sul nodo `opensearch1`:**

```bash
cd /usr/share/opensearch

sudo -u opensearch ./plugins/opensearch-security/tools/securityadmin.sh \
  -cd plugins/opensearch-security/securityconfig/ \
  -icl -nhnv \
  -cacert /etc/opensearch/certs/root-ca.pem \
  -cert /etc/opensearch/certs/admin.pem \
  -key /etc/opensearch/certs/admin-key.pem
```
> **Nota:** Durante l'esecuzione, viene impostata la password per l'utente `admin` al valore `admin`

---

### **Passo 7: Verifica Finale**
```bash
# Controlla lo stato del cluster
curl -XGET --insecure -u 'admin:admin' 'https://localhost:9200/_cluster/health?pretty'
curl -k -u admin:password https://IP_O_HOST:9200/_cluster/health?pretty

# Elenca i nodi
curl -XGET --insecure -u 'admin:admin' 'https://localhost:9200/_cat/nodes?v'
```
---

### **Diagnosi Problemi**

1. **Logs:** `journalctl -u opensearch -f`

2. **Firewall:** 
   ```bash
   sudo firewall-cmd --permanent --add-port={9200/tcp,9300/tcp}
   sudo firewall-cmd --reload
   ```
3. **Connessione tra nodi:** Verifica che i nomi host siano risolvibili.

### **Tuning***

1. **Modificare le impostazioni JVM**:

Il file da modificare è `/etc/opensearch/jvm.options`

```bash
# Esempio di configurazione delle heap memory
-Xms4g	
-Xmx4g
```

Dove:
- `-Xms4g` imposta la memoria heap iniziale a 4 GB
- `-Xmx4g` imposta la memoria heap massima a 4 GB

2. **Raccomandazioni per le dimensioni della memoria**:

- Per server con 8 GB di RAM: usa 4-5 GB per Java
- Per server con 16 GB di RAM: usa 8-10 GB per Java

3. **Lascia sempre memoria libera per il sistema operativo e altri processi**:
- Non assegnare più del 50% della RAM totale a Java
- Impostare valori uguali per `-Xms` e `-Xmx` per evitare ridimensionamenti

4. **Altre ottimizzazioni**:
```bash
# Disabilitare swap
-XX:+DisableExplicitGC

# Utilizzare G1 Garbage Collector
-XX:+UseG1GC
```

Dopo le modifiche, riavvia il servizio OpenSearch:
```bash
sudo systemctl restart opensearch

---


Procedura per resettare la password di un utente interno
1. Genera un nuovo hash
Usa lo script hash fornito dal security plugin di OpenSearch, di solito si trova qui:
Plain Text
/usr/share/opensearch/plugins/opensearch-security/tools/hash.sh
Esegui:

./hash.sh -p NuovaPasswordSicura
Otterrai un hash bcrypt simile a $2a$12$....
2. Modifica internal_users.yml
Apri il file relativo nella cartella security config, es:
Plain Text
text/usr/share/opensearch/config/opensearch-security/internal_users.yml
Sostituisci l’hash dell’utente che vuoi resettare (es. admin) con il nuovo appena ottenuto, tipo:
Plain Text
textadmin:  hash: "$2a$12$..."
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"


3. Applica la configurazione con securityadmin.sh
Dopo aver modificato il file, lancia il comando securityadmin dalla stessa dir:
Plain Text
bashcd /usr/share/opensearch/plugins/opensearch-security/tools/./securityadmin.sh -cd /usr/share/opensearch/config/opensearch-security/ -icl -nhnv
Questa operazione aggiorna la security config del cluster con il nuovo hash.
4. Riavvia (se serve)

Per far sì che il certificato SSL di OpenSearch sia riconosciuto come valido dalla macchina client (senza bisogno di usare -k), devi aggiungere la CA root che ha firmato il certificato tra i trusted del sistema operativo della tua macchina client.

Procedura per aggiungere la CA (Linux, RedHat/CentOS/Rocky)
Copia la CA root sul client

Recupera il file della CA root (di solito un file .pem) che ha firmato i certificati del tuo cluster OpenSearch.

Copia questo file nella tua macchina client, ad esempio in /tmp/root-ca.pem.

Sposta la CA nella cartella dei certificati trusted

Su molte distro, la cartella è /etc/pki/ca-trust/source/anchors/ oppure /usr/local/share/ca-certificates/ (su Ubuntu/Debian).

bash
cp /tmp/root-ca.pem /etc/pki/ca-trust/source/anchors/
oppure

bash
cp /tmp/root-ca.pem /usr/local/share/ca-certificates/root-ca.crt
Aggiorna il trust store

Su RedHat/CentOS/Rocky:

bash
update-ca-trust
Su Debian/Ubuntu:

bash
update-ca-certificates
Testa nuovamente il comando curl (senza -k)

bash
curl -u admin:password "https://192.168.0.31:9200/_cluster/health?pretty" -v

