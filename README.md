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


[^1]: [How can I make RHEL FIPS compliant?](https://access.redhat.com/solutions/137833)
