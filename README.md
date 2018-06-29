**Early replacement of OSSEC Server 2.8 with Wazuh 3.x manager on standalone installs of Security Onion 14.04 or 16.04**

It is already planned for the OSSEC 2.8 package in Security Onion to be completely replaced with a Wazuh 3.x Docker container.
In the meantime, it is possible to swap the latest Wazuh 3.x Manager in place of the Security Onion packaged OSSEC 2.8 Server.

* Sudo to root

  ```sudo su -```

* Stop OSSEC

  ```/var/ossec/bin/ossec-control stop```

* Take the three wazuh-* scripts in this repo and place them in /usr/local/bin/ on your SO standalone system, also making them executable by root.

* Run this once to move the legacy OSSEC files to an inactive location (/var/ossec-so/) so Wazuh can be installed cleanly
  
  ```/usr/local/bin/wazuh-post-soup init```

* Follow this guide for installing Wazuh from source. You cannot do a package install because the Wazuh package will conflict with the SO ossec-hids-server package.
  * https://documentation.wazuh.com/3.x/installation-guide/installing-wazuh-agent/wazuh_agent_sources.html
  * At this point Wazuh is now in /var/ossec and the old OSSEC is in /var/ossec-so so that legacy OSSEC can be temporarily swapped back in place during soup updates and runs of sosetup to avoid disrupting those processes or having Wazuh files overwritten.

* Enable syslog output in Wazuh like SO does with OSSEC

  ```/var/ossec/bin/ossec-control enable client-syslog```
  
* Add this to /var/ossec/etc/ossec.conf directly below the `</global>` line
  ```
  <syslog_output>
    <server>127.0.0.1</server>
  </syslog_output>
  ```
Set these values in Wazuh's /var/ossec/etc/ossec.conf to what your old OSSEC setup has (see /var/ossec-so/etc/ossec.conf)
  * `<logall>` - will be yes
  * `<log_alert_level>` - defaults to 1
  * `<email_alert_level>` - defaults to 7

* Replace this in /var/ossec/etc/ossec.conf 
  ```
  <!--
    <active-response>
      active-response options here
    </active-response>
    -->
  ````
  with whatever `<active-response>` sections you have in the old /var/ossec-so/etc/ossec.conf file.

* Carry across any desired missing `<localfile>` sections from old /var/ossec-so/etc/ossec.conf to new /var/ossec/etc/ossec.conf

* Add this line to the ``<syscheck>`` section of /var/ossec/etc/ossec.conf
  * `<directories check_all="yes">/var/ossec/etc</directories>`

* If you had any custom rules in OSSEC, then copy them to their new location in Wazuh:
  ```cp /var/ossec-so/rules/local_rules.xml /var/ossec/etc/rules/```

* If you had set up an OSSEC centralized configuration, then copy it over to the new Wazuh location:
  ```cp /var/ossec-so/etc/shared/* /var/ossec/etc/shared/default/```

* Copy over the securityonion OSSEC rule file as well as symlink the old /var/ossec/rules directory to Wazuh's location for local rules.
  ```
  cp /var/ossec-so/rules/securityonion_rules.xml /var/ossec/etc/rules/
  ln -s /var/ossec/etc/rules/ /var/ossec/rules
  mkdir /var/ossec/rules/backup
  ```

* Start Wazuh Manager
   ```/var/ossec/bin/ossec-control restart```

* If you have any OSSEC agents that were reporting to SO's OSSEC server, then uninstall OSSEC agent from those systems and replace it with the latest Wazuh agent.  Then re-register them with the Wazuh manager much like you did with OSSEC.  

* If instead you want to preserve the old agent registrations, this should probably work:
  * On SO copy /var/ossec-so/etc/client.keys to /var/ossec/etc/client.keys and restart Wazuh manager with **/var/ossec/bin/ossec-control restart**.  
  * Also backup each agent's local client.keys file before removing OSSEC and installing Wazuh agent, and then restore each agent's old client.keys to its original location and restart Wazuh agent.

The initial run of **wazuh-post-soup** patched the stock SO **soup** script so that it will run **wazuh-pre-soup** before updating, and **wazuh-post-soup** after updating.  This is so that during thr update process, the legacy OSSEC files are temporarily located back in their original /var/ossec/ location, and after the update  they are moved back to /var/ossec-so/ and Wazuh is moved back into /var/ossec/.  Updated versions of the **soup** script itself should automatically be re-patched by **wazuh-post-soup**.

Note if you run **sosetup**, you need to first run **wazuh-pre-soup** and after **sosetup** is done, run **wazuh-post-soup**.  The **sosetup** is not patched to add these steps automatically.

If you want to go back to OSSEC, just run **wazuh-pre-soup**.  That will push Wazuh over to /var/ossec-wazuh/ and restore the legacy OSSEC to /var/ossec.  Also unpatch **soup** with this:
 ```
  sed -i '/wazuh-pre-soup/d' /usr/sbin/soup
  sed -i '/wazuh-post-soup/d' /usr/sbin/soup
 ```
OSSEC will remain active unless you run a subsequent **wazuh-post-soup** at which point Wazuh will be active again.
