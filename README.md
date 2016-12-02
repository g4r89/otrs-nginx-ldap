# otrs
centos7+otrs+nginx+fgci wrapper
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
server_name otrs.example.com otrs;
root /opt/otrs/var/httpd/htdocs;
index index.html;

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

yum install fcgi-devel spawn-fcgi -y
cd /usr/local/src/
git clone git://github.com/gnosek/fcgiwrap.git
cd fcgiwrap
autoreconf -i
./configure
make
make install

cat <<EOF> /etc/sysconfig/spawn-fcgi
FCGI_SOCKET=/var/run/fcgiwrap.socket
FCGI_PROGRAM=/usr/local/sbin/fcgiwrap
FCGI_USER=nginx
FCGI_GROUP=nginx
FCGI_EXTRA_OPTIONS="-M 0700"
OPTIONS="-u $FCGI_USER -g $FCGI_GROUP -s $FCGI_SOCKET -S $FCGI_EXTRA_OPTIONS -F 1 -P /var/run/spawn-fcgi.pid -- $FCGI_PROGRAM"

systemctl enable --now spawn-fcgi

cat <<EOF> /etc/my.cnf.d/otrs.cnf
[mysqld]
max_allowed_packet   = 20M
query_cache_size     = 32M
innodb_log_file_size = 256M
EOF

systemctl enable --now mariadb
/usr/bin/mysql_secure_installation

yum install perl perl-core perl-Archive-Zip perl-Crypt-Eksblowfish perl-Crypt-SSLeay perl-Date-Format perl-DBD-MySQL perl-IO-Socket-SSL perl-JSON-XS perl-Mail-IMAPClient perl-Net-DNS perl-LDAP perl-Template-Toolkit perl-Text-CSV_XS perl-XML-LibXML perl-XML-LibXSLT perl-XML-Parser perl-YAML-LibYML

wget -qO- http://ftp.otrs.org/pub/otrs/otrs-5.0.14.tar.gz | tar xvz -C /opt/
mv /opt/otrs-5.0.14 /opt/otrs && cd /opt/otrs
cp Kernel/Config.pm.dist Kernel/Config.pm

perl -cw /opt/otrs/bin/cgi-bin/index.pl
perl -cw /opt/otrs/bin/cgi-bin/customer.pl
perl -cw /opt/otrs/bin/otrs.Console.pl

useradd -d /opt/otrs/ -s /sbin/nologin -c 'OTRS System User' otrs
usermod -G nginx otrs
bin/otrs.SetPermissions.pl --web-group=nginx

sed -i.bak '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
setenforce 0

sytemctl enable --now perl-fcgi
sytemctl enable --now nginx
