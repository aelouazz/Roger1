# Roger_SkyLine_1

***author: aelouazz***

Introduction to the basics of system and network administration.

This project will allow for the installation of a Virtual Machine,
to discover the system and network bases as well as the many services used on a server machine.

Road to administrate a web server.

---

# Mandatory Part

## Virtual Machine Part

- [ ]  **A disk size of 8 GB**
- [ ]  **a main 4.2GB partition**
- [ ]  **up to date and installed all the packages and installing dependencies as root**

        apt-get update -y && apt upgrade -y
        apt-get install sudo vim ufw fail2ban portsentry apache2 mailutils postfix -y

## Network and Security Part

- [ ]  **create an non root user**

        sudo adduser aymane

- [ ]  **use sudo with this user**

    we need to add the privilege to the new user in "/etc/sudoers"

        aymane  ALL=(ALL:ALL)  NOPASSWD:ALL

---

### Setup a static IP/30

- [ ]  **configuring a static IP and a netmask /30**

    we need first to change the primary network interfaceint the file /etc/network/interfaces to the following:

        #FILE : /etc/network/interfaces
        #The primary network interface
        auto enp0s3

        # This file describes the network interfaces available on your system
        # and how to activate them. For more information, see interfaces(5).

        source /etc/network/interfaces.d/*

        # The loopback network interface
        auto lo
        iface lo inet loopback

        # The primary network interface
        auto	enp0s3

    then we setup the auto configuration with the static IP:

        #FILE: /etc/network/interfaces.d/enp0s3
        iface enp0s3 inet static
        	address 10.12.254.253
        	netmask 255.255.255.252
        	gateway 10.12.254.254

        #restart the networking service:
        sudo service networking restart

        #Check the current ip:
        ip addr

---

### Setup SSH configuration

- [ ]  **change the default SSH port:**

        #FILE: /etc/ssh/sshd_config    LINE: 13
        Port 55555

- [ ]  **Setup SSH the public and private RSA keys for SSH**
    - generate the public and private key pair on the client machine

            ssh-keygen -t rsa

    - copy the public key to the server

            ssh-copy-id -i id_rsa.pub user@10.11.200.247 -p 55555

- [ ]  **remove the SSH root login and the password authentication**

        #FILE: /etc/ssh/sshd.config
        #LINE32:
        PermitRootLogin no
        #LINE56:
        PasswordAuthentication no

- [ ]  **restart the sshd service**

        sudo service sshd restart

---

### limiting the open port using UFW FireWall

- [ ]  **enable ufw**

        #check if ufw already enabled:
        sudo ufw status
        #enable ufw in case its not:
        sudo ufw enable

- [ ]  **Specify the FireWall rules:**

        # SSH PORT
        sudo ufw allow 55555/tcp
        # HTTP PORT
        sudo ufw allow 80/tcp
        # HTTPS PORT
        sudo ufw allow 443

---

### Protection against DOS attack using Fail2ban

- [ ]  **Setup the Jail for SSH and HTTP-DOS**

        # FILE : /etc/fail2ban/jail.conf

        #Enable the following jail
        [sshd]
        enabled = true
        port = 42
        logpath = %(sshd_log)s
        backend = %(sshd_backend)s
        maxretry = 3
        bantime = 600

        #Add the following jail (we will setup its filter in the next step)
        [http-get-dos]
        enabled = true
        port = http,https
        filter = http-get-dos
        logpath = /var/log/apache2/access.log
        maxretry = 300
        findtime = 300
        bantime = 600
        action = iptables[name=HTTP, port=http, protocol=tcp]

- [ ]  **Setup the http-get-dos filter**

        # FILE : /etc/fail2ban/filter.d/http-get-dos.conf

        [Definition]
        failregex = ^<HOST> -.*"(GET|POST).*
        ignoreregex =

- [ ]  **reload the new firewall and dos-filter settings**

        sudo ufw reload
        sudo service fail2ban restart

---

### Setting up Port Scaning protection using Portsentry

- [ ]  **configuring the default Portsentry**

        # FILE : /etc/default/portsentry

        TCP_MODE="atcp"
        UDP_MODE="audp"

- [ ]  **Editing the portsentry configuration file**

        # FILE /etc/portsentry/portsentry.conf
        BLOCK_UDP="1"
        BLOCK_TCP="1"
        #Uncomment the following and comment the default
        KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
        #
        #Comment the following
        KILL_HOSTS_DENY="ALL: $TARGET$ : DENY

    - [ ]  then we need to restart portsentry to apply the changes

            sudo service portsentry restart

---

### Stopping the stupid useless services:

    sudo systemctl disable console-setup.service
    sudo systemctl disable keyboard-setup.service
    sudo systemctl disable apt-daily.timer
    sudo systemctl disable apt-daily-upgrade.timer
    sudo systemctl disable syslog.service
    #to list all services:
    sudo service --status-all

### Update packages

- [ ]  **Create two scripts to update and upgrade the packages using**

        sudo touch /var/log/update_script.log
        sudo chmod 777 /var/log/update_script.log
        #also they will save the log in a logfile:
        sudo touch /etc/scripts/update.sh
        sudo chmod 777 /etc/scripts/update.sh
        echo "sudo apt-get update -y >> /var/log/update_script.log" >> /etc/scripts/update.sh
        echo "sudo apt-get upgrade -y >> /var/log/update_script.log" >> /etc/scripts/update.sh

- [ ]  **make a scheduled task for the update**

        #FILE: /etc/crontab
        SHELL=/bin/bash
        PATH=/sbin:/bin:/usr/sbin:/usr/bin
        @reboot sudo /etc/scripts/update.sh
        0 4 * * 6 sudo /etc/scripts/update.sh

### monitoring Crontab modifications

- [ ]  **create a monitoring script**

    we need first to install postfix to be able to send the email:

        sudo apt install postfix -y

        #!/bin/bash
        # FILE: /etc/scripts/cronMonitor.sh
        SUM_COPY="/var/tmp/checksum"
        CRON_FILE="/etc/crontab"
        SUM=$(sudo md5sum $CRON_FILE)
        if [ ! -f $SUM_COPY ]
        then
        echo "$SUM" > $SUM_COPY
        exit 0;
        fi;
        if [ "$SUM" != "$(cat $SUM_COPY)" ];
        then
        echo "$SUM" > $SUM_COPY
        echo "$CRON_FILE has been modified ! '*_*" | mail -s "$CRON_FILE modified !" root@debian
        fi;

    add the following task to crontab

        0 0 * * * sudo /etc/scripts/cronMonitor.sh

---

---

# Optional Part

## Web Part

### Creating the SSL Certificate

    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt

- **openssl**

     ****a command for creating and maintaining certs, keys and more

- **req**

    means we want to use X.509 certificate signing request (CSR) management. The “X.509” is a public key infrastructure standard that SSL and TLS adheres to for its key and certificate management. We want to create a new X.509 cert. ****

- **-x509**

    This further modifies the previous subcommand by telling the utility that we want to make a self-signed certificate instead of generating a certificate signing request, as would normally happen.

- **-nodes**

    skip the option to secure our certificate with a passphrase. We need Apache to be able to read the file, without user intervention, when the server starts up. A passphrase would prevent this from happening because we would have to enter it after every restart.

- **-days 365**

    This option sets the length of time that the certificate will be considered valid. We set it for one year here.

- **-newkey**

    generate a new certificate and a new key at the same time. We did not create the key that is required to sign the certificate in a previous step, so we need to create it along with the certificate.

- **rsa:2048**

    tells it to make an RSA key that is 2048 bits long.

- **-keyout**

    This line tells OpenSSL where to place the generated private key file that we are creating.

- **-out**

    This tells OpenSSL where to place the certificate that we are creating.

---

after entering the command you will have to fill the following informations:

    Country Name (2 letter code) [AU]:US
    State or Province Name (full name) [Some-State]:New York
    Locality Name (eg, city) []:New York City
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Bouncy Castles, Inc.
    Organizational Unit Name (eg, section) []:Ministry of Water Slides
    Common Name (e.g. server FQDN or YOUR name) []:server_IP_address
    Email Address []:admin@your_domain.com

---

### Configuring Apache to Use SSL

**Creating an Apache Configuration Snippet with Strong Encryption Settings**

Create a new snippet in the /etc/apache2/conf-available directory. We will name the file ssl-params.conf to make its purpose clear:

    # FILE: /etc/apache2/conf-available/ssl-params.conf

    SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
    SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
    SSLHonorCipherOrder On
    # Disable preloading HSTS for now.  You can use the commented out header line that includes
    # the "preload" directive if you understand the implications.
    # Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Frame-Options DENY
    Header always set X-Content-Type-Options nosniff
    # Requires Apache >= 2.4
    SSLCompression off
    SSLUseStapling on
    SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
    # Requires Apache >= 2.4.11
    SSLSessionTickets Off

**Modifying the Default Apache SSL Virtual Host File**

Before we go any further, let’s back up the original SSL Virtual Host file

    sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf.bak

    # FILE : /etc/apache2/sites-available/default-ssl.conf

    <IfModule mod_ssl.c>
            <VirtualHost _default_:443>
                    ServerAdmin aelouazz@student.1337.ma
                    ServerName 10.12.254.253

                    DocumentRoot /var/www/html

                    ErrorLog ${APACHE_LOG_DIR}/error.log
                    CustomLog ${APACHE_LOG_DIR}/access.log combined

                    SSLEngine on

                    SSLCertificateFile      /etc/ssl/certs/apache-selfsigned.crt
                    SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

                    <FilesMatch "\.(cgi|shtml|phtml|php)$">
                                    SSLOptions +StdEnvVars
                    </FilesMatch>
                    <Directory /usr/lib/cgi-bin>
                                    SSLOptions +StdEnvVars
                    </Directory>

            </VirtualHost>
    </IfModule>

**Modifying the HTTP Host File to Redirect to HTTPS**

    # FILE : /etc/apache2/sites-available/000-default.conf

    <VirtualHost *:80>
    	ServerAdmin webmaster@localhost
    	DocumentRoot /var/www/html
    	**Redirect "/" "https://10.12.254.253/"**
    	ErrorLog ${APACHE_LOG_DIR}/error.log
    	CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>

**To load the new Configuration we run the following commands**

    # Enable mod_ssl, the Apache SSL module, and mod_headers,
    # which is needed by some of the settings in our SSL snippet,
    # with the a2enmod command:
    sudo a2enmod ssl
    sudo a2enmod headers

    # enable your SSL Virtual Host
    sudo a2ensite default-ssl.conf

    # enable your ssl-params.conf file, to read in the values you’ve set
    sudo a2enconf ssl-params

    systemctl reload apache2

**check to make sure that there are no syntax errors in our files**

    sudo apache2ctl configtest
    # output SHOULD BE:  Syntax OK

### Deployment of the website using GIT

***ON SERVER***

- [ ]  **Install git**

        sudo apt-get install git

- [ ]  **setup a location to push and pull from (Repo)**

        cd /var/repo
        mkdir website1.com.git
        cd website1.com.git

        # we need to initialize this repo now
        # --bare: make sure the repo doesn't contain any of the files
        #  but just the versions because the source code is in /var/www/html
        git init --bare
        sudo chmod 777 -R ../website.com.git
        # enter the file hooks that got created by the git init
        cd hooks

        # post-receive is one of the hooks called after receiving pushed code
        vim post-receive

        #################################################################
        # FILE : /var/repo/website1.com.git/hooks/post-receive

        #!/bin/sh
        git --work-tree=/var/www/website1.com --git-dir=/var/repo/website1.com.git checkout -f
        #################################################################

        # initialize the GIT inside the directory of the project
        chmod +x post-receive

---

***ON CLIENT***

    # initialize the GIT inside the directory of the project
    git init

    # specifies the list of files and dirs to ignore when pushing
    vim .gitignore

    # in order for git to know where to push the files we need to set it up
    # live is just a name.
    git remote add live ssh://aymane@10.12.254.13/var/repo/website1.com.git

    # DONE bellow are some usefull commands:
    # to deploy:
    git add .
    git status
    git commit -m 'website is done'
    git push live master
    # to undo push:
    git log
    git revert [commit id]
    git push live master%
