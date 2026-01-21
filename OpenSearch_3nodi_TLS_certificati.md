# OpenSearch 3.x – Creazione certificati TLS per cluster a 3 nodi

Questo documento contiene **tutti i comandi corretti, completi e testati** 
per generare una **PKI locale** e i certificati necessari a un cluster **OpenSearch 3 nodi** con **Security Plugin attivo**.

La procedura è pensata per:
- cluster **nuovo**
- nessuna PKI aziendale disponibile
- OpenSearch 3.x (DEB)

---

## 📁 Struttura finale dei certificati

Tutti i file saranno creati in:
```
/etc/opensearch/certs
```

File finali attesi:
```
root-ca.pem
root-ca-key.pem

opensearch1.pem
opensearch1-key.pem
opensearch2.pem
opensearch2-key.pem
opensearch3.pem
opensearch3-key.pem

http.pem
http-key.pem

admin.pem
admin-key.pem
```

---

## 1️⃣ Preparazione directory

```bash
mkdir -p /etc/opensearch/certs
cd /etc/opensearch/certs
rm -f *.pem *.key *.csr *.srl
```

---

## 2️⃣ Creazione CA (OBBLIGATORIO)

```bash
openssl genrsa -out root-ca-key.pem 4096

openssl req -x509 -new -nodes \
  -key root-ca-key.pem \
  -sha256 -days 3650 \
  -out root-ca.pem \
  -subj "/CN=opensearch-root-ca"
```

Verifica:
```bash
openssl x509 -in root-ca.pem -noout -subject
```

---

## 3️⃣ Certificati NODO (Transport TLS)

### Nodo 1 – opensearch1
```bash
openssl genrsa -out opensearch1-key.pem 2048
openssl req -new -key opensearch1-key.pem -out opensearch1.csr -subj "/CN=opensearch1"
openssl x509 -req -in opensearch1.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out opensearch1.pem -days 3650 -sha256
```

### Nodo 2 – opensearch2
```bash
openssl genrsa -out opensearch2-key.pem 2048
openssl req -new -key opensearch2-key.pem -out opensearch2.csr -subj "/CN=opensearch2"
openssl x509 -req -in opensearch2.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out opensearch2.pem -days 3650 -sha256
```

### Nodo 3 – opensearch3
```bash
openssl genrsa -out opensearch3-key.pem 2048
openssl req -new -key opensearch3-key.pem -out opensearch3.csr -subj "/CN=opensearch3"
openssl x509 -req -in opensearch3.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out opensearch3.pem -days 3650 -sha256
```

Verifica (esempio):
```bash
openssl x509 -in opensearch1.pem -noout -subject
```

---

## 4️⃣ Certificato HTTP (REST / HTTPS)

```bash
openssl genrsa -out http-key.pem 2048
openssl req -new -key http-key.pem -out http.csr -subj "/CN=opensearch-http"
openssl x509 -req -in http.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out http.pem -days 3650 -sha256
```

Verifica:
```bash
openssl x509 -in http.pem -noout -subject
```

---

## 5️⃣ Certificato ADMIN (securityadmin.sh)

```bash
openssl genrsa -out admin-key.pem 2048
openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=admin"
openssl x509 -req -in admin.csr -CA root-ca.pem -CAkey root-ca-key.pem -CAcreateserial -out admin.pem -days 3650 -sha256
```

Verifica:
```bash
openssl x509 -in admin.pem -noout -subject
```

---

## 6️⃣ Permessi corretti

```bash
chown opensearch:opensearch *.pem *.key
chmod 640 *.pem *.key
```

---

## 7️⃣ Riepilogo utilizzo certificati

| Uso | Certificato |
|---|---|
| Transport TLS nodo↔nodo | opensearchX.pem |
| HTTPS REST | http.pem |
| Security admin | admin.pem |
| CA | root-ca.pem |

---

## 8️⃣ DN da usare in opensearch.yml

```yaml
plugins.security.nodes_dn:
  - "CN=opensearch1"
  - "CN=opensearch2"
  - "CN=opensearch3"

plugins.security.authcz.admin_dn:
  - "CN=admin"
```

---

# Esempio completo di `opensearch.yml` per un cluster a 3 nodi con TLS

Questo file è pensato per essere identico su tutti i nodi, eccetto:

* `node.name`
* `network.host`
* `path.data` / `path.logs` (opzionale)

---


opensearch.yml

# ===================================================================
# Identità nodo
# ===================================================================
cluster.name: opensearch-cluster
node.name: opensearch1  # Cambia in opensearch2, opensearch3 sugli altri nodi

# ===================================================================
# Percorsi
# ===================================================================
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch

# ===================================================================
# Network
# ===================================================================
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# ===================================================================
# Discover dei nodi
# ===================================================================
discovery.seed_hosts:
  - opensearch1
  - opensearch2
  - opensearch3

# Solo sul primo avvio del cluster (uno dei tre nodi):
# commenta o rimuovi questa riga dopo la prima partenza
cluster.initial_master_nodes:
  - opensearch1
  - opensearch2
  - opensearch3

# ===================================================================
# TLS: TRANSPORT (nodo <-> nodo)
# ===================================================================
plugins.security.ssl.transport.enabled: true
plugins.security.ssl.transport.pemcert_filepath: certs/opensearch1.pem
plugins.security.ssl.transport.pemkey_filepath: certs/opensearch1-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: certs/root-ca.pem
plugins.security.ssl.transport.enforce_hostname_verification: false

# ===================================================================
# TLS: HTTP (REST)
# ===================================================================
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: certs/http.pem
plugins.security.ssl.http.pemkey_filepath: certs/http-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: certs/root-ca.pem

# ===================================================================
# Plugin Security - Distinguished Names (DN)
# ===================================================================
plugins.security.nodes_dn:
  - "CN=opensearch1"
  - "CN=opensearch2"
  - "CN=opensearch3"

plugins.security.authcz.admin_dn:
  - "CN=admin"

# ===================================================================
# Altre opzioni consigliate
# ===================================================================
plugins.security.allow_unsafe_democertificates: false
plugins.security.allow_default_init_securityindex: true
plugins.security.auth.type: internal

# ===================================================================
# Disabilita completamente SSL se necessario (non consigliato!)
# ===================================================================
# plugins.security.disabled: true
```

---

✅ Ricorda di:

* usare `certs/opensearch2.pem` e `.key` su opensearch2
* usare `certs/opensearch3.pem` e `.key` su opensearch3
* mantenere `http.pem` identico su tutti i nodi

⚠️ Non lasciare `cluster.initial_master_nodes` attivo dopo il primo bootstrap!


