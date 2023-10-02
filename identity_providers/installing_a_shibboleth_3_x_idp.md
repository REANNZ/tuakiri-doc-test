---
redirect_from:
 - Installing+a+Shibboleth+3.x+IdP
 - Installing+a+Shibboleth+3x+IdP
id: identity_providers/installing_a_shibboleth_3_x_idp
---
# Installing a Shibboleth 3.x IdP

This page is a guide to installing a Shibboleth 3.x IdP - based on the [Installing a Shibboleth 2.x IdP](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985355/Installing+a+Shibboleth+2.x+IdP) page, but updated for Shibboleth IdP version 3. This page assumes the IdP would be installed on a minimal-OS-install-only Linux system (typically a _virtual machine_) and follows from that point on. The IdP will be installed with the [Shibboleth IdP](http://shibboleth.internet2.edu/) application.

This guide is periodically updated as new versions of the software installed become available. This guide is current for IdP 3.4.0, the latest versions available as of October 2018. The guide assumes the Linux distribution would be CentOS/RHEL 7 with Tomcat7 (required). It should be possible to also use this for other Linux distribution / other operating systems, varying as needed.

If you are interested in upgrading an existing IdP to the latest release, please see [Upgrading a Shibboleth 3.x IdP](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985149/Upgrading+a+Shibboleth+3.x+IdP).

For instructions on upgrading a 2.x IdP to 3.x, please see [Upgrading a 2.x IdP to 3.x](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985147/Upgrading+a+2.x+IdP+to+3.x).

If you are interested in linking an existing IdP into Tuakiri, please see [Configuring a Shibboleth Identity Provider to join the Tuakiri Federation](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985363/Configuring+a+Shibboleth+Identity+Provider+to+join+the+Tuakiri+Federation).

1. TOC
{:toc}

# Prerequisites

*   A Linux system (VM) preinstalled with base OS distribution, with current updates installed.
    *   Minimum hardware recommendation: 2GB RAM, 20GB diskspace
    *   Linux distribution: recommended RHEL 7 or CentOS 7

*   Network requirements:
    *   Allocating a hostname (typically `idp.institution.domain.ac.nz`) and a static IP address.
    *   Setup firewall rules:
        *   Enable incoming and outgoing TCP connections to ports 443 and 8443. Note that these connections **MUST** be direct connections and cannot go through a proxy. In particular, the port 8443 connections use client SSL authentication and will not work with proxies that try to step in the middle of an SSL connection, even if the server certificate is pre-loaded onto the proxy.
        *   Enable connections to an external NTP server or provide an internal NTP server.

*   Provide an X509 certificate for the allocated hostname issued by a CA trusted by major browsers.
    *   (There are no further requirements on the certificate, it would be only accessed from inside a browser)

*   Access to an institutional LDAP server (with a system account allowed to read all attributes needed for the federation). This typically includes the following information:
    *   LDAP server hostname and port number
    *   Search base (subtree DN)
    *   Bind DN for a generic reader account
    *   Bind password for this account
    *   Search filter - i.e. what attribute contains the user's login name?

## Modifications to Identity Management system

Your Identity Management System (IdMS) will very likely have most of the attributes asked for by the federation - or will have enough information to synthesize the specific attribute values on the fly inside the IdP. But for some attributes, the IdMS might not have enough information. The following information should be considered for adding into your IdMS:

*   **eduPersonEntitlement**: The eduPersonEntitlement attribute is a storage container for values representing privileges to access resources within the federation. It is a multi-valued string attribute. The values will have the form of a URI - with specific values that are yet to be defined. The attribute definition details are (source: [Attribute Recommendation 2.1 (PDF)](http://www.aaf.edu.au/wp-content/uploads/2012/05/auEduPerson_attribute_vocabulary_v02-1-0.pdf), page 14):
    
    ```
    Origin/ObjectClass:   eduPerson [eduPerson]
    OID:                  1.3.6.1.4.1.5923.1.1.1.7
    SAML attribute name:  urn:mace:dir:attribute-def:eduPersonEntitlement
    LDAP syntax:          directoryString [1.3.6.1.4.1.1466.115.121.1.15]
    Number of values:     Multiple
    Example values:       eduPersonEntitlement: urn:mace:washington.edu:confocalMicroscope
                          eduPersonEntitlement: http://publisher.example.com/contract/GL123
    ```
    
    This attribute can also be defined as a static attribute. If you would prefer not to modify your IdMS schema and do not have any eduPersonEntitlement values to release, it is OK to initially only define the attribute as static inside the IdP. For more information on this option, please see the notes on [configuring eduPersonEntitlement as a static attribute](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-eduPersonEntitlement) further below in this document.
    

*   **auEduPersonSharedToken**: The auEduPersonSharedToken uniquely identifies users when accessing certain resources - particularly within the _computational grid_ and _data grid_. The values should be **opaque**, **non-reassignable** and **persistent** - and **transferrable** when a user moves between institutions. Even though the values are typically created as hash-values on first use, they MUST be stored and each institution must be ready to accept values users already have when coming from another institution. The attribute can be stored in either the IdMS directly (preferred) or in a database. The attribute definition details are (source: [Attribute Recommendation 2.1 (PDF)](http://www.aaf.edu.au/wp-content/uploads/2012/05/auEduPerson_attribute_vocabulary_v02-1-0.pdf), pages 9-10, with OID updated to correct value):
    
    ```
    Origin/ObjectClass:   auEduPerson
    OID:                  1.3.6.1.4.1.27856.1.2.5
    SAML attribute name   urn:mace:federation.org.au:attribute:auEduPersonSharedToken
    LDAP syntax:          directoryString [1.3.6.1.4.1.27856.1.2.5]
    Number of values:     Single
    Example values:       ZsiAvfxa0BXULgcz7QXknbGtfxk
    ```
    
    *   See also the [auEduPerson LDAP Schema Definition](http://www.aaf.edu.au/wp-content/uploads/2012/05/auEduPerson_attribute_vocabulary_v02-1-0.pdf) (pages 45-52) for exact LDAP definition snippets.
        
        This auEduPersonSharedToken values can also be stored locally in a MySQL database. If you would prefer not to modify your IdMS schema, you can also choose the local database option - at the cost of not having all of your primary identity information in your IdMS. Please see the instructions on [defining the sharedToken attribute](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-DefinesharedToken) further below for more detail.
        

*   **eduPersonAssurance**: This attribute represents the [Levels of Assurance](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808986245/Levels+of+Assurance). Either add the attribute into the IdMS directly, or start collecting enough information to synthesize the values later in a _scripted attribute definition_ (like done for Affiliation below).  The attribute definition details are (source: [Attribute Recommendation 2.1 (PDF)](http://www.aaf.edu.au/wp-content/uploads/2012/05/auEduPerson_attribute_vocabulary_v02-1-0.pdf), page 13):
    
    ```
    Origin/ObjectClass:   eduPerson
    OID:                  1.3.6.1.4.1.5923.1.1.1.11
    SAML attribute name:  urn:oid:1.3.6.1.4.1.5923.1.1.1.11
    LDAP syntax:          directoryString [1.3.6.1.4.1.1466.115.121.1.15]
    Number of values:     multiple
    Example values:       See AAF IdentityLoA Vocabulary
    ```
    
    *   For an overview of the requirements for individual assurance levels, please refer to the AAF documentation at [http://www.aaf.edu.au/technical/levels-of-assurance/](http://www.aaf.edu.au/technical/levels-of-assurance/)
    *   For detailed information on the requirements, please refer to the [NIST SP 800-63-1\* standard](http://csrc.nist.gov/publications/nistpubs/800-63-1/SP-800-63-1.pdf)
        
        As the federation is moving to centralized management of Levels-Of-Assurance, it is recommend to at this moment only define the attribute as a static attribute releasing the floor-of-trust values. Please see the notes on [configuring eduPersonAssurance as a static attribute](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-eduPersonAssurance) further below in this document.
        

# Preliminaries

## System firewall configuration

A typical default firewall configuration on RedHat systems permits only incoming SSH.  To permit incoming connections to ports 443 and 8443:

*   On RHEL/CentOS 7 with **firewalld** (and with a single default zone) run:
    
    ```
    firewall-cmd --add-service=https
    firewall-cmd --add-port=8443/tcp
    firewall-cmd --permanent --add-service=https
    firewall-cmd --permanent --add-port=8443/tcp
    ```
    
*   On systems using plain iptables (more likely on RHEL/CentOS 6), edit `/etc/sysconfig/iptables` and add rules to permit incoming traffic to ports 443 and 8443; add the following just below the rule for port 22:
    
    ```
    -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT
    -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 8443 -j ACCEPT
    ```
    
      
    
    *   And restart iptables:
        
        ```
        service iptables restart
        ```
        

Please remember that besides the incoming connections discussed here, the IdP also needs outgoing connections to TCP ports 80 and 443, and also to UDP port 514 for Centralized logging (more details below)

Outgoing connections are open in the default configuration of a local RHEL/CentOS firewall, but please make sure the perimeter firewall permits these connections too.

  

# Bootstrapping the VM

We assume a standard install of either CentOS or RHEL, version 7. The IdP web application (as of version 3) needs Tomcat at least at version 7 (or the upstream instructions recommend Jetty). Also, the IdP java binaries are compiled in class format 51.0 and need Java7.  However, given that Java7 is being phased out, the IdP works well with Java8, and the compatibility issues (see the Scripted Attributes section below) are easy to resolve, we recommend installing Java8 on platforms where it's available (which does include CentOS/RHEL 7).

There are known issues with LDAP on Java versions higher then Java8.  We recommend running a 3.x IdP with Java8.  (In IdP 4.x, which requires Java 11, [this issue is taken care of](https://wiki.shibboleth.net/confluence/display/IDP4/LDAPonJava)).

If you need to run a 3.x IdP with Java version higher then Java8, please see the [upstream documentation](https://wiki.shibboleth.net/confluence/display/IDP30/LDAPonJava%3E8).

  

## Install packages

*   Apache with mod\_ssl, Java8 (with javac):
    
    ```
    yum install httpd mod_ssl java-1.8.0-openjdk java-1.8.0-openjdk-devel
    ```
    

*   MySQL (needed for storing sharedToken values if shared Token is stored in MySQL)
    *   MySQL is not strictly required and an alternative database system may be used if already available on site - just use the corresponding JDBC drivers for the alternative database. As of CentOS 7, the available MySQL server package is MariaDB
        
        ```
        yum install mariadb mariadb-server
        ```
        

*   Install NTP (if time synchronization is to be done inside the VM)
    
    ```
    yum install ntp
    ```
    

*   Useful debugging / sysadmin tools
    
    ```
    yum install openldap-clients wireshark-gnome mc strace subversion
    ```
    
*   Install Tomcat:
    
    ```
    yum install tomcat
    ```
    
    Historically, this document was recommending to install Tomcat7 from JPackage.  As this document is being updated for CentOS/RHEL 7, and the default tomcat on these systems is version 7, this is no longer needed.  (Also, the JPackage repository appears to be no longer maintained).  And even for CentOS/RHEL 6, EPEL now provides Tomcat7. The original instructions are at [Install Tomcat 7 on CentOS 6](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985124/Install+Tomcat+7+on+CentOS+6)
    

## Local configuration

*   Configure NTP: time synchronization is crucial for the Shibboleth IdP to correctly interact with Service Providers in the federation. Please gather beforehand the hostname or IP address of the time server provided within your organisation. Further on, we'll assume it's `ntp.institution.domain.ac.nz`. If your organisation is not providing one, you may use an external source (including the default servers listed in `/etc/ntp.conf`) - but make sure your firewall allows the NTP traffic through (outgoing UDP port 123 traffic + reply packets).
    *   Do a one-off synchronization:
        
        ```
        ntpdate -s ntp.institution.domain.ac.nz
        ```
        
    *   Edit `/etc/ntp.conf` and:
        
        *   Comment out local server (`server 127.127.1.0` and `fudge 127.127.1.0`)
        *   Comment out CentOS servers (all lines starting with `server`)
        *   The local time server:
            
            ```
            server ntp.institution.domain.ac.nz
            ```
            
    *   Make ntpd start on system startup:
        
        ```
        chkconfig ntpd on
        ```
        
    *   Start ntpd now:
        
        ```
        service ntpd start
        ```
        
    *   Check ntpd is running and synchronizing: run
        
        ```
        ntpdc -p
        ```
        

SELinux

RHEL/CentOS distributions (both 6 and 7) come with SELinux.

SELinux improves the security of the system and we recommend leaving SELinux turned on.

Up until late 2017, Tomcat was running in the unconfined context and the IdP web application did not directly benefit from SELinux, but even so, at least Apache interactions were controlled by SELinux.  And the only SELinux-specific step this manual had to cover was permitting the Apache to LDAP communication needed for ECP.

As of late 2017 (RHEL/CentOS 7.4), Tomcat runs in a confined domain.  This has serious impact on operating an IdP - as the IdP web application running inside Tomcat now runs under SELinux restrictions, it now needs all actions explicitly permitted.

We still recommend running with SELinux in enforcing mode.  A new section ([Configuring SELinux for Tomcat](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-ConfiguringSELinuxforTomcat)) further below details the steps needed.  Please follow the instructions carefully - otherwise, the IdP would fail to start.

  

## Securing the MySQL server

On RHEL/CentOS7, the `mariadb-server` package by default configures the MySQL server with:

*   several separate root accounts for different forms of the local hostname ( `'localhost'`, `'127.0.0.1'`, `'::1'`, and also the hostname of the system).  These accounts have no initial password set.
*   "anonymous" accounts with blank username - one for `'localhost'`, one for the hostname of the system,
*   a `test` database where the anonymous account has full permissions.

Actual ways of securing the MySQL server may depend on local security policy, but overall, our recommendations are:

*   Setting a password for all root accounts.
*   Removing the anonymous accounts.
*   Removing the `test` database.

The following MySQL code does that - just substitute your chosen password - you can create one e.g. with: `openssl rand -base64 24`

```
DROP DATABASE test;
DROP USER ''@'localhost';
DELETE FROM mysql.user WHERE user='' and host=@@global.hostname;
UPDATE mysql.user SET password=PASSWORD('MySQL-root-password') WHERE user='root';
FLUSH PRIVILEGES;
```

And, you can check the list of users with:

```
SELECT user,host,password FROM mysql.user;
```

There should be no anonymous users (username blank) and all accounts should have a password set (displayed as hash, but should be non-blank).

# Basic Shibboleth IdP installation

## Rationale and planning

From the very beginning, we will install the Shibboleth IdP:

*   as a web application WAR file managed by Tomcat
*   with a Tomcat login screen used for authentication

Having done these steps early in the installation prevents them from slipping later on.

Please note that historically, this guide was separately documenting installation of the uApprove application for users to grant consent over attribute release. This application has now been integrated into the main IdP application as the _consent_ module - and is therefore installed (and enabled) in a basic IdPV3 deployment.

The IdP installation paths (IDP\_HOME) will be `/opt/shibboleth-idp` (default)

Your IdP hostname will be **idp.****_institution.domain.ac.nz_**  
Your scope, home organization name and security domain will be **_institution.domain.ac.nz_**  
Your IdP **entityId** will be `https://idp.institution.domain.ac.nz/idp/shibboleth`

*   The Shibboleth IdP _installation directory_ (from where Shibboleth IdP software is installed) will be `/root/inst/shibboleth-identity-provider-<version>` - and the name of the directory will be stored in `SHIB_INST_HOME` (here, we assume version is `3.2.1`)

*   Configure the corresponding environment variable: create /etc/profile.d/shib.sh with the following content (and make the file executable):
    
    ```
    IDP_VERSION="3.4.0"
    SHIB_HOME=/opt/shibboleth-idp
    SHIB_INST_HOME=/root/inst/shibboleth-identity-provider-$IDP_VERSION
    
    IDP_HOME=/opt/shibboleth-idp
    JAVA_HOME=/usr/lib/jvm/java
    
    export SHIB_HOME IDP_HOME JAVA_HOME SHIB_INST_HOME IDP_VERSION
    ```
    

## Basic Shibboleth Installation

*   Check for the most recent version of Shibboleth IdP at [https://shibboleth.net/downloads/identity-provider/](https://shibboleth.net/downloads/identity-provider/)

*   Create an installation directory and download Shibboleth
    
    ```
    mkdir /root/inst
    cd /root/inst
    wget http://shibboleth.net/downloads/identity-provider/latest/shibboleth-identity-provider-${IDP_VERSION}.tar.gz
    tar xzf shibboleth-identity-provider-${IDP_VERSION}.tar.gz
    cd $SHIB_INST_HOME
    ```
    

*   Invoke installer
    
    ```
    sh ./bin/install.sh
    ```
    
    *   Answer the following questions:
        *   Source (Distribution) Directory: confirm the current directory
        *   Installation directory: accept the proposed value of `/opt/shibboleth-idp` if suitable.
        *   Hostname: enter the user-facing hostname of the IdP, typically **idp.****_institution.domain.ac.nz_**
        *   SAML Entity ID: accept the value derived from the hostname, [https://idp.institution.domain.ac.nz/idp/shibboleth](https://idp.institution.domain.ac.nz/idp/shibboleth)
        *   Attribute Scope: set this to the domain name of your institution ( **institution.domain.ac.nz** in the above example). The value offered based on the hostname should be already correct, but please check and adjust as needed.
        *   Enter the passphrase to protect the generated keystores (back-channel and cookie encryption). It is acceptable to use the phrase "changeit" (as the files are protected by filesystem permissions and the phrase itself is also stored in files on the same filesystem).
            
            *   Note that the installer generates three separate certificates+keypairs for back-channel, signing, and encryption, respectively - and also an encryption key for cookie. Only the back-channel private key is encrypted, the other two private keys will be stored on disk unencrypted.  The passphrase for the cookie encryption key is stored in the generated idp.properties file.  Therefore, there is no need to choose a secure keystore password and instead, it is important to secure access to the system...
            
              
            
    *   This installs the Shibboleth IdP web application into `/opt/shibboleth-idp/war/idp.war`

About re-running the installer

In general, it should not be necessary to re-run the installer.

*   Re-running the installer would overwrite system files under $IDP\_HOME/system, but would preserve configuration files under $IDP\_HOME/conf
*   But the installer would only be re-run to perform an upgrade - running the installer from a newer version **distribution** directory would overwrite system files in the **installation** directory `$IDP_HOME`, but would preserve the (assuming compatible) configuration files in `$IDP_HOME/conf`.

To just rebuild the WAR file, run `$IDP_HOME/bin/build.sh` instead.

  

*   The installer has generated three separate certificates+keypairs for back-channel, signing, and encryption.  The back-channel private key was only stored in a Java keystore, but for Apache, we need it converted to the PEM format.  Run the following command (and when prompted for the keystore passphrase, enter the default passphrase "changeit"):
    
    ```
    openssl pkcs12 -in $IDP_HOME/credentials/idp-backchannel.p12 -out $IDP_HOME/credentials/idp-backchannel.key -nocerts -nodes
    ```
    

## Configure Tomcat and deploy the IdP WAR

*   Create `/etc/tomcat/Catalina/localhost/idp.xml` with the following content:
    
    ```
          <Context docBase="/opt/shibboleth-idp/war/idp.war"
                   unpackWAR="false"
                   swallowOutput="true" />
    ```
    

Historically, the installation process involved deploying XML parser libraries as _endorsed_ libraries in Tomcat. Since IdP 2.4.3, this is no longer needed and the step has been removed.

  

*   Connectors: in `/etc/tomcat/server.xml`, define a new AJP connector at port 8009.
    
    Note
    
    Tomcat7 already has this connector defined, but in an insecure way that would be opening the connector to outside connections as well. Comment the original definition out and instead put this definition in.
    
    ```
       <Connector port="8009" address="127.0.0.1"
                  enableLookups="false" redirectPort="443" protocol="AJP/1.3"
                  tomcatAuthentication="false"
                  secretRequired="false" />
    ```
    
*   And comment out the existing http connector defined for port 8080:
    
    ```
        <!--
        <Connector port="8080" maxHttpHeaderSize="8192"
                   maxThreads="150" minSpareThreads="25" maxSpareThreads="75"
                   enableLookups="false" redirectPort="8443" acceptCount="100"
                   connectionTimeout="20000" disableUploadTimeout="true" />
        -->
    ```
    

*   Tweak Tomcat memory settings: increase the default Java minimal and maximal heap size settings (increasing the maximum to at least 1GB - or more, depending the total RAM in your VM)
    *   Edit `/etc/sysconfig/tomcat` and add (reasonably adjust according to the VM size):
        
        ```
        JAVA_OPTS="-Xms768m -Xmx1536m"
        ```
        
        Historically, due to a [bug](http://www.jpackage.org/bugzilla/show_bug.cgi?id=377) in JPackage bundle of Tomcat7, settings in `/etc/tomcat7/tomcat7.conf` were ignored and we therefore recommended putting all Tomcat settings into `/etc/sysconfig/tomcat7`.
        
        With RHEL/CentOS7 Tomcat being managed by systemd, this bug (in startup script interaction) is now irrelevant.  However, we still recommend putting the memory settings into `/etc/sysconfig/tomcat`, as it is compatible with Tomcat per-instance configuration.
        
*   The IdPV3 code relies on the web application container having support for JSTL, but Tomcat7 comes packaged without JSTL.  Therefore, install JSTL (version 1.2.1, API and implementation jars) into `/usr/share/tomcat/lib`: download from [search.maven.org/remotecontent?filepath=javax/servlet/jsp/jstl/javax.servlet.jsp.jstl-api/1.2.1/javax.servlet.jsp.jstl-api-1.2.1.jar](http://search.maven.org/remotecontent?filepath=javax/servlet/jsp/jstl/javax.servlet.jsp.jstl-api/1.2.1/javax.servlet.jsp.jstl-api-1.2.1.jar) and [search.maven.org/remotecontent?filepath=org/glassfish/web/javax.servlet.jsp.jstl/1.2.1/javax.servlet.jsp.jstl-1.2.1.jar](http://search.maven.org/remotecontent?filepath=org/glassfish/web/javax.servlet.jsp.jstl/1.2.1/javax.servlet.jsp.jstl-1.2.1.jar) (credits: [http://stackoverflow.com/tags/jstl/info](http://stackoverflow.com/tags/jstl/info)):
    
    ```
    wget -O /usr/share/tomcat/lib/javax.servlet.jsp.jstl-api-1.2.1.jar 'http://search.maven.org/remotecontent?filepath=javax/servlet/jsp/jstl/javax.servlet.jsp.jstl-api/1.2.1/javax.servlet.jsp.jstl-api-1.2.1.jar'
    wget -O /usr/share/tomcat/lib/javax.servlet.jsp.jstl-1.2.1.jar 'http://search.maven.org/remotecontent?filepath=org/glassfish/web/javax.servlet.jsp.jstl/1.2.1/javax.servlet.jsp.jstl-1.2.1.jar'
    ```
    

## Configure Apache

Apache needs to be configured to:

*   Listen on ports 443 and 8443 - this is done via separate configuration files idp.conf and idp8443.conf (bypassing parts of the default configuration in ssl.conf)

Note: historically, this guide used to recommend to disable SSL session cache to work around a [bug](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPTroubleshootingCommonErrors#NativeSPTroubleshootingCommonErrors-error:1408F06B:SSLroutines:SSL3_GET_RECORD:baddecompression) - while the bug was never fully tracked, the current version of OpenSSL/ShibbolethSP/httpd/mod\_ssl no longer demonstrate this bug, so for performance reasons, we recommend keep SSL session caching turned on. Also, as the bug was in the back-channel communication which is rarely used with SAML2, even the impact in the unlikely case the bug reoccurs should be quite minimal.

  

*   To make Apache listen at ports 443 and 8443, create `/etc/httpd/conf.d/ports.conf` (or install from [ports.conf](https://raw.githubusercontent.com/REANNZ/Tuakiri-public/master/shibboleth-idp/apache/ports.conf))  with
    
    ```
    Listen 443
    Listen 8443 https
    ```
    

*   OPTIONAL: as it is no longer necessary to listen on port 80 (all incoming connections will be on ports 443 or 8443), comment out the `Listen 80` directive in `/etc/httpd/conf/httpd.conf`
*   To configure the two virtual hosts for the two ports, download the files [idp.conf](https://github.com/REANNZ/Tuakiri-public/raw/master/shibboleth-idp/apache/idp.conf) and [idp8443.conf](https://github.com/REANNZ/Tuakiri-public/raw/master/shibboleth-idp/apache/idp8443.conf) into `/etc/httpd/conf.d`
    *   In both files, replace all occurrences of idp.example.org with the hostname of your IdP
    *   In `idp.conf`, configure the SSL VirtualHost to use the commercial certificate issued for your IdP
        
        Apache 2.4.8+ marks `SSLCertificateChainFile` as deprecated.  The recommended approach on Apache 2.4.8+ for inserting intermediate CA certificates is to append them to the server certificate specified with `SSLCertificateFile` - [sorted from leaf to root](http://httpd.apache.org/docs/2.4/mod/mod_ssl.html#sslcertificatefile).
        
        However, as RHEL/CentOS7 only comes with Apache 2.4.6 (as of August 2016), the recommended approach on these systems is to still use a separate CA certificate chain file with the `SSLCertificateChainFile` directive.
        
    *   Note that idp.conf is configured to disable all access to `<Location /idp/Authn/RemoteUser>` - all users will be authenticating via the Tomcat login screen.
    *   Note that idp.conf (as of January 2018) defaults to only allowing TLS 1.2 (dropping support for TLS 1.0 and TLS 1.1).
        *   This cuts off old legacy clients, including Android<=4.3, IE<=10, Java <=6u45 or 7u25, Safari 5 or 6, or openssl <=0.9.8y
        *   If you need support for these clients, comment out the first `SSLProtocol` line and uncomment the second `SSLProtocol` line
    *   Note that idp8443.conf is using the back-channel certificate generated when deploying Shibboleth IdP, stored in `/opt/shibboleth-idp/credentials`

*   Earlier versions of `idp.conf` (between January and December 2018) were enabling [HSTS](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) by default - instructing browsers to remember to only accept secure connections to the IdP
    
    *   Browser would automatically rewrite all plain HTTP connections to the IdP to HTTPS
    *   If the browser detects a certificate error, it will only display an error message and will NOT offer the option to add an exception.
    *   This is to protect users from being tricked into accepting a malicious site pretending to be the IdP
    
    Since IdP 3.4.0, the IdP provides this functionality (setting the Strict-Transport-Security header) and so the HSTS setting has been retracted from the `idp.conf` template - and we instead recommend to use the functionality built into the IdP application.
    
*   Make a backup of your ssl.conf
    
    ```
    cd /etc/httpd/conf.d/
    cp ssl.conf ssl.conf.dist
    ```
    
*   Edit your ssl.conf and:
    *   **Delete the whole default <VirtualHost> section** from `ssl.conf` (the definitions in idp.conf and idp8443.conf will be used instead).
    *   Delete the entry for `**Listen 443**` (because we now have the directive in `ports.conf`).

## Basic Shibboleth Configuration

Earlier documentation was instructing to create backup copies of all configuration file before modifying them - to easily identify locally made changes. This is no longer necessary since the version 3 installer creates pristine copies of all configuration files under `/opt/shibboleth-idp/dist/conf`

Earlier documentation was instructing to modify the generated metadata and fix the scope. This is no longer necessary because:

*   The installer now specifically asks for scope (instead of just guessing it from the hostname), so the generated metadata should contain the correct scope
*   The generated local IdP metadata is no longer used by the IdP directly.

However, the contents of `$IDP_HOME/metadata/idp-metadata.xml` is served by the IdP at the URL corresponding to the default IdP entityID - `https://idp.institution.ac.nz/idp/shibboleth`  

If this URL is used anywhere to obtain the IdP metadata (we recommend against this practice, as it is insecure and fragile, but this gets used in some bilateral setups), this file will have to be kept up-to-date with the actual IdP metadata.  However, the installer should initially create it with the correct contents.

![](https://reannz.atlassian.net/wiki/plugins/servlet/confluence/placeholder/unknown-macro?name=multiexcerpt-include&locale=en_US&version=2)

  

  

Please note that historically (in IdP 2.x), the IdP was  also loading its own metadata.  This is no longer needed and the `$IDP_HOME/metadata/idp-metadata.xml` file now exists only for informative purposes.

  

Configure the IdP to use secure cookies: by default Shibboleth IdP 3.x uses session cookies not marked as secure - which means the browser would also send them over a plain unencrypted HTTP connection.

Mark the cookies as **secure** by adding the following line into `$IDP_HOME/conf/idp.properties`:

```
idp.cookie.secure = true
```

  

  

## Configure LDAP Authentication

The IdP can authenticate against a number of sources, but here we assume the authentication would be done against an LDAP directory server. For other options, please see theprimary documentation at [https://wiki.shibboleth.net/confluence/display/IDP30/AuthenticationConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/AuthenticationConfiguration).

By default IdPV3 already comes with a login screen (which needs to be branded with institutional logo, as discussed later on below).

While an IdP uses an LDAP server for both authentication and for resolving user attributes, these are configured as separate connections. However, the way configuration files are organized in IdPV3, both of them can be configured in `$IDP_HOME/conf/ldap.properties` - and this way, it is enough to configure the settings just once for the authentication part - and the attribute resolver by default copies the authentication settings.

The exact settings would very much depend on your exact LDAP server settings (and e.g., whether it requires TLS/SSL, whether it's using a self-signed certificate or a private root CA), but the settings below are what would typically need to be done (see the snippet just below for concrete examples):

*   Configure the LDAP URL, base DN, user Filter, bindDN and bindDNCredential in the corresponding properties.
    *   The user filter may need to be adjusted based on which attribute contains the user name in your LDAP - and whether you want to restrict the search with an additional filter.
    *   Note that by default, subtree search is set to off - for LDAPs with accounts scattered across a tree, this must be turned on.
*   Set the authenticator method to bindSearchAuthenticator (i.e., the LDAP client needs to bind with a system account before it can search for user accounts)
*   **In IdP 3.2.x only:** Set the list of attributes to retrieve from LDAP in `idp.attribute.resolver.LDAP.returnAttributes`
*   Configure the TLS and certificate settings according to the specifics of your LDAP server, with the following caveats:
    *   TLS is turned on by default.
    *   The IdP has three different ways of establishing trust for the certificate presented by the LDAP server, specified in the `idp.authn.LDAP.sslConfig` property - defaulting to `certificateTrust`):
        *   `jvmTrust`: use the trust store that comes with the Java VM (default set of trusted CA certificates)
        *   `certificateTrust`: trust a single certificate pointed to by `idp.authn.LDAP.trustCertificates` (defaults to `%{idp.home}/credentials/ldap-server.crt`, can be either the LDAP server certificate directly or a local CA root).
        *   `keyStoreTrust`: trust certificates in a separate keystore, pointed to by `idp.authn.LDAP.trustStore` (defaults to `%{idp.home}/credentials/ldap-server.truststore`, can be either the LDAP server certificate directly or a local CA root).
    *   Even if TLS is turned off, the `idp.authn.LDAP.sslConfig` setting must still point to a policy that can successfully initialize.  And the default `certificateTrust` policy does not initialize if the certificate does not exist.  So with switching TLS off, it is also necessary to to set `idp.authn.LDAP.sslConfig=jvmTrust`

These are typical settings for a simple LDAP server:

```
idp.authn.LDAP.authenticator                    = bindSearchAuthenticator
idp.authn.LDAP.ldapURL                          = ldap://ldap.example.org
idp.authn.LDAP.baseDN                           = ou=People,dc=example,dc=org
idp.authn.LDAP.bindDN                           = cn=read,dc=example,dc=org
idp.authn.LDAP.bindDNCredential                 = PASSWORD-GOES-HERE
# in IdP 3.2.x only
#idp.attribute.resolver.LDAP.returnAttributes    = displayName,mail,uid
idp.authn.LDAP.useStartTLS                      = false
# As we are not setting trustCertificates/trustStore, switch the sslConfig to point to jvmTrust to avoid errors for broken references....
idp.authn.LDAP.sslConfig                        = jvmTrust
```

And these are the settings for an Active Directory server:

```
idp.authn.LDAP.authenticator                    = bindSearchAuthenticator
# explicit failover over multiple DCs
idp.authn.LDAP.ldapURL                          = ldap://dc1.example.org ldap://dc2.example.org ldap://dc3.example.org
idp.authn.LDAP.baseDN                           = ou=People,dc=example,dc=org
idp.authn.LDAP.bindDN                           = cn=read,dc=example,dc=org
idp.authn.LDAP.bindDNCredential                 = PASSWORD-GOES-HERE
# in IdP 3.2.x only
# idp.attribute.resolver.LDAP.returnAttributes    = displayName,mail,sAMAccountName
idp.authn.LDAP.subtreeSearch                    = true
idp.authn.LDAP.userFilter                       = (sAMAccountName={user})
# leave TLS on
# deploy local root CA certificate as $IDP_HOME/credentials/ldap-server.crt
```

The default behavior for LDAP is to perform a case-insensitive search for username.  And the default behavior for the IdP is to accept the username in exactly the form as entered by the user.  Which can lead to inconsistent behavior of the IdP if the user changes the form how the username is entered (lower-case/upper-case/mixed-case...)

Force the IdP to normalize to lower-case by setting `shibboleth.authn.Password.Lowercase` to `TRUE` in `/opt/shibboleth-idp/conf/authn/password-authn-config.xml`:

```
    <util:constant id="shibboleth.authn.Password.Lowercase" static-field="java.lang.Boolean.TRUE"/>
```

## Link your Attribute Resolver to your LDAP server

The attribute resolver is configured in `/opt/shibboleth-idp/conf/attribute-resolver.xml`. In the next section, we will define attributes that the IdP would be releasing about the user.

The first step is to connect the attribute resolver to the LDAP server. Edit `/opt/shibboleth-idp/conf/attribute-resolver.xml` and copy in the definition of the LDAP `DataConnector` from the sample configuration file `$IDP_HOME/conf/attribute-resolver-ldap.xml`. 

This definition is using the properties defined in ldap.properties file (configured in the previous section), so the key connection parameters should be already set.

However, it may be necessary to make some customizations:

*   If the connection is not using a trust certificate, remove the `trustFile` attribute.
*   If the LDAP server is an Active Directory server, it may be necessary to add the property `java.naming.referral` and set it to `"follow"`:
    
    ```
    <LDAPProperty name="java.naming.referral" value="follow"/>
    ```
    
*   And any other customizations as needed - for binary attributes, we for example recommend explicitly listing them as binary:
    
    ```
    <LDAPProperty name="java.naming.ldap.attributes.binary" value="objectGUID objectSid msExchMailboxGuid msExchMailboxSecurityDescriptor mSMQDigests mSMQSignCertificates"/>
    ```
    
    Earlier versions of the IdP were using the `urn:mace:shibboleth:2.0:resolver:dc` namespace for connector definitions, and this type definition required a strict order of elements - e.g., the `dc:LDAPProperty` had to come after `FilterTemplate`.
    
    As of IdP 3.3.0, the definitions moved into the main resolver namespace, `urn:mace:shibboleth:2.0:resolver` - and at the same time, switched to the convention of not using explicit XML namespace prefixes (instead relying on the default namespace being set to this single namespace).  This new type definition permits arbitrary order of the elements.
    
    The examples here have been updated to the new syntax.
    

For completeness, the definition to copy from `attribute-resolver-ldap.xml` is:

```
    <DataConnector id="myLDAP" xsi:type="LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}" 
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
        useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}"
        connectTimeout="%{idp.attribute.resolver.LDAP.connectTimeout}"
        trustFile="%{idp.attribute.resolver.LDAP.trustCertificates}"
        responseTimeout="%{idp.attribute.resolver.LDAP.responseTimeout}">
        <FilterTemplate>
            <![CDATA[
                %{idp.attribute.resolver.LDAP.searchFilter}
            ]]>
        </FilterTemplate>
        <ConnectionPool
            minPoolSize="%{idp.pool.LDAP.minSize:3}"
            maxPoolSize="%{idp.pool.LDAP.maxSize:10}"
            blockWaitTime="%{idp.pool.LDAP.blockWaitTime:PT3S}"
            validatePeriodically="%{idp.pool.LDAP.validatePeriodically:true}"
            validateTimerPeriod="%{idp.pool.LDAP.validatePeriod:PT5M}"
            expirationTime="%{idp.pool.LDAP.idleTime:PT10M}"
            failFastInitialize="%{idp.pool.LDAP.failFastInitialize:false}" />
    </DataConnector>
```

# Configure Attribute Resolver - define attributes

All attribute configuration is done in `/opt/shibboleth-idp/conf/attribute-resolver.xml`.

IdPV3 provides two additional files that can be used as an example, `attribute-resolver-full.xml` and `attribute-resolver-ldap.xml`.  We recommend copying individual snippets (attribute and connector definitions) from these files into `attribute-resolver.xml`.

Edit the file and implement the following changes:

Earlier versions of the IdP were using several namespaces under `urn:mace:shibboleth:2.0:resolver` (`urn:mace:shibboleth:2.0:resolver:ad`, `urn:mace:shibboleth:2.0:resolver:dc`, and also neighbouring `urn:mace:shibboleth:2.0:attribute:encoder`).

As of IdP 3.3.0, the definitions moved into the main resolver namespace, `urn:mace:shibboleth:2.0:resolver` - and at the same time, switched to the convention of not using explicit XML namespace prefixes (instead relying on the default namespace being set to this single namespace).  This new type definition permits arbitrary order of the elements.

This also applies to the XSI types - these are now also specified without prefixes, and some have had their name changed (namely: `ad:Script` changed to `ScriptedAttribute`).

One exception is the `SharedToken` connector, which is a custom extension to the IdP and is defined in a separate namespace - the recommended sharedToken definition now uses the `st:` prefix for its custom namespace.

All other attribute definitions in this section here have been updated to the new syntax.

IdP 3.4.0 replaces **Dependencies** of Attribute Definitions and Data Connectors on other Attribute Definitions and Data Connectors with instead declaring them as **inputs** - and also shifts the `sourceAttributeID` from the attribute definition into the input specification.  This has strong implications only for very specific edge cases (where multiple dependencies produced attributes of the same name) - and otherwise is a syntactic change only.

The syntactic change is trivial: as per the [upstream instructions](https://wiki.shibboleth.net/confluence/display/IDP30/Dependency#Dependency-deprecated):

*   replace a `Dependency` referencing an attributes with an `InputAttributeDefinition` referencing the same attribute.
*   replace a `Dependency` referencing a data connector in a definition with a `sourceAttributeID` with an `InputDataConnector` referencing the same connector and selecting the attribute from the `sourceAttributeID` definition:
    
    *   e.g., turn
        
        ```
            <AttributeDefinition id="commonName" xsi:type="Simple" sourceAttributeID="cn">
                <Dependency ref="myLDAP" />
        ```
        
    *   into
        
        ```
            <AttributeDefinition id="commonName" xsi:type="Simple">
                <InputDataConnector ref="myLDAP" attributeNames="cn" />
        ```
        
*   replace a `Dependency` referencing a data connector in a definition with no `sourceAttributeID` with an `InputDataConnector` referencing the same connector and selecting all attributes:
    
    ```
    <InputDataConnector ref="myLDAP" allAttributes="true" />
    ```
    

This documentation has already been updated to this new syntax - but existing attribute resolver configuration files will need to be transformed accordingly.  Version 3.4 will work with legacy configuration, but would produce deprecation warnings.  And support for legacy configuration will be removed in V4.

  

## Delete existing definitions

The default `attribute-resolver.xml` defines the following attributes as derived from the authenticated username. While useful as interesting examples, delete the following definitions:

```
eduPersonPrincipalName
uid
mail
eduPersonScopedAffiliation
```

## Link existing attributes

*   Define attributes available directly in LDAP - just copy in the relevant `AttributeDefinition` element for each of these attributes from `attribute-resolver-full.xml`:
    
    ```
    surname
    givenName
    ```
    

*   Define the `email` attribute by copying in the `mail` attribute definition and changing the attribute `id` from `mail` to `email` (that is how it's called in the federation attribute vocabulary and also in the attribute filter generated by the Federation Registry)  
      
    
*   Define attributes available as an attribute in the LDAP _with a slightly different name_ - typically, in ActiveDirectory, cn would be the user's login name, while displayName would be the user's actual preferred (common) name.  And specifically for the common name, even when `cn` in LDAP contains the common name, this attribute should be called `commonName` on the Tuakiri side.  So for attributes that need renaming, copy the relevant `AttributeDefinition` element from `attribute-resolver-full.xml` and change the sourceAttributeID XML attribute to the name of the original attribute.  
      
    If connecting to Active Directory:  
    
    *   Define `commonName` based on the AD `displayName` (sourceAttributeID="displayName")
    *   Define `uid` based on the AD `cn`
    *   Define `displayName` based on the AD `displayName`.
    
      
    For LDAP systems other than Active Directory, define these attributes with the sources as applicable to your deployment.  
      
    
    CommonName definition
    
    The `attribute-resolver-full.xml` file is unfortunately missing a sample definition of the commonName attribute.  Please use the following snippet - or customize as necessary:
    
    ```
        <AttributeDefinition id="commonName" xsi:type="Simple" >
            <InputDataConnector ref="myLDAP" attributeNames="cn" />
            <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:cn" />
            <AttributeEncoder xsi:type="SAML2String" name="urn:oid:2.5.4.3" friendlyName="cn" />
        </AttributeDefinition>
    ```
    
    The key part to preserver is the attribute `id` (`"commonName"`) and the encoders (SAML1 and SAML2 attribute names).  The `sourceAttributeID`, `Dependency` and  even `type` can be customized as needed...
    

*   Define `eduPersonPrincipalName` define based on the attribute containing the username (typically, `uid` or `cn` or `sAMAccountName`) with your institution's _scope_ (in the form **_institution.domain.ac.nz_**).  IdPV3 allows setting the scope by expanding the `%{idp.scope}` property, so set the scope with:
    
    ```
    scope="%{idp.scope}" sourceAttributeID="uid"
    ```
    

*   If either `eduPersonAssurance` or `eduPersonEntitlement` are available in your IdMS, expose them via the IdP by copying in their respective definitions from `attribute-resolver-full.xml`

## Define static attributes

We need to define several attributes that would have a static value for each user. We do so by first defining a `StaticDataConnector` (expand on the definition included in `attribute-resolver.xml`):

*   Provide actual meaningful values for your institution: full organization name, home organization name (same as scope), and home Organization Type.
    *   For full list of homeOrganizationType values, see the [SCHAC URN registry](https://wiki.refeds.org/display/STAN/SCHAC+URN+Registry) and the [NZ schacHomeOrganizationType extensions](https://tuakiri.ac.nz/confluence/display/Tuakiri/schacHomeOrganizationType).  The relevant values are:
        *   **`urn:schac:homeOrganizationType:nz:university`** 
        *   **`urn:schac:homeOrganizationType:nz:research-institution`**
        *   **`urn:schac:homeOrganizationType:nz:vho`**
        *   `**urn:schac:homeOrganizationType:int:NREN**`
            
        *   `**urn:schac:homeOrganizationType:int:other**`
            

```
    <!-- Static Connector -->
    <DataConnector id="staticAttributes" xsi:type="Static">
        <Attribute id="o">
            <Value>The Institution</Value>
        </Attribute>
        <Attribute id="homeOrganization">
            <Value>institution.domain.ac.nz</Value>
        </Attribute>
        <Attribute id="homeOrganizationType">
            <Value>urn:schac:homeOrganizationType:nz:university</Value>
        </Attribute>
    </DataConnector>
```

Now define the attributes:

*   Define `organizationName` by copying in the definition and changing the connector dependency from "myLDAP" to "staticAttributes"
    *   Note: the attribute constructed by the IdP is called `organizationName`, while the attribute defined in the `staticAttributes` connector used as the source for this attribute is called just `o`. Don't get confused.

*   Pasting in the definitions for `homeOrganization` and `homeOrganizationType`, not provided in the default configuration:

```
    <AttributeDefinition id="homeOrganization" xsi:type="Simple" >
        <InputDataConnector ref="staticAttributes" attributeNames="homeOrganization" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:oid:1.3.6.1.4.1.25178.1.2.9" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.25178.1.2.9" friendlyName="homeOrganization" />
    </AttributeDefinition>

    <AttributeDefinition id="homeOrganizationType" xsi:type="Simple" >
        <InputDataConnector ref="staticAttributes" attributeNames="homeOrganizationType" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:oid:1.3.6.1.4.1.25178.1.2.10" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.25178.1.2.10" friendlyName="homeOrganizationType" />
    </AttributeDefinition>
```

## Define scripted attributes

The `eduPersonAffiliation` and `eduPersonPrimaryAffiliation` describe the affiliation type of the user authenticating. The typical question is: _"Is this person staff or student?"_ The full set of values (with definitions available in the [AAF Attribute Recommendation: eduPersonAffiliation](http://wiki.aaf.edu.au/tech-info/attributes/edupersonaffiliation)) is: faculty / student / staff / employee / member / affiliate / alum / library-walk-in. The `eduPersonAffiliation` attribute is multi-valued and should include all values that apply to the user, while `eduPersonPrimaryAffiliation` should contain only a single, the most relevant value.

Typically, the LDAP server will not be directly providing values for these attributes, but it would have some attributes that allow to determine at least some of the affiliation type values applying to the user - e.g. to tell whether the user is a staff member or a student.

In this case, we recommend defining the attributes using a scriptlet, synthesizing the value from the LDAP attributes available.

This would be very specific for each institution. We illustrate this in an example (based on an actual setup), where we assume the LDAP server has attributes:

*   isUnderGrad (boolean)
*   isPostGrad (boolean)
*   isStaff (boolean)

We decide to give anyone with `isStaff=TRUE` the `"staff"` affiliation, and to anyone with `isUnderGrad=TRUE OR isPostGrad=TRUE` the `"student"` affiliation. Also, anyone who is staff or student also gets the `"member"` affiliation.

*   We first define these attributes at the Shibboleth level, importing them from LDAP, using the following definitions. Note that as these attributes are not expected to be passed in Shibboleth assertions, the definitions don't have any `AttributeEncoder` elements. Otherwise, we would have to decide on attribute names / OIDs to use in the encoder definitions.

```
    <!-- prerequisite to scripted eduPersonAffiliation -->
    <AttributeDefinition id="isUnderGrad" xsi:type="Simple">
        <InputDataConnector ref="myLDAP" attributeNames="isUnderGrad" />
        <!-- no encoder needed -->
    </AttributeDefinition>

    <AttributeDefinition id="isPostGrad" xsi:type="Simple">
        <InputDataConnector ref="myLDAP" attributeNames="isPostGrad" />
        <!-- no encoder needed -->
    </AttributeDefinition>

    <AttributeDefinition id="isStaff" xsi:type="Simple">
        <InputDataConnector ref="myLDAP" attributeNames="isStaff" />
        <!-- no encoder needed -->
    </AttributeDefinition>
```

*   We follow by defining `eduPersonAffiliation` using an `AttributeDefinition` of type `Script`:
    *   In this script, dependencies (attributes and connectors) are visible as variables (object references)
    *   The attribute being constructed (eduPersonAffiliation) is also directly visible as a variable - with an initially empty collection of attributes.
    *   The script thus adds values to the output attribute.
    *   Earlier versions of the code were also constructing the attribute object if it doesn't exist yet, but:
        *   the syntax for importing packages and creating objects changed significantly between Java7 and Java8 (replacing the Rhino scripting engine with Nashorn)
        *   Java8 supports a compatibility mode to behave like Java7:
            
            ```
            load("nashorn:mozilla_compat.js");
            ```
            
        *   However, as the attribute object is already created and provided to the script, this is not really needed.
        *   See the Shibboleth project wiki for further information: [https://wiki.shibboleth.net/confluence/display/SHIB2/IdPJava1.8](https://wiki.shibboleth.net/confluence/display/SHIB2/IdPJava1.8)

```
    <AttributeDefinition id="eduPersonAffiliation" xsi:type="ScriptedAttribute">
        <InputAttributeDefinition ref="isUnderGrad" />
        <InputAttributeDefinition ref="isPostGrad" />
        <InputAttributeDefinition ref="isStaff" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:eduPersonAffiliation" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.1" friendlyName="eduPersonAffiliation" />
        <Script>
        <![CDATA[
                is_UnderGrad = isUnderGrad != null && isUnderGrad.getValues().size()>0 && isUnderGrad.getValues().get(0).equals("TRUE");
                is_PostGrad = isPostGrad != null && isPostGrad.getValues().size()>0 && isPostGrad.getValues().get(0).equals("TRUE");
                is_Staff = isStaff != null && isStaff.getValues().size()>0 && isStaff.getValues().get(0).equals("TRUE");

                if (is_Staff) { eduPersonAffiliation.getValues().add("staff"); };
                if (is_UnderGrad || is_PostGrad ) { eduPersonAffiliation.getValues().add("student"); };
                if (is_UnderGrad || is_PostGrad || is_Staff ) { eduPersonAffiliation.getValues().add("member"); };
        ]]>
        </Script>
    </AttributeDefinition>
```

*   And a similar definition for `eduPersonPrimaryAffiliation` (which has to be a single valued attribute, hence the logic in the script is slightly different):

```
    <AttributeDefinition id="eduPersonPrimaryAffiliation" xsi:type="ScriptedAttribute">
        <InputAttributeDefinition ref="isUnderGrad" />
        <InputAttributeDefinition ref="isPostGrad" />
        <InputAttributeDefinition ref="isStaff" />
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:eduPersonPrimaryAffiliation" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.5" friendlyName="eduPersonPrimaryAffiliation" />
        <Script>
        <![CDATA[
                is_UnderGrad = isUnderGrad != null && isUnderGrad.getValues().size()>0 && isUnderGrad.getValues().get(0).equals("TRUE");
                is_PostGrad = isPostGrad != null && isPostGrad.getValues().size()>0 && isPostGrad.getValues().get(0).equals("TRUE");
                is_Staff = isStaff != null && isStaff.getValues().size()>0 && isStaff.getValues().get(0).equals("TRUE");

                if (is_Staff) { eduPersonPrimaryAffiliation.getValues().add("staff"); }
                else if (is_UnderGrad || is_PostGrad ) { eduPersonPrimaryAffiliation.getValues().add("student"); };
        ]]>
        </Script>
    </AttributeDefinition>
```

*   Finally, we define the `eduPersonScopedAffiliation` attribute by copying the definition from `attribute-resolver-full.xml` and:  
    *   changing the dependency from LDAP to the previously defined attribute `eduPersonAffiliation`:
        
        ```
                <InputAttributeDefinition ref="eduPersonAffiliation" />
        ```
        
    *   Note: it is no longer needed to set scope, as the default configuration now refers to the `%{idp.scope}` property set in `idp.properties`.  
          
        

## Define sharedToken

The instructions below are based on the original ARCS instructions, now archived at [https://github.com/REANNZ/arcs-shibext/blob/master/INSTALL-SharedToken.md](https://github.com/REANNZ/arcs-shibext/blob/master/INSTALL-SharedToken.md)

The shared token value **MUST** be stored in either the LDAP server itself (preferred, keeps the values alongside primary identity information), or alternatively in a MySQL database (easier to get going by running a MySQL server directly on the IdP).

*   The benefits of storing the generated SharedToken values in the LDAP or in a database are to:
    *   keep track of values issued
    *   release consistent values in case of changes to the code or input generating values
    *   have a means of inserting a specific value.
*   Therefore, using on-the-fly generated values is highly discouraged and MAY only be used in development mode.

In this section, some instructions are specific to storing the shared token values in an LDAP server, some are specific to storing the values in a local MySQL server - please choose accordingly.

The arcs-shib-ext module versions 1.5.x and older are only compatible with IdP v2.x - and are not compatible with IdP V3.

The most recent version is 1.9.1 (as of July 2020) and this version is compatible only with IdP 3.4.0.

For IdP 3.2.x and 3.3.x, use version 1.8.4.

Please note that earlier versions (up to and including 1.8.2) break with Tomcat 7.0.66+, so to avoid issues with Tomcat updates (as they appear in the RHEL7 update stream as of January 2017), please update the plugin to the latest version suitable for your IdP version).

Older versions compatible with older 3.x releases are 1.7.x for IdP 3.1.x+ and 1.6.x for IdP 3.0.x

Please see [https://github.com/REANNZ/arcs-shibext/releases](https://github.com/REANNZ/arcs-shibext/releases) for up-to-date information.

  

*   Download binary from [https://github.com/REANNZ/arcs-shibext/releases/download/1.9.1/arcs-shibext-1.9.1.jar](https://github.com/REANNZ/arcs-shibext/releases/download/1.9.1/arcs-shibext-1.9.1.jar)
    
    ```
    wget https://github.com/REANNZ/arcs-shibext/releases/download/1.9.1/arcs-shibext-1.9.1.jar
    ```
    

*   Copy the JAR into `$SHIB_HOME/edit-webapp/WEB-INF/lib` and (later) rebuild the webapp WAR archive.
    
    ```
    cp arcs-shibext-*.jar $IDP_HOME/edit-webapp/WEB-INF/lib/
    cd $IDP_HOME
    sh $IDP_HOME/bin/build.sh
    ```
    

### Configuring a MySQL database for storing sharedToken values

*   If configuring with a MySQL database, make sure the MySQL JDBC driver is present. As it would be later (in section [Configure Database Storage](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-ConfigureDatabaseStorage)) used with a JDBC Pool DataSource (loaded as part of Tomcat outside of the IdP web application), the JDBC driver has to be loaded outside of the application as well.  The correct way (repeated also below) is: 
    
    ```
    yum install mysql-connector-java
    ln -s /usr/share/java/mysql-connector-java.jar /usr/share/tomcat/lib/
    ```
    

*   Create a MySQL user: run "mysql" as root and enter the following commands:
    
    ```
     create user 'idp_admin'@'localhost' identified by 'IDP_ADMIN_PASSWORD';
     grant all privileges on idp_db.* to 'idp_admin'@'localhost';
    ```
    
*   Create a MySQL user: run "mysql" as idp\_admin and enter the following commands:
    
    ```
     mysql -u idp_admin -p
    
     CREATE DATABASE idp_db CHARACTER SET utf8 COLLATE utf8_bin;
     use idp_db;
    
     CREATE TABLE tb_st (
     uid VARCHAR(100) NOT NULL,
     sharedToken VARCHAR(50),
     PRIMARY KEY  (uid)
     );
    ```
    

### Defining sharedToken attribute (both LDAP and MySQL)

*   Add schema definition to `attribute-resolver.xml`: add the following to the list of schema locations at the top of the file:
    
    ```
    urn:mace:arcs.org.au:shibboleth:2.0:resolver:dc classpath:/schema/arcs-shibext-dc.xsd
    ```
    

*   Add connector definition to `attribute-resolver.xml`(sharedToken) with the following customizations:
    *   idPIdentifier - your IdP entityId
    *   sourceAttributeID: list of source attribute IDs (comma-separated) forming a combination of attributes that is unique, non-reassignable and if possible persistent - if usernames are not reused at your institution, username is perfectly fine - so the "cn" or "uid" attribute would do
        *   Make sure the source attribute is also included as inputs (formerly dependencies).
        *   The below example uses `sAMAccountName` from the `myLDAP` DataConnector - but it is also possible to use `InputAttributeDefinition` to reference other existing attribute definitions.
    *   **If using a MySQL server**, enter the database connection details, with the IDP\_ADMIN\_PASSWORD as defined just above
    *   **If storing sharedToken in an LDAP server**:
        *   Omit the `DatabaseConnection` element from the below snippet
            
        *   Change the values to `**storeLdap="true"**` and `**storeDatabase="false"**` and
            
        *   Make sure the LDAP connector is listed as an InputDataConnector (formerly Dependency)
        *   If storing the sharedToken value in an attribute other than `auEduPersonSharedToken`, set this name in `storedAttributeName`.
        *   Set the **ldapConnectorId** attribute to the ID of the connector (this is required to identify it among the dependencies).
            
            Please note that when storing the sharedToken values in LDAP, the sharedToken module would be accessing the LDAP server from the connector specified in the dependency - so the same LDAP server as used for retrieving all other attributes, **with the same credentials**. Therefore, in order for this module to succeed in storing the generated sharedToken values, the account specified in the LDAP connector needs to have the permissions to write into the `auEduPersonSharedToken` attribute (or the attribute set in `storedAttributeName`).
            
    *   The connector definition is:
        
        ```
            <!-- ==================== auEduPersonSharedToken data connector ================== -->
        
            <DataConnector xsi:type="st:SharedToken" xmlns:st="urn:mace:arcs.org.au:shibboleth:2.0:resolver:dc"
                                id="sharedToken"
                                idpIdentifier="https://idp.institution.domain.ac.nz/idp/shibboleth"
                                sourceAttributeID="sAMAccountName"
                                storeLdap="false"
                                ldapConnectorId="myLDAP"
                                storedAttributeName="auEduPersonSharedToken"
                                storeDatabase="true"
                                salt="SALT-GOES-HERE">
                <InputDataConnector ref="myLDAP" attributeNames="auEduPersonSharedToken sAMAccountName"/>
        
                <st:DatabaseConnection jdbcDriver="com.mysql.jdbc.Driver"
                                    jdbcURL="jdbc:mysql://localhost/idp_db?useSSL=false"
                                    jdbcUserName="idp_admin"
                                    jdbcPassword="IDP_ADMIN_PASSWORD"
                                    preferredTestQuery="/* ping */ SELECT 1;"
                                    testConnectionOnCheckout="true"
                                    primaryKeyName="uid"/>
        
            </DataConnector>
        ```
        
          
        
          
        
          
        
*   Note: on the first install, generate a suitable salt value with:
    
    ```
     openssl rand -base64 36 
    ```
    
    *   On subsequent installs, reuse the same value (stored somewhere carefully)
*   Note also that the SharedToken value depends on the IdP entityID - which could be picked up from the environment, but is better set in the configuration.
*   Database reconnection
    
    Earlier versions of this documentation were instructing to adjust the JdbcURL to set the `"autoReconnect=true"` option and the `wait_timeout` session variable.
    
    Version 1.8.2 of arcs-shib-ext, released on August 1st, 2016, introduces settings for configuring connection testing on checkout in the DataSource.  These are now included in the example above - and with these settings in place, it is no longer necessary to set the increased wait\_timeout OR the autoReconnect option.
    

*   Add attribute definition to attribute-resolver.xml (auEduPersonSharedToken)
    
    ```
        <!-- ==================== auEduPersonSharedToken attribute definition ================== -->
        <AttributeDefinition id="auEduPersonSharedToken" xsi:type="Simple">
            <InputDataConnector ref="sharedToken" attributeNames="auEduPersonSharedToken" />
            <AttributeEncoder xsi:type="SAML1String" name="urn:mace:federation.org.au:attribute:auEduPersonSharedToken" />
            <AttributeEncoder xsi:type="SAML2String" name="urn:oid:1.3.6.1.4.1.27856.1.2.5" friendlyName="auEduPersonSharedToken" />
        </AttributeDefinition>
    ```
    

*   Release attribute in attribute-filter.xml (auEduPersonSharedToken)
    *   See attribute release later.

## eduPersonTargetedID / PersistentNameID

Persistent targeted ID attributes are used to uniquely identify a user when visiting a site (an SP), but each value is targeted to that SP - and when visiting another site (a different SP), the value is different. The value can be either calculated on the fly as a hash (through the ComputeID connector), or stored in a database. We strongly recommend storing the values in a database, as it:

*   allows keeping track of the values issued (and e.g., sites visited by each user)
*   makes possible to preserve the values when redeploying the IdP (e.g., when entityId or salt values used as input to the hash change)
*   allows to revoke individual values if a particular user needs to discontinue their identity at a particular site.

Historically, there were several different (conflicting) ways of defining the persistent targeted ID attribute as the eduPersonTargetedID attribute. Earlier versions of this document were referring to [https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPTargetedID](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPTargetedID) (instructing to define eduPersonTargetedID in the "new" format only, ignoring the "old" format).

However, as IdPv3 deprecates eduPersonTargetedID ( see [ComputedIdConnector](https://wiki.shibboleth.net/confluence/display/IDP30/ComputedIdConnector) and [StoredIdConnector](https://wiki.shibboleth.net/confluence/display/IDP30/StoredIdConnector) documentation) and instead recommends using a Persistent NameID in the SAML envelope. This would make the IdP interoperable with other SAML implementations - and is still compatible with existing Shibboleth SP deployments - in the default configuration, a Shibboleth SP accepts both SAML2 Persistent Name ID **and** eduPersonTargetedID values as the `persistent-id`. However, it is crucial that an IdP only issues **either** the SAML2 Persistent Name ID **or** eduPersonTargetedID **but not both** - otherwise, the Shibboleth SP would accept both (identical) values as multiple values of the `persitent-id` attribute - and would present these values to applications in a misformatted way, concatenating them with a semicolon into a single string.

As part of the IdPv3 upgrade, we strongly encourage all IdPs to switch to SAML2 Persitent Name ID.

Earlier versions of this manual were instructing when performing an upgrade from a 2.x IdP that was using ComputedIdConnector for eduPersonTargetedID (i.e., not storing the values in a database), to first follow the instructions at [Configuring an 2.x IdP to use StoredID Connector](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808986243/Configuring+an+2.x+IdP+to+use+StoredID+Connector).

While it is still strongly recommended to store the values in a database, it is no longer deemed necessary to change the configuration of the version 2 IdP, as the version 3 IdP would be producing the same values.

However, if the existing persistent ID values are not stored in a database, it is crucial to use the identical salt value on the old 2.x and the new 3.x IdP.

  

To configure SAML2 Persistent NameID (based on [https://wiki.shibboleth.net/confluence/display/IDP30/NameIDGenerationConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/NameIDGenerationConfiguration)):

*   Edit `conf/saml-nameid.properties`  and set the following properties:
    *   `idp.persistentId.sourceAttribute`: set this to a local attribute that is unique, persistent, and **non-reassignable**. Typically, the username attribute (`cn`, `uid` or `sAMAccountName`) would be a good choice.  However, if usernames are reused at your institution, you must choose a different attribute (e.g., `objectGUID` on AD servers works well for this purpose)
    *   `idp.persistentId.salt`: at the first install generate a salt value with the following command (on subsequent installs, reuse the same value!):
        
        ```
        openssl rand -base64 36
        ```
        
    *   `idp.persistentId.generator = shibboleth.StoredPersistentIdGenerator`  
        (this selects the StoredPersistentIdGenerator defined in `saml-nameid.xml`
        
    *   `idp.persistentId.encoding = BASE64`
        
        IdP version 3.3.x introduces a new property, `idp.persistentId.encoding`, which defaults to `BASE32`.  While this default has the intention to improve interoperability (with SPs which do not preserve case), it would break transition from a 2.x IdP which did not store the generated values.  If upgrading from a V2 IdP, please change this new property to `BASE64` to preserve the same values.
        
    *   `idp.persistentId.dataSource = shibboleth.JPAStorageService.DataSource`  
        (this selects the `DataSource` bean that will be defined in `conf/global.xml` when configuring database storage)
        
        Please note that `conf/saml-nameid.properties` also allows to set `idp.persistentId.store = PersistentIdStore` (and this is what earlier versions of this document recommended).  However, this strategy does not work well with enabling the reverse lookup, so we now recommend to use `idp.persistentId.dataSource`, for linking `StoredPersistentIDGenerator` to the database.  With having `idp.persistentId.store` set as above, it is not necessary to set `idp.persistentId.dataSource`
        
        The earlier instructions were also including defining the persistent store bean: also assuming the JPA Storage Service will be configured further below, the instructions were define the PersistentIdStore just as a reference to the JPAStorageService DataSource:
        
        ```
            <bean id="PersistentIdStore" parent="shibboleth.JDBCPersistentIdStore"
                p:dataSource-ref="shibboleth.JPAStorageService.DataSource"
                p:queryTimeout="PT2S" />
        ```
        
*   Edit `conf/saml-nameid.xml` and:
    *   In the definition of `shibboleth.SAML2NameIDGenerators`, uncomment the reference to `shibboleth.SAML2PersistentGenerator`: 
        
        ```
            <util:list id="shibboleth.SAML2NameIDGenerators">
        
                <ref bean="shibboleth.SAML2TransientGenerator" />
        
                <!-- Uncommenting this bean requires configuration in saml-nameid.properties. -->
                <ref bean="shibboleth.SAML2PersistentGenerator" />
        
                <!-- ...  -->
        
            </util:list>
        ```
        
    *   And within the database pointed to by the JPAStorageService DataSource ( `idb_db` as per below, and also as possibly created earlier for sharedToken), create the `shibpid` table with the following DDL (taken from [https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration)):
        
        ```
        CREATE TABLE shibpid (
            localEntity VARCHAR(255) NOT NULL,
            peerEntity VARCHAR(255) NOT NULL,
            persistentId VARCHAR(50) NOT NULL,
            principalName VARCHAR(50) NOT NULL,
            localId VARCHAR(50) NOT NULL,
            peerProvidedId VARCHAR(50) NULL,
            creationDate TIMESTAMP NOT NULL,
            deactivationDate TIMESTAMP NULL,
            PRIMARY KEY (localEntity, peerEntity, persistentId)
        );
        ```
        
*   Add the source attribute selected above.  The attribute must be configured in Attribute Resolver in order to be visible to the PersistentIDGenerator.  
    *   If not already defined, define the attribute in `conf/attribute-resolver.xml`.  If not intending to release this attribute to any SPs, the attribute definition does not need any SAML encoders - so that will also stop the attribute from being included in the outgoing AttributeStament, it would just be accessible to the PersistentIDGenerator.  A sample definition of the `uid` attribute would be:
        
        ```
            <AttributeDefinition id="uid" xsi:type="Simple" >
                <InputDataConnector ref="myLDAP" attributeNames="uid" />
            </AttributeDefinition>
        ```
        
        No longer needed
        
        Earlier versions of this documentation were also instructing to explicitly release this attribute to make it visible to the PersistentID generator.  This was necessary for older versions of IdPv3, but for 3.2.0 and newer is no longer needed - see the last line in the ComputedID section of the upstream [PersistentNameID documentation](https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration#PersistentNameIDGenerationConfiguration-ComputedIDs).
        
        The original instructions were:
        
        *   Edit `conf/attribute-filter.xml` and add a rule to release the source Attribute:
            
            ```
                <!-- release uid to all SPs, so we can calculate Persistent NameID.
                     As we do not use an encoder in that definition, 
                     the attribute will not really be released. -->
                <AttributeFilterPolicy id="uid2all">
                    <PolicyRequirementRule xsi:type="ANY" />
            
                    <AttributeRule attributeID="uid">
                        <PermitValueRule xsi:type="ANY" />
                    </AttributeRule>
                </AttributeFilterPolicy>
            ```
            
              
            
              
            
        *   If the attribute is not released as per above, the IdP would log a message:
            
            ```
            2015-05-05 14:49:27,360 - INFO [net.shibboleth.idp.saml.nameid.impl.PersistentSAML2NameIDGenerator:218] - Attribute sources [uid] did not produce a usable source identifier
            ```
            
        
    *   See [https://issues.shibboleth.net/jira/browse/IDP-714](https://issues.shibboleth.net/jira/browse/IDP-714) for further information.
*   To support mapping SAML2 Persistent NameIDs back to username (which is important for some advanced use cases - e.g., an SP making an explicit AttributeQuery about a user or an SP tries to confirm whether a user still exists at the home institution and still has the same privileges), please edit `$IDP_HOME/conf/c14n/subject-c14n.xml` and uncomment the reference to the `c14n/SAML2Persistent` bean:
    
    ```
    <ref bean="c14n/SAML2Persistent" />
    ```
    
*   And also, to avoid warnings for deprecated elements, edit `$IDP_HOME/conf/c14n/subject-c14n.xml` and comment out the `c14n/LegacyPrincipalConnector` bean:
    
    ```
    <!-- <ref bean="c14n/LegacyPrincipalConnector" /> -->
    ```
    

![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Historical eduPersonTargetedID documentation

 For historical purposes, we also include the original documentation on setting up eduPersonTargetedID.  However, as per above, we strongly encourage the migration to SAML2 Persistent NameID.

  

1.  Add the following attribute definition into `attribute-resolver.xml` (unfortunately, IdPV3 does NOT provide any template definition in the default configuration files):
    
    ```
        <resolver:AttributeDefinition xsi:type="ad:SAML2NameID" id="eduPersonTargetedID" 
                                      nameIdFormat="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent" sourceAttributeID="computedID">
            <resolver:Dependency ref="StoredIDConnector" />
            <resolver:DisplayName xml:lang="en">Targeted ID (opaque per-service username)</resolver:DisplayName>
            <resolver:AttributeEncoder xsi:type="enc:SAML1XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" />
            <resolver:AttributeEncoder xsi:type="enc:SAML2XMLObject" name="urn:oid:1.3.6.1.4.1.5923.1.1.1.10" friendlyName="eduPersonTargetedID" />
        </resolver:AttributeDefinition>
    ```
    
2.  Add the connector definition.  To use the recommended `StoredIDConnector` connector, add the following definition:
    
    ```
        <resolver:DataConnector id="StoredIDConnector" 
                                xsi:type="dc:StoredId" 
                                sourceAttributeID="uid"
                                salt="SALSALSALTTTSALT"
                                generatedAttributeID="computedID">
            <resolver:Dependency ref="myLDAP" />
            <dc:ApplicationManagedConnection
                jdbcDriver="com.mysql.jdbc.Driver"
                jdbcURL="jdbc:mysql://localhost/idp_db?autoReconnect=true&amp;sessionVariables=wait_timeout=31536000"
                jdbcUserName="idp_admin"
                jdbcPassword="IDP_ADMIN_PASSWORD"
            />
        </resolver:DataConnector>
    ```
    
      
    making the following changes as needed:
    
    *   Adjust the database connection accordingly (the above snippet assumes it would be reusing the `idp_db` database created for storing SharedToken values - storing the eduPersonTargetedID values in a separate table `shibpid`in the same database.
        *   If choosing a different database, create that database.
        *   Create the table `shibpid` with the following DDL code (coming from [https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration) , and setting ENGINE to InnoDB to work around key length restrictions):
            
            ```
            CREATE TABLE shibpid (
                localEntity VARCHAR(255) NOT NULL,
                peerEntity VARCHAR(255) NOT NULL,
                persistentId VARCHAR(50) NOT NULL,
                principalName VARCHAR(50) NOT NULL,
                localId VARCHAR(50) NOT NULL,
                peerProvidedId VARCHAR(50) NULL,
                creationDate TIMESTAMP NOT NULL,
                deactivationDate TIMESTAMP NULL,
                PRIMARY KEY (localEntity, peerEntity, persistentId)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
            ```
            
    *   If also adding storage support via JPA (see section Database Storage further below), it is possible to reuse the DataSource definition created for JPA instead of having a duplicate database connection definition in the `ApplicationManagedConnection` inside the `StoredIDConnector`.
        
        ```
         <dc:BeanManagedConnection>shibboleth.JPAStorageService.DataSource</dc:BeanManagedConnection>
        ```
        
    *   Using just the username attribute (typically `cn`, `uid` or `sAMAccountName`) as the source attribute.
        *   Note: the attribute chosen must be unique, persistent, and **non-reassignable** - if usernames are reused at your institution, you must choose a different attribute (e.g., `objectGUID` on AD servers works well for this purpose)
    *   Using a value of salt generated (only at the first install, to be reused later) with:
        
        ```
        openssl rand -base64 36
        ```
        
          
        
        *   Note: if converting from an existing configuration using `ComputeId` connector, **reuse** the existing salt values.  The `StoredIdConnector` is designed to be backwards-compatible and if provided with the same salt, will generate the same values as `ComputeId` connector.  
              
            
    *   Make sure the `id` and the `generatedAttributeID` in the connector definition match the `dependency ref` and the `sourceAttributeID` in the attribute definition.
3.  If it is not possible to use the StoredIDConnector, use instead the `ComputedId` connector:
    
    ```
        <DataConnector xsi:type="ComputedId"
                                id="computedID"
                                generatedAttributeID="computedID"
                                sourceAttributeID="uid"
                                salt="abcef123YOURVALUE">
            <resolver:Dependency ref="myLDAP" />
        </DataConnector>
    ```
    
      
    
    *   Remember to still populate the sourceAttributeID and salt values appropriately (using the same instructions as for `StoredIdConnector` above)
    *   Update the `dependency ref` in the attribute definition to match the connector `id="computedID"`

  

  

  

For reference, to deactivate a particular value for a particular user, set the `deactivationDate` timestamp on that value's record directly in the database - e.g. with the following SQL code:

```
UPDATE shibpid SET deactivationDate=NOW() WHERE principalName='user123' AND peerEntity='https://sp.example.org/shibboleth';
```

  

  

## eduPersonAssurance

The eduPersonAssurance expresses both levels of identity assurance and authentication assurance (i.e., both how sure an IdMS is about the identity of a user and how strong authentication mechanisms were used to establish the session). These attributes would ideally have per-user values, based on information captured about the users in the IdMS - e.g., an attribute tracking whether the business process for creating the user's credentials involved checking official photoID documents and e.g. what kind of password policy applies to the user.

In the absence of such information in the IdMS, we recommend to at least define this attribute as a static attribute (providing the same value for all users) at level 1 (floor of trust). The basic requirements for level 1 are that the institution has documented processes for issuing credentials and is following good practice in managing credentials.

To configure the level 1 values as static attributes:

*   Add the level1 identity and authentication assurance values to the `staticAttributes` connector (defined above) with `id="eduPersonAssurance"`:
    
    ```
        <DataConnector id="staticAttributes" xsi:type="Static">
            <Attribute id="eduPersonAssurance">
                <Value>urn:mace:aaf.edu.au:iap:id:1</Value>
                <Value>urn:mace:aaf.edu.au:iap:authn:1</Value>
            </Attribute>
        ...
        </DataConnector>
    ```
    
*   Copy in the `eduPersonAssurance` attribute definition from `attribute-resolver-full.xml` and change the connector dependency from `"myLDAP"` to `"staticAttributes"`

## eduPersonEntitlement

The `eduPersonEntitlement` attribute is a multivalued container for arbitrary strings (URNs) that identify privileges. Most values are yet-to-be-defined, one commonly used value is `urn:mace:dir:entitlement:common-lib-terms`. If not adding the `eduPersonEntitlement` to your IdMS, we recommend defining `eduPersonEntitlement` as a static attribute with this value (representing "this user has library privileges") being the only value defined.

*   Copy in the `eduPersonEntitlement` attribute definition from `attribute-resolver-full.xml` and change the connector dependency from `"myLDAP"` to `"staticAttributes"`
*   Add single value to be released (can be multivalued if desired) to the `staticAttributes` connector (defined above) with `id="eduPersonEntitlement"`:
    
    ```
        <DataConnector id="staticAttributes" xsi:type="Static">
            <Attribute id="eduPersonEntitlement">
                <Value>urn:mace:dir:entitlement:common-lib-terms</Value>
            </Attribute>
        ...
        </DataConnector>
    ```
    

# Configuring Attribute Release

*   Attribute release is configured in `/opt/shibboleth-idp/conf/attribute-filter.xml`
*   The file can contain multiple policies.
*   Each policy can apply to a number of hosts or hostgroups (federations) - linked with the `OR` policy.
*   Attributes are referred to by the "friendly" ID they get assigned in `attribute-resolver.xml`.
*   Additional documentation on policy rules is at [https://wiki.shibboleth.net/confluence/display/IDP30/AttributeFilterConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/AttributeFilterConfiguration).

*   We strongly recommend controlling the attribute release through the automatic release rules generated by the Tuakiri Federation Registry further below.
*   If desired, the following two policies can help with testing an IdP while its being deployed (before it is fully registered into Tuakiri and before the automatic attribute release is configured).  
    These policies release:  
    *   All required attributes to the Tuakiri Federation Registry - necessary for logging in the Federation Registry.
    *   All available attributes to the Tuakiri Attribute Reflector - very useful for testing.
        
         Expand source
        
        ```
            <AttributeFilterPolicy id="federationRegistryPolicy" >
                <PolicyRequirementRule xsi:type="Requester" value="https://registry.tuakiri.ac.nz/shibboleth" />
        
                <AttributeRule attributeID="displayName" permitAny="true"/>
                <AttributeRule attributeID="surname" permitAny="true"/>
                <AttributeRule attributeID="givenName" permitAny="true"/>
                <AttributeRule attributeID="email" permitAny="true"/>
                <AttributeRule attributeID="homeOrganization" permitAny="true"/>
                <AttributeRule attributeID="homeOrganizationType" permitAny="true"/>
                <AttributeRule attributeID="eduPersonTargetedID" permitAny="true"/>
                <AttributeRule attributeID="auEduPersonSharedToken" permitAny="true"/>
            </AttributeFilterPolicy>
        
            <AttributeFilterPolicy id="attributesValidatorPolicy" >
                <PolicyRequirementRule xsi:type="Requester" value="https://attributes.tuakiri.ac.nz/shibboleth" />
        
                <AttributeRule attributeID="displayName" permitAny="true"/>
                <AttributeRule attributeID="commonName" permitAny="true"/>
                <AttributeRule attributeID="surname" permitAny="true"/>
                <AttributeRule attributeID="givenName" permitAny="true"/>
                <AttributeRule attributeID="email" permitAny="true"/>
                <AttributeRule attributeID="eduPersonPrincipalName" permitAny="true"/>
                <AttributeRule attributeID="eduPersonScopedAffiliation" permitAny="true"/>
                <AttributeRule attributeID="eduPersonAffiliation" permitAny="true"/>
                <AttributeRule attributeID="eduPersonAssurance" permitAny="true"/>
                <AttributeRule attributeID="eduPersonPrimaryAffiliation" permitAny="true"/>
                <AttributeRule attributeID="homeOrganization" permitAny="true"/>
                <AttributeRule attributeID="homeOrganizationType" permitAny="true"/>
                <AttributeRule attributeID="organizationName" permitAny="true"/>
                <AttributeRule attributeID="eduPersonTargetedID" permitAny="true"/>
                <AttributeRule attributeID="auEduPersonSharedToken" permitAny="true"/>
                <AttributeRule attributeID="eduPersonEntitlement">
                    <PermitValueRule xsi:type="Value" value="urn:mace:dir:entitlement:common-lib-terms" />
                </AttributeRule>
        
                <AttributeRule attributeID="auEduPersonAffiliation" permitAny="true"/>
                <AttributeRule attributeID="auEduPersonLegalName" permitAny="true"/>
        
                <AttributeRule attributeID="mobileNumber" permitAny="true"/>
                <AttributeRule attributeID="postalAddress" permitAny="true"/>
                <AttributeRule attributeID="organizationalUnit" permitAny="true"/>
                <AttributeRule attributeID="telephoneNumber" permitAny="true"/>
        
            </AttributeFilterPolicy>
        ```
        

*   You may also wish to configure additional attribute release policies - e.g., if establishing bilateral relations with some service providers outside Tuakiri or if registering your IdP into another federation that does not generate a per-SP attribute filter (in that case, releasing a set of attributes to all hosts in the federation via an RequesterInEntityGroup rule might be a good choice). For more information on such configuration, please see the [Shibboleth Project IdP attribute filter documentation](https://wiki.shibboleth.net/confluence/display/IDP30/AttributeFilterConfiguration).

# Register the IdP into the federation

Please follow the instructions on [registering an IdP into the Tuakiri federation](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985363/Configuring+a+Shibboleth+Identity+Provider+to+join+the+Tuakiri+Federation#ConfiguringaShibbolethIdentityProvidertojointheTuakiriFederation-RegisteringanIdPintotheFederationRegistry) (using Federation Registry URL [https://registry.tuakiri.ac.nz/federationregistry/](https://registry.tuakiri.ac.nz/federationregistry/) for the Tuakiri federation or [https://registry.test.tuakiri.ac.nz/federationregistry/](https://registry.test.tuakiri.ac.nz/federationregistry/) for Tuakiri-TEST)

For IdP version 3, these instructions need to be slightly adjusted, as IdPv3 generates three certificates/keypairs: signing, encryption and back-channel:

*   On the initial registration form, paste the certificate from `$IDP_HOME/credentials/idp-signing.crt`.
*   Afterwards, update the registration and add the back-channel certificate `$IDP_HOME/credentials/idp-backchannel.crt` as an additional certificate for **signing** and `$IDP_HOME/credentials/idp-encryption.crt` as an **encryption** certificate.
*   Otherwise, proceed as with an IdPV2 (2.4.x) registration - and select that version if the Federation Registry is not listing 3.0.0 as an explicit choice.
*   Also, when updating the registration entry, add the Single Log Out (SLO) endpoints as documented in the [Configuring Single Logout](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-ConfiguringSingleLogout) section.

![](https://reannz.atlassian.net/wiki/plugins/servlet/confluence/placeholder/unknown-macro?name=multiexcerpt-include&locale=en_US&version=2)

# Advanced IdP Configuration

## Configure Database Storage

The IdPV3 comes with several components that need a storage service, and several implemented and configured storage services.

However, the storage services that come enabled by default only store data in either client-side cookies (session or long-lived, possibly cryptographically encrypted and sealed), or in server memory (discarded during a server restart).

In this section, we document setting up a database storage service through JPA (Java Persistence API) and Hibernate. This sequence follows the documentation at [https://wiki.shibboleth.net/confluence/display/IDP30/StorageConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/StorageConfiguration), adding in minor bits (that are being integrated into the documentation).

We assume the database is MySQL and for Tomcat deployments, we recommend the Tomcat JDBC pool implementation for connection pooling (defining a DataSource).

The steps to configure the database storage are:

*   Create a database. The data would be stored in a single table called `StorageService`, we recommend creating database `idp_db` (which, as per other section of this document, can also host the `shibpid` table for storing the values of the of the PersistentNameID / eduPersonTargetedID attribute and the `tb_st` table storing the `auEduPersonSharedToken` values).
    
    ```
     CREATE DATABASE idp_db CHARACTER SET utf8 COLLATE utf8_bin;
     CREATE USER 'idp_admin'@'localhost' IDENTIFIED BY 'IDP_ADMIN_PASSWORD';
     GRANT ALL PRIVILEGES ON idp_db.* TO 'idp_admin'@'localhost';
    ```
    
    It is strongly encouraged to create the databases with `utf8` character encoding and `utf8_bin` collation (sorting).
    
    Earlier versions of this document did not specify these settings and the database (and all tables) would be created with the default system encoding and collation.
    
    To convert these to `utf8` / `utf8_bin`, please run:
    
    ```
    ALTER DATABASE idp_db DEFAULT CHARACTER SET = utf8 COLLATE = utf8_bin;
    ALTER TABLE tb_st CONVERT TO CHARACTER SET utf8 COLLATE utf8_bin;
    ALTER TABLE StorageRecords CONVERT TO CHARACTER SET utf8 COLLATE utf8_bin;
    ALTER TABLE shibpid CONVERT TO CHARACTER SET utf8 COLLATE utf8_bin;
    ```
    
*   Create the `StorageRecords` table.  This part can be tricky, as different versions of IdP ship with different versions of Hibernate, which use different database mapping / field names.  IdP 3.0.0 used column names `key` (and `expiration`) instead of`id` (and `expires`).  The key issue with that was `key` is a reserved word in MySQL - and therefore, the column name then must be quoted in all SQL statements.  IdP 3.1.1 reverts back to `id` (and `expires`), avoiding the clash with MySQL reserved words.  For IdP 3.1.1+, create the table with:
    
    ```
    CREATE TABLE `StorageRecords` (
     `context` varchar(255) NOT NULL,
     `id` varchar(255) NOT NULL,
     `expires` bigint(20) DEFAULT NULL,
     `value` longtext NOT NULL,
     `version` bigint(20) NOT NULL,
     PRIMARY KEY (`context`,`id`));
    ```
    
*   Add the following beans to `$IDP_HOME/conf/global.xml` - instead of duplicating them here, please use the MySQL versions from the [IdPv3 Storage documentation](https://wiki.shibboleth.net/confluence/display/IDP30/StorageConfiguration#StorageConfiguration-Installation) (section Installation, unfold the snippets under **DB-independent Configuration** and **MySQL Configuration**). 
    
    As of September 2022, the snippets in the linked IdPv3 documentation may fail to unfold.  As an alternative, get the snippets intead from the [IdPv4 documentation](https://wiki.shibboleth.net/confluence/display/IDP4/StorageConfiguration#Storage-Implementations) (unfold JPAStorageService under Storage Implementations).
    
      
    The beans to add are:
    
    ```
    shibboleth.JPAStorageService
    shibboleth.JPAStorageService.EntityManagerFactory
    shibboleth.JPAStorageService.JPAVendorAdapter
    shibboleth.JPAStorageService.DataSource
    ```
    
      
    
    *   Customize the  `shibboleth.JPAStorageService.DataSource`bean with database connection parameters: 
        *   Set the `class` to match the Tomcat JDBC pool (already comes preinstalled with Tomcat as `tomcat-jdbc.jar`), `org.apache.tomcat.jdbc.pool.DataSource`
        *   Set the connection URL, username, password and driverClassName to match your database connection.
        *   For local MySQL connections, explicitly turn off SSL (as the server likely does not have SSL configured, but the driver would produce warnings about SSL not being used - these are not emitted when SSL is explicitly disabled).
        *   Note that with Tomcat JDCBC pool, the JDBC URL property name is just `url`, not `jdbcUrl`
        *   Set the [validationQuery property](https://tomcat.apache.org/tomcat-7.0-doc/jdbc-pool.html#Common_Attributes) to a simple query that would probe (validate) the database connection before it is handed out from the pool.  Note that the special syntax, starting with `/* ping */` is crucial - this triggers a ping in the database driver; see the [MySQL JDBC Driver documentation](http://dev.mysql.com/doc/connector-j/5.1/en/connector-j-usagenotes-j2ee-concepts-connection-pooling.html#idm139787873937472).
        *   Set the `testOnBorrow` property to actually turn on connection validation on checkout.
        *   With all these modifications, the bean could look like: 
            
            ```
            <bean id="shibboleth.JPAStorageService.DataSource"
                class="org.apache.tomcat.jdbc.pool.DataSource" destroy-method="close" lazy-init="true"
                p:driverClassName="com.mysql.jdbc.Driver"
                p:url="jdbc:mysql://localhost:3306/idp_db?useSSL=false"
                p:validationQuery="/* ping */ SELECT 1;"
                p:testOnBorrow="true"
                p:username="idp_admin"
                p:password="IDP_ADMIN_PASSWORD" />
            ```
            
        *   If using a remote database server, set the [validationQueryTimeout property](https://tomcat.apache.org/tomcat-7.0-doc/jdbc-pool.html#Common_Attributes) to a reasonable value (3-5s) to detect connection failures possibly caused by a firewall positioned between the IdP and the database server - this would add to the above definition:
            
            ```
                p:validationQueryTimeout="3"
            ```
            
    *   And remember to install the database driver.  Note that as the driver will be used by classes outside the web application (the Tomcat JDBC pool), the driver also needs to be installed outside the web application.  The following will work on RHEL/CentOS 7 systems:
        
        ```
        yum install mysql-connector-java
        ln -s /usr/share/java/mysql-connector-java.jar /usr/share/tomcat/lib/
        ```
        

Now that the JPAStorageService is configured, we can start reconfiguring the IdP to use this storage service in various parts - all configured in `idp.properties` via properties that should expand to the name of the storage service bean ( `shibboleth.JPAStorageService` ) :

*   IdP consent storage: `idp.consent.StorageService`
*   IdP sessions: `idp.session.StorageService`
*   IdP replay cache: `idp.replayCache.StorageService`
*   IdP artifact map: `idp.artifact.StorageService`

To set all of these four modules to use the JPA Storage service, add the relevant directives into `$IDP_HOME/conf/idp.properties`: 

```
idp.session.StorageService=shibboleth.JPAStorageService
idp.consent.StorageService=shibboleth.JPAStorageService
idp.replayCache.StorageService = shibboleth.JPAStorageService
idp.artifact.StorageService = shibboleth.JPAStorageService
```

## Enabling automatic reload

The default `services.properties` file makes most services reload their configuration every 15 minutes.  This should be considered sufficient for deploying configuration changes on a production system without having to restart the web application; for development, we recommend tweaking this setting to a lower value - e.g., to reload the attribute resolver configuration more frequently when developing attribute mappings, set:

```
idp.service.attribute.resolver.checkInterval = PT5S
```

Please note that refresh intervals configured in `services.properties` only apply to resources directly referenced from the service configuration ( `services.xml` ) - but in the case of Metadata service, not to actual metadata - this applies only to the metadata-providers.xml file itself. See the documentation on configuring metadata above for details on adjusting metadata refresh intervals.

  

## Load Attribute Filter

This step should be done only after fully registering your IdP into the federation.

To automatically release attributes to new services registered in the federation, we will configure the IdP to load an additional attribute filter generated by the Federation Registry. Because of how resources are configured in IdPV3 (just a list of URIs), it is not possibly configure a locally-backed HTTP resource. We therefore configure the IdP to load the attribute filter from a local file, and set up independent refreshing of the local file.

If registering the IdP into multiple federations (such as into Tuakiri and AAF), load the attribute filter for each of the federations according to the respective instructions.

To configure each additional attribute filter, follow these steps:

*   First, determine the URL of your remote filter.  
    For **Tuakiri**: please follow the instructions at [Configuring a Shibboleth Identity Provider to join the Tuakiri Federation#Configure attribute release/filtering through the federation](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985363/Configuring+a+Shibboleth+Identity+Provider+to+join+the+Tuakiri+Federation#ConfiguringaShibbolethIdentityProvidertojointheTuakiriFederation-Configureattributerelease/filteringthroughthefederation) (this process would provide you with a custom URL to download the metadata from).

![](https://reannz.atlassian.net/wiki/plugins/servlet/confluence/placeholder/unknown-macro?name=multiexcerpt-include&locale=en_US&version=2)

  

Alternatively, set up the fetching via an external script and configure the IdP to only load an additional local file:

*   Download the [fetch-xml.sh](https://github.com/REANNZ/Tuakiri-public/raw/master/scripts/fetch-xml.sh) script into `/opt/shibboleth-idp/bin`
    
    ```
    wget -O /opt/shibboleth-idp/bin/fetch-xml.sh https://github.com/REANNZ/Tuakiri-public/raw/master/scripts/fetch-xml.sh
    chmod +x /opt/shibboleth-idp/bin/fetch-xml.sh
    ```
    
*   Determine a local email address that should be receiving notifications when fetching a fresh copy of the attribute filter fails.  
    
      
    
*   Run the `fetch-xml.sh` script once to download a copy of your attribute filter into `/opt/shibboleth-idp/conf/tuakiri-attribute-filter.xml` (substituting the correct local values - URL and email address - into the following line):
    
    ```
    /opt/shibboleth-idp/bin/fetch-xml.sh http://directory.tuakiri.ac.nz/attribute-filter/institution.domain.ac.nz.xml /opt/shibboleth-idp/conf/tuakiri-attribute-filter.xml errors@institution.domain.ac.nz
    ```
    
*   Create a cron job to periodically (every 2 hours) download the metadata and the attribute filter: run `crontab -e` and add the following entry (matching the command you had run on the command line earlier):
    
    ```
    02 */2 * * * /opt/shibboleth-idp/bin/fetch-xml.sh https://directory.tuakiri.ac.nz/attribute-filter/institution.domain.ac.nz.xml /opt/shibboleth-idp/conf/tuakiri-attribute-filter.xml errors@institution.domain.ac.nz
    ```
    
*   Edit `$IDP_HOME/conf/services.xml` and add the additional attribute filter as an additional value in the `shibboleth.AttributeFilterResources` `util:list` bean:
    
    ```
    <value>%{idp.home}/conf/tuakiri-attribute-filter.xml</value> 
    ```
    

  

For both federations, please note:

*   The attribute names used in the download policy file must match the local attribute names. Check that the attribute names in the downloaded attribute filter against their names in existing configuration files:
    *   **attribute-resolver.xml** (attribute definitions, match against the ID in the AttributeDefinition element)
    *   **attribute-filter.xml** (local attribute filter)
*   But please note that no attributes need to be renamed with the Federation Registry since December 2010 (current as of April 2015).

## ECP support

To allow your IdP to be used with the [ECP](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985390/ECP) profile (access via non-browser clients) to let your users access ECP-enabled services in the federation:

*   In IdPV3, no configuration effort on the IdP side is needed - ECP is enabled by default.
    
*   Protect the /idp/profile/SAML2/SOAP/ECP location on your IdP with authentication against your LDAP - add the following (customized for your LDAP server) into `/etc/httpd/conf.d/idp.conf`:
    
    ```
            <Location /idp/profile/SAML2/SOAP/ECP>
                    AuthType Basic
                    AuthName "Example Institution Shibboleth Identity Provider - ECP profile"
                    AuthBasicProvider ldap
                    AuthLDAPURL ldap://ldap.example.org/ou=People,dc=example,dc=org?uid
                    AuthLDAPBindDN "cn=read,dc=example,dc=org"
                    AuthLDAPBindPassword "password"
                    Require valid-user
                    # enable this only over SSL -  not needed when defined in the context of a https VirtualHost
                    SSLRequireSSL
            </Location>
    ```
    
    *   Note: the Apache LDAP module cannot handle LDAP referrals. When connecting to an Active Directory server (which typically includes referrals to other domains in the AD forest in the results), you will need to connect to the Global Catalog at port 3268.
    *   On RHEL/CentOS7, Apache 2.4 by default does not include `mod_ldap` - so install `mod_ldap` explicitly:
        
        ```
        yum install mod_ldap
        ```
        
    *   If you have SELinux enabled, you will also need to explicitly permit the LDAP communication from Apache:
        
        ```
        setsebool -P httpd_can_connect_ldap on
        ```
        
    *   When using an LDAP server using a self-signed certificate or a private root CA, configure `mod_ldap` to trust this CA.  Assuming the certificate is stored in `/opt/shibboleth/credentials/ldap-server.crt`, add the following into the above snippet (as per [mod\_ldap](https://httpd.apache.org/docs/2.4/mod/mod_ldap.html#ldaptrustedclientcert) documentation):
        
        ```
         LDAPTrustedClientCert CA_BASE64 /opt/shibboleth-idp/credentials/ldap-server.crt
        ```
        
    *   When configuring multiple LDAP servers to be used as fall-back, enter them inside a single `AuthLDAPURL` with multiple space-separated hostname-port tuples (and enclose the URL in quotes) e.g.: 
        
        ```
        AuthLDAPURL "ldaps://dc01.instlocal:636 dc02.inst.local:636/ou=People,dc=example,dc=org?uid"
        ```
        
    *   Please see the [IdP3 ECP documentation](https://wiki.shibboleth.net/confluence/display/IDP30/IDP3+ECP+with+Tomcat+and+Apache-Managed+Authentication) for further information:

*   When registering your IdP in the Federation Registry, advertise also the ECP endpoint.  
    ![](https://reannz.atlassian.net/wiki/plugins/servlet/confluence/placeholder/unknown-macro?name=multiexcerpt-include&locale=en_US&version=2)

In order for the ECP handler (running as part of the IdP web application inside Tomcat) to receive the REMOTE\_USER variable set by Apache, the AJP connector in Tomcat must have the `tomcatAuthentication="false"` as instructed above.

ECP will not work if the AJP connector is left with the default settings.

For information on protecting the ECP endpoint from within Tomcat instead, please see [https://wiki.shibboleth.net/confluence/display/IDP30/ECPConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/ECPConfiguration)

## Configuring Single Logout

The Shibboleth IdP supports at least a minimalist implementation of [Single Log Out (SLO)](https://tuakiri.ac.nz/confluence/pages/viewpage.action?pageId=9601405). (This support has been added in 2.4.0 and provides the same functionality in the 3.0.x and 3.1.x release branches)

![](https://reannz.atlassian.net/wiki/plugins/servlet/confluence/placeholder/unknown-macro?name=multiexcerpt-include&locale=en_US&version=2)

The software side of the SLO implementation comes enabled out of the box on IdPV3 installations, however, we recommend making the following changes:

**Step 1:** Make the following adjustments to the settings in `$IDP_HOME/conf/idp.properties`:

*   Configure the IdP to track SP sessions, create a secondary index of the SPs by their SAML IDs, and display the elaborate list on the logout page:
    
    ```
    idp.session.trackSPSessions = true
    idp.session.secondaryServiceIndex = true
    idp.logout.elaboration = true
    ```
    
*   We strongly recommend configure the JPA/Hibernate storage service as per above and store the session information in the JPA storage service:
    
    ```
    idp.session.StorageService = shibboleth.JPAStorageService
    ```
    
      
    
    *   If not configuring the JPA storage service, to get SLO and SP session tracking the work, it is necessary to at least switch the session storage to the in-memory-only (discarded on server restart) storage service (the default client-side cookie storage would not work for the logout functionality):
        
        ```
        idp.session.StorageService = shibboleth.StorageService
        ```
        
*   We also recommend increasing the duration for storing SP sessions: the default on the SP side is 8 hours (while the IdP default expectation of that value would be 2H), and some instances might be using longer value - so set the defaultSPlfetime property to the actual default SP value and add in a buffer to err on the side of caution:
    
    ```
    idp.session.defaultSPlifetime = PT8H
    idp.session.slop = P1D
    ```
    

**Step 2**: Customize the Logout page (`$IDP_HOME/views/logout.vm`). 

*   The most important step is to remove the boilerplate notice saying the page should be customized.  
    Remove the notice (expand here to see the exact changes): 
    
    ![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand...
    
    ```
    --- logout.vm.orig    2018-10-12 15:08:08.726832762 +1300
    +++ logout.vm    2018-10-17 13:41:20.651782569 +1300
    @@ -39,11 +39,6 @@
      
             <div class="content">
               <div class="column one">
    -            <p>This page is displayed when a logout operation at the Identity Provider completes. This page is an example
    -            and should be customized. It is not fully internationalized because the presentation will be a highly localized
    -            decision, and we don't have a good suggestion for a default.</p>
    -            <br>
    -   
                 #if ($rpContext)
                     <p>#springMessageText("idp.logout.sp-initiated", "You have been logged out of the following service:")</p>
                     <blockquote>
    ```
    
    If you enable Single Logout without customizing your Logout page (i.e., leaving in the original default page), users utilizing the Logout functionality will see an unflattering page saying: **"This page is an example and should be customized."**
    
*   Optionally, also add additional notices to the landing logout pages: (`$IDP_HOME/views/logout-propagate.vm`  and `$IDP_HOME/views/logout-complete.vm`).  You may wish to inform the users that sessions derived by the applications on the SPs might not have been terminated and that the only safe way to terminate the session is to close the browser.  Expand here to see our suggestion at the wording:
    
    ![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand...
    
    ```
    --- logout-complete.vm.orig    2018-10-12 15:08:08.725832770 +1300
    +++ logout-complete.vm    2018-10-17 13:43:03.532974734 +1300
    @@ -33,6 +33,8 @@
             <div class="content">
               <div class="column one">
                 <p>#springMessageText("idp.logout.local", "You elected not to log out of all the applications accessed during your session.")</p>
    +            <br>
    +            <p>We recommend to close your browser to close all the sessions at the Service Providers.</p>
               </div>
               <div class="column two">
                 <ul class="list list-help">
    --- logout-propagate.vm.orig    2018-10-12 15:08:08.726832762 +1300
    +++ logout-propagate.vm    2018-10-17 13:43:03.545974633 +1300
    @@ -37,6 +37,10 @@
               <div class="column one">
                   <p>#springMessageText("idp.logout.attempt", "Attempting to log out of the following services:")</p>
                   #parse("logout/propagate.vm")
    +              <br>
    +              <p>The logout process has now completed with the above results.</p>
    +              <br>
    +              <p> However, as application sessions derived from the initial login might have stayed on at these services, we still <strong>recommend to close your browser</strong> to reliably close all the sessions.</p>
               </div>
               <div class="column two">
                 <ul class="list list-help">
    ```
    

  

**Step 3**: Register the following additional endpoints as Single Logout Service in your IdP metadata the Federation Registry, with the following bindings names and URL values (substituting your IdP hostname in the URLs):

|     |     |
| --- | --- |
| Binding | URL |
| urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect | https://idp.example.org/idp/profile/SAML2/Redirect/SLO |
| urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST | https://idp.example.org/idp/profile/SAML2/POST/SLO |
| urn:oasis:names:tc:SAML:2.0:bindings:SOAP | https://idp.example.org:8443/idp/profile/SAML2/SOAP/SLO |

On IdP 3.2.1 only, it may be necessary to apply a fix to the Logout webflow.  The issue has already been fixed upstream and the fix will be included with IdP 3.3.0 once released - this patch is to be applied to 3.2.1 only.

To avoid getting a NullPointerException from stale HttpRequest objects, make the following change to `/opt/shibboleth-idp/system/flows/logout/logout-flow.xml`:

```
--- /root/inst/shibboleth-identity-provider-3.2.1/system/flows/logout/logout-flow.xml      2015-12-19 21:48:00.000000000 +1300
+++ system/flows/logout/logout-flow.xml    2016-09-13 12:53:04.632080786 +1200
@@ -70,21 +70,21 @@
         <transition on="proceed" to="NextRelyingPartyContext" />
     </action-state>
     <view-state id="LogoutView" view="logout">
-        <on-entry>
+        <on-render>
             <evaluate expression="WriteAuditLog" />
             <evaluate expression="environment" result="viewScope.environment" />
             <evaluate expression="opensamlProfileRequestContext" result="viewScope.profileRequestContext" />
             <evaluate expression="opensamlProfileRequestContext.getSubcontext(T(net.shibboleth.idp.session.context.LogoutContext))" result="viewScope.logoutContext" />
             <evaluate expression="opensamlProfileRequestContext.getSubcontext(T(net.shibboleth.idp.profile.context.MultiRelyingPartyContext))" result="viewScope.multiRPContext" />
             <evaluate expression="T(net.shibboleth.utilities.java.support.codec.HTMLEncoder)" result="viewScope.encoder" />
             <evaluate expression="flowRequestContext.getExternalContext().getNativeRequest()" result="viewScope.request" />
             <evaluate expression="flowRequestContext.getExternalContext().getNativeResponse()" result="viewScope.response" />
             <evaluate expression="flowRequestContext.getActiveFlow().getApplicationContext().containsBean('shibboleth.CustomViewContext') ? flowRequestContext.getActiveFlow().getApplicationContext().getBean('shibboleth.CustomViewContext') : null" result="viewScope.custom" />
-        </on-entry>
+        </on-render>
         <transition on="propagate" to="LogoutPropagateView" />
         <transition on="end" to="LogoutCompleteView" />
     </view-state>
     <!-- Terminus -->
```

Please see [https://issues.shibboleth.net/jira/browse/IDP-956](https://issues.shibboleth.net/jira/browse/IDP-956) for further information.

## Configuring Consent Module

Privacy laws, depending on the jurisdiction where the IdP is deployed, may require the IdP to get an explicit consent from the user before releasing personal information about the user to other parties. Thus, before proceeding with a SAML login and sending the user's attributes to the Service Provider, the IdP must first obtain the user's consent. This was historically done through a separate application, uApprove, which has now (in IdPV3) been integrated into the IdP as the consent module.

The consent module steps in the first time the user is logging in to a service, and asks the user for permission to release the required (and desired) attributes to the service. There are several options (discussed below) on when the user would be asked again - these depend on the choices made available in the configuration and on the selection user makes on the consent screen.

The consent module is turned on by default and can be used as it is, we however recommend doing (or at least considering) the following recommendations:

*   Check for value changes. In the default setting, the user would be asked to reconfirm only when the list of attributes being released changes. By turning on this feature, the user would be also asked when the values of the attributes change. Turn this feature on by adding the following line to `$IDP_HOME/conf/idp.properties`:
    
    ```
    idp.consent.compareValues = true
    ```
    
*   Configure server-side database storage. By default, consent information would be stored in client-side cookies. That might be unexpected behavior for end-users - consent once granted would disappear when they switch browsers or devices or otherwise discard cookies. We also do not recommend server-side in memory storage (the only other pre-configured option), where contents of the storage would disappear when the server is restarted. We recommend configuring the JPA/Hibernate storage as per above and then configuring the consent module to user the JPA storage with:
    
    ```
    idp.consent.StorageService=shibboleth.JPAStorageService
    ```
    
*   Remove the limit on the number of consent records held (by default, `10`) by setting the limit to `-1` (no limit):
    
    ```
    idp.consent.maxStoredRecords = -1
    ```
    
*   The following property can be accepted with their default value, or they could be adjusted as needed:
    
    ```
    #idp.consent.storageRecordLifetime = P1Y
    #idp.consent.allowDoNotRemember = true
    #idp.consent.allowGlobal = true
    #idp.consent.allowPerAttribute = false
    ```
    
      
    
    *   The Storage Record Life Time of 1 year should be sufficient - consent records would expire after a year.
    *   The users would be allowed to select to be asked again at next login ("do not remember")
    *   The users would be allowed to grant "global consent" - i.e., not to be asked again, regardless of the services being accesses, or attributes and values being passed.
    *   The users would not be allowed to make per attribute selection - this could make the user interface too confusing for ordinary users, even though it would give the users interested in doing so the power to hand-select which attributes get released.  
          
        
*   We also recommend to configure the order in which the attributes are rendered on the consent screen.  As of 3.2.1, the consent module allows to [set the attribute order](https://wiki.shibboleth.net/confluence/display/IDP30/ConsentConfiguration#ConsentConfiguration-AttributeDisplayOrder) through the order in which the attributes are listed in the `AttributeDisplayOrder` bean.  Edit `IDP_HOME/conf/intercept/consent-intercept-config.xml` and uncomment the `shibboleth.consent.attribute-release.AttributeDisplayOrder` bean and set the contents to:
    
    ```
        <util:list id="shibboleth.consent.attribute-release.AttributeDisplayOrder">
            <value>cn</value>
            <value>commonName</value>
            <value>displayName</value>
            <value>auEduPersonLegalName</value>
            <value>givenName</value>
            <value>sn</value>
            <value>surname</value>
            <value>mail</value>
            <value>email</value>
            <value>eduPersonPrincipalName</value>
            <value>samlSubjectID</value>
            <value>uid</value>
            <value>auEduPersonSharedToken</value>
            <value>eduPersonTargetedID</value>
            <value>eduPersonEntitlement</value>
            <value>eduPersonAssurance</value>
            <value>eduPersonAffiliation</value>
            <value>eduPersonScopedAffiliation</value>
            <value>eduPersonPrimaryAffiliation</value>
            <value>auEduPersonAffiliation</value>
            <value>o</value>
            <value>organizationName</value>
            <value>schacHomeOrganization</value>
            <value>schacHomeOrganizationType</value>
            <value>homeOrganization</value>
            <value>homeOrganizationType</value>
            <value>ou</value>
            <value>organizationalUnit</value>
            <value>postalAddress</value>
            <value>telephoneNumber</value>
            <value>mobile</value>
            <value>mobileNumber</value>
        </util:list>
    ```
    
    Note: if setting this list, we recommend it includes all attributes defined by the IdP.  If some attributes are not included, they'd be displayed in random order after the attributes included in the list - which would not be a consistent user experience.
    
    No longer needed
    
    Earlier versions of this documentation were instructing to explicitly release the attribute used for calculating the persistentNameID  to make it visible to the PersistentID generator - and to hide this attribute from the consent screen. This was necessary for older versions of IdPv3, but for 3.2.0 and newer is no longer needed - see the last line in the ComputedID section of the upstream [PersistentNameID documentation](https://wiki.shibboleth.net/confluence/display/IDP30/PersistentNameIDGenerationConfiguration#PersistentNameIDGenerationConfiguration-ComputedIDs).
    
    This documentation has been updated (in December 2016) in the [configuring Persistent NameID](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-eduPersonTargetedID/PersistentNameID) section above to no longer add rules to explicitly release the attributes.  As long as these rules are not present, it is also no longer necessary to hide the attribute from the consent screen as the original documentation was instructing here.
    
    For completeness, the original instructions were:
    
    *   Edit `/opt/shibboleth-idp/dist/conf/intercept/consent-intercept-config.xml.dist`, locate the `shibboleth.consent.attribute-release.BlacklistedAttributeIDs` and add the `uid` attribute to the list:
        
        ```
        --- /opt/shibboleth-idp/dist/conf/intercept/consent-intercept-config.xml.dist   2016-07-28 11:44:39.060776510 +1200
        +++ consent-intercept-config.xml        2016-07-29 14:56:08.804113011 +1200
        @@ -55,6 +55,7 @@
             </util:list>
        
             <util:list id="shibboleth.consent.attribute-release.BlacklistedAttributeIDs">
        +        <value>uid</value>
                 <value>transientId</value>
                 <value>persistentId</value>
                 <value>eduPersonTargetedID</value>
        
        ```
        
    

This completes the consent configuration.  In IdPV3, the login screen already comes configured with a check-box for resetting the consent information - which erases all consent information, including the global consent (if enabled and granted).

For additional information on configuring the consent module, please see the upstream documentation at [https://wiki.shibboleth.net/confluence/display/IDP30/ConsentConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/ConsentConfiguration)

### Excluding SPs from requiring consent

An institution may decide to bypass the consent module for Service Providers (SPs) operated by the institution internally - as the attributes being released are not crossing organizational boundaries. This can be configured by defining a conditon and attaching the condition as `activationCondition` to the `intercept/attribute-release` flow in `/opt/shibboleth-idp/conf/intercept/profile-intercept.xml`.

*   This step is optional - to be done only as needed in the local deployment circumstances.
*   The default condition defined in `system/conf/profile-intercept-system.xml` activates the flow if attributes would be included in the SSO (true in most practical cases) or if per attribute consent is enabled (not enabled by default).
*   For compatibility, it is recommended to include this default condition in our overriding condition: the overriding condition would be an `AND` of the original condition and an expression over the RelyingPartyId (NOT matching a pattern describing SPs local to the institution).
*   The condition is defined as a new bean, also in `conf/intercept/profile-intercept.xml`
*   The steps therefore are: in `conf/intercept/profile-intercept.xml`:
    *   Add the condition bean as per the below example (adjusting the regular expression match as appropriate)
    *   Add a reference to the condition to the `intercept/attribute-release` flow:  
             `p:activationCondition-ref="AttributeReleaseActivationCondition"`

![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand the activationCondition bean definition

**AvailableInterceptFlows bean after adding activationCondition**

```
    <bean id="shibboleth.AvailableInterceptFlows" parent="shibboleth.DefaultInterceptFlows" lazy-init="true">
        <property name="sourceList">
            <list merge="true">
                <bean id="intercept/context-check" parent="shibboleth.InterceptFlow" />
                <bean id="intercept/terms-of-use" parent="shibboleth.consent.TermsOfUseFlow" />
                <bean id="intercept/attribute-release" parent="shibboleth.consent.AttributeReleaseFlow" p:activationCondition-ref="AttributeReleaseActivationCondition" />
            </list>
        </property>
    </bean>
```

**AttributeReleaseActivationCondition bean definition**

```
    <bean id="AttributeReleaseActivationCondition" parent="shibboleth.Conditions.AND">
        <constructor-arg>
            <!-- The default condition from system/conf/profile-intercept-system.xml -->
            <bean parent="shibboleth.Conditions.OR">
                <constructor-arg>
                    <bean parent="shibboleth.Conditions.NOT">
                        <constructor-arg value="%{idp.consent.allowPerAttribute:false}" />
                    </bean>
                </constructor-arg>
                <constructor-arg>
                    <bean class="net.shibboleth.idp.saml.profile.config.logic.IncludeAttributeStatementPredicate" />
                </constructor-arg>
            </bean>
        </constructor-arg>
        <constructor-arg>
            <!-- A custom condition -->
            <bean parent="shibboleth.Conditions.NOT">
                <constructor-arg>
                    <bean parent="shibboleth.Conditions.RelyingPartyId">
                        <constructor-arg>
                            <bean class="com.google.common.base.Predicates" factory-method="containsPattern"
                                c:pattern="^https://[^/]+\.institution\.domain\.ac\.nz/shibboleth$" />
                        </constructor-arg>
                    </bean>
                </constructor-arg>
            </bean>
        </constructor-arg>
    </bean>
```

  
  

## Friendly attribute names

By default, uApprove would be representing attributes by their local alias - which may not provide the best possible user experience. Names like "eduPersonPrincipalName" look quite cryptic to an ordinary user. The metadata syntax allows to provide friendly names (even locale-specific multiple names) in the `attribute-resolver.xml` file, and uApprove will pick the attribute names from there.

The syntax for specifying the attribute names is:

```
    <AttributeDefinition xsi:type="Simple" id="email" >
        <InputDataConnector ref="myLDAP" attributeNames="mail" />
        <DisplayName xml:lang="en">Email address</DisplayName>
        <AttributeEncoder xsi:type="SAML1String" name="urn:mace:dir:attribute-def:mail" />
        <AttributeEncoder xsi:type="SAML2String" name="urn:oid:0.9.2342.19200300.100.1.3" friendlyName="mail" />
    </AttributeDefinition>
```

Add the following attribute descriptions for the respective attributes into `attribute-resolver.xml`, right between the `Dependency` and `AttributeEncoder` elements:

```
uid:                           <DisplayName xml:lang="en">Local user ID</DisplayName>
email:                         <DisplayName xml:lang="en">Email address</DisplayName>
commonName:                    <DisplayName xml:lang="en">Common name</DisplayName>
surname:                       <DisplayName xml:lang="en">Surname</DisplayName>
givenName:                     <DisplayName xml:lang="en">Given name</DisplayName>
eduPersonPrincipalName:        <DisplayName xml:lang="en">Global username (EPPN)</DisplayName>
samlSubjectID                  <DisplayName xml:lang="en">Unique ID</DisplayName>
samlPairwiseID                 <DisplayName xml:lang="en">Pairwise ID</DisplayName>
displayName:                   <DisplayName xml:lang="en">Display name</DisplayName>
organizationName:              <DisplayName xml:lang="en">Institution name</DisplayName>
organizationalUnit:            <DisplayName xml:lang="en">Organisational Unit</DisplayName>
homeOrganization:              <DisplayName xml:lang="en">Institution domain</DisplayName>
homeOrganizationType:          <DisplayName xml:lang="en">Institution type</DisplayName>
eduPersonAffiliation:          <DisplayName xml:lang="en">Affiliation type</DisplayName>
eduPersonScopedAffiliation:    <DisplayName xml:lang="en">Affiliation type (with institution)</DisplayName>
eduPersonPrimaryAffiliation:   <DisplayName xml:lang="en">Primary affiliation type</DisplayName>
eduPersonEntitlement:          <DisplayName xml:lang="en">Entitlements</DisplayName>
eduPersonAssurance:            <DisplayName xml:lang="en">Identity assurance level</DisplayName>
eduPersonTargetedID:           <DisplayName xml:lang="en">Targeted ID (opaque per-service username)</DisplayName>
auEduPersonSharedToken:        <DisplayName xml:lang="en">Shared token</DisplayName>
auEduPersonLegalName:          <DisplayName xml:lang="en">Legal name</DisplayName>
auEduPersonAffiliation         <DisplayName xml:lang="en">Affiliation type (Australian extensions)</DisplayName>
postalAddress                  <DisplayName xml:lang="en">Business postal address</DisplayName>
telephoneNumber                <DisplayName xml:lang="en">Business phone number</DisplayName>
mobileNumber                   <DisplayName xml:lang="en">Mobile phone number</DisplayName>
```

## DataSealer Key Refreshing

The IdP uses an encryption mechanism, primarily used for the client-side storage (storing information in encrypted client-side cookies), where the IdP relies on an AES secret key. This key needs to be periodically refreshed.  The IdP can be configured to keep around a given number of past versions of the key (defaulting to 30).  Any new information gets encrypted with the newest key; any information received encrypted with older key can still be decrypted as long as the old key is still retained.  For further information, please see [https://wiki.shibboleth.net/confluence/display/IDP30/SecretKeyManagement](https://wiki.shibboleth.net/confluence/display/IDP30/SecretKeyManagement).

In the Database Storage section above, we have recommended to configure server-side database storage to use instead of the client-side cookie storage service.  If this has been done and the client-side storage is not being used, this step can be skipped.

Otherwise, configure a cron job that would be periodically refreshing the encryption key.  The recommended command to refresh they key is:

```
IDP_HOME=/opt/shibboleth-idp JAVA_HOME=/usr/lib/jvm/java /opt/shibboleth-idp/bin/seckeygen.sh --versionfile /opt/shibboleth-idp/credentials/sealer.kver --storefile /opt/shibboleth-idp/credentials/sealer.jks --storepass changeit  --alias secret 
```

To run this as a cronjob, run `crontab -e` as either `root` or `tomcat` and add this entry to rotate the key each night:

```
3 3 * * * IDP_HOME=/opt/shibboleth-idp JAVA_HOME=/usr/lib/jvm/java /opt/shibboleth-idp/bin/seckeygen.sh --versionfile /opt/shibboleth-idp/credentials/sealer.kver --storefile /opt/shibboleth-idp/credentials/sealer.jks --storepass changeit  --alias secret 
```

  

  

## Centralized Usage Logging

Reporting on federation usage is difficult, in particular because not all services use the centralized Discovery Service.

To address this, IdPv3 adds supporting for sending anonymized usage data in the [F-TICKS format](https://tools.ietf.org/html/draft-johansson-fticks-00) to a centralized log service (via syslog messages to UDP port 514).

Tuakiri runs such a service on hosts:

*   **logs.tuakiri.ac.nz** for Tuakiri Production Federation
*   **logs.test.tuakiri.ac.nz** for the Tuakiri Staging Environment / TEST Federation

Once configured, an IdP would send a syslog message to the centralized log service for each session established. The message includes:

*   A timestamp
*   Entity Id of the service provider
*   Entity Id of the IdP
*   An anonymized username (hashed with a secret salt). This does not reveal the user identity, however it allows to generate statistics on the number of unique users.

A sample message may look like:

```
F-TICKS/Tuakiri-TEST/1.0#TS=1467772333#RP=https://attributes.staging.tuakiri.ac.nz/shibboleth#AP=https://virtualhome.test.tuakiri.ac.nz/idp/shibboleth#PN=b61fca51c22df2e5ddb961bb5dfec9a672e627668b4bf2c45de0a719540179cc#RESULT=OK#
```

  

To enable this service, please make the following changes (based on upstream instructions at [https://wiki.shibboleth.net/confluence/display/IDP30/FTICKSLoggingConfiguration](https://wiki.shibboleth.net/confluence/display/IDP30/FTICKSLoggingConfiguration)):

*   In `idp.properties`, uncomment and set:
    *   `idp.fticks.federation` to `Tuakiri` for PROD deployments and `Tuakiri-TEST` for DEV/TEST IdPs
    *   `idp.fticks.algorithm` - just uncomment it, leaving it set to the `SHA-256` hashing algorithm
    *   `idp.fticks.salt` - set it to a value generated with:
        
        ```
        openssl rand -base64 24
        ```
        
    *   `idp.fticks.loghost`: set it to either `logs.tuakiri.ac.nz` or `logs.test.tuakiri.ac.nz` as per above (note that prior to 3.3.0, this setting did not exist in `idp.properties` and had to be added - from 3.3.0 onwards, it is present exists in commented out form)
    *   The resulting configuration may look like:
        
        ```
        idp.fticks.federation=Tuakiri
        idp.fticks.algorithm=SHA-256
        idp.fticks.salt=piKN3aBHKMmyWEDn4zYxA71rZxouQibq
        idp.fticks.loghost=logs.tuakiri.ac.nz
        ```
        
        Earlier versions of this document were instructing to set `idp.fticks.loghost` in `$IDP_HOME/conf/logback.xml` - it is easier and simpler to have all logging configuration in `idp.properties`.
        
*   Optionally, if your IdP has non-Tuakiri traffic (such as bilateral arrangements with individual service providers) that should not be sent to the centralized log collection servers (if it is, it would still get discarded in subsequent processing), please follow these instructions: 
    
    ![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand the instructions to filter usage logging messages...
    
    *   Define an ActivationCondition bean called `TuakiriFTicksCondition` in `$IDP_HOME/conf/global.xml`  (in Tuakiri-TEST, replace the `tuakiri.ac.nz`  labels with `test.tuakiri.ac.nz` ): 
        
        ```
            <!-- based on https://wiki.shibboleth.net/confluence/display/IDP30/ActivationConditions#ActivationConditions-RelyingPartiesByGroup -->
            <bean id="TuakiriFTicksCondition" parent="shibboleth.Conditions.EntityDescriptor">
                <constructor-arg name="pred">
                    <bean class="org.opensaml.saml.common.profile.logic.EntityGroupNamePredicate">
                        <constructor-arg>
                            <list>
                                <value>tuakiri.ac.nz</value>
                                <value>tuakiri.ac.nz/edugain-verified</value>
                            </list>
                        </constructor-arg>
                    </bean>
                </constructor-arg>
            </bean>
        ```
        
    *   In `$IDP_HOME/conf/idp.properties` , set the `idp.fticks.condition` property to use this bean: 
        
        ```
        idp.fticks.condition = TuakiriFTicksCondition
        ```
        
        This property was only introduced in [IdP 4.1.0](https://issues.shibboleth.net/jira/browse/IDP-1643).  If your IdP is running on an earlier version (4.0.x), you can still activate this setting by directly modifying the activationCondition of the `WriteFTICKSLog` bean in `$IDP_HOME/``system/flows/saml/saml-abstract-beans.xml` , but changes to files under `system/` should be avoided - and will be overwritten on IdP upgrades. 
        
        However, if this change gets lost in an upgrade, the IdP will not break, just the behavior would return to default - send all FTICKS messages. 
        
        Preserving these instructions for historical reference (and for sites still on 4.0.x).
        
        ![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand the instructions for filtering usage logs on IdP 4.0.x
        
        *   Edit the definition of `WriteFTICKSLog`  bean in `$IDP_HOME/system/flows/saml/saml-abstract-beans.xml` and instead of the default in-line activation condition, pass a reference to the bean defined above:
            
            ```
            --- system/flows/saml/saml-abstract-beans.xml.dist	2020-07-14 18:02:32.742299470 +1200
            +++ system/flows/saml/saml-abstract-beans.xml	2020-07-21 17:06:41.030977466 +1200
            @@ -337,7 +337,7 @@
                     p:httpServletRequest-ref="shibboleth.HttpServletRequest" />
            
                 <bean id="WriteFTICKSLog" class="net.shibboleth.idp.saml.audit.impl.WriteFTICKSLog" scope="prototype"
            -        p:activationCondition="#{'%{idp.fticks.federation:null}' != 'null'}"
            +        p:activationCondition-ref="TuakiriFTicksCondition"
                     p:federationId="#{'%{idp.fticks.federation:Undefined}'.trim()}"
                     p:digestAlgorithm="#{'%{idp.fticks.algorithm:SHA-256}'.trim()}" p:salt="%{idp.fticks.salt:}" />
            
            ```
            
        
    
*   And restart the IdP - by restarting Tomcat, `service tomcat restart`

`Please make sure the firewall permits outgoing UDP packets to port 514 (at least for the Tuakiri log collection server)`

  

## Configuring SELinux for Tomcat

We recommend operating an IdP with SELinux in enforcing mode.

Since CentOS 7.4, Tomcat runs inside a confined domain, providing additional security.  However, as a result, there are additional configuration steps that need to be taken go give the IdP application  (or to Tomcat as the application server) permissions to access certain files.

Implementing these steps is required - otherwise, the IdP would fail to start.

The IdP needs R/W access to `/opt/shibboleth-idp` - primarily read access, but write access for updating local copies of federation metadata and attribute filters.

Tomcat has R/W access to content labeled as `tomcat_var_lib_t`.  However, as the IdP also needs access to the back-channel certificate in `/opt/shibboleth-idp/credentials`, we need to use the `cert_t` label here - to give both Apache and Tomcat read access to the certificate.

*   Install the `policycoreutils-python` package to get the `semanage` command:
    
    ```
    yum install policycoreutils-python
    ```
    
*   Create file-context rules for labeling files as per above:
    
    ```
    semanage fcontext -a -t tomcat_var_lib_t '/opt/shibboleth-idp(/.*)?'
    semanage fcontext -a -t cert_t '/opt/shibboleth-idp/credentials(/.*)?'
    ```
    
*   Relabel existing files under `/opt/shibboleth-idp` to match these rules:
    
    ```
    restorecon -R /opt/shibboleth-idp
    ```
    
*   Allow Tomcat to talk to the dabase (new in CentOS/RHEL 7.6)
    
    ```
    setsebool -P tomcat_can_network_connect_db on
    ```
    

On CentOS 7.4 only, it is necessary to implement a workaround to give Tomcat permission to connect to MySQL.

On CentOS 7.5, the missing permission has been added and this workaround is no longer necessary - you can skip the rest of this section.

If you are running CentOS 7.4, please unfold the box below to see the details of the workaround - archived otherwise for histroical purposes only.

![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand...

Tomat needs to be able to connect to MySQL - and unfortunately, this has been so far omitted in the SELinux policy for target.  While we expect this to be fixed in future updates to the SELinux policy, for now, we have to use a workaround - create a custom policy module.

*   Create a policy type enforcement file defining a policy module `tomcat-to-mysql` - in a working directory (e.g., `/root/inst`) create `tomcat-to-mysql.te` with the following contents:
    
    ```
    module tomcat-to-mysql 1.0;
    
    require {
            type tomcat_t;
            type mysqld_port_t;
            class tcp_socket name_connect;
    }
    
    #============= tomcat_t ==============
    allow tomcat_t mysqld_port_t:tcp_socket name_connect;
    ```
    
*   Compile, package and load the module with:
    
    ```
    checkmodule -m -M -o tomcat-to-mysql.mod tomcat-to-mysql.te
    semodule_package -o tomcat-to-mysql.pp -m tomcat-to-mysql.mod
    semodule -i tomcat-to-mysql.pp
    ```
    

With these in place, Tomcat should have all the SELinux permissions required and the IdP should operate normally.

  

## Enabling HSTS

Since version 3.4.0, the IdP supports HSTS - HTTP Strict Transport Security:

*   Browser will automatically rewrite all plain HTTP connections to the IdP to HTTPS
*   If the browser detects a certificate error, it will only display an error message and will NOT offer the option to add an exception.
*   This is to protect users from being tricked into accepting a malicious site pretending to be the IdP

This setting is controlled by the `idp.hsts` property in `idp.properties`.

To enable HSTS, change its max-age property from the default value 0 (disabled) to a reasonably long value (industry practice is at least 6 months, preferably 1 year):

  

```
idp.hsts = max-age=31536000
```

## Additional Information

Please see the [IdPv3 wiki](https://wiki.shibboleth.net/confluence/display/IDP30/) for further information. Useful pages for additional configuration options are:

*   [https://wiki.shibboleth.net/confluence/display/IDP30/Configuration](https://wiki.shibboleth.net/confluence/display/IDP30/Configuration)
*   [https://wiki.shibboleth.net/confluence/display/IDP30/ConfigurationFileSummary](https://wiki.shibboleth.net/confluence/display/IDP30/ConfigurationFileSummary)

  
Earlier versions of this documentation included a workaround needed for IdP 3.1.x only.

![](https://reannz.atlassian.net/wiki/images/icons/grey_arrow_down.png)Click here to expand historical IdP 3.1.x compatbility workaround... (no action required on IdP 3.2.0+)

Login breaks on IdP 3.1.x with SPs misconfigured to request AuthenticationType `urn:oasis:names:tc:SAML:2.0:ac:classes:unspecified`.

Version 2 IdP just ignores that misconfiguration and login works. Version 3.2.0 includes a proper workaround, allowing to add a list of contexts to be ignored, with this value being included by default.

On IdP 3.1.x, as an interim workaround, modify `$IDP_HOME/system/conf/general-authn-system.xml` and add this value to the list of `supportedPrincipals` in the `shibboleth.AuthenticationFlow` bean:

```
--- /opt/shibboleth-idp/system/conf/general-authn-system.xml.dist 2015-11-18 10:55:36.713111653 +1300
+++ /opt/shibboleth-idp/system/conf/general-authn-system.xml    2015-11-18 10:55:39.520094266 +1300
@@ -40,6 +40,8 @@
                     c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport" />
                 <bean parent="shibboleth.SAML2AuthnContextClassRef"
                     c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:Password" />
+                <bean parent="shibboleth.SAML2AuthnContextClassRef"
+                    c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:unspecified" />
                 <bean parent="shibboleth.SAML1AuthenticationMethod"
                     c:method="urn:oasis:names:tc:SAML:1.0:am:password" />
             </list>
```

For further details, please see [https://issues.shibboleth.net/jira/browse/IDP-780](https://issues.shibboleth.net/jira/browse/IDP-780)

Note that as this interim workaround is applied to a file under `$IDP_HOME/system/`, it would get overwritten in an upgrade - but, the next version the upgrade would be introducing should already have the proper permanent workaround included.

**No action is required on IdP 3.2.0+**

# Starting the IdP

*   Make all files under /opt/shibboleth-idp owned by Tomcat:
    
    ```
    chown -R tomcat.tomcat /opt/shibboleth-idp
    ```
    

*   For extra security, you may consider making files where passwords to MySQL are stored readable only to the Tomcat users:
    
    ```
    chmod 600 /opt/shibboleth-idp/conf/ldap.properties /opt/shibboleth-idp/conf/attribute-resolver.xml
    ```
    

*   Start MySQL, Tomcat and Apache and make them auto start:
    
    ```
    systemctl enable mariadb
    systemctl start mariadb
    
    systemctl enable tomcat
    systemctl start tomcat
    
    systemctl enable httpd
    systemctl start httpd
    ```
    

# Customization and Branding

In a default install, the IdP login screen is displaying the Shibboleth project logo, a default prompt for username and password, and text (embedded in the logo) saying that this screen should be customized. To establish trust with users, this page should at the very least have the proper institution logo and name and the instructions to customize the page should be removed. Each institution may also wish to customize the graphics to match the style of their login pages - and also the consent pages, logout pages and error pages to give a consistent professional look.

Historically, all of the pages have been JSP pages. While JSP is still left as an option, the default way IdPV3 renders these pages is from a Velocity template under `$IDP_HOME/views`. The main advantage is not having to rebuild the IdP WAR file after a change to the dynamic pages.

However, any images, css files (as well as any other static HTML content) are served from within the WAR file. But, for development of the branding, this can be worked around with Apache aliases to serve the files directly - see below.

We therefore recommend customizing the Velocity login page (and a few additional pages), adding supplementary images and CSS files (or modify existing) as needed, and leave out the JSP files.

The Velocity templates can be configured through message properties defined in message property files. The exact location depends on the IdP version.

*   Up to 3.2.1, the message properties were configured in **`$IDP_HOME/messages/error-messages.properties`.**
*   From 3.3.0 onwards, the three separate property files have merged into one and moved to `$IDP_HOME/system/messages/messages.properties` - which should NOT be modified - and any customizations should instead be inserted into **`$IDP_HOME/messages/messages.properties`** (initially blank).

We recommend configuring at least the following properties:

*   `idp.title` - HTML TITLE to use across all of the IdP page templates.  We recommend settings this to something like `University of Example Login Service`
    
*   `idp.logo` - relative path (within the application) to the logo to render on the templates.  E.g., `/images/logo.jpg`.
    *   The logo image has to be installed into `$IDP_HOME/edit-webapp/images` and the web application WAR file has to be rebuilt with `$IDP_HOME/bin/build.sh`
*   `idp.logo.alt-text` - the ALT text for your logo.  Should be changed from the default value (where the text asks for the logo to be replaced).  Suggested value: `University of Example Logo`
*   `idp.footer` - footer that displays on (almost) all pages.  Suggested value: anything appropriate for your institution, e.g., `Copyright University of Example`.
*   `root.footer` - footer that displays on some error pages.  Suggested value: same as for `idp.footer`, e.g., `Copyright University of Example`.

Overall, the changed message properties might look like:

```
idp.title = University of Example Login Service
idp.logo = /images/logo.jpg
idp.logo.alt-text = University of Example logo
idp.footer = Copyright University of Example
root.footer = Copyright University of Example
```

This would cover the basic customization.  Depending on branding requirements, it may be sufficient to edit the CSS files in `$IDP_HOME/edit-webapp/css`, or it may be necessary to start editing the template pages.

Most of the branding can be done with CSS and with customizing messages in `$IDP_HOME/messages/auth-messages.properties` (and also `consent-messages.properties` and `error-messages.properties` in the same directory.

Please note that the consent module uses a separate CSS file: while the login page and most other pages use `$IDP_HOME/edit-webapp/css/main.css`, the consent module uses `$IDP_HOME/edit-webapp/css/consent.css` with different element names.

Note that if using CSS to customize the logo appearance, while the consent module pages use the `federation_logo` class on the logo `<img>` element, the other pages (login and logout) do not.  It may help to edit these templates and add `class="federation_logo"` to the logo `<img>` tag.

Besides the logo, the login page (and several other pages) display a toolbox on the right with placeholders for links to password-reset and help-desk pages.  These can be customized by editing `$IDP_HOME/messages/authn-messages.properties` and setting (with values appropriate to local context):

```
idp.url.password.reset = http://help.institution.ac.nz/ChangePassword/
idp.url.helpdesk = http://help.institution.ac.nz/
```

Alternatively, it is also possible to hide the whole toolbox (the whole `<div class="column two">` element) from all of the relevant pages (essentially, `login.vm` and all (three) logout pages: `logout.vm`, `logout-complete.vm` and `logout.propagate`).  This can be easily done by adding the following CSS snippet into `$IDP_HOME/edit-webapp/css/main.css`:

```
.column.two {
    display: none;
}
```

  

Overall, the pages under `$IDP_HOME/views` we recommend customizing are:

*   `login.vm` - the main login page
*   `logout.vm`, `logout-complete.vm`, and `logout-propagate.vm` - the pages used in Single Log Out (SLO)
*   `error.vm` - generic error-handling page

For any change done to one page, we recommend applying the change to all of the above pages.

An additional page to customize (but based on a different template) is:

*   `intercept/attribute-release.vm` - attribute release page.

This page still reuses the Logo settings from `error-messages.properties` above, but does not have the toolbox and footer used by the other three pages above.  It is a deployment decision whether to unify the look of these pages or keep the slightly different look of the attribute release page.

We however recommend to at least change the width of the `.box` element in `/opt/shibboleth-idp/edit-webapp/css/consent.css` from 600px to 800px to accommodate the full width of the  friendly attribute names:

```
diff --git a/idp-war/src/main/webapp/css/consent.css b/idp-war/src/main/webapp/css/consent.css
index 5daabeed0..a5b6313b3 100644
--- edit-webapp/css/consent.css.orig
+++ edit-webapp/css/consent.css
@@ -1,5 +1,5 @@
 .box {
-    width:600px;
+    width:800px;
     margin-left: auto;
     margin-right: auto;
     margin-top: 50px;
```

Earlier versions of this documentation were recommending to aid with developing the branding by temporarily serving the CSS and image files out of the `/opt/shibboleth-idp/webapp` directory directly.  However, since Shibboleth IdP 3.4.0, the `webapp` directory is considered a temporary artififact of the build process and the build script deletes it after building the WAR file.  (And this technique was also getting more complicated with Apache and Tomcat being in different SELinux containers).  Therefore, these instructions have been removed.

When done with changes to the images and css directories, remember to rebuild the WAR file and restart Tomcat:

```
$IDP_HOME/bin/build.sh
service tomcat restart
```

# Administrative Interface

The IdPV3 software comes with an administrative interface, that provides several functions:

*   Checking the IdP status
*   Testing the attribute resolver with a particular user ID
*   Triggering the reload of a particular metadata source
*   Triggering the reload of a service within the IdP

For all of these functions, the IdP uses a (configurable) access control mechanism - which is initially the same across all the above functions and is by default IP addressed based, permitting only localhost.

The policy in use can be configured in the `$IDP_HOME/conf/admin/general-admin.xml` file.

The policy selection used to be in `$IDP_HOME/conf/idp.properties` with the below properties - but this has been removed in 3.3.x.  Now the policy selection is done in `$IDP_HOME/conf/admin/general-admin.xml` directly:

```
idp.status.accessPolicy= AccessByIPAddress
idp.resolvertest.accessPolicy= AccessByIPAddress
idp.reload.accessPolicy= AccessByIPAddress
```

The policy itself is configured in `$IDP_HOME/conf/access-control.xml` and we recommend adding at least the external address of the IdP itself to allow resolver testing to work. And it would be also recommended to add the IP address of the local monitoring system to permit monitoring access, and possibly also the IP address of a management console from where an admin could trigger the reload of the services (this can be the IdP itself the administrator is able to run a browser there).

The IP addresses are specified as a comma-separated list of CIDR expressions - the example below adds a single IPv4 address to the default configuration (permitting IPv4 and IPv6 localhost):

```
    <util:map id="shibboleth.AccessControlPolicies">

        <entry key="AccessByIPAddress">
            <bean parent="shibboleth.IPRangeAccessControl"
                p:allowedRanges="#{ {'127.0.0.1/32', '::1/128', '192.168.0.1/32' } }" />
        </entry>

    </util:map>
```

  

After configuring access, the administrative functions can be accessed with either a browser or any HTTP capable tool, by accessing the following URLs:

*   Status: https://idp.institution.ac.nz/idp/status
*   ResolverTest: https://idp.institution.domain.ac.nz/idp/profile/admin/resolvertest?requester=https://attributes.tuakiri.ac.nz/shibboleth&principal=userid , where requester should be the entityId of the target SP and principal should be the local user ID of the target user
*   Metadata Reload: https://idp.institution.ac.nz/idp/profile/admin/reload-metadata?id=TuakiriMetadata , where id is the ID of the MetadataProvider definition (as in metadata-providers.xml) to be reloaded
*   Service Reload: https://idp.institution.ac.nz/idp/profile/admin/reload-service?id=shibboleth.AttributeFilterService , where id is the ID of the service to be reloaded - see `$IDP_HOME/system/conf/services-system.xml` for the full list of services (or the ouput of the status page above)

Earlier versions of the IdP (2.x) had a publicly accessible URL /idp/profile/Status that would return simple "ok".  This URL was long marked as deprecated and has been removed in IdPV3...

# Testing

The best way to test an IdP is to try to log into a Service Provider in the federation. Tuakiri provides a service specifically designed for this purpose, the **Attribute Validator**.  The Attribute Validator displays all of the attributes received from the IdP, and in addition to that performs a number of checks to confirm the values are valid.  Tuakiri is running an attribute reflector in both Tuakiri-production and Tuakiri-TEST federations. Both reflectors are running as standalone services at URLs listed below and are requesting all attributes available in the federation - hence, they work very well for testing all the attributes released by your IdP.

The attribute reflector URLs are:

*   **Tuakiri**: [https://attributes.tuakiri.ac.nz/attributes/](https://attributes.tuakiri.ac.nz/attributes/)
*   **Tuakiri-TEST**: [https://attributes.staging.tuakiri.ac.nz/attributes/](https://attributes.staging.tuakiri.ac.nz/attributes/)

An alternative tool for testing IdP is the _Attribute Authority Command Line Interface_ client - `aacli.sh`.

The tool output is the whole set of attributes that would be released for this user to the given user (the Tuakiri Attribute Validator in the example below - if testing for another service, pass the entityId of the service in the `--requester` option).

Contrary to how this tool functioned in earlier version (2.x - where it launched the full IdP engine in parallel), the tool uses the resolvertest functionality on the IdP itself (see Administrative Interface above - also for instructions on setting up access control).

We recommend connecting the aacli.sh tool to the IdP via the public-facing HTTPS interface (as the default Apache idp.conf does not map the plain-http space to the IdP tomcat web application).  Therefore, run the aacli tool as:

```
$IDP_HOME/bin/aacli.sh --principal TEST-USER --requester https://attributes.tuakiri.ac.nz/shibboleth --url https://idp.instition.domain.ac.nz/idp
```

where the parameters are:

*   principal: the local username of a test user
*   requester: the entityID of the target SP (we recommend the Tuakiri Attribute Validator - use https://attributes.tuakiri.ac.nz/shibboleth for Tuakiri Production environment and https://attributes.test.tuakiri.ac.nz/shibboleth for Tuakiri Staging environment (Test Federation).
*   url: the base URL of the IdP (externally facing https URL)

Alternatively, it is also possible to access the resolvertest URL directly via a browser (as documented above in [Administrative Interface](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-AdministrativeInterface)).   These URLs are subject to access control (by default accessible only from localhost); the options for gaining access include:

*   Explicitly adding an entry to the AccessByIPAddress bean as per the instructions above in the  section  [Administrative Interface](https://reannz.atlassian.net/wiki/spaces/Tuakiri/pages/3808985394/Installing+a+Shibboleth+3.x+IdP#InstallingaShibboleth3.xIdP-AdministrativeInterface).
    *   This would be required even for running the `aacli.sh` command locally - by default, only localhost is on the ACL, so even the external IP address of the IdP host would have to be explicitlly added for connections going via the external hostname.
*   Running the browser on the IdP
    *   Note that by default, only `localhost` is in the `AccessByIPAddress` ACL, so it would be required to either add the external address of the IdP host to the ACL, or connect via `localhost`
*   Using SSH port forwarding (e.g., `ssh idp -L 8443:localhost:443` and then accessing the IdP at [https://localhost:8443/idp/profile/admin/resolvertest](https://localhost:8443/idp/profile/admin/resolvertest)

Other options for testing including using the [UnsolicitedSSO](https://wiki.shibboleth.net/confluence/display/IDP30/UnsolicitedSSOConfiguration): invoking a URL directly at the IdP and pretending as if an SP initiated the request. This can be used for testing even in situations where the IdP is not yet registered in the federation (but is already loading the federation metadata).  While the SSO would not be accepted by the SP in that case (the IdP would not be trusted through the federation), it allows to test the full login and attribute release process at the IdP.  To test SSO against the Tuakiri Attribute Reflector, open the following URL (substituting the correct IdP hostname): [https://idp.institution.ac.nz/idp/profile/SAML2/Unsolicited/SSO?providerId=https://attributes.tuakiri.ac.nz/shibboleth](https://idp.institution.ac.nz/idp/profile/SAML2/Unsolicited/SSO?providerId=https://attributes.tuakiri.ac.nz/shibboleth)
