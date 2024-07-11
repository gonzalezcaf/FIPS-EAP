# Setting FIPS 140-2 Cryptography on RHEL 7 and EAP 7.4

### OBJECTIVE 
Test the steps of configuration for enabling FIPS-2 cryptography securing EAP management interfaces.
&nbsp;

#### Environment:
* RHEL 9.4
* EAP 7.4.17 (user `jboss` for running the service)
* OpenJDK 11 (Red_Hat-11.0.23.0.9-2)
* Documentation:
[Enable FIPS 140-2 Cryptography for SSL/TLS on Red Hat Enterprise Linux 7 and Later](https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/7.4/html-single/how_to_configure_server_security/index#configure_ssl_fips_rhel7)
&nbsp;

#### Description:
This setting is a set of steps to get the correct configuration:

 1. **Enabling FIPS on RHEL**
One of the requeriments is enabling RHEL for FIPS-2, which can be read through the following article [^1]
~~~bash
	# fips-mode-setup --enable
~~~
##### Output:
~~~bash
	[root@dev-tools ~]# fips-mode-setup --enable
	Kernel initramdisks are being regenerated. This might take some time.
	Setting system policy to FIPS
	Note: System-wide crypto policies are applied on application start-up.
	It is recommended to restart the system for the change of policies
	to fully take place.
	FIPS mode will be enabled.
	Please reboot the system for the setting to take effect.
	[root@dev-tools ~]#	
	[root@dev-tools ~]#	reboot
	[root@dev-tools ~]#	
	[root@dev-tools ~]# fips-mode-setup --check
	FIPS mode is enabled.
	[root@dev-tools ~]#
~~~
&nbsp;
&nbsp;

 2. **Location of the configuration files**
For this testing I've used:
* A JBoss EAP base dir named: *`fips`*
* Location of the file standalone.xml: *`JBOSS_HOME/fips/configuration/`*
* Location of the NSS database files: *`JBOSS_HOME/fips/configuration/nssdb/`* 
&nbsp;
&nbsp;

 3. **Configure the NSS database**
**Note:** maybe the *`modutil`* tool isn't installed. So please install using *`yum/dnf`* command and the package: *`nss-tools.x86_64`* [^2]
~~~bash
	$ mkdir -p fips/configuration/nssdb
	$ chown jboss fips/configuration/nssdb
	$ modutil -create -dbdir fips/configuration/nssdb
~~~
##### Output:
~~~bash
	[jboss@dev-tools eap7417]$ modutil -create -dbdir fips/configuration/nssdb

	WARNING: Performing this operation while the browser is running could cause
	corruption of your security databases. If the browser is currently running,
	you should exit browser before continuing this operation. Type 
	'q <enter>' to abort, or <enter> to continue: 
	[jboss@dev-tools eap7417]$ 

	[jboss@dev-tools eap7417]$ tree fips/configuration/nssdb/
	fips/configuration/nssdb/
	├── cert9.db
	├── key4.db
	└── pkcs11.txt

	0 directories, 3 files
~~~
&nbsp;
Create the file *`nss_pkcsll_fips.cfg`* with the following contents y location of the NSS database.
##### Output:
~~~bash
	[jboss@dev-tools eap7417]$ cat > fips/configuration/nss_pkcsll_fips.cfg << EOF
	name = nss-fips
	nssLibraryDirectory=/usr/lib64
	nssSecmodDirectory=/opt/eap7417/fips/configuration/nssdb
	nssDbMode = readOnly
	nssModule = fips
	EOF
	[jboss@dev-tools eap7417]$
~~~
&nbsp;
&nbsp;

 4. **Edit the java security configuration file**
There is a file by default in the OpenJDK installation [^3], but it's better if you define a custom file.
* Location of the file java.security: *`JBOSS_HOME/fips/configuration/`*

	The file `java.security` created is a copy of the default file found in the OpenJDK 		installation *`/usr/lib/jvm/java-11-openjdk-11.0.23.0.9-3.el9.x86_64/conf/security`*

	According to the documentation you have to modify the value for the the following keys:

~~~bash
	security.provider.1=sun.security.pkcs11.SunPKCS11 /opt/eap7417/fips/configuration/nss_pkcsll_fips.cfg
	security.provider.5=com.sun.net.ssl.internal.ssl.Provider SunPKCS11-nss-fips
~~~
&nbsp;

but according to the tests I had to modify a line more:
~~~bash
	fips.provider.1=SunPKCS11 /opt/eap7417/fips/configuration/nss_pkcsll_fips.cfg
~~~

##### Content of the file java.security:
~~~bash
....
#security.provider.1=SUN
security.provider.1=sun.security.pkcs11.SunPKCS11 /opt/eap7417/fips/configuration/nss_pkcsll_fips.cfg
....
#security.provider.5=SunJCE
security.provider.5=com.sun.net.ssl.internal.ssl.Provider SunPKCS11-nss-fips
....
#fips.provider.1=SunPKCS11 ${java.home}/conf/security/nss.fips.cfg
fips.provider.1=SunPKCS11 /opt/eap7417/fips/configuration/nss_pkcsll_fips.cfg
....
~~~
&nbsp;
&nbsp;

 5. **Enable FIPS mode through the NSS database**
~~~bash
	$ modutil -fips true -dbdir fips/configuration/nssdb
~~~
##### Output:
~~~bash
	[jboss@dev-tools eap7417]$ modutil -fips true -dbdir fips/configuration/nssdb/

	WARNING: Performing this operation while the browser is running could cause
	corruption of your security databases. If the browser is currently running,
	you should exit browser before continuing this operation. Type 
	'q <enter>' to abort, or <enter> to continue: 

	FIPS mode already enabled.
	[jboss@dev-tools eap7417]$
~~~
&nbsp;
&nbsp;

 6. **Set the NSS database password**
~~~bash
	$ modutil -changepw "NSS FIPS 140-2 Certificate DB" -dbdir fips/configuration/nssdb/
~~~
##### Output:
~~~bash
	[jboss@dev-tools eap7417]$ modutil -changepw "NSS FIPS 140-2 Certificate DB" 	-dbdir fips/configuration/nssdb/
	
	WARNING: Performing this operation while the browser is running could cause
	corruption of your security databases. If the browser is currently running,
	you should exit browser before continuing this operation. Type 
	'q <enter>' to abort, or <enter> to continue: 
	
	Enter new password: 
	Re-enter new password: 
	Token "NSS FIPS 140-2 Certificate DB" password changed successfully.
	[jboss@dev-tools eap7417]$
~~~
&nbsp;
&nbsp;

6. **Create a certificate stored in the NSS database**
**Note:** you need provide the full path for the NSS database location or you cab get an error message[^4]
~~~bash
	$ certutil -S -k rsa -n undertow -t "u,u,u" -x -s "CN=dev-tools.lab.xee.com, OU=XEE - SBR JSEC, O=RED HAT, L=LA CALERA, ST=CUNDINAMARCA, C=CO" -d /opt/eap7417/fips/configuration/nssdb
~~~
##### Output:
~~~bash
	[jboss@dev-tools eap7417]$ certutil -S -k rsa -n undertow -t "u,u,u" -x -s 	"CN=dev-tools.lab.xee.com, OU=XEE - SBR JSEC, O=RED HAT, L=LA CALERA, 	ST=CUNDINAMARCA, C=CO" -d /opt/eap7417/fips/configuration/nssdb
	Enter Password or Pin for "NSS FIPS 140-2 Certificate DB":
	
	A random seed must be generated that will be used in the
	creation of your key.  One of the easiest ways to create a
	random seed is to use the timing of keystrokes on a keyboard.
	
	To begin, type keys on the keyboard until this progress meter
	is full.  DO NOT USE THE AUTOREPEAT FUNCTION ON YOUR KEYBOARD!
	
	Continue typing until the progress meter is full:
	|************************************************************|
	Finished.  Press enter to continue: 
	Generating key.  This may take a few moments...
	Notice: Trust flag u is set automatically if the private key is present.
	[jboss@dev-tools eap7417]$
~~~
&nbsp;
&nbsp;

7. **Check the private key in the PKCS11 keystore**
You need to check the keytool command can read the private key for the PKCS11 keystore stored in the NSS database.
~~~bash
	$ keytool -list -storetype pkcs11
~~~
##### Output:
~~~bash
[jboss@dev-tools eap7417]$ keytool -list -storetype pkcs11
Enter keystore password:  
Keystore type: PKCS11
Keystore provider: SunPKCS11-NSS-FIPS

Your keystore contains 0 entries

[jboss@dev-tools eap7417]$
~~~
&nbsp;
&nbsp;


[^1]: [How can I make RHEL FIPS compliant?](https://access.redhat.com/solutions/137833)

[^2]: ~~~bash
	[root@dev-tools ~]# yum install nss-tools.x86_64
	Updating Subscription Management repositories.
	Last metadata expiration check: 1:48:10 ago on Wed 10 Jul 2024 03:21:53 PM -05.
	Dependencies resolved.
	...
	...
	Installed:
		nss-tools-3.90.0-7.el9_4.x86_64                                                                                                                                                                                                             
	Complete!
	[root@dev-tools ~]#
	~~~

[^3]: ~~~bash
	[jboss@dev-tools security]$ pwd
	/usr/lib/jvm/java-11-openjdk-11.0.23.0.9-3.el9.x86_64/conf/security
	[jboss@dev-tools security]$ ls -ltrh
	total 72K
	-rw-r--r--. 1 root root 2.3K Apr 11 17:37 java.policy
	-rw-r--r--. 1 root root  195 Apr 11 17:40 nss.fips.cfg
	-rw-r--r--. 1 root root  139 Apr 11 17:40 nss.cfg
	-rw-r--r--. 1 root root  58K Apr 12 12:33 java.security ====>THIS
	drwxr-xr-x. 4 root root   56 Jun  4 11:31 policy
	[jboss@dev-tools security]$
	~~~

[^4]: ~~~bash
	[jboss@dev-tools eap7417]$ certutil -S -k rsa -n undertow -t "u,u,u" -x -s 	"CN=dev-tools.lab.xee.com, OU=XEE - SBR JSEC, O=RED HAT, L=LA CALERA, 	ST=CUNDINAMARCA, C=CO" -d fips/nssdb
	certutil: function failed: SEC_ERROR_BAD_DATABASE: security library: bad database.
	[jboss@dev-tools eap7417]$
	~~~
 
