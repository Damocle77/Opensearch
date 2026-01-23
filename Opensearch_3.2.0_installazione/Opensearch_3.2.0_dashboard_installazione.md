Guida pratica (Ubuntu 24.04.3) per installare **OpenSearch Dashboards 3.2.0** via **pacchetto `.deb`** e 
collegarlo a un cluster **OpenSearch 3.2.0** già configurato (TLS + Security plugin).

> Obiettivo: avere Dashboards raggiungibile su **https://<IP_DASH>:5601** e connesso al cluster su **https://<IP_NODE>:9200**.  
> Versioni: **Dashboards e OpenSearch devono combaciare** (3.2.0 ↔ 3.2.0).  
> Riferimenti ufficiali: installazione Dashboards su Debian/Ubuntu e TLS per Dashboards.  
> (Fonti: documentazione OpenSearch)  

---

## 0) Prerequisiti
- Cluster OpenSearch 3.2.0 funzionante (già bootstrap + security init completati).
- Connettività rete da Dashboards verso i nodi OpenSearch sulle porte **9200** (HTTP API) e **9300** (transport, se serve in altre operazioni).
- Una macchina/VM dove installare Dashboards (consigliato **1 sola Dashboards** per cluster; può stare su `opensearch1` o su una VM dedicata).
- Certificati/CA del cluster disponibili (almeno la **root CA** che firma i cert dei nodi OpenSearch).

---

## 1) Scarica il pacchetto `.deb` di OpenSearch Dashboards 3.2.0
OpenSearch fornisce i pacchetti “bundle” su `artifacts.opensearch.org`. Per Debian/Ubuntu, l’installazione è via download del `.deb` e `dpkg -i`.  
Fonte ufficiale: “Installing OpenSearch Dashboards (Debian)”.

### 1.1 Scegli l’architettura
Verifica architettura:

```bash
dpkg --print-architecture
uname -m
```

- x86_64 / amd64 → usa `linux-x64.deb`
- arm64 / aarch64 → usa `linux-arm64.deb`

### 1.2 Download (x64) — esempio
```bash
cd /tmp
curl -SLO https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.2.0/opensearch-dashboards-3.2.0-linux-x64.deb
```

### 1.3 (Opzionale ma consigliato) download firma `.sig` e verifica
La guida ufficiale include la verifica fingerprint / firma per assicurarsi che il pacchetto non sia stato alterato.

```bash
curl -SLO https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/3.2.0/opensearch-dashboards-3.2.0-linux-x64.deb.sig
# Segui la sezione di fingerprint/signature verification della doc ufficiale.
```

---

## 2) Installazione con `dpkg` (stesso stile di OpenSearch)

```bash
sudo dpkg -i /tmp/opensearch-dashboards-3.2.0-linux-x64.deb

# Se dpkg segnala dipendenze mancanti:
sudo apt-get -f install -y
```

---

## 3) Abilita e avvia il servizio
Dopo installazione: `daemon-reload`, enable, start, status.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now opensearch-dashboards
sudo systemctl status opensearch-dashboards --no-pager
```

Log live:

```bash
sudo journalctl -u opensearch-dashboards -f
```

---

## 4) TLS su Dashboards (HTTPS su porta 5601)
Se vuoi Dashboards in **HTTPS** (consigliato), devi configurare:
- `server.ssl.enabled: true`
- `server.ssl.certificate` e `server.ssl.key`


### 4.1 Prepara cartella cert
```bash
sudo mkdir -p /etc/opensearch-dashboards/certs
sudo chown -R opensearch-dashboards:opensearch-dashboards /etc/opensearch-dashboards/certs
sudo chmod 755 /etc/opensearch-dashboards/certs
```

### 4.2 Copia la CA del cluster (root-ca.pem)
Questa CA serve per far fidare Dashboards dei certificati dei nodi OpenSearch (quando `opensearch.hosts` è `https://...`).  
(Se nella tua guida cluster la CA è in `/etc/opensearch/certs/root-ca.pem`, copia da lì.)

```bash
sudo cp /etc/opensearch/certs/root-ca.pem /etc/opensearch-dashboards/certs/
sudo chown opensearch-dashboards:opensearch-dashboards /etc/opensearch-dashboards/certs/root-ca.pem
sudo chmod 644 /etc/opensearch-dashboards/certs/root-ca.pem
```

### 4.3 Crea un certificato server per Dashboards (firmato dalla tua CA)
Esempio con SAN (IP + DNS). Adatta IP/DNS alla tua installazione.

> Nota: questo richiede la **chiave privata** della CA (`root-ca-key.pem`). 
Se non vuoi tenerla su questa macchina, genera CSR qui e firma altrove.

```bash
cd /tmp

openssl genrsa -out dashboard-key.pem 2048
openssl req -new -key dashboard-key.pem -out dashboard.csr \
  -subj "/C=IT/ST=Lazio/L=Roma/O=OpenSearch/CN=opensearch-dashboards"

cat > dashboard.ext <<'EOF'
subjectAltName=IP:192.168.0.31,DNS:opensearch1,DNS:opensearch-dashboards
EOF

sudo openssl x509 -req -in dashboard.csr \
  -CA root-ca.pem \
  -CAkey root-ca-key.pem \
  -CAcreateserial -out dashboard-cert.pem -days 3650 -extfile dashboard.ext
  
sudo mv dashboard-cert.pem dashboard-key.pem /etc/opensearch-dashboards/certs/
sudo chown opensearch-dashboards:opensearch-dashboards /etc/opensearch-dashboards/certs/*
sudo chmod 600 /etc/opensearch-dashboards/certs/dashboard-key.pem

# Creare una fullchain dashboard-cert.pem + root-ca.pem:

cat dashboard-cert.pem root-ca.pem > dashboard-fullchain.pem
sudo chown opensearch-dashboards:opensearch-dashboards /etc/opensearch-dashboard/certs/dashboard-fullcahin.pem
sudo chmod 600 /etc/opensearch-dashboard/certs/dashboard-fullcahin.pem
```

---

## 5) Configura `/etc/opensearch-dashboards/opensearch_dashboards.yml`
Il file di config include:
- bind e porta (`server.host`, `server.port`)
- HTTPS (`server.ssl.*`)
- connessione al cluster (`opensearch.hosts`)
- fiducia CA (`opensearch.ssl.certificateAuthorities`)
- verifica TLS (`opensearch.ssl.verificationMode`)
- metodo auth (basic vs OpenID)


### 5.1 Backup e modifica
```bash
sudo cp /etc/opensearch-dashboards/opensearch_dashboards.yml \
  /etc/opensearch-dashboards/opensearch_dashboards.yml.bak.$(date +%F_%H%M)

sudo nano /etc/opensearch-dashboards/opensearch_dashboards.yml
```

### 5.2 Config “base” (TLS + cluster HTTPS)
Esempio minimalista, **senza** OIDC (basic auth). Adatta IP/host.
```
-------------------------------------------------------------------------------------------------------------
server.host: 0.0.0.0
server.port: 5601

server.ssl.enabled: true
server.ssl.certificate: /etc/opensearch-dashboards/certs/dashboard-fullchain.pem
server.ssl.key: /etc/opensearch-dashboards/certs/dashboard-key.pem

opensearch.hosts:
  - https://192.168.0.31:9200
  - https://192.168.0.32:9200
  - https://192.168.0.33:9200

opensearch.ssl.certificateAuthorities:
  - /etc/opensearch-dashboards/certs/root-ca.pem

# Consigliato: full (verifica hostname). Se usi IP e il certificato dei nodi non ha SAN IP,
# puoi temporaneamente usare "certificate" mentre sistemi i SAN.
opensearch.ssl.verificationMode: certificate

# Basic auth (inserisci user e password di opensearch)
opensearch.username: "admin"
opensearch.password: "Temporanea.123!"

opensearch.requestHeadersWhitelist:
  - authorization
  - securitytenant

opensearch_security.enabled: true
opensearch_security.multitenancy.enabled: true
opensearch_security.multitenancy.tenants.enable_private: true
opensearch_security.multitenancy.tenants.enable_global: true
opensearch_security.multitenancy.tenants.preferred:
  - Private
  - Global

# Use this setting if you are running opensearch-dashboards without https
#opensearch_security.cookie.secure: false
-------------------------------------------------------------------------------------------------------------
```

> Se usi `verificationMode: full`, **hostnames e certificati devono combaciare** (SAN + DNS).  
> Se sei in fase “bring-up” e i cert non hanno SAN corretti, `certificate` è spesso un compromesso

---

## 6) OpenID Connect (Keycloak) - se necessario
Per abilitare login via IdP OIDC, si usa `opensearch_security.auth.type: openid` + blocco `opensearch_security.openid.*`.  

Esempio (adatta i tuoi endpoint Keycloak):

```yaml
opensearch_security.auth.type: openid
opensearch_security.openid.connect_url: https://keycloak.example/realms/REALM/.well-known/openid-configuration
opensearch_security.openid.client_id: opensearch-dashboards
opensearch_security.openid.client_secret: "SEGRETO_CLIENT"
opensearch_security.openid.scope: "openid profile email"
opensearch_security.openid.base_redirect_url: https://192.168.0.31:5601
opensearch_security.openid.logout_url: https://keycloak.example/realms/REALM/protocol/openid-connect/logout
```

Checklist Keycloak (alta probabilità di “bug” se manca qualcosa):
- Redirect URI coerente con `base_redirect_url` (es. `https://192.168.0.31:5601/*`)
- Client di tipo “confidential” se usi `client_secret`
- Realm/issuer e discovery URL corretti (well-known)

---

## 7) Riavvia e valida
```bash
sudo systemctl restart opensearch-dashboards
sudo systemctl status opensearch-dashboards --no-pager
```

Test porta locale:

```bash
curl -vk https://192.168.0.31:5601
```

Apri da browser:
- `https://192.168.0.31:5601`

---

## 8) Troubleshooting rapido (quando la Forza ti abbandona)
### 8.1 Dashboards non parte
```bash
sudo journalctl -u opensearch-dashboards -n 200 --no-pager
```

### 8.2 Dashboards parte ma non si collega al cluster
- Verifica `opensearch.hosts` raggiungibili
- Verifica CA corretta in `opensearch.ssl.certificateAuthorities`
- Se `verificationMode: full` fallisce per SAN/hostname → prova temporaneamente `certificate` e poi rigenera cert con SAN corretti.

### 8.3 OIDC loop / 401 / redirect errati
- Controlla `base_redirect_url` e i redirect URI del client in Keycloak
- Controlla `connect_url` (well-known) e issuer

---

## 9) Hardening consigliato (da “lab” a “produzione”)
- Evita utente `admin` in Dashboards: crea un **utente servizio** con ruoli minimi.
- Mantieni `opensearch.ssl.verificationMode: full` (hostname verification) quando possibile. 
- Metti Dashboards dietro reverse proxy (opzionale) con rate limit / WAF.
- Abilita TLS moderno (`TLSv1.2`/`TLSv1.3`) se vuoi stringere le viti.

---

