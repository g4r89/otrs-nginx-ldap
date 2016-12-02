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

mv /etc/nginx/nginx.conf{,.bak}
mv /etc/nginx/conf.d/default.conf{,.bak}
cp nginx.conf /etc/nginx/nginx.conf
cp default.conf /etc/nginx/conf.d/default.conf
cp fastcgi-wrapper /usr/local/bin/fastcgi-wrapper.pl

wget -O- http://nginxlibrary.com/downloads/perl-fcgi/fastcgi-wrapper | sed '/OpenSocket/s/127\.0\.0\.1\:8999/\/var\/run\/perl-fcgi\/perl-fcgi.sock/' > /usr/local/bin/fastcgi-wrapper.pl

cat <<EOF> /lib/systemd/system/perl-fcgi.service
[Unit]
Description=FastCGI Perl Wrapper
After=network.target
[Service]
User=otrs
Group=nginx
Type=simple
ExecStart=/usr/local/bin/fastcgi-wrapper.pl
[Install]
WantedBy=multi-user.target
EOF

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
