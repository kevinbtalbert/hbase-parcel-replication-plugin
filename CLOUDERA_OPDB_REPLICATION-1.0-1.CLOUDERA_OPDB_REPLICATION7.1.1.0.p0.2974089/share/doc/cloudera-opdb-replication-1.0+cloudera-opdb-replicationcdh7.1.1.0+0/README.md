# cloudera-opdb-replication  
A plugin for Apache HBase 2.2.3+ to simplify ssl authenticated replication between clusters. Basically we use a custom replication endpoint, to allow it to specify a different sasl ticket when establishing the connection to the remote cluster. 

## Building the project
> mvn -s ~/.m2/settings.xml clean install

The ~/.m2/settings.xml should contain necessary settings to access dependencies:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <mirrors>
    <mirror>
      <id>public</id>
      <mirrorOf>*,!IN-QA,!LW-IN-QA,!cloudera.snapshots.repo,!cloudera.repos</mirrorOf>
      <url>http://nexus-private.hortonworks.com/nexus/content/groups/public</url>
    </mirror>
  </mirrors>
  <pluginGroups>
    <pluginGroup>org.sonatype.plugins</pluginGroup>
  </pluginGroups>
<profiles>
  <profile>
     <id>allow-snapshots</id>
        <activation><activeByDefault>true</activeByDefault></activation>
     <repositories>
       <repository>
         <id>snapshots-repo</id>
         <url>https://oss.sonatype.org/content/repositories/snapshots</url>
         <releases><enabled>true</enabled></releases>
         <snapshots><enabled>true</enabled></snapshots>
       </repository>
     </repositories>
   </profile>
</profiles>
</settings>
```
## Installation
Installation requires a user with root access. The following steps are used when replicating to a DataHub cluster. In any other case Step 3 will be different.
1. Copy the necessary jars to **each** RegionServer on both the source and the destination cluster
	- create the directory **/usr/local/lib/hbase-cdp-jars**
	- copy **target/cloudera-opdb-replication-`<version>`-bin.tar.gz** to the remote machine and extract it
	> tar -xvf cloudera-opdb-replication-`<version>`-bin.tar.gz
	- copy the extracted jars to the new directory
	> cp cloudera-opdb-replication-`<version>`/* /usr/local/lib/hbase-cdp-jars
	- set **hbase** as the owner
	> chown -R hbase:hbase /usr/local/lib/hbase-cdp-jars
2. On **both** the source and the destination cluster find the following hbase config properties using CM: 
	- HBase RegionServer Advanced Configuration Snippet (Safety Valve) for hbase-site.xml
	- HBase Client Environment Advanced Configuration Snippet (Safety Valve) for `hbase-env.sh`
To each add the new key-value pair:
	- Key: HBASE_CLASSPATH
	- Value: "$HBASE_CLASSPATH:/usr/local/lib/hbase-cdp-jars/*"
3. On **any** of the destination cluster's RegionServers create the system user. Please note you will have to use the **csso_$USER** for this step, the 'cloudbreak' user doesn't have the right privileges on the environment ipa:
	> ipa user-add hbase-replication --first=HBase --last=Replication --displayname="HBase Replication"
	- Set the password for the new user (i.e.: "2020-Cloudera-Hbase-CDP-rep") The password defined here should be at least 8 chars long, and include alphanumeric values. Please note we will need this password shortly when generate the keystore
	> ipa passwd hbase-replication
	- Before we could configure **hbase-replication** user for replication, IPA requires a password reset. So we should kinit here. IPA will show a "password expired" message and request a password reset. You may define a new password, or define the same one, as long as it follows the rules mentioned above. In this guide we keep using the same password.
	> kinit hbase-replication
4. On the machine from previous set generate the keystore, with the password step for **hbase-replication** (replace “2020-Cloudera-Hbase-CDP-rep” with the pasword used if different):
	> sudo -u hbase hbase com.cloudera.hbase.security.token.CldrReplicationSecurityTool -sharedkey cloudera -password 2020-Cloudera-Hbase-CDP-rep -keystore localjceks://file/tmp/credentials.jceks
5. Add the freshly generated keystore to the hdfs on **both** clusters
	> hdfs dfs -mkdir /hbase-replication

	> hdfs dfs -put /tmp/credentials.jceks /hbase-replication

	> hdfs dfs -chown -R hbase:hbase /hbase-replication
6. Map hostnames/ip. Collect every RegionServer and ZooKeeper instance's hostname and ip from the destination cluster and add them to the /etc/hosts files on **each** source RegionServer. Please note this might not be needed if name resolution between hosts from the two clusters is already set, say, via a common DNS for example
7. On **both** the source and the destination cluster find the following hbase config property using CM:
	- HBase RegionServer Advanced Configuration Snippet (Safety Valve) for hbase-site.xml:
	Click "View as XML" and add:
	```xml
	<property>
	  <name>hbase.security.external.authenticator</name>
	  <value>com.cloudera.hbase.security.pam.PamAuthenticator</value>
	</property>
	<property>
	  <name>hbase.security.replication.credential.provider.path</name>
	  <value>cdprepjceks://hdfs@ns1/hbase-replication/credentials.jceks</value>
	</property>
	<property>
	  <name>hbase.security.replication.user.name</name>
	  <value>hbase-replication</value>
	</property>
	<property>
	  <name>hbase.client.sasl.provider.extras</name>
	  <value>com.cloudera.hbase.security.provider.CldrPlainSaslClientAuthenticationProvider</value>
	</property>
	<property>
	  <name>hbase.client.sasl.provider.class</name>  
	  <value>com.cloudera.hbase.security.provider.CldrPlainSaslAuthenticationProviderSelector</value>
	</property>
	<property>
	  <name>hbase.server.sasl.provider.extras</name>
	  <value>com.cloudera.hbase.security.provider.CldrPlainSaslServerAuthenticationProvider</value>
	</property>
	```
	- Please note you will have to customize the **hbase.security.replication.credential.provider.path** value. "ns1" should be replaced with the name service value if HA, or with the active NameNode address otherwise. The values on each cluster will be different!
8. Using the hbase shell define replication peer at source cluster, specifying the pluggable replication endpoint implementation:
	> hbase shell
	
	> add_peer '1', ENDPOINT_CLASSNAME => 'com.cloudera.hbase.replication.CldrHBaseInterClusterReplicationEndpoint', CLUSTER_KEY => '10.65.19.121:2181:/hbase'
	
9. At this point replication setup is done. The only thing left is to enable replication by setting the REPLICATION_SCOPE for the selected table and collum family in the hbase shell. Please note `enable_table_replication` command would not work and should not be used:
	> hbase shell
	  
	>  alter 'my-table', {NAME=>'cf', REPLICATION_SCOPE => '1'}

## Non-DataHub destination cluster
To make it work Linux PAM have to be already configured (This is already covered in our CloudCat clusters). Also replace Step 3 of the installation process with:
- Create the technical user on **every** destination RegionServer
    > useradd hbase-replication
    - Set the password for the new user (i.e.: "2020-Cloudera-Hbase-CDP-rep") The password defined here should be at least 8 chars long, and include alphanumeric values. Please note we will need this password shortly when generate the keystore                                                       
    > passwd hbase-replication
