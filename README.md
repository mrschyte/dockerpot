# Description

Dockerpot is docker based honeypot. For a better summary visit
http://www.itinsight.hu/blog/posts/2015-05-04-creating-honeypots-using-docker.html

# Installation

## Install the necessary software

~~~ shell
$ sudo apt-get update
$ sudo apt-get install docker.io socat xinetd auditd

$ # for installing nsenter
$ docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter
~~~

## Install the honeypot scripts 

Copy `honeypot` to `/usr/bin/honeypot` and `honeypot.clean` to
`/usr/bin/honeypot.clean` and make them executable. You may have to
customize the ports in the iptables rules, the memory limit of the
container and the network quota if you want to run anything other than
an SSH honeypot on port 22.

## Configure crond, xinetd and auditd

### crond

Add the following line to `/etc/crontab`. This runs the cleanup script
to check for old containers every 5 minutes.

~~~ shell
*/5 * * * * /usr/honeypot/honeypot.clean
~~~

### xinetd

Create the following service file in `/etc/xinetd.d/honeypot` and add
the line `honeypot 22/tcp` to `/etc/services` to keep xinetd happy.

~~~ shell
# Container launcher for an SSH honeypot
service honeypot
{
        disable         = no
        instances       = UNLIMITED
        server          = /usr/bin/honeypot
        socket_type     = stream
        protocol        = tcp
        port            = 22
        user            = root
        wait            = no
        log_type        = SYSLOG authpriv info
        log_on_success  = HOST PID
        log_on_failure  = HOST
}
~~~

### auditd

Enable logging the execve systemcall in auditd by appending the following lines to `/etc/audit/audit.rules`.

~~~ shell
-a exit,always -F arch=b64 -S execve
-a exit,always -F arch=b32 -S execve
~~~

## Create a base image for the honeypot

Create and configure a base image for the honeypot. The container will
be run using the command /sbin/init so place your initialization
script there or configure an init system of your choice. Make sure to
commit the image as "honeypot:latest". You should also create an
account named `user` and give it a weak password like `123456` to let
brute-force attackers crack your host. The ip address of the
attacker's host is passed to the container in the environment variable
"REMOTE_HOST". For logging you might want to additionally configure an
rsyslog instance to forward logs to the host machine at 172.17.42.1.

