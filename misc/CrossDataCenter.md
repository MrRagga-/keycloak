Test Cross-Data-Center scenario (test with external JDG server)
===============================================================

These are temporary notes. This docs should be removed once we have cross-DC support finished and properly documented. 

These steps are already automated for embedded Undertow, see Cross-DC tests section in [HOW-TO-RUN.md](../testsuite/integration-arquillian/HOW-TO-RUN.md) document. For Wildfly they are not yet automated.
Following instructions are related to Wildfly server.
 
What is working right now is:
- Propagating of invalidation messages for `realms`, `users` and `authorization` caches
- sessions, offline sessions and login failures are propagated between datacenters 
  

Basic setup
===========

This is the example setup simulating 2 datacenters `site1` and `site2` . Each datacenter consists of 1 infinispan server and 2 Keycloak servers. 
So 2 infinispan servers and 4 Keycloak servers are totally in the testing setup. 

* Site1 consists of infinispan server `jdg1` and 2 Keycloak servers `node11` and `node12` .

* Site2 consists of infinispan server `jdg2` and 2 Keycloak servers `node21` and `node22` .
 
* Infinispan servers `jdg1` and `jdg2` are connected with each other through the RELAY protocol and "backup" based infinispan caches in
similar way as described in the infinispan documentation - https://access.redhat.com/documentation/en-us/red_hat_jboss_data_grid/7.1/html-single/administration_and_configuration_guide/#configure_cross_datacenter_replication_remote_client_server_mode .
 
* Keycloak servers `node11` and `node12` forms cluster with each other, but they don't communicate with any server in `site2` .
They communicate with infinispan server `jdg1` through the HotRod protocol (Remote cache).
  
* Same applies for `node21` and `node22` . They have cluster with each other and communicate just with `jdg2` server through the HotRod protocol.
  
TODO: Picture on blog
  
* For example when some object (realm, client, role, user, ...) is updated on `node11`, the `node11` will send invalidation message. It does it by saving special cache entry to the remote cache `work` on `jdg1` .
  The `jdg1` notifies client listeners in same DC (hence on `node12`) and propagate the message to it. But `jdg1` is connected through backup with `jdg2` too.
  So the entry is saved on `jdg2` too and `jdg2` will notify client listeners on nodes `node21` and `node22`.
  All the nodes know that they should invalidate the updated object from their caches. The caches with the actual data (`realms`, `users` and `authorization`) are infinispan local caches.     

TODO: Picture and better explanation?

* For example when some userSession is created/updated/removed on `node11` it is saved in cluster on current DC, so the `node12` can see it. But it's saved also to remote cache on `jdg1` server. 
 The userSession is then automatically seen on `jdg2` server through the backup cache `sessions` between `jdg1` and `jdg2` . Server `jdg2` then notifies nodes `node21` and `node22` through
 the client listeners (Feature of Remote Cache and HotRod protocol. See infinispan docs for details). The node, who is owner of the userSession (either `node21` or `node22`) will update userSession in the cluster 
 on `site2` . Hence any user requests coming to Keycloak nodes on `site2` will see latest updates.
 
TODO: Picture and better explanation?

Example setup assumes all 6 servers are bootstrapped on localhost, but each on different ports.


Infinispan Server setup
-----------------------

1) Download Infinispan 8.2.8 server and unzip to some folder

2) Change those things in the `JDG1_HOME/standalone/configuration/clustered.xml` in the configuration of JGroups subsystem:

2.a) Add the `xsite` channel, which will use `tcp` stack, under `channels` element:

```xml
<channels default="cluster">
    <channel name="cluster"/>
    <channel name="xsite" stack="tcp"/>
</channels>
```

2.b) Add `relay` element to the end of the `udp` stack:

```xml
<stack name="udp">
    ...
    <relay site="site1">
        <remote-site name="site2" channel="xsite"/>
    </relay>
</stack>
```

2.c) Configure `tcp` stack to use TCPPING instead of MPING . Just remove MPING element and replace with the TCPPING like this:

```xml
<stack name="tcp">
    <transport type="TCP" socket-binding="jgroups-tcp"/>
    <protocol type="TCPPING">
        <property name="initial_hosts">localhost[8610],localhost[9610]"</property>
        <property name="ergonomics">false</property>
    </protocol>
    <protocol type="MERGE3"/>
    ...
</stack>
```

3) Add this into `JDG1_HOME/standalone/configuration/clustered.xml` under cache-container named `clustered` :

```xml   
<cache-container name="clustered" default-cache="default" statistics="true">
        ...
        <replicated-cache-configuration name="sessions-cfg" mode="SYNC" start="EAGER" batching="false">
            <transaction mode="NON_XA" locking="PESSIMISTIC"/>
            <backups>
                <backup site="site2" failure-policy="FAIL" strategy="SYNC" enabled="true"/>
            </backups>
        </replicated-cache-configuration>

        <replicated-cache name="work" configuration="sessions-cfg"/>
        <replicated-cache name="sessions" configuration="sessions-cfg"/>
        <replicated-cache name="offlineSessions" configuration="sessions-cfg"/>
        <replicated-cache name="actionTokens" configuration="sessions-cfg"/>
        <replicated-cache name="loginFailures" configuration="sessions-cfg"/>
                
</cache-container>
```
        
4) Copy the server into the second location referred later as `JDG2_HOME`

5) In the `JDG2_HOME/standalone/configuration/clustered.xml` exchange `site1` with `site2` and viceversa in the configuration of `relay` in the
JGroups subsystem and in configuration of `backups` in the cache-subsystem.

NOTE: It's currently needed to have different configuration files for both sites as Infinispan subsystem doesn't support
replacing site name with expressions. See https://issues.jboss.org/browse/WFLY-9458 for more details.

6) Start server `jdg1`:

```
cd JDG1_HOME/bin
./standalone.sh -c clustered.xml -Djava.net.preferIPv4Stack=true \
-Djboss.socket.binding.port-offset=1010 -Djboss.default.multicast.address=234.56.78.99 \
-Djboss.node.name=jdg1
```

7) Start server `jdg2` . There is different multicast address, so the `jdg1` and `jdg2` servers are not in "direct" cluster with each other,
but they are just connected through the RELAY protocol and TCP JGroups stack is used for communication between them. So the startup command is like this:

```
cd JDG2_HOME/bin
./standalone.sh -c clustered.xml -Djava.net.preferIPv4Stack=true \
-Djboss.socket.binding.port-offset=2010 -Djboss.default.multicast.address=234.56.78.100 \
-Djboss.node.name=jdg2
```

8) To verify that channel works at this point, you may need to use JConsole and connect either to JDG1 or JDG2 running server. When
use the MBean `jgroups:type=protocol,cluster="cluster",protocol=RELAY2` and operation `printRoutes`, you should see the output like this:

```
site1 --> _jdg1:site1
site2 --> _jdg2:site2
```

Keycloak servers setup
----------------------
1) Download Keycloak 3.3.0.CR1 and unzip to some location referred later as `NODE11`

2) Configure shared database for KeycloakDS datasource. Recommended to use MySQL, MariaDB or PostgreSQL. See Keycloak docs for more details
 
3) Edit `NODE11/standalone/configuration/standalone-ha.xml` :

3.1) Add attribute `site` to the JGroups UDP protocol:
  
```xml
                  <stack name="udp">
                      <transport type="UDP" socket-binding="jgroups-udp" site="${jboss.site.name}"/>
```  

3.2) Add this `module` attribute under `cache-container` element of name `keycloak` :
 
```xml
 <cache-container name="keycloak" jndi-name="infinispan/Keycloak" module="org.keycloak.keycloak-model-infinispan">
``` 

3.3) Add the `remote-store` under `work` cache:

```xml
<replicated-cache name="work" mode="SYNC">
    <remote-store cache="work" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="false" shared="true">
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</replicated-cache>
```

3.5) Add the `remote-store` like this under `sessions` cache:

```xml
<distributed-cache name="sessions" mode="SYNC" owners="1">
    <remote-store cache="sessions" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="false" shared="true">   
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</distributed-cache>
```

3.6) Same for `offlineSessions`, `clientSessions`, `offlineClientSessions`, `loginFailures`, and `actionTokens` caches (the only difference
from `sessions` cache is that `cache` property value are different):

```xml
<distributed-cache name="offlineSessions" mode="SYNC" owners="1">
    <remote-store cache="offlineSessions" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="false" shared="true">
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</distributed-cache>

<distributed-cache name="clientSessions" mode="SYNC" owners="1">
    <remote-store cache="clientSessions" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="false" shared="true">
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</distributed-cache>

<distributed-cache name="offlineClientSessions" mode="SYNC" owners="1">
    <remote-store cache="offlineClientSessions" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="false" shared="true">
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</distributed-cache>

<distributed-cache name="loginFailures" mode="SYNC" owners="1">
    <remote-store cache="loginFailures" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="false" shared="true">
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</distributed-cache>

<distributed-cache name="actionTokens" mode="SYNC" owners="2">
    <eviction max-entries="-1" strategy="NONE"/>
    <expiration max-idle="-1" interval="300000"/>
    <remote-store cache="actionTokens" remote-servers="remote-cache" passivation="false" fetch-state="false" purge="false" preload="true" shared="true">
        <property name="rawValues">true</property>
        <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property>
    </remote-store>
</distributed-cache>
```

3.7) Add outbound socket binding for the remote store into `socket-binding-group` configuration:

```xml
<outbound-socket-binding name="remote-cache">
    <remote-destination host="${remote.cache.host:localhost}" port="${remote.cache.port:11222}"/>
</outbound-socket-binding>
```

3.8) The configuration of distributed cache `authenticationSessions` and other caches is left unchanged.

3.9) Optionally enable DEBUG logging under `logging` subsystem:

```xml
<logger category="org.keycloak.cluster.infinispan">
    <level name="DEBUG"/>
</logger>
<logger category="org.keycloak.connections.infinispan">
    <level name="DEBUG"/>
</logger>
<logger category="org.keycloak.models.cache.infinispan">
    <level name="DEBUG"/>
</logger>
<logger category="org.keycloak.models.sessions.infinispan">
    <level name="DEBUG"/>
</logger>
```

4) Copy the `NODE11` to 3 other directories referred later as `NODE12`, `NODE21` and `NODE22`.
 
5) Start `NODE11` :
 
```
cd NODE11/bin
./standalone.sh -c standalone-ha.xml -Djboss.node.name=node11 -Djboss.site.name=site1 \
-Djboss.default.multicast.address=234.56.78.100 -Dremote.cache.port=12232 -Djava.net.preferIPv4Stack=true \
-Djboss.socket.binding.port-offset=3000

```

6) Start `NODE12` :

````
cd NODE12/bin
./standalone.sh -c standalone-ha.xml -Djboss.node.name=node12 -Djboss.site.name=site1 \
-Djboss.default.multicast.address=234.56.78.100 -Dremote.cache.port=12232 -Djava.net.preferIPv4Stack=true \
-Djboss.socket.binding.port-offset=4000
````

The cluster nodes should be connected. This should be in the log of both NODE11 and NODE12:

```
Received new cluster view for channel keycloak: [node11|1] (2) [node11, node12]
```

7) Start `NODE21` :

```
cd NODE21/bin
./standalone.sh -c standalone-ha.xml -Djboss.node.name=node21 -Djboss.site.name=site2 \
-Djboss.default.multicast.address=234.56.78.101 -Dremote.cache.port=13232 -Djava.net.preferIPv4Stack=true \
-Djboss.socket.binding.port-offset=5000
```

It shouldn't be connected to the cluster with `NODE11` and `NODE12`, but to separate one:

```
Received new cluster view for channel keycloak: [node21|0] (1) [node21]
```

8) Start `NODE22` :

```
cd NODE22/bin
./standalone.sh -c standalone-ha.xml -Djboss.node.name=node22 -Djboss.site.name=site2 \
-Djboss.default.multicast.address=234.56.78.101 -Dremote.cache.port=13232 -Djava.net.preferIPv4Stack=true \
-Djboss.socket.binding.port-offset=6000
```

It should be in cluster with `NODE21` :

```
Received new cluster view for channel keycloak: [node21|1] (2) [node21, node22]
```

9) Test:

9.1) Go to `http://localhost:11080/auth/` and create initial admin user

9.2) Go to `http://localhost:11080/auth/admin` and login as admin to admin console

9.3) Open 2nd browser and go to any of nodes `http://localhost:12080/auth/admin` or `http://localhost:13080/auth/admin` or `http://localhost:14080/auth/admin` . After login, you should be able to see 
the same sessions in tab `Sessions` of particular user, client or realm on all 4 servers

9.4) After doing any change (eg. update some user), the update should be immediatelly visible on any of 4 nodes as caches should be properly invalidated everywhere.


9.5) Check server.logs if needed. After login or logout, the message like this should be on all the nodes `NODEXY/standalone/log/server.log` :

```
2017-08-25 17:35:17,737 DEBUG [org.keycloak.models.sessions.infinispan.remotestore.RemoteCacheSessionListener] (Client-Listener-sessions-30012a77422542f5) Received event from remote store. 
Event 'CLIENT_CACHE_ENTRY_REMOVED', key '193489e7-e2bc-4069-afe8-f1dfa73084ea', skip 'false'
```

This is just a starting point and the instructions are subject to change. We plan various improvements especially around performance. If you 
have any feedback regarding cross-dc scenario, please let us know on keycloak-user mailing list referred from [Keycloak home page](http://www.keycloak.org/community.html).