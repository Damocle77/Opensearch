I FIles:

config.yml, internal_users.yml, roles.yml, roles_mapping.yml, tenants.yml (e compagnia)

Servono se vuoi:

inizializzare la security (bootstrap della security index) su un cluster nuovo
modificare utenti/ruoli/tenants e applicare le modifiche
tenere traccia della config in un repo e riprodurre l’ambiente

Dove si mettono (su installazione .deb)

La posizione “canonica” per i file di configurazione che dai in pasto a securityadmin.sh è:

/usr/share/opensearch/plugins/opensearch-security/securityconfig/

Creare la directory (se manca):

sudo mkdir -p /usr/share/opensearch/plugins/opensearch-security/securityconfig
sudo chown -R opensearch:opensearch /usr/share/opensearch/plugins/opensearch-security/securityconfig

Copiare dentro i file:

config.yml
internal_users.yml
roles.yml
roles_mapping.yml
tenants.yml

(a volte anche action_groups.yml, nodes_dn.yml)

Applichare la config al cluster con:

cd /usr/share/opensearch/plugins/opensearch-security/tools

sudo -E ./securityadmin.sh \
  -cd /usr/share/opensearch/plugins/opensearch-security/securityconfig/ \
  -icl -nhnv \
  -cacert /etc/opensearch/certs/root-ca.pem \
  -cert /etc/opensearch/certs/admin.pem \
  -key  /etc/opensearch/certs/admin-key.pem \
  -h 192.168.0.31

Nota importantissima!

Una volta che securityadmin.sh ha caricato i YAML, puoi anche cancellarli dal nodo: OpenSearch continuerà a funzionare uguale perché la config è nel suo indice di security.
