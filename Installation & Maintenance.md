# Installation & Maintenance (v5.*)

## Installation

    [install Java]
    [download solr-<ver>.tgz from] apache.org/dyn/closer.cgi/lucene/solr/
    tar xzf solr-<ver>.tgz solr-<ver>/bin/install_solr_service.sh --strip-components=2
    sudo bash ./install_solr_service.sh solr-<ver>.tgz

## Maintenance

    [status] sudo service solr status
    [restart] sudo service solr restart
    [create a core] sudo su - solr -c '/opt/solr/bin/solr create -c <corename> -d <configparentdirname>'
    [create a core from basic schema-config] sudo su - solr -c '/opt/solr/bin/solr create -c <corename> -d basic_configs'
    [create a core from basic schemaless-config] sudo su - solr -c '/opt/solr/bin/solr create -c <corename> -d data_driven_schema_configs'
    [delete a core] sudo su - solr -c '/opt/solr/bin/solr delete -c <corename>'
    [reload a core] curl 'http://localhost:8983/solr/admin/cores?action=RELOAD&core=<corename>'
    [unload a core] curl 'http://localhost:8983/solr/admin/cores?action=UNLOAD&core=<corename>'
