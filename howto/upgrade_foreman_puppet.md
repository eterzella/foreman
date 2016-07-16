Processo de atualização do foreman 1.11 -> 1.12

1) Executar backup

2) Instalação do repositorio do foreman 1.12


yum remove foreman-release
yum install https://yum.theforeman.org/releases/1.12/el7/x86_64/foreman-release.rpm
yum clean all
yum install foreman-release-scl
yum upgrade tfm\* ruby\* foreman\*


3) Atualizando o puppet server

rpm -ivh https://yum.puppetlabs.com/puppetlabs-release-pc1-el-7.noarch.rpm
yum install puppetserver
cp -rpf /var/lib/puppet/ssl /etc/puppetlabs/puppet
cp -rpf /var/lib/puppet/ssl /etc/puppetlabs/puppet/ssl
mv /etc/httpd/conf.d/25-puppet.conf /root/25-puppet.conf

Remove 8140 from the Apache ports in /etc/httpd/conf/ports.conf or /etc/apache2/ports.conf

Update SSL paths in /etc/httpd/conf.d/05-foreman-ssl.conf or /etc/apache2/sites-available/05-foreman-ssl.conf, changing /var/lib/puppet/ssl to /etc/puppetlabs/puppet/ssl 

systemctl restart httpd

cd /etc/puppet
cp autosign.conf puppet.conf node.rb foreman.yaml /etc/puppetlabs/puppet/

4) Alterar os parametros no arquivo do puppet.conf

in the 'main' section:
vardir = /opt/puppetlabs/puppet/cache
logdir = /var/log/puppetlabs/puppet
rundir = /var/run/puppetlabs
ssldir = /etc/puppetlabs/puppet/ssl
environmentpath = /etc/puppetlabs/code
basemodulepath = /etc/puppetlabs/code/environments/common:/etc/puppetlabs/code/modules:/opt/puppetlabs/puppet/modules

in the 'master' section:
autosign = /etc/puppetlabs/puppet/autosign.conf { mode = 0644 }
external_nodes = /etc/puppetlabs/puppet/node.rb
ssldir = /etc/puppetlabs/puppet/ssl

5) Alterando arquivo puppetserver.conf

edit /etc/puppetlabs/puppetserver/conf.d/puppetserver.conf
change master-var-dir to /opt/puppetlabs/puppet/cache
uncomment/set use-legacy-auth-conf: false


6) Editando arquivo auth.conf

edit /etc/puppetlabs/puppetserver/conf.d/auth.conf

If using a smart proxy to import classes, edit /etc/puppetlabs/puppetserver/conf.d/auth.conf, search for /puppet/v3/environments and add a new section below it:

        {
            match-request: {
                path: "/puppet/v3/resource_type" 
                type: path
                method: [get, post]
            }
            allow: "*" 
            sort-order: 500
            name: "puppetlabs resource type" 
        },

7) Configurando o serviço do puppetserver, para iniciar automaticamente.

 /opt/puppetlabs/bin/puppet resource service puppetserver ensure=running enable=true


8) ENC Files

	cp /etc/puppet/foreman.yaml /etc/puppetlabs/puppet/foreman.yaml and change:
	replace /var/lib/puppet/ssl with /etc/puppetlabs/puppet/ssl
	:puppetdir: /opt/puppetlabs/puppet/cache

mv /etc/puppet/node.rb /etc/puppetlabs/puppet/

9) Foreman Settings

edit /etc/foreman/settings.yaml and change :puppetssldir: /etc/puppetlabs/puppet/ssl
change websockets_* settings to use /etc/puppetlabs/puppet/ssl and also ssl_* if specified
restart Foreman by running touch ~foreman/tmp/restart.txt
check in Administer > Settings > Auth in the Foreman UI that SSL certificate, private key and CA file all use /etc/puppetlabs/puppet/ssl, else change them


10) Smart proxy settings

edit /etc/foreman-proxy/settings.d/puppet.yml and set :puppet_version to the version of Puppet currently installed, e.g. 4.5.0
look up the version of puppet-agent (rpm -q puppet-agent or dpkg -l puppet-agent) and check Release contents
edit /etc/foreman-proxy/settings.d/puppet_proxy_puppet_api.yml and change /var/lib/puppet/ssl to /etc/puppetlabs/puppet/ssl
edit /etc/foreman-proxy/settings.d/puppetca.yml and change:
:ssldir: /etc/puppetlabs/puppet/ssl
:puppetdir: /etc/puppetlabs/puppet
edit /etc/foreman-proxy/settings.yml and change /var/lib/puppet/ssl to /etc/puppetlabs/puppet/ssl
restart foreman-proxy



11) 

 wget https://raw.githubusercontent.com/theforeman/puppet-foreman/master/files/foreman-report_v2.rb

https://raw.githubusercontent.com/theforeman/puppet-foreman/master/files/foreman-report_v2.rb


Site referência:

http://projects.theforeman.org/projects/foreman/wiki/Upgrading_from_Puppet_3_to_4