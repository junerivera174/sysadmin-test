# sysadmin-test
This project used ansible to automate the deployment of two apache webservers, mysql database, and a trading_bot backend service. This project also includes a script which collects system metrics

################################
DOCUMENTATION - Ansible playbook
################################

Before being able to use this playbook, ensure your environment is configured correctly! 

(1) Ensure your ansible-user exists has the appropriate privileges on the control node and all managed nodes
(2) Ensure you have properly set-up ssh-key authentication between your ansible control node and all managed nodes 
(3) Ensure your local-dns file (/etc/hosts) is configured properly and resolves the ip addresses of all managed nodes
(4) Lastly, ensure your ansible control node is able to properly communicate with all the managed nodes using the ping module 
(5) All ansible playbooks are run with the command >> ansible-playbook <name_of_playbook> >> from the same project directory you are in

***
if you are unsure of how to complete these tasks - reference the playbook titled "ansible_setup.yml" 
***

When looking at the ansible-playbooks, there are a few things to take into consideration when trying to deploy it:

(1) The ansible playbook breaks down the different tasks into seperate roles (located within the roles directory) > this allows for more flexible deployment in case of future changes

the four roles include:
 - deploy_webserver, deploy_database, deploy_dns, deploy_backend

(2) When possible, variables have been included to avoid hard-coding the syntax into the ansible playbook > this allows for more flexible deployment in case of future changes. However, there are still some aspects of the roles you should take into consideration:

All ip addresses and passwords used are just for example purposes, when trying to implement the playbook in your own environment you should change the ip addresses as needed and create passwords in accordance with your companies password policies. This applies to the roles deploy_database and deploy_dns (shown below). 

deploy_database (applies to the following tasks) 

- name: create database users
  community.mysql.mysql_user:
    name: "{{ item.name }}"
    password: "{{ item.password }}"
    priv: "database.*:SELECT,INSERT,UPDATE,DELETE" (NOTE - you can change these privileges to your needs) 
    host: "{{ item.host }}"
    state: present
   loop:
    - name: webserver1 (# generic name >> change to your needs)
      password: secret_password_1 (# generic password >> change to your needs) 
      host: 192.168.0.102 (# generic ip_address >> change to your needs) 
    - name: webserver2 (# generic name >> change to your needs)
      password: secret_password_2 (# generic password >> change to your needs)
      host: 192.168.0.103 (# generic ip_address >> change to your needs)

- name: create backend user
  community.mysql.mysql_user:
    name: backend (# generic name >> change to your needs) 
    password: backend_password (# generic password >> change to your needs) 
    priv: "database.*:ALL" (NOTE - you can change these privileges to your needs)
    host: 192.168.0.105 (# generic ip_address >> change to your needs) 
    state: present

deploy_dns (applies to the following task) 

- name: allow dns queries from same subnet
  ansible.builtin.lineinfile:
    path: /etc/named.conf
    regex: "^allow-query { localhost; };"
    line: "allow-query { localhost; 192.168.0.0/24; };" (# change the subnet to your needs) 

- name: configure the zone file
  ansible.builtin.blockinfile:
    path: /var/named/dns.example.com
    create: true
    block: |
      $TTL 1h
      @   IN  SOA dns.example.com. admin.dns.example.com. (    (# change the admin email to your needs) 
              2025082201 ; Serial (YYYYMMDDnn)
              1h         ; Refresh
              15m        ; Retry
              1w         ; Expire
              1h )       ; Minimum TTL

      ; Authoritative name server
          IN  NS  dns.example.com.

      ; A record for the nameserver
          dns IN  A   {{ hostvars['dnsserver']['ansible_facts']['default_ipv4']['address'] }} 

      ; Round-robin A records for the web servers
      @   IN  A   {{ hostvars['webserver1']['ansible_facts']['default_ipv4']['address'] }}
      @   IN  A   {{ hostvars['webserver2']['ansible_facts']['default_ipv4']['address'] }}

change the names after hostvars >> dnsserver & webserver >> to how your managed nodes are defined within the /etc/hosts file

#################################
DOCUMENTATION - Monitoring Script
#################################

This project also includes a bash script which will collect system metrics on cpu_usage, memory_usage, and free_disk space in the root partition into a single line csv file. For ease of finding the script file, it has the .sh extension

Tips on running the script: 

(1) make sure the script is exectutable (sudo chmod +x script_name)
(2) you can type the absolute path to the script file (assuming you are in a different directory)
(3) you can type ./script_name or bash script_name (assuming you are in the same directory as the script itself) 

Tips on how to interpret the data for analysis: 

(1) cpu_usage = this metric allows you to see how overloaded your system might be. A consistently high cpu_usage may indicate that your system is overloaded and can't handle the amount of processes it is being asked to execute. This may be indicative of performance issues - due to things like slow disks, memory issues, database queries, compiling code, system backups - which can lead to system instability and performance delays. This metric will allow you to see if your system is being overloaded which you can then troubleshoot 

(2) memory_usage = this metric allows you to see how much memory an application or process is consuming. A consistenly high memory_usage may indicate that your system is running out of memory and can't allocate sufficient memory to other applications or processes to be executed. This may be indicative of performance issues - due to things like memory leaks from processes which dont release memory once finished, insufficient RAM (heavy swapping), misconfigured applications (mysql databases) - which can lead to system instability and performance delays. This metric will allow you to see if your system is starving for memory which you can then troubleshoot

(3) free_disk_space = this metric allows you to see how much free space your system has. A consistently low free_disk_space may indicate that your system is running out of space and will eventually not be able to store or write new information. Applications like databases (mysql) need space to write new information and filesystems need space to store and format information (ext4, xfs). This metric will allow you to see if your system is running out of space which you can then troubleshoot. 

