cloud_ddns
==========

Setup a custom dynamic DNS for instances in an OpenStack cloud

==========

SUMMARY of DDNS scripts
The “ddns” set of scripts allow one to create a dynamic DNS for OpenStack cloud instances. Cloud providers may default to using unique and long hostnames, like:
instance.ff0f61549f3c4c6cb72a64f16572a524.compute.subdomain.domain.com

One might prefer to use easier to remember hostnames, like:
	instance.subdomain.domain.com
 The “ddns” set of scripts allows one to use simpler hostnames. A cloud admin can use these scripts to to set this up a hostname scheme and dynamic DNS for all users in an OpenStack cloud. 

This solution includes the following:
    •	setup file
        o	setup_ddns.sh
    •	configuration file
        o	ddns_config.yaml
    •	ddns.sh executable
        o	this main script will be autogenerated by setup_ddns.sh
    •	python routines that are executed by scripts
        o	ddns.py, gen_bind_files.py, gen_ddns_sh.py and ddns_common.py
    •	templates directory, including templates for generating the following:
        o	ddns.sh, named.conf, forward and reverse zone files

PRE-REQUISITES
1. Pre-Requisites for the DNS Instance
    1.	Deploy a Centos/RHEL 6.5 instance for the DNS. It should be sufficient to choose a smaller flavor like “n1.small”.
    2.	Add a Floating IP
    3.	Setup security group rules for the DNS instance. The following ports should be open: TCP 22 (SSH), TCP 53 and UDP 53 (for DNS). 
    4.	If the iptables service is on, it may be easiest to turn it off
    5.	Ensure that the OpenStack CLI clients are installed 


PART 1 - HOW TO INSTALL

1. Install BIND
Install the bind and bind-utils packages if they have not already been installed. Note that bind sets up the DNS server and bind-utils has the nsupdate tools.

  [centos@ddns ~]$ sudo yum update -y
  [centos@ddns ~]$ sudo yum install -y bind bind-utils

2. Get the Cloud Admin’s OpenStack API Credentials, e.g. openrc.sh
Using the Admin’s credentials will allow the “DDNS “ scripts to see instances across projects. If openrc.sh prompts for a password, remove the prompt and add the password in the file. Including the password in the file will be required for the scripts to run automatically. Below is an example of how to test the openrc.sh file using the OpenStack CLI.

  [centos@ddns ~]$ source openrc.sh 
  [centos@ddns ~]$ nova list

3. Unzip the DDNS scripts
For example,
  [centos@ddns ~]$ unzip /tmp/ddns.zip

4. Setup ddns_config.yaml
  “ddns_config.yaml” has all of the configuration parameters that will be used to configure BIND and the “DDNS” scripts. Edit this file and input customized information. Then save the file.  It is important to make sure there are no errors, like no over lapping IP ranges in this file. 

Note that all fields can take only one value with the exception of “forwarders” and “ip_ranges”. Multiple forwarders and IP address ranges can be included. The IP address ranges are for both the fixed and floating OpenStack networks. These scripts require that the ranges be on octet boundaries, e.g. /24, /16 or /8.

Here is an already filled in sample of the required fields.

    domain_name: cloud.myuniverse.org
    dns_shortname: bigbang
    dns_fixed_ip: 10.130.52.121
    dns_floating_ip: 10.130.56.248
    forwarders:
      - 10.130.0.1
    ip_ranges:
      - 10.130.52.0/24

5. Run setup_ddns.sh to complete the DNS setup
setup_ddns.sh creates configuration files for the DNS updates as well as configuration files for BIND. It then moves the BIND files to the appropriate directories and restarts the DNS (“named”) service.  Feel free to view the script to see what it does. Run this script using “sudo”. 

  [centos@ddns ~]$ sudo ./setup_ddns.sh

6. Run ddns.sh to complete the setup of the Dynamic DNS
Now the main executable script, ddns.sh, has been created. It will use the OpenStack APIs to query instances and then update the DNS appropriately. Run the script.

  [centos@ddns ~]$ ./ddns.sh

7. Add ddnn.sh to cron to update the DNS automatically
The following example shows how to setup cron to call ddns.sh every minute.

  [centos@ddns ~]$ crontab –e  

Once in the file, add  “* * * * * <path-to-script>”.

To monitor if the cron is working monitor /var/log/messages with “tail –f /var/log/messages”. Use ping and dig or nslookup to test the name server. 

8. Test the DNS Server
Use ping and dig or nslookup to test the DNS server

PART 2 - HOW TO CONNECT INSTANCES
An instance needs to be configured to resolve using the new DNS. The easiest way to do this is to configure the image(s). In the image, change /etc/resolv.conf and /etc/sysconfig/network-scripts/ifcfg-eth0 as described below. These changes can also be done for an individual instance. 

Edit /etc/resolv.conf
First, change the contents of resolv.conf as follows. The “nameserver_ip” can be either the fixed or floating IP of the DNS instance.
  search <domain_name>
  nameserver <nameserver_ip>
For example:
  search cloud.mycompany.com
  nameserver 10.130.52.121

Add PEERDNS=”no” to /etc/sysconfig/network-scripts/ifcfg-eth0.
For example, type:
    echo "PEERDNS=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0


