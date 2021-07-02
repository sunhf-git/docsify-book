# 使用场景
nifi单机ETL出现了性能瓶颈，任务过多数量过大时会导致cpu100%并且jvm出现卡顿，并以TLS用户认证对公网开通s2s功能。nifi集群原理请转到[官网](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html#clustering) 阅读, [中文文档](https://nifichina.github.io/general/AdminGuide.html#%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AE)

# 解决方案
当优化任务流和JVM参数仍然不能满足需求后，可采用NIFI集群方案解决问题，可以根据实际任务压力对nifi进行水平扩展，甚至可以进行弹性扩容。

# NIFI集群
- 系统：centos 7.8
- 机器配置： 2G内存+4核心CPU+120G硬盘 * 3
- nifi版本：1.11.4
- 机器1 host: 127.0.0.1 ca.nifiserver.com 0.0.0.0 s61.nifiserver.com 192.168.2.62 s62.nifiserver.com 192.168.2.63 s63.nifiserver.com
- 机器2 host: 192.168.2.61 s61.nifiserver.com 0.0.0.0 s62.nifiserver.com 192.168.2.63 s63.nifiserver.com
- 机器3 host: 192.168.2.61 s61.nifiserver.com 192.168.2.62 s62.nifiserver.com 0.0.0.0 s63.nifiserver.com


# 开始部署
### 安装java
执行以下脚本后重启机器 java即安装完成(地址可能会失效，自己重新下载java安装包就行了)
```shell script
#!/usr/bin/env bash
JAVA_RES="http://down.ownroute.pw/public/jdk-8u131-linux-x64.tar.gz";
JAVA_HOME="/apps/java8"
if ! test -d $JAVA_HOME; then
    echo "start install java..."
    yum install -y wget
    wget -q $JAVA_RES && tar -zxf jdk-8u131-linux-x64.tar.gz && rm -rf jdk-8u131-linux-x64.tar.gz
    rm -rf $JAVA_HOME && mkdir -p $JAVA_HOME && mv jdk*/* $JAVA_HOME && rm -rf jdk*
    echo "
    export JAVA_HOME=$JAVA_HOME
    export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
    export PATH=\$PATH:\$JAVA_HOME/bin
    export JAVA_HOME CLASSPATH PATH
    " >> /etc/profile
    source /etc/profile
    java -version
    echo "java install is done!"
fi
source /etc/profile
java -version
```
## 配置Zookeeper集群
网上资料太多就不写了，参考[zk集群搭建](https://www.jianshu.com/p/a5fda39f20d0) [zk高可用集群搭建](https://www.cnblogs.com/cyfonly/p/5626532.html)


## 申请TLS证书
```shell script
# 登录192.168.2.60机器配置好host后进行以下操作
#启动nifi-toolkit server
bin/tls-toolkit.sh server -c ca.nifiserver.com -t differentKeyAndKeystorePasswords &

mkdir s61 s62 s63
# 生成nifi服务证书
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=s61.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks s61
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=s62.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks s62
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=s63.nifiserver.com,OU=NIFI
mv config.json keystore.jks nifi-cert.pem truststore.jks s63

#生成浏览器用的证书
mkdir browser
bin/tls-toolkit.sh client -c ca.nifiserver.com -t differentKeyAndKeystorePasswords -D CN=browseruser,OU=NIFI -T PKCS12
mv config.json keystore.pkcs12 nifi-cert.pem truststore.jks browser/
```
## 配置NIFI TLS和ZK集群
编辑conf/nifi.properties文件，修改web,security,cluster部分
```properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Core Properties #
nifi.flow.configuration.file=./conf/flow.xml.gz
nifi.flow.configuration.archive.enabled=true
nifi.flow.configuration.archive.dir=./conf/archive/
nifi.flow.configuration.archive.max.time=30 days
nifi.flow.configuration.archive.max.storage=500 MB
nifi.flow.configuration.archive.max.count=
nifi.flowcontroller.autoResumeState=true
nifi.flowcontroller.graceful.shutdown.period=10 sec
nifi.flowservice.writedelay.interval=500 ms
nifi.administrative.yield.duration=30 sec
# If a component has no work to do (is "bored"), how long should we wait before checking again for work?
nifi.bored.yield.duration=10 millis
nifi.queue.backpressure.count=10000
nifi.queue.backpressure.size=1 GB

nifi.authorizer.configuration.file=./conf/authorizers.xml
nifi.login.identity.provider.configuration.file=./conf/login-identity-providers.xml
nifi.templates.directory=./conf/templates
nifi.ui.banner.text=
nifi.ui.autorefresh.interval=30 sec
nifi.nar.library.directory=./lib
nifi.nar.library.autoload.directory=./extensions
nifi.nar.working.directory=./work/nar/
nifi.documentation.working.directory=./work/docs/components

####################
# State Management #
####################
nifi.state.management.configuration.file=./conf/state-management.xml
# The ID of the local state provider
nifi.state.management.provider.local=local-provider
# The ID of the cluster-wide state provider. This will be ignored if NiFi is not clustered but must be populated if running in a cluster.
nifi.state.management.provider.cluster=zk-provider
# Specifies whether or not this instance of NiFi should run an embedded ZooKeeper server
nifi.state.management.embedded.zookeeper.start=false
# Properties file that provides the ZooKeeper properties to use if <nifi.state.management.embedded.zookeeper.start> is set to true
nifi.state.management.embedded.zookeeper.properties=./conf/zookeeper.properties


# H2 Settings
nifi.database.directory=./database_repository
nifi.h2.url.append=;LOCK_TIMEOUT=25000;WRITE_DELAY=0;AUTO_SERVER=FALSE

# FlowFile Repository
nifi.flowfile.repository.implementation=org.apache.nifi.controller.repository.WriteAheadFlowFileRepository
nifi.flowfile.repository.wal.implementation=org.apache.nifi.wali.SequentialAccessWriteAheadLog
nifi.flowfile.repository.directory=./flowfile_repository
nifi.flowfile.repository.partitions=256
nifi.flowfile.repository.checkpoint.interval=2 mins
nifi.flowfile.repository.always.sync=false
nifi.flowfile.repository.encryption.key.provider.implementation=
nifi.flowfile.repository.encryption.key.provider.location=
nifi.flowfile.repository.encryption.key.id=
nifi.flowfile.repository.encryption.key=

nifi.swap.manager.implementation=org.apache.nifi.controller.FileSystemSwapManager
nifi.queue.swap.threshold=20000
nifi.swap.in.period=5 sec
nifi.swap.in.threads=1
nifi.swap.out.period=5 sec
nifi.swap.out.threads=4

# Content Repository
nifi.content.repository.implementation=org.apache.nifi.controller.repository.FileSystemRepository
nifi.content.claim.max.appendable.size=1 MB
nifi.content.claim.max.flow.files=100
nifi.content.repository.directory.default=./content_repository
nifi.content.repository.archive.max.retention.period=12 hours
nifi.content.repository.archive.max.usage.percentage=50%
nifi.content.repository.archive.enabled=true
nifi.content.repository.always.sync=false
nifi.content.viewer.url=../nifi-content-viewer/
nifi.content.repository.encryption.key.provider.implementation=
nifi.content.repository.encryption.key.provider.location=
nifi.content.repository.encryption.key.id=
nifi.content.repository.encryption.key=

# Provenance Repository Properties
nifi.provenance.repository.implementation=org.apache.nifi.provenance.WriteAheadProvenanceRepository
nifi.provenance.repository.debug.frequency=1_000_000
nifi.provenance.repository.encryption.key.provider.implementation=
nifi.provenance.repository.encryption.key.provider.location=
nifi.provenance.repository.encryption.key.id=
nifi.provenance.repository.encryption.key=

# Persistent Provenance Repository Properties
nifi.provenance.repository.directory.default=./provenance_repository
nifi.provenance.repository.max.storage.time=24 hours
nifi.provenance.repository.max.storage.size=1 GB
nifi.provenance.repository.rollover.time=30 secs
nifi.provenance.repository.rollover.size=100 MB
nifi.provenance.repository.query.threads=2
nifi.provenance.repository.index.threads=2
nifi.provenance.repository.compress.on.rollover=true
nifi.provenance.repository.always.sync=false
# Comma-separated list of fields. Fields that are not indexed will not be searchable. Valid fields are:
# EventType, FlowFileUUID, Filename, TransitURI, ProcessorID, AlternateIdentifierURI, Relationship, Details
nifi.provenance.repository.indexed.fields=EventType, FlowFileUUID, Filename, ProcessorID, Relationship
# FlowFile Attributes that should be indexed and made searchable.  Some examples to consider are filename, uuid, mime.type
nifi.provenance.repository.indexed.attributes=
# Large values for the shard size will result in more Java heap usage when searching the Provenance Repository
# but should provide better performance
nifi.provenance.repository.index.shard.size=500 MB
# Indicates the maximum length that a FlowFile attribute can be when retrieving a Provenance Event from
# the repository. If the length of any attribute exceeds this value, it will be truncated when the event is retrieved.
nifi.provenance.repository.max.attribute.length=65536
nifi.provenance.repository.concurrent.merge.threads=2


# Volatile Provenance Respository Properties
nifi.provenance.repository.buffer.size=100000

# Component Status Repository
nifi.components.status.repository.implementation=org.apache.nifi.controller.status.history.VolatileComponentStatusRepository
nifi.components.status.repository.buffer.size=1440
nifi.components.status.snapshot.frequency=1 min

# Site to Site properties
nifi.remote.input.host=s63.nifiserver.com
nifi.remote.input.secure=true
nifi.remote.input.socket.port=10443
nifi.remote.input.http.enabled=true
nifi.remote.input.http.transaction.ttl=30 sec
nifi.remote.contents.cache.expiration=30 secs

# S2S Routing for HTTP
nifi.remote.route.http.nginx1.when=${X-ProxyHost:contains('.nifiserver.com')}
nifi.remote.route.http.nginx1.hostname=p63.nifiserver.com
nifi.remote.route.http.nginx1.port=443
nifi.remote.route.http.nginx1.secure=true

nifi.remote.route.raw.nginx.when=\
${X-ProxyHost:equals('p63.nifiserver.com'):or(\
${s2s.source.hostname:equals('p63.nifiserver.com'):or(\
${s2s.source.hostname:equals('192.168.2.60')})})}
nifi.remote.route.raw.nginx.hostname=p63.nifiserver.com
nifi.remote.route.raw.nginx.port=7444
nifi.remote.route.raw.nginx.secure=true


# web properties #
nifi.web.war.directory=./lib
nifi.web.http.host=
nifi.web.http.port=
nifi.web.http.network.interface.default=
nifi.web.https.host=s63.nifiserver.com
nifi.web.https.port=9443
nifi.web.https.network.interface.default=
nifi.web.jetty.working.directory=./work/jetty
nifi.web.jetty.threads=200
nifi.web.max.header.size=16 KB
nifi.web.proxy.context.path=
nifi.web.proxy.host=

# security properties #
nifi.sensitive.props.key=
nifi.sensitive.props.key.protected=
nifi.sensitive.props.algorithm=PBEWITHMD5AND256BITAES-CBC-OPENSSL
nifi.sensitive.props.provider=BC
nifi.sensitive.props.additional.keys=

nifi.security.keystore=./conf/keystore.jks
nifi.security.keystoreType=jks
nifi.security.keystorePasswd=Y65QmrPkkmJA3yHDok/CEjBaFsqtzfQS1Qu5eaxIk5M
nifi.security.keyPasswd=Y65QmrPkkmJA3yHDok/CEjBaFsqtzfQS1Qu5eaxIk5M
nifi.security.truststore=./conf/truststore.jks
nifi.security.truststoreType=jks
nifi.security.truststorePasswd=zjXeH2IM5VxCvRVeBRWrdnLzL+Gu/4uT+e4NYAMbeHI
nifi.security.user.authorizer=managed-authorizer
nifi.security.user.login.identity.provider=
nifi.security.ocsp.responder.url=
nifi.security.ocsp.responder.certificate=

# OpenId Connect SSO Properties #
nifi.security.user.oidc.discovery.url=
nifi.security.user.oidc.connect.timeout=5 secs
nifi.security.user.oidc.read.timeout=5 secs
nifi.security.user.oidc.client.id=
nifi.security.user.oidc.client.secret=
nifi.security.user.oidc.preferred.jwsalgorithm=
nifi.security.user.oidc.additional.scopes=
nifi.security.user.oidc.claim.identifying.user=

# Apache Knox SSO Properties #
nifi.security.user.knox.url=
nifi.security.user.knox.publicKey=
nifi.security.user.knox.cookieName=hadoop-jwt
nifi.security.user.knox.audiences=

# Identity Mapping Properties #
# These properties allow normalizing user identities such that identities coming from different identity providers
# (certificates, LDAP, Kerberos) can be treated the same internally in NiFi. The following example demonstrates normalizing
# DNs from certificates and principals from Kerberos into a common identity string:
#
# nifi.security.identity.mapping.pattern.dn=^CN=(.*?), OU=(.*?), O=(.*?), L=(.*?), ST=(.*?), C=(.*?)$
# nifi.security.identity.mapping.value.dn=$1@$2
# nifi.security.identity.mapping.transform.dn=NONE
# nifi.security.identity.mapping.pattern.kerb=^(.*?)/instance@(.*?)$
# nifi.security.identity.mapping.value.kerb=$1@$2
# nifi.security.identity.mapping.transform.kerb=UPPER

# Group Mapping Properties #
# These properties allow normalizing group names coming from external sources like LDAP. The following example
# lowercases any group name.
#
# nifi.security.group.mapping.pattern.anygroup=^(.*)$
# nifi.security.group.mapping.value.anygroup=$1
# nifi.security.group.mapping.transform.anygroup=LOWER

# cluster common properties (all nodes must have same values) #
nifi.cluster.protocol.heartbeat.interval=5 sec
nifi.cluster.protocol.is.secure=true

# cluster node properties (only configure for cluster nodes) #
nifi.cluster.is.node=true
nifi.cluster.node.address=s63.nifiserver.com
nifi.cluster.node.protocol.port=11443
nifi.cluster.node.protocol.threads=10
nifi.cluster.node.protocol.max.threads=50
nifi.cluster.node.event.history.size=25
nifi.cluster.node.connection.timeout=5 sec
nifi.cluster.node.read.timeout=5 sec
nifi.cluster.node.max.concurrent.requests=100
nifi.cluster.firewall.file=
nifi.cluster.flow.election.max.wait.time=5 mins
nifi.cluster.flow.election.max.candidates=

# cluster load balancing properties #
nifi.cluster.load.balance.host=
nifi.cluster.load.balance.port=6342
nifi.cluster.load.balance.connections.per.node=4
nifi.cluster.load.balance.max.thread.count=8
nifi.cluster.load.balance.comms.timeout=30 sec

# zookeeper properties, used for cluster management #
nifi.zookeeper.connect.string=192.168.2.61:2181,192.168.2.62:2181,192.168.2.63:2181
nifi.zookeeper.connect.timeout=3 secs
nifi.zookeeper.session.timeout=3 secs
nifi.zookeeper.root.node=/nifi

# Zookeeper properties for the authentication scheme used when creating acls on znodes used for cluster management
# Values supported for nifi.zookeeper.auth.type are "default", which will apply world/anyone rights on znodes
# and "sasl" which will give rights to the sasl/kerberos identity used to authenticate the nifi node
# The identity is determined using the value in nifi.kerberos.service.principal and the removeHostFromPrincipal
# and removeRealmFromPrincipal values (which should align with the kerberos.removeHostFromPrincipal and kerberos.removeRealmFromPrincipal
# values configured on the zookeeper server).
nifi.zookeeper.auth.type=
nifi.zookeeper.kerberos.removeHostFromPrincipal=
nifi.zookeeper.kerberos.removeRealmFromPrincipal=

# kerberos #
nifi.kerberos.krb5.file=

# kerberos service principal #
nifi.kerberos.service.principal=
nifi.kerberos.service.keytab.location=

# kerberos spnego principal #
nifi.kerberos.spnego.principal=
nifi.kerberos.spnego.keytab.location=
nifi.kerberos.spnego.authentication.expiration=12 hours

# external properties files for variable registry
# supports a comma delimited list of file locations
nifi.variable.registry.properties=

# analytics properties #
nifi.analytics.predict.enabled=false
nifi.analytics.predict.interval=3 mins
nifi.analytics.query.interval=5 mins
nifi.analytics.connection.model.implementation=org.apache.nifi.controller.status.analytics.models.OrdinaryLeastSquares
nifi.analytics.connection.model.score.name=rSquared
nifi.analytics.connection.model.score.threshold=.90
```

接着修改conf/authorizers.xml其他几台机器都用同样的配置就可以
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with3  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at
      http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->
<!--
    This file lists the userGroupProviders, accessPolicyProviders, and authorizers to use when running securely. In order
    to use a specific authorizer it must be configured here and it's identifier must be specified in the nifi.properties file.
    If the authorizer is a managedAuthorizer, it may need to be configured with an accessPolicyProvider and an userGroupProvider.
    This file allows for configuration of them, but they must be configured in order:

    ...
    all userGroupProviders
    all accessPolicyProviders
    all Authorizers
    ...
-->
<authorizers>

    <!--
        The FileUserGroupProvider will provide support for managing users and groups which is backed by a file
        on the local file system.

        - Users File - The file where the FileUserGroupProvider will store users and groups.

        - Legacy Authorized Users File - The full path to an existing authorized-users.xml that will be automatically
            be used to load the users and groups into the Users File.

        - Initial User Identity [unique key] - The identity of a users and systems to seed the Users File. The name of
            each property must be unique, for example: "Initial User Identity A", "Initial User Identity B",
            "Initial User Identity C" or "Initial User Identity 1", "Initial User Identity 2", "Initial User Identity 3"

            NOTE: Any identity mapping rules specified in nifi.properties will also be applied to the user identities,
            so the values should be the unmapped identities (i.e. full DN from a certificate).
    -->
    <userGroupProvider>
        <identifier>file-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.FileUserGroupProvider</class>
        <property name="Users File">./conf/users.xml</property>
        <property name="Legacy Authorized Users File"></property>

        <property name="Initial User Identity 1">CN=browseruser, OU=NIFI</property>
        <property name="Initial User Identity 2">CN=s61.nifiserver.com, OU=NIFI</property>
        <property name="Initial User Identity 3">CN=s62.nifiserver.com, OU=NIFI</property>
        <property name="Initial User Identity 4">CN=s63.nifiserver.com, OU=NIFI</property>
    </userGroupProvider>

    <!--
        The LdapUserGroupProvider will retrieve users and groups from an LDAP server. The users and groups
        are not configurable.

        'Authentication Strategy' - How the connection to the LDAP server is authenticated. Possible
            values are ANONYMOUS, SIMPLE, LDAPS, or START_TLS.

        'Manager DN' - The DN of the manager that is used to bind to the LDAP server to search for users.
        'Manager Password' - The password of the manager that is used to bind to the LDAP server to
            search for users.

        'TLS - Keystore' - Path to the Keystore that is used when connecting to LDAP using LDAPS or START_TLS.
        'TLS - Keystore Password' - Password for the Keystore that is used when connecting to LDAP
            using LDAPS or START_TLS.
        'TLS - Keystore Type' - Type of the Keystore that is used when connecting to LDAP using
            LDAPS or START_TLS (i.e. JKS or PKCS12).
        'TLS - Truststore' - Path to the Truststore that is used when connecting to LDAP using LDAPS or START_TLS.
        'TLS - Truststore Password' - Password for the Truststore that is used when connecting to
            LDAP using LDAPS or START_TLS.
        'TLS - Truststore Type' - Type of the Truststore that is used when connecting to LDAP using
            LDAPS or START_TLS (i.e. JKS or PKCS12).
        'TLS - Client Auth' - Client authentication policy when connecting to LDAP using LDAPS or START_TLS.
            Possible values are REQUIRED, WANT, NONE.
        'TLS - Protocol' - Protocol to use when connecting to LDAP using LDAPS or START_TLS. (i.e. TLS,
            TLSv1.1, TLSv1.2, etc).
        'TLS - Shutdown Gracefully' - Specifies whether the TLS should be shut down gracefully
            before the target context is closed. Defaults to false.

        'Referral Strategy' - Strategy for handling referrals. Possible values are FOLLOW, IGNORE, THROW.
        'Connect Timeout' - Duration of connect timeout. (i.e. 10 secs).
        'Read Timeout' - Duration of read timeout. (i.e. 10 secs).

        'Url' - Space-separated list of URLs of the LDAP servers (i.e. ldap://<hostname>:<port>).
        'Page Size' - Sets the page size when retrieving users and groups. If not specified, no paging is performed.
        'Sync Interval' - Duration of time between syncing users and groups (i.e. 30 mins). Minimum allowable value is 10 secs.
        'Group Membership - Enforce Case Sensitivity' - Sets whether group membership decisions are case sensitive. When a user or group
            is inferred (by not specifying or user or group search base or user identity attribute or group name attribute) case sensitivity
            is enforced since the value to use for the user identity or group name would be ambiguous. Defaults to false.

        'User Search Base' - Base DN for searching for users (i.e. ou=users,o=nifi). Required to search users.
        'User Object Class' - Object class for identifying users (i.e. person). Required if searching users.
        'User Search Scope' - Search scope for searching users (ONE_LEVEL, OBJECT, or SUBTREE). Required if searching users.
        'User Search Filter' - Filter for searching for users against the 'User Search Base' (i.e. (memberof=cn=team1,ou=groups,o=nifi) ). Optional.
        'User Identity Attribute' - Attribute to use to extract user identity (i.e. cn). Optional. If not set, the entire DN is used.
        'User Group Name Attribute' - Attribute to use to define group membership (i.e. memberof). Optional. If not set
            group membership will not be calculated through the users. Will rely on group membership being defined
            through 'Group Member Attribute' if set. The value of this property is the name of the attribute in the user ldap entry that
            associates them with a group. The value of that user attribute could be a dn or group name for instance. What value is expected
            is configured in the 'User Group Name Attribute - Referenced Group Attribute'.
        'User Group Name Attribute - Referenced Group Attribute' - If blank, the value of the attribute defined in 'User Group Name Attribute'
            is expected to be the full dn of the group. If not blank, this property will define the attribute of the group ldap entry that
            the value of the attribute defined in 'User Group Name Attribute' is referencing (i.e. name). Use of this property requires that
            'Group Search Base' is also configured.

        'Group Search Base' - Base DN for searching for groups (i.e. ou=groups,o=nifi). Required to search groups.
        'Group Object Class' - Object class for identifying groups (i.e. groupOfNames). Required if searching groups.
        'Group Search Scope' - Search scope for searching groups (ONE_LEVEL, OBJECT, or SUBTREE). Required if searching groups.
        'Group Search Filter' - Filter for searching for groups against the 'Group Search Base'. Optional.
        'Group Name Attribute' - Attribute to use to extract group name (i.e. cn). Optional. If not set, the entire DN is used.
        'Group Member Attribute' - Attribute to use to define group membership (i.e. member). Optional. If not set
            group membership will not be calculated through the groups. Will rely on group membership being defined
            through 'User Group Name Attribute' if set. The value of this property is the name of the attribute in the group ldap entry that
            associates them with a user. The value of that group attribute could be a dn or memberUid for instance. What value is expected
            is configured in the 'Group Member Attribute - Referenced User Attribute'. (i.e. member: cn=User 1,ou=users,o=nifi vs. memberUid: user1)
        'Group Member Attribute - Referenced User Attribute' - If blank, the value of the attribute defined in 'Group Member Attribute'
            is expected to be the full dn of the user. If not blank, this property will define the attribute of the user ldap entry that
            the value of the attribute defined in 'Group Member Attribute' is referencing (i.e. uid). Use of this property requires that
            'User Search Base' is also configured. (i.e. member: cn=User 1,ou=users,o=nifi vs. memberUid: user1)

        NOTE: Any identity mapping rules specified in nifi.properties will also be applied to the user identities.
            Group names are not mapped.
    -->
    <!-- To enable the ldap-user-group-provider remove 2 lines. This is 1 of 2.
    <userGroupProvider>
        <identifier>ldap-user-group-provider</identifier>
        <class>org.apache.nifi.ldap.tenants.LdapUserGroupProvider</class>
        <property name="Authentication Strategy">START_TLS</property>

        <property name="Manager DN"></property>
        <property name="Manager Password"></property>

        <property name="TLS - Keystore"></property>
        <property name="TLS - Keystore Password"></property>
        <property name="TLS - Keystore Type"></property>
        <property name="TLS - Truststore"></property>
        <property name="TLS - Truststore Password"></property>
        <property name="TLS - Truststore Type"></property>
        <property name="TLS - Client Auth"></property>
        <property name="TLS - Protocol"></property>
        <property name="TLS - Shutdown Gracefully"></property>

        <property name="Referral Strategy">FOLLOW</property>
        <property name="Connect Timeout">10 secs</property>
        <property name="Read Timeout">10 secs</property>

        <property name="Url"></property>
        <property name="Page Size"></property>
        <property name="Sync Interval">30 mins</property>
        <property name="Group Membership - Enforce Case Sensitivity">false</property>

        <property name="User Search Base"></property>
        <property name="User Object Class">person</property>
        <property name="User Search Scope">ONE_LEVEL</property>
        <property name="User Search Filter"></property>
        <property name="User Identity Attribute"></property>
        <property name="User Group Name Attribute"></property>
        <property name="User Group Name Attribute - Referenced Group Attribute"></property>

        <property name="Group Search Base"></property>
        <property name="Group Object Class">group</property>
        <property name="Group Search Scope">ONE_LEVEL</property>
        <property name="Group Search Filter"></property>
        <property name="Group Name Attribute"></property>
        <property name="Group Member Attribute"></property>
        <property name="Group Member Attribute - Referenced User Attribute"></property>
    </userGroupProvider>
    To enable the ldap-user-group-provider remove 2 lines. This is 2 of 2. -->

    <!--
        The ShellUserGroupProvider provides support for retrieving users and groups by way of shell commands
        on systems that support `sh`.  Implementations available for Linux and Mac OS, and are selected by the
        provider based on the system property `os.name`.

        'Refresh Delay' - duration to wait between subsequent refreshes.  Default is '5 mins'.
        'Exclude Groups' - regular expression used to exclude groups.  Default is '', which means no groups are excluded.
        'Exclude Users' - regular expression used to exclude users.  Default is '', which means no users are excluded.
        'Legacy Identifier Mode' - preserves the legacy behavior for id generation. Disabling this will ensure that
                                    user and group ids are differentiated to handle the case where a user and group have
                                    the same identity. Default is 'true', which means users and groups are not differentiated.
    -->
    <!-- To enable the shell-user-group-provider remove 2 lines. This is 1 of 2.
    <userGroupProvider>
        <identifier>shell-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.ShellUserGroupProvider</class>
        <property name="Refresh Delay">5 mins</property>
        <property name="Exclude Groups"></property>
        <property name="Exclude Users"></property>
        <property name="Legacy Identifier Mode">true</property>
    </userGroupProvider>
    To enable the shell-user-group-provider remove 2 lines. This is 2 of 2. -->

    <!--
        The CompositeUserGroupProvider will provide support for retrieving users and groups from multiple sources.

        - User Group Provider [unique key] - The identifier of user group providers to load from. The name of
            each property must be unique, for example: "User Group Provider A", "User Group Provider B",
            "User Group Provider C" or "User Group Provider 1", "User Group Provider 2", "User Group Provider 3"

            NOTE: Any identity mapping rules specified in nifi.properties are not applied in this implementation. This behavior
            would need to be applied by the base implementation.
    -->
    <!-- To enable the composite-user-group-provider remove 2 lines. This is 1 of 2.
    <userGroupProvider>
        <identifier>composite-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.CompositeUserGroupProvider</class>
        <property name="User Group Provider 1"></property>
    </userGroupProvider>
    To enable the composite-user-group-provider remove 2 lines. This is 2 of 2. -->

    <!--
        The CompositeConfigurableUserGroupProvider will provide support for retrieving users and groups from multiple sources.
        Additionally, a single configurable user group provider is required. Users from the configurable user group provider
        are configurable, however users loaded from one of the User Group Provider [unique key] will not be.

        - Configurable User Group Provider - A configurable user group provider.

        - User Group Provider [unique key] - The identifier of user group providers to load from. The name of
            each property must be unique, for example: "User Group Provider A", "User Group Provider B",
            "User Group Provider C" or "User Group Provider 1", "User Group Provider 2", "User Group Provider 3"

            NOTE: Any identity mapping rules specified in nifi.properties are not applied in this implementation. This behavior
            would need to be applied by the base implementation.
    -->

    <!-- To enable the composite-configurable-user-group-provider remove 2 lines. This is 1 of 2.
    <userGroupProvider>
        <identifier>composite-configurable-user-group-provider</identifier>
        <class>org.apache.nifi.authorization.CompositeConfigurableUserGroupProvider</class>
        <property name="Configurable User Group Provider">file-user-group-provider</property>
        <property name="User Group Provider 1"></property>
    </userGroupProvider>
    To enable the composite-configurable-user-group-provider remove 2 lines. This is 2 of 2. -->

    <!--
        The FileAccessPolicyProvider will provide support for managing access policies which is backed by a file
        on the local file system.

        - User Group Provider - The identifier for an User Group Provider defined above that will be used to access
            users and groups for use in the managed access policies.

        - Authorizations File - The file where the FileAccessPolicyProvider will store policies.

        - Initial Admin Identity - The identity of an initial admin user that will be granted access to the UI and
            given the ability to create additional users, groups, and policies. The value of this property could be
            a DN when using certificates or LDAP, or a Kerberos principal. This property will only be used when there
            are no other policies defined. If this property is specified then a Legacy Authorized Users File can not be specified.

            NOTE: Any identity mapping rules specified in nifi.properties will also be applied to the initial admin identity,
            so the value should be the unmapped identity. This identity must be found in the configured User Group Provider.

        - Legacy Authorized Users File - The full path to an existing authorized-users.xml that will be automatically
            converted to the new authorizations model. If this property is specified then an Initial Admin Identity can
            not be specified, and this property will only be used when there are no other users, groups, and policies defined.

            NOTE: Any users in the legacy users file must be found in the configured User Group Provider.

        - Node Identity [unique key] - The identity of a NiFi cluster node. When clustered, a property for each node
            should be defined, so that every node knows about every other node. If not clustered these properties can be ignored.
            The name of each property must be unique, for example for a three node cluster:
            "Node Identity A", "Node Identity B", "Node Identity C" or "Node Identity 1", "Node Identity 2", "Node Identity 3"

            NOTE: Any identity mapping rules specified in nifi.properties will also be applied to the node identities,
            so the values should be the unmapped identities (i.e. full DN from a certificate). This identity must be found
            in the configured User Group Provider.

        - Node Group - The name of a group containing NiFi cluster nodes. The typical use for this is when nodes are dynamically
            added/removed from the cluster.

            NOTE: The group must exist before starting NiFi.
    -->
    <accessPolicyProvider>
        <identifier>file-access-policy-provider</identifier>
        <class>org.apache.nifi.authorization.FileAccessPolicyProvider</class>
        <property name="User Group Provider">file-user-group-provider</property>
        <property name="Authorizations File">./conf/authorizations.xml</property>
        <property name="Initial Admin Identity">CN=browseruser, OU=NIFI</property>
        <property name="Legacy Authorized Users File"></property>
        <property name="Node Identity 1">CN=s61.nifiserver.com, OU=NIFI</property>
        <property name="Node Identity 2">CN=s62.nifiserver.com, OU=NIFI</property>
        <property name="Node Identity 3">CN=s63.nifiserver.com, OU=NIFI</property>
        <property name="Node Group"></property>
    </accessPolicyProvider>

    <!--
        The StandardManagedAuthorizer. This authorizer implementation must be configured with the
        Access Policy Provider which it will use to access and manage users, groups, and policies.
        These users, groups, and policies will be used to make all access decisions during authorization
        requests.

        - Access Policy Provider - The identifier for an Access Policy Provider defined above.
    -->
    <authorizer>
        <identifier>managed-authorizer</identifier>
        <class>org.apache.nifi.authorization.StandardManagedAuthorizer</class>
        <property name="Access Policy Provider">file-access-policy-provider</property>
    </authorizer>

    <!--
        NOTE: This Authorizer has been replaced with the more granular approach configured above with the Standard
        Managed Authorizer. However, it is still available for backwards compatibility reasons.

        The FileAuthorizer is NiFi's provided authorizer and has the following properties:

        - Authorizations File - The file where the FileAuthorizer will store policies.

        - Users File - The file where the FileAuthorizer will store users and groups.

        - Initial Admin Identity - The identity of an initial admin user that will be granted access to the UI and
            given the ability to create additional users, groups, and policies. The value of this property could be
            a DN when using certificates or LDAP, or a Kerberos principal. This property will only be used when there
            are no other users, groups, and policies defined. If this property is specified then a Legacy Authorized
            Users File can not be specified.

            NOTE: Any identity mapping rules specified in nifi.properties will also be applied to the initial admin identity,
            so the value should be the unmapped identity.

        - Legacy Authorized Users File - The full path to an existing authorized-users.xml that will be automatically
            converted to the new authorizations model. If this property is specified then an Initial Admin Identity can
            not be specified, and this property will only be used when there are no other users, groups, and policies defined.

        - Node Identity [unique key] - The identity of a NiFi cluster node. When clustered, a property for each node
            should be defined, so that every node knows about every other node. If not clustered these properties can be ignored.
            The name of each property must be unique, for example for a three node cluster:
            "Node Identity A", "Node Identity B", "Node Identity C" or "Node Identity 1", "Node Identity 2", "Node Identity 3"

            NOTE: Any identity mapping rules specified in nifi.properties will also be applied to the node identities,
            so the values should be the unmapped identities (i.e. full DN from a certificate).
    -->
    <!-- <authorizer>
        <identifier>file-provider</identifier>
        <class>org.apache.nifi.authorization.FileAuthorizer</class>
        <property name="Authorizations File">./conf/authorizations.xml</property>
        <property name="Users File">./conf/users.xml</property>
        <property name="Initial Admin Identity"></property>
        <property name="Legacy Authorized Users File"></property>

        <property name="Node Identity 1"></property>
    </authorizer>
    -->
</authorizers>

```
配置conf/state-management.xml,zk-provider添加上zk的集群地址就行了
```xml
 <cluster-provider>
        <id>zk-provider</id>
        <class>org.apache.nifi.controller.state.providers.zookeeper.ZooKeeperStateProvider</class>
        <property name="Connect String">192.168.2.61:2181,192.168.2.62:2181,192.168.2.63:2181</property>
        <property name="Root Node">/nifi</property>
        <property name="Session Timeout">10 seconds</property>
        <property name="Access Control">Open</property>
    </cluster-provider>
```
## 启动NIFI
分别启动bin/nifi.sh start 就可以了，刚开始选举主nifi的时候可能有点慢，选举期间进不去nifi，多等一会。
# 总结

