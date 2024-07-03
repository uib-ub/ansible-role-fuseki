# Ansible Role: Apache Jena/Fuseki SPARQL Server

[![Build Status](https://travis-ci.org/paginagmbh/ansible-role-fuseki.svg?branch=master)](https://travis-ci.org/paginagmbh/ansible-role-fuseki)

An Ansible Role that installs [Apache Jena/Fuseki](https://jena.apache.org/documentation/fuseki2/) on Linux.

## Requirements

A Java runtime has to be installed on the target host. Apache Jena and Apache Jena Fuseki aims to support the last two LTS releases of java and will post a new major release when a LTS is no longer in use.

* Apache Jena  4.X branch supports java 11 to 17
* Apache Jena 5.X branch supports java 17 to 21

The default version if none is set is now 5.0, so a java >= 17 is currently required, if the version isn't lowered.

When used from Enterprise Linux, remember to add selinux boolean for `httpd_can_network_relay` if fuseki is behind an Apache webserver. 

## Example Playbook

``` yaml
- role: geerlingguy.java
- role: fuseki
  fuseki_configurations:
    - name: "test-db"
      read_write: True
      union_default_graph: False
      service_enhancer: true
      shacl: true # creates dataset/shacl service
```

## Tests

To run the test playbook(s):

  1. Install and start Docker.
  1. Download the test shim (see .travis.yml file for the URL) into `tests/test.sh`:
  1. Make the test shim executable: `chmod +x tests/test.sh`.
  1. Run (from the role root directory) `distro=[distro] playbook=[playbook] ./tests/test.sh`

If you don't want the container to be automatically deleted after the test playbook is run, add the following environment variables:

    cleanup=false container_id=$(date +%s)

## License

BSD

## Author Information

This role was created in 2018 by [Pagina GmbH](https://www.pagina.gmbh/).

It draws heavily from best practices as demonstrated in roles authored by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
