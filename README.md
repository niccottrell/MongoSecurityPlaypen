# MongoSecurityPlaypen
Copyright (c) 2016 Paul Done

**WARNING** *This project is intentionally NOT "production secure" to make it easier for people to explore. For example, no firewalls are configured and passwords are passed around on the command line which can be view-able in OS user history and OS process lists. Other potential security holes are likely to exist. It is strongly suggested that you consult the [MongoDB Security Checklist](https://docs.mongodb.com/manual/administration/security-checklist/).* 

MongoSecurityPlaypen is intended to be used for learning, exploring or demo'ing specific [MongoDB security](https://docs.mongodb.com/manual/security/) features, in a safe sandbox environment. The project uses [VirtualBox](https://www.virtualbox.org/), [Vagrant](https://www.vagrantup.com/) & [Ansible](https://www.ansible.com/) to build and run a demo environment on a Laptop/PC.

The project demonstrates the following MongoDB Security capabilities.

* _Client Authentication_ (SCRAM-SHA-1, Certificate, LDAP or Kerberos)
* _Internal Authentication_ (Keyfile or Certificate)
* _Role Based Access Control_
* _Auditing_
* _Encryption-over-the-Wire_ (TLS/SSL)
* _Encryption-at-Rest_ (Keyfile or KMIP)
* _FIPS 140-2 usage_

When the project is run on a Laptop/PC, the following local environment is generated, in a set of 5 Virtual Machines:

![MongoSecurityPlaypen](MongoSecurityPlaypen.png)

OpenLDAP, MIT's Keberos KDC and the PyKMIP Server are all installed and configured on the 'centralit' VM.

*WARNING* This project is licensed using the open source MIT License (refer to the 'LICENSE' file in the root directory of this project). However, when run, the project will download and install the Enterprise version of MongoDB, supplied by MongoDB Inc., which has a commercial licence. By running the 'vagrant up' command of this MongoSecurityPlaypen project, you will be implicitly accepting the terms and conditions of the MongoDB Enterprise licence enforced by MongoDB Inc.. Please consult MongoDB Inc.'s licence documents directly, for more information.


## 1  How To Run

### 1.1 Pre-Requisites / Dependencies

Ensure the following prerequisites are already fulfilled on the host Laptop/PC:

* Host operating system is Mac OS X or Linux
* [VirtualBox](https://www.virtualbox.org/wiki/Downloads) is already installed on the host machine
* [Vagrant](https://www.vagrantup.com/downloads.html) is already installed on the host machine
* Ansible is already installed on the host machine (see OS specific installation guides for [Mac OSX](https://valdhaus.co/writings/ansible-mac-osx/) and the many [Linux](http://docs.ansible.com/ansible/intro_installation.html) variants)
* Host machine is connected to the internet (for the installation/configuration only - once configured, you can re-start the environment offline)

### 1.2 Main Environment Generation Steps

* If required, change any values in the text file 'vars/external_vars.yml' to dictate which security features should be turned on and off
* From the terminal/shell, ensure the current directory is the root directory of this MongoSecurityPlaypen project (ie. the directory containing the file 'Vagrantfile')
* Run the following command to create and configure the 5-virtual-machine environment outlined in the diagram above - this includes the final step of automatically running the Test Client Python Application and listing the results in the console

`> vagrant up`

Notes:
* It may take around 10-15 minutes to complete execution, mainly depending on the speed of the host's internet connection.
* If the internet connection is very slow, the build process may fail with an error due to the CentOS/RedHat package manager (yum) timing out when trying to download binaries.
* Once completed, the results from the Test Client Python Application will have been displayed towards the end of the Vagrant output text in the console, showing some data queried from the MongoDB replica set.


## 2  Tips for Exploring & Playing With the Configured Environment

### 2.1 SSH'ing to each of the 5 VMs

    To connect to the VM hosting OpenLDAP, MIT Kerberos KDC and PyKMIP Server:
    > vagrant ssh centralit

    To connect to the VM hosting the 1st MongoDB Database Replica in the Replica-Set:
    > vagrant ssh dbnode1

    To connect to the VM hosting the 2nd MongoDB Database Replica in the Replica-Set:
    > vagrant ssh dbnode2

    To connect to the VM hosting the 3rd MongoDB Database Replica in the Replica-Set:
    > vagrant ssh dbnode3

    To connect to the VM hosting the Test Client Python Application:
    > vagrant ssh client

### 2.2 Stopping, Re-starting and Clearing Out The Environment

* To shutdown/halt the VMs, allowing them to be re-started at a future time, as is, for use offline, run:
    $ vagrant halt
* To restart the VMs (inc. the MongoDB, OpenLDAP, Kerberos & PyKMIP processes) after previously halting it, just run (this won’t attempt to recreate the VMs - the VMs will just be started up again):
    $ vagrant up 
* To completely remove the VMs, ready to start all over again with 'vagrant up', run: 
    $ vagrant destroy -f

Note: Halt/Up doesn't currently work when the 'encryptdb_enabled' variable is true, because the PyKMIP Server does not have is for testing purposes only and dies not persisted saved keys to disk (see section 3. Project TODOs, below).

### 2.3 Using Mongo Shell to Connect to the Replica Set

The sub-sections below outline the way to connect depending on the type of MongoDB authentication that has been configured.

Notes:
 * For some types of authentication, when connecting via the Mongo Shell, Fully Qualified Domain Names - FQDNs (eg. dbnode1.vagrant.dev), need to be used rather than just hostnames (eg. dbnode1, localhost) or IP addresses (eg. 192.168.14.101, or 127.0.0.1. This is necessary when using Kerberos, Certificates and or TLS.
 * For some types of authentication, when invoking the Mongo Shell, the 'mongo' command has to be run as the 'mongod' OS user because a referenced file (such as a keyfile/certificate or a Kerberos keytab), has been "locked down" to only be visible to the 'mongod' OS user that runs the 'mongod' OS process. Hence the use of 'sudo' in those cases.

#### 2.3.1 Connect with no authentication enabled

    $ vagrant ssh dbnode1
    $ mongo    # If SSL disabled
    $ mongo dbnode1.vagrant.dev:27017 --ssl --sslCAFile /etc/ssl/mongodbca.pem  # If SSL enabled
    > show dbs

#### 2.3.2 Authenticate via Username/Password Challenge (SCRAM-SHA-1)

    $ vagrant ssh dbnode1
    $ mongo    # If SSL disabled
    $ mongo dbnode1.vagrant.dev:27017 --ssl --sslCAFile /etc/ssl/mongodbca.pem  # If SSL enabled
    > db.getSiblingDB("admin").auth(
        {
             mechanism: "SCRAM-SHA-1",
             user: "dbmaster",
             pwd:  "adminpasswd123",
             digestPassword: true
        }
      );
    > show dbs

#### 2.3.3 Authenticate via a Certificate

    $ vagrant ssh dbnode1
    $ sudo -u mongod mongo dbnode1.vagrant.dev:27017 --ssl --sslCAFile /etc/ssl/mongodbca.pem --sslPEMKeyFile /etc/ssl/adminuser_client.pem --sslPEMKeyPassword tlsClientPa55word678
    > db.getSiblingDB("$external").auth(
        {
             mechanism: "MONGODB-X509",
             user: "CN=dbmaster,OU=Human Resources,O=WizzyIndustries,L=London,ST=London,C=GB"
        }
     );
    > show dbs

#### 2.3.4 Authenticate via LDAP Proxy passing Username/Password

    $ vagrant ssh dbnode1
    $ mongo    # If SSL disabled
    $ mongo dbnode1.vagrant.dev:27017 --ssl --sslCAFile /etc/ssl/mongodbca.pem  # If SSL enabled
    > db.getSiblingDB("$external").auth(
        {
             mechanism: "PLAIN",
             user: "dbmaster",
             pwd:  "adminpasswd123",
             digestPassword: false
        }
     );
    > show dbs

#### 2.3.5 Authenticate via Kerberos (GSSAPI)
    $ vagrant ssh dbnode1
    $ sudo -u mongod kinit dbmaster  # Required after running Vagrant 'halt' and then 'up, to obtain a Kerberos ticket again
    $ sudo -u mongod mongo dbnode1.vagrant.dev:27017   # If SSL disabled
    $ sudo -u mongod mongo dbnode1.vagrant.dev:27017 --ssl --sslCAFile /etc/ssl/mongodbca.pem  # If SSL enabled
    > db.getSiblingDB("$external").auth(
        {
             mechanism: "GSSAPI",
             user: "dbmaster@WIZZYINDUSTRIES.COM"
        }
     );
    > show dbs

### 2.4 Investigating the MongoDB Replica Set

* SSH to the host for one of the replicas, eg.:
    $ vagrant ssh dbnode1
* Each mongod process is running as a service using the generated configuration file, including Security settings, at: /etc/mongod.conf
* The output log for each mongod process is viewable at /var/log/mongod/mongod.conf - this needs to viewed as the 'mongod' OS user eg.:
    $ sudo -u mongod less /var/log/mongodb/mongod.log 
* If FIPS 140-2 is enabled, this output log file should contain an output line saying: "FIPS 140-2 mode activated"
* If Kerberos is enabled, the following file has additional environment variables set to specify the location of the Keytab and debug logging files: /etc/sysconfig/mongod
* If Kerberos is enabled, the mongod process will log Kerberos debug info at /var/log/mongodb/krbtrace.log - this needs to viewed as the 'mongod' OS user eg.:
    $ sudo -u mongod less /var/log/mongodb/krbtrace.log
* If Auditing is enabled, the mongod process will log Audit events to: /var/lib/mongo/auditLog.bson - to view these events, run:
    $ bsondump /var/lib/mongo/auditLog.bson | less
* The database is configured with an admin user and a sample user (see vars/external_vars.yml for the usernames & passwords). To view the different access control settings for these users, start the Mongo Shell (see section 2.3) and then run the command:
    // If using Username/Password Challenge (SCRAM-SHA-1) authentication:
    > db.getSiblingDB("admin").runCommand({usersInfo:1})
    // If using Certificate/LDAP/Kerberos authentication:
    > db.getSiblingDB("$external").runCommand({usersInfo:1})
* The MongoDB database/collection that is populated with sample data is: maindata.records
* To see the contents of the sample database collection, start the Mongo Shell (see section 2.3) and run:
    > use maindata
    > db.records.find().pretty()

### 2.5 Investigating the OpenLDAP Server

* The OpenLDAP process (/usr/sbin/slapd) is running as a service on the 'centralit' VM, listening on port 389
* To test the LDAP connection from a host running mongod:
    $ vagrant ssh dbnode1
    $ testsaslauthd -u jsmith -p Pa55word124 -f /var/run/saslauthd/mux -s ldap
* The 'ldapsearch' tool can be used to look at the contents of the LDAP Directory:
    $ vagrant ssh centralit
    # Show contents of whole directory (password: "ldapManagerPa55wd123"):
    $ ldapsearch -x -W -H ldap://centralit/ -D "cn=Manager,dc=WizzyIndustries,dc=com" -b "dc=WizzyIndustries,dc=com" "(objectclass=*)"
    # Prove ability to bind to the directory with the 'jsmith' sample user (password: "Pa55word124"):
    $ ldapsearch -x -W -H ldap://centralit/ -D "cn=jsmith,ou=Users,dc=WizzyIndustries,dc=com" -b "ou=Users,dc=WizzyIndustries,dc=com" "(uid=jsmith)"

### 2.6 Investigating the MIT Kerberos KDC

* The MIT version of a Kerberos Key Distribution Center (KDC) (krb5-server) is running as a service on the 'centralit' VM, listening on port 88
* The generated configuration file for the Kerberos server is at: /etc/krb5.conf
* The main log file for the KDC is at: /var/log/krb5kdc.log
* A keytab of registered principals (just host machines and not users) is generated to the file: /etc/krb5.keytab - this needs to viewed as the 'root' OS user, eg.:
    $  sudo klist -k /etc/krb5.keytab
* The kadmin.local tool can be used on the host VM to list the principals (registered host machines and registered users), eg.:
    $ sudo kadmin.local
    : listprincs
    : quit

### 2.7 Investigating the PyKMIP Server

* PyKMIP (a Python implementation of the Key Management Interoperability Protocol) is a server for maintaining keys & certificates and has been developed for testing and demonstration purposes only. It doesn't persist anything to disk (such as stored keys) and as a result if the host 'centralit' VM is restarted, all stored keys are lost.
* Version 0.4.0 of PyKMIP is used rather than a later version (e.g. 0.4.1) which appears to have a regression preventing it from working with MongoDB.
* PyKMIP is started as a service using a generated file at: /usr/lib/systemd/system/pykmip.service
* The wrapper script that runs the PyKMIP python application is at: /sbin/pykmip_server.py
* The wrapper script uses syslog for the logging output for PyKMIP, therefore PyKMIP events can be viewed using the command:
    $ sudo grep 'PyKMIP' /var/log/messages
* The status of the PyKMIP service and some of its output events can also be viewed using the systemctl command, eg.:
    $ sudo systemctl status -l pykmip

### 2.8 Investigating the Test Client Python Application

* A simple Python client, that uses the PyMongo driver to test the secure connection to the remote replica set and query and print out some data from the database/collection maindata.records, is located in the home directory of the default vagrant OS user in the 'client' VM (ie.: /home/vargant/TestSecPyClient.py)
* The test client application can be simply run, over and over again, by SSH'ing to the 'client' VM and invoking it directly, eg.:
    $ vargant ssh client
    $ ./TestSecPyClient.py
* If Kerberos has been configured, and vagrant halt & up have been run to restart the 'client' VM, when SSH'ing to the VM and BEFORE running the test application application, the OS user must be granted a Kerberos ticket again, using the command (password us "Pa55word124"):
    $ kinit jsmith


## 3  Project TODOs
* Extend the 'yum' timeout duration, to avoid timeout failurs when running 'vagrant up' with a slow internet connection.
* PyKMIP has no built-in persistence, so if vagrant halt and then vagrant up have been run, the mongod replicas won't start properly, if encryption is enabled using KMIP
* Fix when encryptdb-enabled and fips140-2-enabled - can't use generated keys as they use blacklisted algorithms (eg. md5) - may need to use FIPS enabled OpenSSL to generate keys - also when fixed, test if can have TLS false, FIPS true, enc-at-rest true, simultaneously.
* When generating the keytab on the 'centralit' VM, generate separate keytabs for dbnode1, dbnode2 & dbnode3 for better security isolation
* Use more elegant way of waiting for primary to be ready, rather than pausing for 30 seconds and then hoping it is ready
* Configure Open LDAP to use TLS
* Cache the MongoDB Enterprise yum repository's contents locally ready for quicker re-running of 'vagrant up'
* For simpler development/debugging of this project, provide ability to just re-run parts of script without running everything

