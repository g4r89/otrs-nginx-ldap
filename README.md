# otrs
centos7+otrs+nginx+fcgiwrap
# install
```bash
yum update -y
cat <<EOF> /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/\$basearch/
gpgcheck=0
enabled=1
EOF

cat <<EOF> /etc/yum.repos.d/mariadb.repo
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
EOF

yum install epel-release wget nginx mariadb-server -y

cat <<EOF> /etc/nginx/conf.d/otrs.conf
server {
listen 80;
server_name otrs.example.com;
root /opt/otrs/var/httpd/htdocs;

error_log /var/log/nginx/otrs-error.log warn;
location = / {
return 301 https://otrs.HOST/otrs/customer.pl;
}

location /otrs-web {
gzip on;
alias /opt/otrs/var/httpd/htdocs;
}

location ~ ^/otrs/(.*.pl)(/.*)?$ {
fastcgi_pass unix:/var/run/fcgiwrap.sock;
fastcgi_index index.pl;
fastcgi_param SCRIPT_FILENAME /opt/otrs/bin/fcgi-bin/$1;
include fastcgi_params;
}
}
EOF

yum localinstall https://dl.dropboxusercontent.com/u/2709550/FCGIwrap/fcgiwrap-1.1.0-3.20150530git99c942c.el7.centos.x86_64.rpm -y
systemctl enable --now fcgiwrap.socket

cat <<EOF> /etc/my.cnf.d/otrs.cnf
[mysqld]
max_allowed_packet   = 20M
query_cache_size     = 32M
innodb_log_file_size = 256M
EOF

systemctl start mysql
systemctl enable --now mariadb
/usr/bin/mysql_secure_installation

yum install perl perl-core perl-Archive-Zip perl-Crypt-Eksblowfish perl-Crypt-SSLeay perl-Date-Format perl-DBD-MySQL perl-IO-Socket-SSL perl-JSON-XS perl-Mail-IMAPClient perl-Net-DNS perl-LDAP perl-Template-Toolkit perl-Text-CSV_XS perl-XML-LibXML perl-XML-LibXSLT perl-XML-Parser perl-YAML-LibYAML -y

sed -i.bak '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
setenforce 0

wget -qO- http://ftp.otrs.org/pub/otrs/otrs-5.0.14.tar.gz | tar xvz -C /opt/
mv /opt/otrs-5.0.14 /opt/otrs && cd /opt/otrs
useradd -d /opt/otrs/ -g nginx -s /sbin/nologin -c 'OTRS System User' otrs
su otrs -s /bin/bash -c "/opt/otrs/bin/otrs.CheckModules.pl"

cp Kernel/Config.pm.dist Kernel/Config.pm
for foo in var/cron/*.dist; do mv $foo var/cron/`basename $foo .dist`; done
cp .procmailrc.dist .procmailrc
cp .fetchmailrc.dist .fetchmailrc
cp .mailfilter.dist .mailfilter

perl -cw /opt/otrs/bin/cgi-bin/index.pl
perl -cw /opt/otrs/bin/cgi-bin/customer.pl
perl -cw /opt/otrs/bin/otrs.Console.pl

/opt/otrs/bin/otrs.SetPermissions.pl --otrs-user=otrs --web-group=nginx

su otrs -s /bin/bash -c "/opt/otrs/bin/otrs.Console.pl Maint::Config::Rebuild";
su otrs -s /bin/bash -c "/opt/otrs/bin/otrs.Console.pl Maint::Cache::Delete";

cat <<EOF> /etc/systemd/system/otrs.service
[Unit]
Description=OTRS Help Desk.
After=network.target
[Service]
Type=forking
User=otrs
Group=nginx
ExecStart=/opt/otrs/bin/otrs.Daemon.pl start
ExecStop=/opt/otrs/bin/otrs.Daemon.pl stop
[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now nginx
systemctl enable --now otrs.service
reboot
