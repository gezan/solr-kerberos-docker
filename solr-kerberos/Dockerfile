FROM solr:8.4.1

USER root

RUN export DEBIAN_FRONTEND=noninteractive; apt update; apt install -y krb5-user nmap

USER solr
