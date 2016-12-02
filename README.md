# otrs
centos7+otrs+nginx+fcgiwrap
# install nginx and mysql
```bash
sed -i.bak '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
setenforce 0

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

yum install epel-release wget nginx mariadb-server perl perl-core -y

cat <<EOF> /etc/my.cnf.d/otrs.cnf
[mysqld]
max_allowed_packet   = 20M
query_cache_size     = 32M
innodb_log_file_size = 256M
EOF

systemctl start mysql
systemctl enable --now nginx
systemctl enable --now mariadb
/usr/bin/mysql_secure_installation
```
# config nginx and fcgiwrap
```bash
mv /etc/nginx/conf.d/default.conf{,.bak}
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
fastcgi_pass unix:/var/run/otrs/perl_cgi-dispatch.sock;
fastcgi_index index.pl;
fastcgi_param SCRIPT_FILENAME   /opt/otrs/bin/fcgi-bin/$1;
fastcgi_param QUERY_STRING      $query_string;
fastcgi_param REQUEST_METHOD    $request_method;
fastcgi_param CONTENT_TYPE      $content_type;
fastcgi_param CONTENT_LENGTH    $content_length;
fastcgi_param GATEWAY_INTERFACE CGI/1.1;
fastcgi_param SERVER_SOFTWARE   nginx;
fastcgi_param SCRIPT_NAME       $fastcgi_script_name;
fastcgi_param REQUEST_URI       $request_uri;
fastcgi_param DOCUMENT_URI      $document_uri;
fastcgi_param DOCUMENT_ROOT     $document_root;
fastcgi_param SERVER_PROTOCOL   $server_protocol;
fastcgi_param REMOTE_ADDR       $remote_addr;
fastcgi_param REMOTE_PORT       $remote_port;
fastcgi_param SERVER_ADDR       $server_addr;
fastcgi_param SERVER_PORT       $server_port;
fastcgi_param SERVER_NAME       $server_name;
}
}
EOF

yum groupinstall 'Development Tools' -y
yum install fcgi-devel -y
cd /usr/local/src/
git clone git://github.com/gnosek/fcgiwrap.git
cd fcgiwrap
autoreconf -i
./configure
make
make install

cat <<EOF> /lib/systemd/system/fcgiwrap.service
[Unit]
Description=Simple CGI Server
After=nss-user-lookup.target

[Service]
ExecStart=/usr/sbin/fcgiwrap
User=nginx
Group=nginx

[Install]
Also=fcgiwrap.socket
EOF

cat <<EOF> /lib/systemd/system/fcgiwrap.socket
[Unit]
Description=fcgiwrap Socket

[Socket]
ListenStream=/var/run/fcgiwrap.sock

[Install]
WantedBy=sockets.target
EOF

systemctl enable fcgiwrap.socket
systemctl enable --now fcgiwrap.service

service nginx restart
```
# install otrs
```bash
yum install perl perl-core perl-Archive-Zip perl-Crypt-Eksblowfish perl-Crypt-SSLeay perl-Date-Format perl-DBD-MySQL perl-IO-Socket-SSL perl-JSON-XS perl-Mail-IMAPClient perl-Net-DNS perl-LDAP perl-Template-Toolkit perl-Text-CSV_XS perl-XML-LibXML perl-XML-LibXSLT perl-XML-Parser perl-YAML-LibYAML -y

wget -qO- http://ftp.otrs.org/pub/otrs/otrs-5.0.14.tar.gz | tar xvz -C /opt/
mv /opt/otrs-5.0.14 /opt/otrs && cd /opt/otrs
cp Kernel/Config.pm.dist Kernel/Config.pm
useradd -d /opt/otrs/ -g nginx -s /sbin/nologin -c 'OTRS System User' otrs
su otrs -s /bin/bash -c "/opt/otrs/bin/otrs.CheckModules.pl"

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
