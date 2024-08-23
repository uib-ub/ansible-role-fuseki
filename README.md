# Ansible Role: Apache Jena/Fuseki SPARQL Server

[![Build Status](https://travis-ci.org/paginagmbh/ansible-role-fuseki.svg?branch=master)](https://travis-ci.org/paginagmbh/ansible-role-fuseki)

An Ansible Role that installs [Apache Jena/Fuseki](https://jena.apache.org/documentation/fuseki2/) and [Apache Jena Commands](https://jena.apache.org/documentation/tools/index.html) on Linux, and lets you configure it for running it as a systemd service, with the option of enabling additional modules such as jena-text, jena-geosparql, jena-service-enhancer, read-write endpoints and SHACL.

## Requirements

A Java runtime has to be installed on the target host. Apache Jena and Apache Jena Fuseki aims to support the last two LTS releases of java and will post a new major release when a LTS is no longer in use.

* Apache Jena  4.X branch supports java 11 to 17
* Apache Jena 5.X branch supports java 17 to 21

The default version if none is set is now 5.0, so a java >= 17 is currently required, if the version isn't lowered.

When used from Enterprise Linux, remember to add selinux boolean for `httpd_can_network_relay` or `httpd_can_network_connect`, based on the configuration, if fuseki is behind an Apache webserver. Also remember that security is only limited to a localhost filter by default, so reversing to localhost or 127.0.0.1:3030 directly means that the admin api is reachable. The role is often used with reverse-proxying only to the read endpoints e.g localhost:3030/test_db/ from examples below.

The Jena-text module enables optimizations for java 21 > if enabled.

Modules are currently configured to work with shaded artifacts created for version 5.1. Variables can be updated to override these hardcoded modules, but there is no archive to get them from, so they need to be built and configured.

## Example Playbook

Minimal example with no settings set, just a db_name ('test_db')
This sets up a working fuseki in fuseki_home (/opt/fuseki by default) and jena in /opt/jena. These are symlinks from /opt/jena-version etc. The role handles changing between versions. In addition it sets up configurations, databases etc in FUSEKI_BASE (/var/fuseki) as default. The paths can be overriden using `fuseki_home` and `fuseki_base`. Apache Jena is also installed by default since we often use the jena binaries in combination with Apache Jena Fuseki. It can be disabled by setting
 
By default, a read-only query and read-only graph store endpoint is configured in the path `/test_db/(data|query)`. This is persisted/configured for tdb2 as default.

```yaml

- name: "install required and optional dependencies"
  package:
    name:  ["unzip", "java-21-openjdk", "java-21-openjdk-headless"]
  become: true

- import_role:
    name: "fuseki"
  vars:
  fuseki_configurations:
  - name: "test_db"
```

Full example with all features

``` yaml
- name: "install required and optional dependencies"
  package:
    name:  ["unzip", "java-21-openjdk", "java-21-openjdk-headless"]
  become: true

- name: "Example of all features, except overriding base paths"
  hosts: all
  tasks:
    - name: "install required dependencies"
      package:
        name:  ["unzip","which", "java-21-openjdk", "java-21-openjdk-headless"]
      become: true

    - name: "Include fuseki"
      import_role:
        name: "fuseki"
      vars:
        # removing `apache_jena` from fuseki_components, fuseki can also be omited, if only Apache Jena is required.
#       fuseki_components:
        - "apache-jena-fuseki"
        # using older version for demonstration of setting checksums for non-default versions, note that modules won't work since they were built with a newer version (5.1.0)
        fuseki_version: "5.0.0"
        #checksums from https://archive.apache.org/dist/jena/binaries/apache-jena-fuseki-5.0.0.tar.gz.sha512 and  https://archive.apache.org/dist/jena/binaries/apache-jena-5.0.0.tar.gz.sha512
        fuseki_checksum: ""c7b9fa452cdec19c50ee08ee191012b26c7b51f1ee5c5143db3047e0545c007599fbc08481fa61df5aef766a796e43262c209fc42578f2e532c0ab0c19dcbc5
        jena_checksum: "5bbb9a3b613eadcd75beb11671b2d797794b578eea2f0e68b57ba7fd402ca789c7ea3c71206baace8c662581e8e615a22d40d3b5f9461823a8603dd6ee40d912"
        # downloads, unpacking,  etc. happens in home dir for the ansible user by default. It can be changed if needed (e.g if dir doesn't exist for some reason)
        fuseki_workspace: "/tmp/"
        # Use \s seperator for space to be correctly understood by the systemd unit 
        fuseki_java_opts: "-Xmx3g\s-server"
        # drop in for service enhancer needs both guava, and dependencies of guava, so using a shaded jar built locally
        service_enhancer_url: "https://github.com/OyvindLGjesdal/jena/releases/download/service-enhancer-shaded/jena-serviceenhancer-5.2.0-SNAPSHOT.jar"
        fuseki_modules: # additional jars can be supplied by giving an url for download and it will be available on the classpath
        # sis-embedded-data packaged with shading enabled - geosparql related
        # dropping in "https://repo1.maven.org/maven2/org/apache/sis/non-free/sis-embedded-data/1.4/sis-embedded-data-1.4.jar" also requires its dependencies
        - "https://github.com/OyvindLGjesdal/jena/releases/download/service-enhancer-shaded/jena-sis-nonfree-1.0.jar"
                #
                # - "https://repo1.maven.org/maven2/org/apache/sis/non-free/sis-embedded-data/1.4/sis-embedded-data-1.4.jar"
        fuseki_configurations:
        - name: "test_db"
        - name: "test-db_all"
          dataset_type: "tdb" # only tdb|tdb2 (default) implemented in role
          read_write: true # default false, creates a secondary dataset using the same database directory, that appends `_write` to the database_name e.g `test-db_all_write`
          union_default_graph: true # default false
          geosparql: true # default false
          service_enhancer: true # default false
          shacl: true # default false/undefined
          lucene: true # default false/undefined
          luceneStoreValues: true # default false/undefined
          # assembler blocks, prefix is placed before other blocks to be able to add further namespaces
          prefix: |
            @prefix ex: <http://example.org/Schema#> .
          # entity map block used by lucene configuration
          entity_map: |
            text:defaultField     "label" ;
            text:entityField      "uri" ;
            text:uidField         "uid" ;
            text:langField        "lang" ;
            text:graphField       "graph" ;
            text:map (
              [ text:field "label" ;
               text:predicate ex:name ] )
          extra_conf: |
            # generic block to insert extra assembler configuration not covered by booleans for a single service
          # block for insert shiro url rules https://shiro.apache.org/web.html#urls
          fuseki_shiro_url_rules: |
            /$/backup  = authcBasic,user[admin]
        # Can use shiro command line hasher (https://shiro.apache.org/command-line-hasher.html) to generate a password stanza to use for password_stanza
        # The password stanza should/could also be vaulted in your playbook, so it is only viewable in the shiro.ini file on server 
        fuseki_shiro_users:   
        - name: "admin"
          password_stanza: "$shiro2$argon2id$v=19$t=1,m=65536,p=4$Wr/2XKxWeYZt8JE5HCONQw$yev4bLiGzbeIZ8qDWrIY7J2msL2vRO/aYksb4RMeX7Y"
        - name: "reader"
          password_stanza: "{{ reader_password_stanza }}"


```

Each fuseki_configuration creates an assembler file in `$FUSEKI_BASE/configuration/name.ttl` Invalid turtle will cause the service to fail starting.

For more on the configuration `https://jena.apache.org/documentation/assembler/`. Configuration errors often cause warnings or errors that can be seen 
with `journalctl -u fuseki --since today` or `systemctl status fuseki -n100 -l`.

Our CI configuration runs molecule to test that the main features and additions work (graph store protocol publish,standard get queries, geosparql, jena-text, service-enhancer and shiro user configuration authentication for request). @todo Shacl test.

## License

BSD

## Author Information

This role was created in 2018 by [Pagina GmbH](https://www.pagina.gmbh/) and was forked by the [The University of Bergen Library](https://uib.no/ub) for further development.
