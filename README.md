# Setting FIPS 140-2 Cryptography on RHEL 7 and EAP 7.4

### OBJECTIVE 
Test the steps of configuration for enabling FIPS-2 cryptography securing EAP management interfaces.
&nbsp;

#### Environment:
* Fedora 40
* EAP 7.4.17
* OpenJDK 11 (Red_Hat-11.0.23.0.9-1)
* Documentation:
[Enable FIPS 140-2 Cryptography for SSL/TLS on Red Hat Enterprise Linux 7 and Later](https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/7.4/html-single/how_to_configure_server_security/index#configure_ssl_fips_rhel7)
&nbsp;

#### Description:
One of the requeriments is enabling RHEL for FIPS-2, which can be read through the following article [^1]

1. **Enabling FIPS on RHEL**
~~~bash
	# fips-mode-setup --enable
~~~
~~~bash
	root@fgonzalezc:~# fips-mode-setup --enable
	Preserving current FIPS-based policy FIPS.
	Please review the subpolicies to ensure they only restrict, not relax the FIPS policy.
	Setting system policy to FIPS
	Note: System-wide crypto policies are applied on application start-up.
	It is recommended to restart the system for the change of policies
	to fully take place.
	FIPS mode will be enabled.
	Please reboot the system for the setting to take effect.
	root@fgonzalezc:~#
	root@fgonzalezc:~#	
	root@fgonzalezc:~# reboot
	root@fgonzalezc:~#
	root@fgonzalezc:~#	
	root@fgonzalezc:~# fips-mode-setup --check
	FIPS mode is enabled.
	Initramfs fips module is enabled.
	The current crypto policy (FIPS) is based on the FIPS policy.
	root@fgonzalezc:~#
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
~~~bash
	$ mkdir -p fips/configuration/nssdb
	$ chown fgonzalezc fips/configuration/nssdb
	$ modutil -create -dbdir fips/configuration/nssdb
~~~
##### Output:
~~~bash
	fgonzalezc@fgonzalezc:~$ modutil -create -dbdir fips/configuration/nssdb

	WARNING: Performing this operation while the browser is running could cause
	corruption of your security databases. If the browser is currently running,
	you should exit browser before continuing this operation. Type 
	'q <enter>' to abort, or <enter> to continue: 

	fgonzalezc@fgonzalezc:~$ tree fips/nssdb/
	fips/nssdb/
	├── cert9.db
	├── key4.db
	└── pkcs11.txt

	1 directory, 3 files
~~~
&nbsp;
~~~bash
	$ touch fips/configuration/nss_pkcsll_fips.cfg
~~~
Edit the file with the following contents y location of the NSS database. Then, save the file.
##### Output:
~~~bash
	fgonzalezc@fgonzalezc:~$ touch fips/configuration/nss_pkcsll_fips.cfg
	fgonzalezc@fgonzalezc:~$ cat nss_pkcsll_fips.cfg 
	name = nss-fips
	nssLibraryDirectory=/usr/lib64
	nssSecmodDirectory=/home/fgonzalezc/apps/EAP/eap7417/fips/configuration/nssdb
	nssDbMode = readOnly
	nssModule = fips
~~~
&nbsp;
&nbsp;

 4. **Edit the java security configuration file**
There is a file by default in the OpenJDK installation, but it's better if you define a custom file.
* Location of the file java.security: *`JBOSS_HOME/fips/configuration/`*

	The file `java.security` created is a copy of the default file found in the OpenJDK 		installation *`/usr/lib/jvm/java-11-openjdk-11.0.23.0.9-1.fc40.x86_64/conf/security/`*

	According to the documentation you have to modify the value for the the following keys:

~~~bash
	security.provider.1=sun.security.pkcs11.SunPKCS11 /usr/share/jboss-as/nss_pkcsll_fips.cfg
	security.provider.5=com.sun.net.ssl.internal.ssl.Provider SunPKCS11-nss-fips
~~~
&nbsp;

but according to the tests I had to modify a line more:
~~~bash
	fips.provider.1=SunPKCS11 /home/fgonzalezc/apps/EAP/eap7417/fips/configuration/nss_pkcsll_fips.cfg
~~~

##### Content of the file java.security:
~~~bash
....
#security.provider.1=SUN
security.provider.1=sun.security.pkcs11.SunPKCS11 /home/fgonzalezc/apps/EAP/eap7417/fips/configuration/nss_pkcsll_fips.cfg
....
#security.provider.5=SunJCE
security.provider.5=com.sun.net.ssl.internal.ssl.Provider SunPKCS11-nss-fips
....
fips.provider.1=SunPKCS11 /home/fgonzalezc/apps/EAP/eap7417/fips/configuration/nss_pkcsll_fips.cfg
....
~~~
&nbsp;
&nbsp;

 5. **Enable FIPS mode through the NSS database**
~~~bash
	$ modutil -fips true -dbdir /home/fgonzalezc/apps/EAP/eap7417/fips/nssdb
~~~
##### Output:
~~~bash
	fgonzalezc@fgonzalezc:~$ modutil -fips true -dbdir /home/fgonzalezc/apps/EAP/eap7417/fips/nssdb

	WARNING: Performing this operation while the browser is running could cause
	corruption of your security databases. If the browser is currently running,
	you should exit browser before continuing this operation. Type 
	'q <enter>' to abort, or <enter> to continue:    

	FIPS mode enabled.
~~~
&nbsp;
&nbsp;

 6. **Set the NSS database password**
~~~bash
	$ modutil -changepw "NSS FIPS 140-2 Certificate DB" -dbdir /home/fgonzalezc/apps/EAP/eap7417/fips/nssdb
~~~
##### Output:
~~~bash
	fgonzalezc@fgonzalezc:~$ modutil -changepw "NSS FIPS 140-2 Certificate DB" -dbdir /home/fgonzalezc/apps/EAP/eap7417/fips/nssdb

	WARNING: Performing this operation while the browser is running could cause
	corruption of your security databases. If the browser is currently running,
	you should exit browser before continuing this operation. Type 
	'q <enter>' to abort, or <enter> to continue:   

	Enter new password: 
	Re-enter new password: 
	Token "NSS FIPS 140-2 Certificate DB" password changed successfully.
~~~
&nbsp;
&nbsp;

6. **Create a certificate stored in the NSS database**
~~~bash
	$ certut il -S -k rsa -n undertow -t "u,u,u" -x -s "CN=localhost.eap7417-2,	OU=XEE - SBR JSEC, O=RED HAT, L=LA CALERA, ST=CUNDINAMARCA, C=CO" -d /home/fgonzalezc/apps/EAP/eap7417/fips/nssdb
~~~
##### Output:
~~~bash
	fgonzalezc@fgonzalezc:~$ certut il -S -k rsa -n undertow -t "u,u,u" -x -s "CN=localhost.eap7417-2, OU=XEE - SBR JSEC, O=RED HAT, L=LA CALERA, ST=CUNDINAMARCA, C=CO" -d /home/fgonzalezc/apps/EAP/eap7417/fips/nssdb
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
~~~
&nbsp;
&nbsp;


[^1]: [How can I make RHEL FIPS compliant?](https://access.redhat.com/solutions/137833)
