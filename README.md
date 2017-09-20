Docker developer DNS
====================

This DNS purpose is to allow for developers to run their development
environments without the need to update their `/etc/hosts` file. The additional
goal is to allow us to run a debug environment locally on the same domainname
as the 'live' environment. For now this DNS will be aimed at Linux users, over
time the goal is to provide a consistent platform for more operating systems.

This DNS will not try to create DNS entries for all running containers. You
must define what containers you want to be available in the DNS by using
`VIRTUAL_HOST` or `DOMAIN_NAME`.

Usage:

1. [Install developer-dns](#installation)
2. [Resolve developer-dns](#resolve-via-developer-dns)
3. [Register docker containers](#let-a-container-have-a-dns-entry)  

Installation
-----

To run this DNS server we need access to the docker socket so the DNS entries
can be added dynamically.

run:

~~~ sh
$ docker run --name developer-dns \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -p 127.0.0.1:53:53/udp \
    blackikeeagle/developer-dns
~~~

systemd service:

You can find the basic systemd service file in
[contrib/systemd](contrib/systemd/docker.developer-dns.service).

~~~
[Unit]
Description=developer-dns
After=network-online.target docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker rm developer-dns
ExecStart=/usr/bin/docker run --rm --name developer-dns \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -p 127.0.0.1:53:53/udp \
    -p 127.0.0.1:53:53 \
    blackikeeagle/developer-dns
ExecStartPost=-/usr/bin/docker pull blackikeeagle/developer-dns
ExecStop=/usr/bin/docker stop developer-dns

[Install]
WantedBy=multi-user.target
~~~

To add support for changing networks you can add your dhcp received resolv.conf
to the container. To achieve this you must install openresolv and make use of
the [resolvconf.conf](contrib/openresolv/resolvconf.conf) file. In some
distributions you get a default configuration that stores files in tmp space
which can lead to some issues. So it's better to use a target location within
`/etc` as used in the file in contrib.

Add resolv.conf to the container:

~~~
    -v /etc/dnsmasq-resolv.conf:/etc/resolv.conf:ro
~~~

Let a container have a DNS entry
--------------------------------

You can use both `VIRTUAL_HOST` and `DOMAIN_NAME` environment variables to get
an entry or comma seperated to add entries to the developer-dns.

single dns entry:

~~~ sh
$ docker run --env VIRTUAL_HOST=test.dev nginx
$ docker run --env DOMAIN_NAME=memcached.test.dev memcached
~~~

When dealing with webservers you might want to add more:

~~~ sh
$ docker run --env VIRTUAL_HOST=mysite.dev,nl.mysite.dev,fr.mysite.dev,de.mysite.dev nginx
$ docker run --env DOMAIN_NAME=mysite.dev,nl.mysite.dev,fr.mysite.dev,de.mysite.dev nginx
~~~

Personally I would recommend to use `VIRTUAL_HOST` for http related entries and
`DOMAIN_NAME` for others. This way you can share you configuration with for
example OSX users using [dinghy](https://github.com/codekitchen/dinghy).

Resolve via developer-dns
-------------------------

You need to make sure your host can resolve the docker dns entries.
This can be done in many ways, depending on the system you use.

#### resolvconf

On a system using resolvconf (eg. ubuntu) you can add this nameserver to the head.
~~~ sh
# echo "nameserver 127.0.0.1" > /etc/resolvconf/resolv.conf.d/head
~~~

After making changes to resolvconf, you need to trigger a reload:

~~~sh
sudo resolvconf -u
~~~

#### openresolv

On systems where openresolv is used you can use the file from
[contrib](contrib/openresolv/resolvconf.conf) and place it in
`/etc/resolvconf.conf`.

#### systemd resolved

On systems using systemd resolved (eg. Ubuntu 17.04) you need to change:

~~~ sh
# echo "DNS=127.0.0.1" >> /etc/systemd/resolved.conf
~~~

Reload resolved service:

~~~sh
$ sudo systemctl restart systemd-resolved.service
~~~

#### manual

You could also add it manually to your network configuration gui like NetworkManager.
