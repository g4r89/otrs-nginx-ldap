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
yum install epel-release wget nginx -y
mv /etc/nginx/nginx.conf{,.bak}
cp nginx.conf /etc/nginx/nginx.conf
mv /etc/nginx/conf.d/default.conf{,.bak}
cp default.conf /etc/nginx/conf.d/default.conf
cp fastcgi-wrapper /usr/local/bin/fastcgi-wrapper.pl
cat <<EOF> /lib/systemd/system/perl-fcgi.service
[Unit]
Description=otrs-index.pl
[Service]
ExecStart=/usr/local/bin/fastcgi-wrapper.pl
Type=forking
User=otrs
Group=nginx
Restart=always
[Install]
WantedBy=multi-user.target
EOF
