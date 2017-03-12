# Varnish Cache

[Varnish Cache](https://www.varnish-cache.org) is a caching HTTP reverse proxy.
It can be used as a very fast web application accelerator.

## Basic Configuration


| Config Key                | Description                                 | Default                   |
|---------------------------|---------------------------------------------|---------------------------|
| varnish.vcl.conf          | Path on filesystem for the VCL Config file  | /etc/varnish/brooklyn.vcl |
| varnish.vcl.url           | URL for downloading VCL config file         | none                      |
| varnish.vcl.customization | Content to append to VCL configuration file | none                      |
| varnish.port              | Frontend port for Varnish                   | 80                        |
| varnish.admin.ip          | IP for Varnish Admin Console                | 127.0.0.1                 |
| varnish.admin.port        | Port for Varnish Admin Console              | 6082                      |
| varnish.memory            | Size of (malloc) cache"                     | 256M                      |
| varnish.user              | User to run Varnish                         | varnish                   |
| varnish.group             | Group to run Varnish                        | varnish                   |
| varnish.backend.ip        | Comma Separated List of Backend IP Addresses| 127.0.0.1                 |
| varnish.backend.port      | Port for Backend server                     | 8080                      |

"varnish.vcl.url" can be used to provide a file that will serve as the VCL configuration file. 
Any other configuration keys having to do with VCL configuration will be (EG varnish.backend.ips and port)
will be overwritten. 

## Example Use

For a single backend:

```

name: testApp
location: my-location
services:
  - type: 'TomcatServer:4.3.0'
    id: tc
    brooklyn.config:
      wars.root: https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/sample.war
  - type: 'varnish-node:0.1'
    id: varnish
    brooklyn.config:
      varnish.backend.ip: $brooklyn:component("tc").attributeWhenReady("host.address")
      varnish.backend.port: $brooklyn:component("tc").attributeWhenReady("http.port")

```

For multiple backends:

```
name: testApp
location: my-location
services:
  - type: org.apache.brooklyn.entity.stock.BasicApplication
    id: app
    brooklyn.children:
    - type: 'varnish-node:0.1'
      id: varnish
      brooklyn.config:
        varnish.backend.ips: $brooklyn:component("tomcat-cluster").attributeWhenReady("host.address.string")
        customize.latch: $brooklyn:component("tomcat-cluster").attributeWhenReady("service.isUp")
    - type: brooklyn.entity.group.DynamicCluster
      id: tomcat-cluster
      name: Tomcat Cluster
      description: A cluster of Tomcat nodes
      brooklyn.enrichers:
      - type: org.apache.brooklyn.enricher.stock.Aggregator
        brooklyn.config:
          enricher.sourceSensor: $brooklyn:sensor("host.address")
          enricher.targetSensor: $brooklyn:sensor("host.address.list")
          enricher.aggregating.fromMembers: true
      - type: org.apache.brooklyn.enricher.stock.Joiner
        brooklyn.config:
          enricher.sourceSensor: $brooklyn:sensor("host.address.list")
          enricher.targetSensor: $brooklyn:sensor("host.address.string")
          enricher.joiner.quote: false

      brooklyn.config:
        initialSize: 2
        memberSpec:
          $brooklyn:entitySpec:
            type: 'TomcatServer:4.3.0'
            name: Tomcat Node
            brooklyn.config:
              wars.root: https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/sample.war
```
