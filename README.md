# otrs
centos7+otrs+nginx+fgci wrapper
# install
yum update -y
cat <<EOF > /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/\$basearch/
gpgcheck=0
enabled=1
EOF
yum install epel-release wget nginx -y
