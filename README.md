# Freeipa-Proxy
FreeIPA Proxy/Load-Balancer

It is a common configuration to have slapd behind a load balancer to help provide high availability. In these configurations it is often hard to make GSSAPI work correctly.

This is because GSSAPI/KRB5 makes the assumption that when you connect to a DNS name foo.example.com, the Service Ticket held by the service matches /foo.example.com@REALM. Of course, behind a load balancer, this is not the case, because the hosts behind the load balancer will be called arbitrary names like ldap1.example.com, ldap2.example.com.

We will be using [Envoy Proxy](https://www.envoyproxy.io/) to create a proxy/load-balancer for FreeIPA, the proxy will handle http/s, ldap/s and krb traffic.

This guide was tested with:
  1. FreeIPA 4.5.0 (installed on CentOS 7.4)
  2. Envoy 95ba531eef4623eb1ae2f26e483e883f874dbd1b/Clean/RELEASE (installed on Ubuntu 16.04)

This guide assumes you have multiple IPA servers in a multi master enviorment.
FreeIPA server installation if beyond the scope of this guide, please refer to the [docs](https://www.digitalocean.com/community/tutorials/how-to-set-up-centralized-linux-authentication-with-freeipa-on-centos-7) for more information.

# Envoy Proxy
Envoy is an L7 proxy and communication bus designed for large modern service oriented architectures. The project was born out of the belief that:

> The network should be transparent to applications. When network and application problems do occur it should be easy to determine the source of the problem. 

Envoy is built inside a docker environment, so we'll need to install docker:
```
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
# sudo apt-get update
# apt-get install -y docker-ce
```
Build Envoy and link the executable:
```
# cd /opt
# git clone https://github.com/envoyproxy/envoy.git
# cd envoy
# ./ci/run_envoy_docker.sh './ci/do_ci.sh bazel.release.server_only'
# cp -R /tmp/envoy-docker-build /opt
# ln -s /opt/envoy-docker-build/envoy/source/exe/envoy /usr/bin/envoy
```
Create Envoy configuration file, please refer to the [configuration example](https://github.com/alex-zel/freeipa-proxy/blob/master/etc/envoy/envoy.yaml):
```
# mkdir /etc/envoy
# touch /etc/envoy/envoy.yaml
```
Create systemd service file:
```
# vim /usr/lib/systemd/system/envoy.service
[Unit]
Description=Envoy Proxy
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/envoy --config-path /etc/envoy/envoy.yaml --log-path /var/log/envoy/envoy.log

[Install]
WantedBy=multi-user.target

# systemctl daemon-reload
# systemctl status envoy
# systemctl enable envoy
```

# FreeIPA

FreeIPA is an integrated security information management solution combining Linux (Fedora), 389 Directory Server, MIT Kerberos, NTP, DNS, Dogtag (Certificate System). It consists of a web interface and command-line administration tools.

In order to make FreeIPA work properly behind a load-balancer we'll have to make some changes to the default apache and slapd configuration:
1. You must setup A/AAAA records with PTR to the load balancer IP or Virtual IP. You cannot use CNAMEs in this configuration.
2.  All IPA servers should use the same GssapiSessionKey, select a single IPA server to copy the Key from (for example ldap01.example.com), do this for each IPA server (make sure you copy same file to all servers):

    ```
    # scp /etc/httpd/alias/ipasession.key ldap02.example.com:/tmp
    # ssh ldap02.example.com
    # mv /tmp/ipasession.key /etc/httpd/alias/ipasession.key
    ```
3.  We'll also need to change some rewrite rules:

    ```
    # vim /etc/httpd/conf.d/ipa-rewrite.conf
    
    comment out the following lines:
    RewriteRule ^/$ https://ldap01.example.com/ipa/ui [L,NC,R=301]
    RewriteCond %{HTTP_HOST}    !^ldap01.example.com$ [NC]
    RewriteRule ^/ipa/(.*)      http://ldap01.example.com/ipa/$1 [L,R=301]

    add the following line:
    (the referer should matches the fqdn of the IPA serevr you are working on) so for ldap01.example.com we use:
    RequestHeader edit Referer ^https://ldap\.example\.com/ https://ldap01.example.com/
    while on ldap02.example.com we use:
    RequestHeader edit Referer ^https://ldap\.example\.com/ https://ldap02.example.com/


    the end result will look like this:

    RewriteEngine on

    # By default forward all requests to /ipa. If you don't want IPA
    # to be the default on your web server comment this line out.
    #RewriteRule ^/$ https://ldap01.example.com/ipa/ui [L,NC,R=301]
    
    # Redirect to the fully-qualified hostname. Not redirecting to secure
    # port so configuration files can be retrieved without requiring SSL.
    #RewriteCond %{HTTP_HOST}    !^ldap01.example.com$ [NC]
    #RewriteRule ^/ipa/(.*)      http://ldap01.example.com/ipa/$1 [L,R=301]

    # Redirect to the secure port if not displaying an error or retrieving
    # configuration.
    RewriteCond %{SERVER_PORT}  !^443$
    RewriteCond %{REQUEST_URI}  !^/ipa/(errors|config|crl)
    RewriteCond %{REQUEST_URI}  !^/ipa/[^\?]+(\.js|\.css|\.png|\.gif|\.ico|\.woff|\.svg|\.ttf|\.eot)$
    RewriteRule ^/ipa/(.*)      https://ldap01.example.com/ipa/$1 [L,R=301,NC]

    # Rewrite for plugin index, make it like it's a static file
    RewriteRule ^/ipa/ui/js/freeipa/plugins.js$    /ipa/wsgi/plugins.py [PT]

    # Rewrite rule for load balancer
    RequestHeader edit Referer ^https://ldap\.example\.com/ https://ldap01.example.com/
    ```
4. We'll need to extract and combine the keytab for the load-balancer and each IPA server.

    If your loadbalancer is not part of the domain, or an appliance you will need to add a fake host for this:
    ```
    # ipa host-add ldap.example.com --random --force
    ```
    Otherwise [install ipa-client](https://www.digitalocean.com/community/tutorials/how-to-configure-a-freeipa-client-on-ubuntu-16-04) on the server

    Create and delegate the principal for the load-balancer:
    ```
    # ipa service-add --force ldap/ldap.example.com
    # ipa service-add-host ldap/ldap.example.com --hosts=ldap01.example.com
    ```
    Extract and combine the keytabs (Important is the –retrieve flag to prevent the keytab contents changing):
    ```
    kinit <account with admins privilige>
    ipa-getkeytab -s ldap01.example.com -p ldap/ldap.example.com -k /etc/dirsrv/slapd-localhost/ldap.keytab --retrieve
    ipa-getkeytab -s ldap01.example.com -p ldap/ldap01.example.com -k /etc/dirsrv/slapd-localhost/ldap.keytab --retrieve
    ```
    Reconfigure slapd to be aware of the keytab:
    ```
    # vim /etc/sysconfig/dirsrv-<instance>
    add the following line
    KRB5_KTNAME=/etc/dirsrv/slapd-localhost/ldap.keytab
    ```
    Finally restart FreeIPA and test that you have working GSSAPI:
    ```
    # ipactl restart
    # ldapsearch -H ldap://ldap.example.com -Y GSSAPI
    # ldapsearch -H ldap://ldap01.example.com -Y GSSAPI
    ````
