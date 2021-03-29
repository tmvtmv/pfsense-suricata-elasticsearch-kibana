# Sending Suricata events of your pfSense firewall to Elasticsearch and Kibana using filebeat

## Notice

* I DO NOT provide ANY warranty in case something breaks on your machine.
* These steps are NOT supported by pfSense.
* These steps worked for me in combination with pfSense 2.5.0-RELEASE and are intended to get you going on your quest of getting Suricata to work for you on your pfSense.

## Prerequisites

* Your pfSense (including Suricata package) is already up-and-running.
* Your ELK-stack is already up-and-running somewhere.

## pfSense setup 

* To enable you to copy-paste easier to pfSense:
  * On pfSense console: select `Option 14 to Enable SSH`
  * `ssh admin@[your-firewall-ip]`

    E.g: ssh admin@192.168.10.1

* Select "Option 8: Shell"
* Enable the regular FreeBSD repository:

  `vi /usr/local/share/pfSense/pkg/repos/pfSense-repo.conf`

  Set  **FreeBSD: {enabled: }** to **yes**

  ```
  FreeBSD: { enabled: yes }
  
  pfSense-core: {
    url: "pkg+https://packages.netgate.com/pfSense_v2_5_0_amd64-core",
    mirror_type: "srv",
    signature_type: "fingerprints",
    fingerprints: "/usr/local/share/pfSense/keys/pkg",
    enabled: yes
  }
  
  pfSense: {
    url: "pkg+https://packages.netgate.com/pfSense_v2_5_0_amd64-pfSense_v2_5_0",
    mirror_type: "srv",
    signature_type: "fingerprints",
    fingerprints: "/usr/local/share/pfSense/keys/pkg",
    enabled: yes
  }
  ```

* Update repository contents:

  `pkg update`

* Install the required packages:

  `pkg install -y beats7 rubygem-elasticsearch-xpack wget`

### These steps are for adding the Suricata-module to Filebeat

* Get the latest beats from Github repository as a ZIP-file:

  * `cd /root`
  * `wget https://github.com/elastic/beats/archive/refs/heads/master.zip`

* Copy the required files to the directories within pfSense:

  * `unzip master.zip`
  * `cp -R /root/beats-master/x-pack/filebeat/module/* /usr/local/share/beats/filebeat/module/`
  * `cp -R /root/beats-master/x-pack/filebeat/modules.d/* /usr/local/etc/beats/filebeat.modules.d/`

* Check to see if all modules are available now:

  ```
  /usr/local/sbin/filebeat --path.config /usr/local/etc/beats --path.data /var/db/beats/filebeat --path.logs /var/log/beats --path.home /usr/local/share/beats/filebeat modules list
  ```

* To enable Suricata type:

  ```
  /usr/local/sbin/filebeat --path.config /usr/local/etc/beats --path.data /var/db/beats/filebeat --path.logs /var/log/beats --path.home /usr/local/share/beats/filebeat modules enable suricata
  ```

* Edit the file /usr/local/etc/beats/filebeat.modules.d/suricata.yml to the following contents:

  ```
  - module: suricata
    eve:
      enabled: true
      var.paths: ["/var/log/suricata/suricata_*/eve.json"]
  ```

### And then finish the usual setup of filebeat

* `cp /usr/local/etc/beats/filebeat.yml.sample /usr/local/etc/beats/filebeat.yml`

* `vi /usr/local/etc/beats/filebeat.yml`
  Edit the settings as you normally would for Filebeat.

* Check your config:
  `/usr/local/sbin/filebeat -c /usr/local/etc/beats/filebeat.yml test config`
  
  Expected response:
   `Config ok`

* Check communication to your ELK-server(s):
  `/usr/local/sbin/filebeat -c /usr/local/etc/beats/filebeat.yml test output`

  Expected repsonse:

  ```
  elasticsearch: https://elk.somedomain.com:9200...
    parse url... OK
    connection...
      parse host... OK
      dns lookup... OK
      addresses: 1.2.3.4
      dial up... OK
    TLS...
      security: server's certificate chain verification is enabled
      handshake... OK
      TLS version: TLSv1.2
      dial up... OK
    talk to server... OK
    version: 7.12.0
  ```

* Provision ElasticSearch and Kibana for Suricata data and dashboards:

  ```
  /usr/local/sbin/filebeat --path.config /usr/local/etc/beats --path.data /var/db/beats/filebeat --path.logs /var/log/beats --path.home /usr/local/share/beats/filebeat setup
  ```

* Ensure filebeat will run again after reboot:
  `service filebeat enable`

* Start filebeat :
  `service filebeat start`

* Optional: Check /var/log/beats/filebeat for clues if something doesn't work as expected.

And you're done. Have fun!
