Shibboleth IdP 3.2.1 on Debian Jessie using Jetty and Postgres 
============

disordered mashed install notes installing the following software:
* shibboleth IdP 3.2.1
* oracle java 8
* Jetty 9.3
* Postgres

For an eduGAIN IdP you will need roughly 3 Gig ram dual core VM
mainly for the needs of unoptimized shibboleth metadata parsing.

## Why Postgres ?

It is the more free database software around and you can implement
foreign connectors for nearly the totality of data soures that fills
the Identity management galaxy of your Institution. You can also
easily implement "caching tables" so that IdP does not fail
authentication because one of their backend is not reachable: old data
in most cases is preferrable to total authentication failure.


## Install Oracle Java 8 (shortcut)
-----------------------------------
```su -```

#### Add a debian-working ppa.launchpad/ubuntu java installer repository

```
echo "deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main" | tee /etc/apt/sources.list.d/webupd8team-java.list
echo "deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main" | tee -a /etc/apt/sources.list.d/webupd8team-java.list
```
#### Import signing key

```apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys EEA14886```
#### Update package list
```apt-get update```
#### Install Java jvm and Cryptography Extensions
```apt-get install oracle-java8-installer oracle-java8-unlimited-jce-policy```
#### Add the following environment variable in /etc/profile.d/java.sh
```
echo "export JAVA_HOME=/usr/lib/jvm/java-8-oracle" >> /etc/profile.d/java.sh
exit
```
#### Make your session aware of this environment (may be useful further on in Shibboleth installation)
```export JAVA_HOME=/usr/lib/jvm/java-8-oracle```


## Install and first configure Shibboleth
---------------------

#### Download shibboleth idp from `https://shibboleth.net/products/` e.g.:
```
cd /opt
wget https://shibboleth.net/downloads/identity-provider/latest/shibboleth-identity-provider-3.2.1.tar.gz
```

Verify SHA1 signature

Unpack the downloaded archive.
`tar xf shibboleth-identity-provider-3.2.1.tar.gz`

Go to the root of shibboleth dir and issue `./bin/install.sh` command.
Following configuration is referred to an IdP named `idemfero.units.it`, please  it to your needs.


```
/opt/shibboleth-identity-provider-3.1.1#  ./bin/install.sh
Source (Distribution) Directory: [/opt/shibboleth-identity-provider-3.2.1]
<enter>
Installation Directory: [/opt/shibboleth-idp]
<enter>
Hostname: [gainfero.localdomain]
idemfero.units.it
SAML EntityID: [https://idemfero.units.it/idp/shibboleth]

Attribute Scope: [localdomain]
*.units.it
Backchannel PKCS12 Password:
<backchanpassword>
Re-enter password: 
<backchanpassword>
Cookie Encryption Key Password: 
<cookiepassword>
Re-enter password: 
<cookiepassword>
Warning: /opt/shibboleth-idp/bin does not exist.
Warning: /opt/shibboleth-idp/dist does not exist.
Warning: /opt/shibboleth-idp/doc does not exist.
Warning: /opt/shibboleth-idp/system does not exist.
Warning: /opt/shibboleth-idp/webapp does not exist.
Generating Signing Key, CN = idemfero.units.it URI = https://idemfero.units.it/idp/shibboleth ...
...done
Creating Encryption Key, CN = idemfero.units.it URI = https://idemfero.units.it/idp/shibboleth ...
...done
Creating Backchannel keystore, CN = idemfero.units.it URI = https://idemfero.units.it/idp/shibboleth ...
...done
Creating cookie encryption key files...
...done
Rebuilding /opt/shibboleth-idp/war/idp.war ...
...done

BUILD SUCCESSFUL
Total time: 2 minutes 36 seconds
```
Modify /opt/jetty/jetty-base/start.d/idp.ini appending:
```
Path to IdP WAR (dir or file), relative to ${jetty.base} directory
# jetty.war.path=../webapp
jetty.war.path=../../shibboleth-idp/war/idp.war
```

## Install and configure Jetty
----------------

#### Download Jetty. e.g.
```
cd /opt
wget http://download.eclipse.org/jetty/stable-9/dist/jetty-distribution-9.3.7.v20160115.tar.gz
```

Verify SHA1 signature

unpack jetty
```
tar xf jetty-distribution-9.3.7.v20160115.tar.gz
ln -s jetty-distribution-9.3.7.v20160115 jetty
cd /opt/jetty
cp bin/jetty.sh /etc/init.d/jetty
echo JETTY_HOME=`pwd` > /etc/default/jetty
echo "JETTY_BASE=/opt/jetty/jetty-base" >> /etc/default/jetty
echo "TMPDIR=/tmp" >> /etc/default/jetty
```

#### Quick-Start a Jetty Service - Section
[Jetty as a unix Service](http://www.eclipse.org/jetty/documentation/9.1.2.v20140210/startup-unix-service.html Jetty Startup) 

To avoid this error during startup: `insserv: warning: script 'jetty' missing LSB tags and overrides`, add this on top of /etc/init.d/jetty:

```
### BEGIN INIT INFO
# Provides:          jetty
# Required-Start:    $local_fs $network
# Required-Stop:     $local_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Jetty startscript.
# Description:       Starts our Jetty server.
### END INIT INFO
```

Enable Debian Systemd service configuration

```systemctl enable jetty```

and start the service

```service jetty start```

#### Redirect https traffic from port 443 to Jetty port 8443

Jetty, by default, runs as an unprivileged user (very good). But non root
users have no access to ports below 1024 such as the 443 https one.

So we need to statically redirect port 443 to the default unprivileged Jetty 8443 port.
This can be easily done 
Add the previous lines in a shell script and recall it using post-up
directive under the corresponding interface in /etc/network/interfaces.
e.g. (see last line):

```
iface eth0 inet static
    address 10.108.45.152
    gateway 10.108.45.254
    netmask 255.255.255.0
    broadcast 10.108.45.255
    post-up /etc/adminscripts/jetty-redirect.sh
```
Where `jetty-redirect.sh` has the following contents:

```
#!/bin/sh
iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443
```

Now that the IdP installation prepared the `/opt/jetty/jetty-base` container,
we can go on customizing Jetty features for our IdP webapp.

Modify `/opt/jetty/jetty-base/start.ini` as described in https://wiki.shibboleth.net/confluence/display/IDP30/Jetty93

**Note:** Memory for java is dimensioned on a 25 MB federation metadata file. 
      The metadata parsing process seems to be the most memory intensive operation for Shibboleth.


#### Setup SSL
See (http://www.eclipse.org/jetty/documentation/9.3.0.v20150612/configuring-ssl.html)

Collect key, cert and cacert in, say, /opt/certs/ directory
```mkdir /opt/certs/```


#### Get a signed certificate
Since Jetty manages the user/browser-facing channel, we need a certificate
signed by one of the Certificate Authorities trusted by browsers.
Please remember to include in this request all possible shibboleth cluster FQDNs.

`# openssl req -new -out idemfero.csr -nodes`

This also generates the private key `privkey.pem`.

`chmod 000 privkey.pem`

#### Prepare certs
Once obtained the signed certificate, prepare the full chain file and enclose all in a pkcs12 keystore:

```
cat idemfero.crt intermediate.crt [intermediate2.crt]... rootCA.crt > cert-chain.txt
openssl pkcs12 -export -inkey privkey.pem -in cert-chain.txt -out idemfero.p12
```

You will be asked a password to protect the key store.

edit `/opt/jetty/etc/jetty-ssl-context.xml` and modify the following attributes accordingly
`KeyManagerPassword`
`KeyStorePassword`

In `/opt/jetty/jetty-base/start.d/idp.ini` customize the following entries:

```
# Absolute path to keystores
#jetty.backchannel.keystore.path=../credentials/idp-backchannel.p12
jetty.backchannel.keystore.path=/opt/shibboleth-idp/credentials/idp-backchannel.p12
#jetty.browser.keystore.path=../credentials/idp-userfacing.p12
jetty.browser.keystore.path=/opt/certs/idemfero.p12

# Keystore passwords
jetty.backchannel.keystore.password=changeit
jetty.browser.keystore.password=changeit


jetty.http.host=0.0.0.0
jetty.host=0.0.0.0
```

#### Jetty and Strict Transport Security (optional)

HSTS is an emerging web-server-to-browser standard to mitigate man in the middle ssl-strip attacks
tipically found around client network. (https://it.wikipedia.org/wiki/HTTP_Strict_Transport_Security)
We can easily enable HSTS in Jetty:

edit /opt/jetty/jetty-base/etc/jetty-rewrite.xml and add a Rule as 
```
<Call name="addRule">
    <Arg>
        <New class="org.eclipse.jetty.rewrite.handler.HeaderPatternRule">
            <Set name="pattern">*</Set>
            <Set name="name">Strict-Transport-Security</Set>
            <Set name="value">max-age=15768000; includeSubDomains</Set>
        </New>
    </Arg>
</Call>
```
**WARNING** Please nothe that you need to have no hosts serving http named something.yourIdP_FQDN
or you will have *serious service disruption* client-side implementing this directives.

Add the rewrite module to /opt/jetty/jetty-base/etc/start.ini

```
# Module to enable header rewriting and support HSTS
--module=rewrite
```

#### Place and link static content and pesonalizations

Put static content such as favicon.ico and logos in 
`/opt/jetty/static/`

Disable Jetty mad default favicon servicing:

In `/opt/jetty/jetty-base/etc/jetty.xml` change

`<New id="DefaultHandler" class="org.eclipse.jetty.server.handler.DefaultHandler" />`

to

`<New id="DefaultHandler" class="org.eclipse.jetty.server.handler.DefaultHandler"> <Set name="serveIcon">false</Set> </New>`

and do a

`service jetty restart`

#### test-run jetty

`root@gainfero:/opt/jetty/jetty-base# java -jar ../start.jar`

-----------------------------------------------------------------------------------------------------
#### Copy Shibboleth jetty-base inside Jetty dir /opt/jetty
`# cp -Rp /opt/shibboleth-identity-provider-3.2.1/embedded/jetty-base /opt/jetty/`

-----------------------------------------------------------------------------------------------------

## Database Connection Poolers and Drivers

Download connection pooling and db client JARs:

```
wget http://search.maven.org/remotecontent?filepath=com/zaxxer/HikariCP/2.4.3/HikariCP-2.4.3.jar
wget https://jdbc.postgresql.org/download/postgresql-9.4.1208.jar
```

Place the driver jar and connection pooling jar in `/opt/shibboleth-idp/edit-webapp/WEB-INF/lib`
chdir to the IdP basedir `/opt/shibboleth-idp` and then execute `bin/build.sh`

Files will be moved in place:

```
/opt/shibboleth-idp/edit-webapp/WEB-INF/lib/
/opt/shibboleth-idp/webapp/WEB-INF/lib/
/opt/jetty/jetty-base/lib/
```

**Note:** Java version incompatibilities and missing jars are poorly or not at all reported in logs.
      Pay attention at what you do here.

-----------------------------------------------------------------------------------------------------

# IDP Configuration

## IdP admin interface access control

In order to reload parts (services) of the IdP runtime and to show running stats,
you'll need to add local admin clients to `access-control.xml` e.g.:

```
    <util:map id="shibboleth.AccessControlPolicies">

        <entry key="AccessByIPAddress">
            <bean parent="shibboleth.IPRangeAccessControl"
                p:allowedRanges="#{ {'127.0.0.1/32', '::1/128', '170.10.11.0/28'} }" />
        </entry>

    </util:map>
```

and test it going to 

`https://<your server>/idp/status/`


## Consent configuration

It's possible to give users a one-time per cookie-lifetime
(_idp.properties => idp.consent.storageRecordLifetime = P1Y_) Terms-of-Use overview
and collect their consent.

Modify `intercept/consent-intercept-config.xml` accordingly:

```
<!-- <alias alias="shibboleth.consent.terms-of-use.Key" name="shibboleth.RelyingPartyIdLookup.Simple" /> -->
<bean id="shibboleth.consent.terms-of-use.Key" class="com.google.common.base.Functions" factory-method="constant">
   <constructor-arg value="my-terms"/>
</bean>
```

And `relying-party.xml`:
```
<!--
Default configuration, with default settings applied for all profiles, and enables
the attribute-release consent flow.
-->
<bean id="shibboleth.DefaultRelyingParty" parent="RelyingParty">
    <property name="profileConfigurations">
        <list>
<!-- defaulto no tou                <bean parent="Shibboleth.SSO" p:postAuthenticationFlows="attribute-release" /> -->
            <bean parent="Shibboleth.SSO" p:postAuthenticationFlows="#{ {'terms-of-use', 'attribute-release'} }" />
            <ref bean="SAML1.AttributeQuery" />
            <ref bean="SAML1.ArtifactResolution" />
<!-- defaulto no tou                 <bean parent="SAML2.SSO" p:postAuthenticationFlows="attribute-release" /> -->
            <bean parent="SAML2.SSO" p:postAuthenticationFlows="#{ {'terms-of-use', 'attribute-release'} }" />
	            <ref bean="SAML2.ECP" />
            <ref bean="SAML2.Logout" />
            <ref bean="SAML2.AttributeQuery" />
            <ref bean="SAML2.ArtifactResolution" />
            <ref bean="Liberty.SSOS" />
        </list>
    </property>
</bean>

```

#### Translate user-facing messages

Put the appropriate files for your languagein `/opt/shibboleth-idp/messages`
 folder.

Some languages are translated and available from:
`https://wiki.shibboleth.net/confluence/display/IDP30/MessagesTranslation`

UTF-8 language files needs adding the corresponding encoding in `system/conf/global-System.Xml` in the `<id bean = "MessageSource" ... />` block add:

 `p: defaultEncoding = "UTF-8"`



#### CSS and other customizations
Graphic appearance and other css drivem customization can be easily done modifying files under

`/opt/shibboleth-idp/edit-webapp/css`

Change or upload images like this `edit-webapp/images/file.png`

Then rebuild the app

`bin/build.sh`

and restart the app container Jetty

`service jetty restart`

#### Enable the generation of the persistent identifier

This replaces the legacy SAML1 attribute *eduPersonTargetedID*.
There is an ongoing discussion on deprecating *eduPersonTargetedID*
attribute in favour of using the Subject assertion of *NameID:persistent*.

For some legacy SPs is stil necessary to release the persistent identifier
in both ways. For some misconfigured new apps, is necessary to suppress *eduPersonTargetedID*.

Edit `/opt/shibboleth-idp/conf/saml-nameid.properties`
    (the *sourceAttribute* MUST BE an attribute, or a list of comma-separated attributes, that uniquely identify the subject of the generated ```persistent-id```. It MUST BE: **Stable**, **Permanent** and **Not-reassignable**).
     It usually is _uid_ when using openLDAP and _sAMAccountName_ when using Active Directory.

```
idp.persistentId.sourceAttribute = sAMAccountNAme
...
idp.persistentId.salt = ### result of 'openssl rand -base64 36'###
...
idp.persistentId.generator = shibboleth.StoredPersistentIdGenerator
...
idp.persistentId.store = PersistentIDpostgresStore
idp.persistentId.dataSource = MyPersistentPostgresStore
...
idp.persistentId.computed = shibboleth.ComputedPersistentIdGenerator
```

#### Enable the **SAML2PersistentGenerator**:

By default, only NameID:transient is released.

Edit `/opt/shibboleth-idp/conf/saml-nameid.xml`

Add the following two beans:

```
<!-- ========================= SAML NameID Generation ========================= -->

<!--            p:dataSourceClassName="org.postgresql.Driver" -->
<!--            p:dataSourceClassName="org.postgresql.ds.PGSimpleDataSource" -->
<!-- POSTGRES connection pooling bean configuration for PersistentIdStore -->
<!-- <bean id="MyPersistentPostgresStore" class="net.shibboleth.idp.saml.nameid.impl.JDBCPersistentIdStore"> -->
<bean id="MyPersistentPostgresStore" class="com.zaxxer.hikari.HikariDataSource">
<property name="dataSource">
    <bean class="com.zaxxer.hikari.HikariDataSource"
        p:dataSourceClassName="org.postgresql.ds.PGSimpleDataSource"
        p:jdbcUrl="jdbc:postgresql://localhost:5432/shibboleth"
        p:username="shibboleth"
        p:password="l4ll4m4n70v4n12011"
        p:connectionTestQuery="select 1"
        p:connectionTimeout="30000" p:idleTimeout="300000" p:maxLifetime="1800000" p:maximumPoolSize="10"
        />
</property>
</bean>

<!-- A "store" bean suitable for use in the idp.persistentId.store property. -->
<bean id="PersistentIDpostgresStore" parent="shibboleth.JDBCPersistentIdStore"
p:dataSource-ref="MyPersistentPostgresStore"
p:queryTimeout="PT2S"
p:retryableErrors="#{{'23505'}}" />
```

**Remove** the comment from the line containing:

`<ref bean="shibboleth.SAML2PersistentGenerator" />`

Edit `vim /opt/shibboleth-idp/conf/c14n/subject-c14n.xml`

**Remove** the comment to the bean called "**c14n/SAML2Persistent**".

Modify the ***DefaultRelyingParty*** to releasing of the "persistent-id" to all, ever:
`vim /opt/shibboleth-idp/conf/relying-party.xml`

```
<bean id="shibboleth.DefaultRelyingParty" parent="RelyingParty">
<property name="profileConfigurations">
  <list>
      <bean parent="Shibboleth.SSO" p:postAuthenticationFlows="attributerelease" />
      <ref bean="SAML1.AttributeQuery" />
      <ref bean="SAML1.ArtifactResolution" />
      <bean parent="SAML2.SSO" p:postAuthenticationFlows="attribute-release" p:nameIDFormatPrecedence="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent" />
      <ref bean="SAML2.ECP" />
      <ref bean="SAML2.Logout" />
      <ref bean="SAML2.AttributeQuery" />
      <ref bean="SAML2.ArtifactResolution" />
      <ref bean="Liberty.SSOS" />
  </list>
</property>
</bean>
```


# Addendum to simple configuration
===================================

# Developement and debugging tricks

* Poison your client /etc/hosts file  to test your developement idp.
* Use an http proxy to reach your production IdP from the poisoned client.
Under Firefox you can use Foxyproxy extension to rapidly switch production and developement environments.
* Prepare an php sp exposing all attributes to check the asssertion consumer point of view.


# Multiple ldap backends (needs testing)

Create a new ldap instance inside password-authn-config.xml
```
cp -p ldap-authn-config.xml ldap-personal-authn-config.xml
sed -i s/idp.authn.LDAP/idp.authn.LDAP-personalized/g ldap-personal-authn-config.xml
```
In _ldap.properties_ duplicte the ldap section renaming all the config parameter names:

`idp.authn.LDAP => idp.authn.LDAP-personalized`

idp.pool.LDAP attrs remains defined only once.

-----------------------------------------------------------------------------------------------------

Reference links:

http://en.wikipedia.org/wiki/Spring_Framework
http://en.wikipedia.org/wiki/Java_Persistence_API
https://www.switch.ch/aai/support/presentations/techupdate-2014/06_IdPv3.pdf
https://wiki.shibboleth.net/confluence/display/CONCEPT/IdPSkills
https://wiki.shibboleth.net/confluence/display/CONCEPT/ECP
http://docs.oasis-open.org/security/saml/Post2.0/saml-ecp/v2.0/cs01/
https://wiki.shibboleth.net/confluence/display/DEV/IdP3Details
http://www.slideshare.net/Codemotion/jetty-9-the-next-generation-servlet-container
https://github.com/Unicon/unicon-shibboleth-idp-v3-template

-----------------------------------------------------------------------------------------------------

# Implementing Oracle foreign Data conrctor for Postgres
#### Oracle clent

Download

```
libs instantclient-basic-linux.x64-12.1.0.2.0.zip
exe instantclient-sqlplus-linux.x64-12.1.0.2.0.zip
headers instantclient-sdk-linux.x64-12.1.0.2.0.zip
```
Unzip it in /opt/oracle/instantclient_12_1

In /etc/profilei.d/oracle  add:

`export PATH=/opt/oracle/instantclient_12_1:$PATH`

and run it from commandliane.

#### Update library path

```
echo "/opt/oracle/instantclient_12_1" > /etc/ld.so.conf.d/instaclient.conf
ldconfig
```

Test using

`sqlplus ORAUSER/password@//<oracle db fqdn>:1521/username`


### Download and install oracle_fdw - A Foreign Data Wrapper for Oracle

Download from http://pgxn.org/dist/oracle_fdw/

```
wget http://api.pgxn.org/dist/oracle_fdw/1.3.0/oracle_fdw-1.3.0.zip
apt-get install postgresql-server-dev-9.4
export ORACLE_HOME=/opt/oracle/instantclient
cd /opt/oracle/instantclient
make
make install
/etc/init.d/postgresql restart
```

Activate the foreign data wrapper and map a local table to a remote one.
Following is just an example table to get out a tax ID number from a DB.

```
# su - postgres
$ psql
psql (9.4.6)
Type "help" for help.

postgres=# \c shibboleth
You are now connected to database "shibboleth" as user "postgres".

postgres=# CREATE EXTENSION oracle_fdw;
CREATE EXTENSION

postgres=# CREATE SERVER dataugov FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver '//<oracle db fqdn>:1521/ugov');
CREATE SERVER
postgres=#  GRANT USAGE ON FOREIGN SERVER dataugov TO postgres;
GRANT
postgres=#  CREATE USER MAPPING FOR postgres SERVER dataugov OPTIONS (user 'orauser', password 'orapassword');
CREATE USER MAPPING

postgres=# CREATE FOREIGN TABLE oracle_TUTTI_AUTENT (
          USER_ID   character varying(80),
          AD        integer,
          GRP_ID    integer,
          DESCR_GRP character varying(10),
          IDENTIFICATIVO  integer,
          COGNOME   character varying(400),
          NOME	    character varying(400),
          COD_FIS   character varying(64)
       ) SERVER dataugov OPTIONS (schema 'QUERYESSE3', table 'TUTTI_AUTENT');


postgres=# CREATE MATERIALIZED VIEW TUTTI_AUTENT AS SELECT * from oracle_TUTTI_AUTENT;
SELECT 170374

postgres=# REFRESH MATERIALIZED VIEW TUTTI_AUTENT;
REFRESH MATERIALIZED VIEW

(mettere a cron di postgres)
14 4 * * * /bin/echo "REFRESH MATERIALIZED VIEW TUTTI_AUTENT;" | /usr/bin/psql -d shibboleth

postgres=# GRANT SELECT ON TUTTI_AUTENT TO shibboleth;
```


#### jar Oracle client

Alternatively, you can configure the oracle connector directly for us in shibboleth.

You should be able to see the client jar under the following locations:
```
/opt/shibboleth-idp/edit-webapp/WEB-INF/lib/ojdbc7.jar
/opt/shibboleth-idp/webapp/WEB-INF/lib/ojdbc7.jar
/opt/jetty/jetty-base/lib/ojdbc7.jar
```

# Credits
- Daniele Albrizio - University of Trieste
- Marco Malavolti - GARR



