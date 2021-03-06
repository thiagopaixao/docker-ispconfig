#
#                    ##        .            
#              ## ## ##       ==            
#           ## ## ## ##      ===            
#       /""""""""""""""""\___/ ===        
#  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
#       \______ o          __/            
#         \    \        __/             
#          \____\______/                
# 
#          |          |
#       __ |  __   __ | _  __   _
#      /  \| /  \ /   |/  / _\ | 
#      \__/| \__/ \__ |\_ \__  |
#
# Dockerfile for ISPConfig with MariaDB database
#
# https://www.howtoforge.com/tutorial/perfect-server-debian-8-jessie-apache-bind-dovecot-ispconfig-3/
#
FROM debian:jessie

MAINTAINER Uni Mock <unimock@gate-m.de>

# Let the container know that there is no tty
ENV DEBIAN_FRONTEND noninteractive

# --- set timezone
ARG BUILD_TZ="Europe/Berlin"
RUN echo "$BUILD_TZ" > /etc/timezone && \
    dpkg-reconfigure -f noninteractive tzdata


# --- 1 Preliminary
RUN apt-get -y update && apt-get -y upgrade && apt-get -y install rsyslog rsyslog-relp logrotate supervisor git sendemail rsnapshot heirloom-mailx
RUN touch /var/log/cron.log
# Create the log file to be able to run tail
RUN touch /var/log/auth.log

# --- 2 Install the SSH server
RUN apt-get -y install ssh openssh-server rsync

# --- 3 Install a shell text editor
RUN apt-get -y install nano vim-nox

# --- 5 Update Your Debian Installation
ADD ./etc/apt/sources.list /etc/apt/sources.list
RUN apt-get -y update && apt-get -y upgrade

# --- 6 Change The Default Shell
RUN echo "dash  dash/sh boolean no" | debconf-set-selections
RUN dpkg-reconfigure dash

# --- 7 Synchronize the System Clock
RUN apt-get -y install ntp ntpdate


# --- 8 Install Postfix, Dovecot, MySQL, phpMyAdmin, rkhunter, binutils
ARG BUILD_MYSQL_PW="pass"
RUN echo "mysql-server mysql-server/root_password password ${BUILD_MYSQL_PW}"           | debconf-set-selections
RUN echo "mysql-server mysql-server/root_password_again password ${BUILD_MYSQL_PW}"     | debconf-set-selections
RUN echo "mariadb-server mariadb-server/root_password password ${BUILD_MYSQL_PW}"       | debconf-set-selections
RUN echo "mariadb-server mariadb-server/root_password_again password ${BUILD_MYSQL_PW}" | debconf-set-selections
RUN apt-get -y install postfix postfix-mysql postfix-doc mariadb-client mariadb-server openssl getmail4 rkhunter binutils dovecot-imapd dovecot-pop3d dovecot-mysql dovecot-sieve dovecot-lmtpd sudo
ADD ./etc/postfix/master.cf /etc/postfix/master.cf
RUN service postfix restart
RUN service mysql restart

# --- 9 Install Amavisd-new, SpamAssassin And Clamav
RUN apt-get -y install amavisd-new spamassassin clamav clamav-daemon zoo unzip bzip2 arj nomarch lzop cabextract apt-listchanges libnet-ldap-perl libauthen-sasl-perl clamav-docs daemon libio-string-perl libio-socket-ssl-perl libnet-ident-perl zip libnet-dns-perl
ADD ./etc/clamav/clamd.conf /etc/clamav/clamd.conf
RUN freshclam
RUN service spamassassin stop
RUN systemctl disable spamassassin

# --- 10 Install Apache2, PHP5, phpMyAdmin, FCGI, suExec, Pear, And mcrypt
RUN echo 'phpmyadmin phpmyadmin/dbconfig-install boolean true' | debconf-set-selections
# RUN echo 'phpmyadmin phpmyadmin/app-password-confirm password your-app-pwd' | debconf-set-selections
RUN echo "phpmyadmin phpmyadmin/mysql/admin-pass password ${BUILD_MYSQL_PW}" | debconf-set-selections
# RUN echo 'phpmyadmin phpmyadmin/mysql/app-pass password your-app-db-pwd' | debconf-set-selections
RUN echo 'phpmyadmin phpmyadmin/reconfigure-webserver multiselect apache2' | debconf-set-selections
RUN service mysql restart && apt-get -y install apache2 apache2.2-common apache2-doc apache2-mpm-prefork apache2-utils libexpat1 ssl-cert libapache2-mod-php5 php5 php5-common php5-gd php5-mysql php5-imap phpmyadmin php5-cli php5-cgi libapache2-mod-fcgid apache2-suexec php-pear php-auth php5-mcrypt mcrypt php5-imagick imagemagick libruby libapache2-mod-python php5-curl php5-intl php5-memcache php5-memcached php5-pspell php5-recode php5-sqlite php5-tidy php5-xmlrpc php5-xsl memcached libapache2-mod-passenger
RUN a2enmod suexec rewrite ssl actions include dav_fs dav auth_digest cgi

# --- 12 XCache and PHP-FPM
RUN apt-get -y install php5-xcache
# php5 fpm (non-free)
# RUN apt-get -y install libapache2-mod-fastcgi php5-fpm
# RUN a2enmod actions fastcgi alias
# RUN service apache2 restart

# --- 13 Install Mailman
RUN echo 'mailman mailman/default_server_language en' | debconf-set-selections
RUN apt-get -y install mailman
# RUN ["/usr/lib/mailman/bin/newlist", "-q", "mailman", "mail@mail.com", "pass"]
ADD ./etc/aliases /etc/aliases
RUN newaliases
RUN service postfix restart
RUN ln -s /etc/mailman/apache.conf /etc/apache2/conf-enabled/mailman.conf

# --- 14 Install PureFTPd And Quota
# install package building helpers
RUN apt-get -y --force-yes install dpkg-dev debhelper openbsd-inetd
# install dependancies
RUN apt-get -y build-dep pure-ftpd
# build from source
RUN mkdir /tmp/pure-ftpd-mysql/ && \
    cd /tmp/pure-ftpd-mysql/ && \
    apt-get source pure-ftpd-mysql && \
    cd pure-ftpd-* && \
    sed -i '/^optflags=/ s/$/ --without-capabilities/g' ./debian/rules && \
    dpkg-buildpackage -b -uc
# install the new deb files
RUN dpkg -i /tmp/pure-ftpd-mysql/pure-ftpd-common*.deb
RUN dpkg -i /tmp/pure-ftpd-mysql/pure-ftpd-mysql*.deb
# Prevent pure-ftpd upgrading
RUN apt-mark hold pure-ftpd-common pure-ftpd-mysql
# setup ftpgroup and ftpuser
RUN groupadd ftpgroup
RUN useradd -g ftpgroup -d /dev/null -s /etc ftpuser
RUN apt-get -y install quota quotatool
ADD ./etc/default/pure-ftpd-common /etc/default/pure-ftpd-common
RUN echo 1 > /etc/pure-ftpd/conf/TLS
RUN mkdir -p /etc/ssl/private/

# --- 15 Install BIND DNS Server
RUN apt-get -y install bind9 dnsutils

# --- 16 Install Vlogger, Webalizer, And AWStats
RUN apt-get -y install vlogger webalizer awstats geoip-database libclass-dbi-mysql-perl
ADD etc/cron.d/awstats /etc/cron.d/

# --- 17 Install Jailkit
RUN apt-get -y install build-essential autoconf automake libtool flex bison debhelper binutils
RUN cd /tmp && wget http://olivier.sessink.nl/jailkit/jailkit-2.17.tar.gz && tar xvfz jailkit-2.17.tar.gz && cd jailkit-2.17 && ./debian/rules binary
RUN cd /tmp && dpkg -i jailkit_2.17-1_*.deb && rm -rf jailkit-2.17*

# --- 18 Install fail2ban
RUN apt-get -y install fail2ban
ADD ./etc/fail2ban/jail.local /etc/fail2ban/jail.local
ADD ./etc/fail2ban/filter.d/pureftpd.conf /etc/fail2ban/filter.d/pureftpd.conf
ADD ./etc/fail2ban/filter.d/dovecot-pop3imap.conf /etc/fail2ban/filter.d/dovecot-pop3imap.conf
RUN echo "ignoreregex =" >> /etc/fail2ban/filter.d/postfix-sasl.conf
RUN service fail2ban restart

# --- 19 Install squirrelmail
#RUN apt-get -y install squirrelmail
#ADD ./etc/apache2/conf-enabled/squirrelmail.conf /etc/apache2/conf-enabled/squirrelmail.conf
#ADD ./etc/squirrelmail/config.php /etc/squirrelmail/config.php
#RUN chown root:www-data /etc/squirrelmail/config.php && chmod g+r /etc/squirrelmail/config.php
#RUN mkdir /var/lib/squirrelmail/tmp
#RUN chown www-data /var/lib/squirrelmail/tmp
#RUN service mysql restart

# --- 19 Install roundcube
ARG BUILD_ROUNDCUBE="1.1.6"
RUN mkdir /opt/roundcube && cd /opt/roundcube && \
    wget https://github.com/roundcube/roundcubemail/releases/download/${BUILD_ROUNDCUBE}/roundcubemail-${BUILD_ROUNDCUBE}.tar.gz && \
    tar xfz roundcubemail-${BUILD_ROUNDCUBE}.tar.gz && mv roundcubemail-${BUILD_ROUNDCUBE}/* . && \
    mv roundcubemail-${BUILD_ROUNDCUBE}/.htaccess . && rmdir roundcubemail-${BUILD_ROUNDCUBE} && rm roundcubemail-${BUILD_ROUNDCUBE}.tar.gz && \
    chown -R www-data:www-data /opt/roundcube
RUN service mysql restart && mysql -h localhost -uroot -ppass -e "CREATE DATABASE roundcubemail;"
# RUN service mysql restart && mysql -h localhost -uroot -ppass -e "CREATE DATABASE roundcubemail; GRANT ALL PRIVILEGES ON roundcubemail.* TO roundcube@localhost IDENTIFIED BY 'secretpassword' ; flush privileges;"
RUN service mysql restart && mysql -h localhost -uroot -p${BUILD_MYSQL_PW} roundcubemail < /opt/roundcube/SQL/mysql.initial.sql
RUN cd /opt/roundcube/config && cp -pf config.inc.php.sample config.inc.php
#RUN sed -i "s/\$config[\'db_dsnw\'] = \'mysql:\/\/roundcube:pass@localhost\/roundcubemail';/\$config[\'db_dsnw\'] = \'mysql:\/\/roundcube:secretpassword@localhost\/roundcubemail\';/g" /opt/roundcube/config/config.inc.php
ADD ./etc/apache2/roundcube.conf /etc/apache2/conf-enabled/roundcube.conf
RUN service apache2 restart
RUN service mysql restart
#
# --- 20 Install ispconfig plugins for roundcube
RUN git clone https://github.com/w2c/ispconfig3_roundcube.git /tmp/ispconfig3_roundcube/ && mv /tmp/ispconfig3_roundcube/ispconfig3_* /opt/roundcube/plugins && rm -Rvf /tmp/ispconfig3_roundcube
run echo "\$rcmail_config['plugins'] = array(\"jqueryui\", \"ispconfig3_account\", \"ispconfig3_autoreply\", \"ispconfig3_pass\", \"ispconfig3_spam\", \"ispconfig3_fetchmail\", \"ispconfig3_filter\");" >> opt/roundcube/config/config.inc.php
RUN cd /opt/roundcube/plugins && mv ispconfig3_account/config/config.inc.php.dist ispconfig3_account/config/config.inc.php
# --- 20 Install ISPConfig 3
ARG BUILD_ISPCONFIG="ISPConfig-3-stable"
RUN cd /tmp && cd . && wget http://www.ispconfig.org/downloads/${BUILD_ISPCONFIG}.tar.gz
RUN cd /tmp && tar xfz ${BUILD_ISPCONFIG}.tar.gz
RUN service mysql restart

# --- 7 Free SSL RUN mkdir /opt/certbot
RUN mkdir -p /opt/certbot && cd /opt/certbot && wget https://dl.eff.org/certbot-auto && chmod a+x ./certbot-auto
#RUN ./certbot-auto 

# --- 22 Install inotify-tools, less, apache-svn 
RUN apt-get -y install inotify-tools less subversion libapache2-svn


# --- 23 Install printing stuff
ARG BUILD_PRINTING="no"
RUN if [ "$BUILD_PRINTING" = "yes" ] ; then  apt-get install --fix-missing  -y libdmtx-utils dblatex latex-make cups-client lpr ; fi ;

# ADD ./etc/mysql/my.cnf /etc/mysql/my.cnf
ADD ./etc/postfix/master.cf /etc/postfix/master.cf

RUN echo "export TERM=xterm" >> /root/.bashrc

EXPOSE 20 21 22 53/udp 53/tcp 80 443 953 8080 30000 30001 30002 30003 30004 30005 30006 30007 30008 30009 3306
##################################################################################################
# starting ispconfig3_install/install/install.php
#
ADD ./autoinstall.ini /tmp/ispconfig3_install/install/autoinstall.ini
ARG BUILD_HOSTNAME="myhost.test.com"
RUN sed -i "s/^hostname=server1.example.com$/hostname=${BUILD_HOSTNAME}/g"                         /tmp/ispconfig3_install/install/autoinstall.ini
RUN sed -i "s/^ssl_cert_common_name=server1.example.com$/ssl_cert_common_name=${BUILD_HOSTNAME}/g" /tmp/ispconfig3_install/install/autoinstall.ini
RUN service mysql restart && php -q /tmp/ispconfig3_install/install/install.php      --autoinstall=/tmp/ispconfig3_install/install/autoinstall.ini
################################################################################################
# the key and cert for pure-ftpd should be available :
#
RUN cd /usr/local/ispconfig/interface/ssl ; cat ispserver.key ispserver.crt > ispserver.pem
RUN cd /etc/ssl/private ; ln -sf /usr/local/ispconfig/interface/ssl/ispserver.pem pure-ftpd.pem
################################################################################################
#ADD ./ISPConfig_Clean-3.0.5 /tmp/ISPConfig_Clean-3.0.5
#RUN cp -r /tmp/ISPConfig_Clean-3.0.5/interface /usr/local/ispconfig/
#RUN service mysql restart && mysql -p${BUILD_MYSQL_PW} < /tmp/ISPConfig_Clean-3.0.5/sql/ispc-clean.sql
############################################################################################
#
# docker-extensions
#
RUN mkdir -p /usr/local/bin
COPY ./bin/*             /usr/local/bin/
RUN chmod a+x /usr/local/bin/*


#
# establisch supervisord
#
ADD /supervisor /etc/supervisor
# link old /etc/init.d/ startup scripts to supervisor
RUN ls -m1    /etc/supervisor/services.d | while read i; do mv /etc/init.d/$i /etc/init.d/$i-orig ;  ln -sf /etc/supervisor/super-init.sh /etc/init.d/$i ; done
RUN ln -sf    /etc/supervisor/systemctl /bin/systemctl
RUN chmod a+x /etc/supervisor/* /etc/supervisor/*.d/*
copy /supervisor/invoke-rc.d /usr/sbin/invoke-rc.d 
#
# create directory for service volume
#
RUN mkdir -p /service ; chmod a+rwx /service
ADD track.gitignore /.gitignore

#
# Create bootstrap archives
#
RUN cp -v /etc/passwd /etc/passwd.bootstrap
RUN cp -v /etc/shadow /etc/shadow.bootstrap
RUN cp -v /etc/group  /etc/group.bootstrap
RUN mkdir -p /bootstrap ;  tar -C /var/vmail -czf /bootstrap/vmail.tgz .
RUN mkdir -p /bootstrap ;  tar -C /var/www   -czf /bootstrap/www.tgz  .
ENV TERM xterm
#
# startup script
#
ADD ./start.sh /start.sh
RUN chmod 755 /start.sh
CMD ["/start.sh"]
