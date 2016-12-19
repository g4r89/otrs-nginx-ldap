# otrs
centos7+otrs+nginx+perl-fcgi
# install nginx and mysql
```bash
sed -i.bak '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
setenforce 0

yum update -y

yum install epel-release nginx mariadb-server perl perl-core -y

cat <<EOF> /etc/my.cnf.d/otrs.cnf
[mysqld]
max_allowed_packet   = 20M
query_cache_size     = 32M
innodb_log_file_size = 256M
EOF

systemctl enable --now nginx
systemctl enable --now mariadb
/usr/bin/mysql_secure_installation

mysql -u root -p
create database `otrs-db` character set utf8;
create user 'USER'@'localhost' identified by 'PASS';
GRANT ALL PRIVILEGES ON `otrs-db`.* to `USER`@`localhost`;
FLUSH PRIVILEGES;
exit;
```
# config nginx and fcgiwrap
```bash
vi /etc/nginx/conf.d/default.conf
cat <<EOF> /etc/nginx/conf.d/otrs.conf
server {
listen 80;
server_name otrs.sk2.su otrs;
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
include fastcgi_params;
}
}
EOF

cat <<EOF> /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
}
EOF

cat <<EOF> /usr/local/bin/fastcgi-wrapper.pl
#!/usr/bin/perl

use FCGI;
#perl -MCPAN -e 'install FCGI'
use Socket;
use POSIX qw(setsid);
#use Fcntl;

require 'syscall.ph';

&daemonize;

#this keeps the program alive or something after exec'ing perl scripts
END() { } BEGIN() { }
*CORE::GLOBAL::exit = sub { die "fakeexit\nrc=".shift()."\n"; }; 
eval q{exit}; 
if ($@) { 
	exit unless $@ =~ /^fakeexit/; 
};

&main;

sub daemonize() {
    chdir '/'                 or die "Can't chdir to /: $!";
    defined(my $pid = fork)   or die "Can't fork: $!";
    exit if $pid;
    setsid                    or die "Can't start a new session: $!";
    umask 0;
}

sub main {
        #$socket = FCGI::OpenSocket( "127.0.0.1:8999", 10 ); #use IP sockets
        $socket = FCGI::OpenSocket( "/var/run/nginx/perl_cgi-dispatch.sock", 10 ); #use UNIX sockets - user running this script must have w access to the 'nginx' folder!!
        $request = FCGI::Request( \*STDIN, \*STDOUT, \*STDERR, \%req_params, $socket );
        if ($request) { request_loop()};
            FCGI::CloseSocket( $socket );
}

sub request_loop {
        while( $request->Accept() >= 0 ) {
            
           #processing any STDIN input from WebServer (for CGI-POST actions)
           $stdin_passthrough ='';
           $req_len = 0 + $req_params{'CONTENT_LENGTH'};
           if (($req_params{'REQUEST_METHOD'} eq 'POST') && ($req_len != 0) ){ 
                my $bytes_read = 0;
                while ($bytes_read < $req_len) {
                        my $data = '';
                        my $bytes = read(STDIN, $data, ($req_len - $bytes_read));
                        last if ($bytes == 0 || !defined($bytes));
                        $stdin_passthrough .= $data;
                        $bytes_read += $bytes;
                }
            }

            #running the cgi app
            if ( (-x $req_params{SCRIPT_FILENAME}) &&  #can I execute this?
                 (-s $req_params{SCRIPT_FILENAME}) &&  #Is this file empty?
                 (-r $req_params{SCRIPT_FILENAME})     #can I read this file?
            ){
		pipe(CHILD_RD, PARENT_WR);
		my $pid = open(KID_TO_READ, "-|");
		unless(defined($pid)) {
			print("Content-type: text/plain\r\n\r\n");
                        print "Error: CGI app returned no output - Executing $req_params{SCRIPT_FILENAME} failed !\n";
			next;
		}
		if ($pid > 0) {
			close(CHILD_RD);
			print PARENT_WR $stdin_passthrough;
			close(PARENT_WR);

			while(my $s = <KID_TO_READ>) { print $s; }
			close KID_TO_READ;
			waitpid($pid, 0);
		} else {
	                foreach $key ( keys %req_params){
        	           $ENV{$key} = $req_params{$key};
                	}
        	        # cd to the script's local directory
	                if ($req_params{SCRIPT_FILENAME} =~ /^(.*)\/[^\/]+$/) {
                        	chdir $1;
                	}

			close(PARENT_WR);
			close(STDIN);
			#fcntl(CHILD_RD, F_DUPFD, 0);
			syscall(&SYS_dup2, fileno(CHILD_RD), 0);
			#open(STDIN, "<&CHILD_RD");
			exec($req_params{SCRIPT_FILENAME});
			die("exec failed");
		}
            } 
            else {
                print("Content-type: text/plain\r\n\r\n");
                print "Error: No such CGI app - $req_params{SCRIPT_FILENAME} may not exist or is not executable by this process.\n";
            }

        }
}
EOF

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
