# freeipa-proxy
FreeIPA Proxy/Load-Balancer

It is a common configuration to have slapd behind a load balancer to help provide high availability. In these configurations it is often hard to make GSSAPI work correctly.

This is because GSSAPI/KRB5 makes the assumption that when you connect to a DNS name foo.example.com, the Service Ticket held by the service matches /foo.example.com@REALM. Of course, behind a load balancer, this is not the case, because the hosts behind the load balancer will be called arbitrary names like ldap1.example.com, ldap2.example.com.

We will be using [Envoy Proxy](https://www.envoyproxy.io/) to create a proxy/load-balancer for FreeIPA, the proxy will handle http/s, ldap/s and krb traffic.

This guide was tested with:
  1. FreeIPA 4.5.0 (installed on CentOS 7.4)
  2. Envoy 95ba531eef4623eb1ae2f26e483e883f874dbd1b/Clean/RELEASE (installed on Ubuntu 16.04)

This guide assumes you have multiple IPA servers in a multi master enviorment.
FreeIPA server installation if beyond the scope of this guide, please refer to the [docs](https://www.digitalocean.com/community/tutorials/how-to-set-up-centralized-linux-authentication-with-freeipa-on-centos-7) for more information.
