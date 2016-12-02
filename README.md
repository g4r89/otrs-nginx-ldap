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

cat <<EOF> /lib/systemd/system/perl-fcgi.service
[Unit]
Description=perl fastcgi service
[Install]
WantedBy=multi-user.target
[Service]
User=otrs
Group=nginx
Type=simple
Restart=always
PermissionsStartOnly=true
ExecStartPre=/usr/bin/mkdir -p /var/run/perl-fcgi
ExecStartPre=/usr/bin/chown otrs.nginx /var/run/perl-fcgi
ExecStart=/usr/local/bin/fastcgi-wrapper.pl
ExecStop=/usr/bin/rm -rf /var/run/perl-fcgi
EOF

sed -i.bak '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
setenforce 0

cat <<EOF> /etc/my.cnf.d/otrs.cnf
[mysqld]
max_allowed_packet   = 20M
query_cache_size     = 32M
innodb_log_file_size = 256M
EOF

systemctl start mariadb
/usr/bin/mysql_secure_installation

/bin/rm /var/lib/mysql/ib_logfile*

yum install apr procmail

