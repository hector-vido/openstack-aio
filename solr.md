# Solr

```bash
SOLR_VERSION=9.8.0

wget https://www.apache.org/dyn/closer.lua/solr/solr/${SOLR_VERSION}/solr-${SOLR_VERSION}-slim.tgz?action=download -O /tmp/solr-${SOLR_VERSION}-slim.tgz
tar -C /root/ -xzf /tmp/solr-${SOLR_VERSION}-slim.tgz --strip-components=2 solr-${SOLR_VERSION}-slim/bin/install_solr_service.sh

apt-get update
apt-get install -y openjdk-17-jre

/root/install_solr_service.sh /tmp/solr-9.8.0-slim.tgz
sed -Ei 's,#SOLR_JETTY_HOST=.*,SOLR_JETTY_HOST=0.0.0.0,' /etc/default/solr.in.sh

systemctl enable solr
systemctl restart solr

su - solr -- /opt/solr/bin/solr create -c openshift
```
