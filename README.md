# bad.example.com Round#2

## Servers

- `bad.example.com` authorized nameserver on `example.com`
  with IP `192.168.57.2/24`
- `worst.example.com` secondary server with IP
  `192.168.57.3/24`

## Network

- Private network `192.168.57.0/24`.
- Requires a Host-only network adapter in your host.

## Deploy

    vagrant up


## Changes to bad

### named.conf.options
We have to allow recursion by changing `recursion no;` to `recursion yes;`, this is because if not, recursive queries would be acepted.
We should also change `dnssec-validation yes;` to `dnssec-validation no;` because it is not configured.
I also added `allow-query { any; };` because i wanted to make sure that anyone from anywhere could be allowed to make queries with the DNS.
In order to apply the security measures that are required, we have to add `allow-recursion { localhost; 192.168.57.0/24; };`, so we only allow recursed queries with the dns server to the machines that we trust.

### named.conf.local
It had the same issues than the previous exercise 1, the typo in the file `bin` instead of `bind`, and the extra period at the end of the other file `"/var/lib/bind/example.com.dns."` instead of `"/var/lib/bind/example.com.dns"`
We only have to add `allow-transfer { 192.168.57.3; };` in order to restrict transfers to only that specific ip, which is the ip of the worst dns machine.  

### example.com.dns
It also has the same errors as before.
`$ORIGIN example.org.`should be `$ORIGIN example.com.`.
I must remove the period from here from `bad.  IN  A       192.168.57.2` to `bad  IN  A       192.168.57.2`.
This part of the file `@     IN  NS      bad.` should be fully qualified like this `@     IN  NS      bad.example.com.`
I also added `@     IN  NS      worst.example.com.` so it can recognize the worst dns machine, now, every NS has to have a A register, so the table can identify its address, so we have to add it like this `worst IN  A		  192.168.57.3`.
Because the practice tells us to add them, we have to add both an alias for the worst server and a mail server with priority 90.
We just have to add this `www   IN  CNAME   worst.example.com.` for the alias (canonical name), and this
`@     IN  MX  90  worst.example.com.` for the mail server.

### 192.168.57.dns
As well as the other ones, this also have the same issues than its predecesor.
`2   IN	PTR	bad.example.com` and `3   IN  PTR     worst.example.com`, should have a period at the end, it should look like
`2   IN	PTR	bad.example.com.` and `3   IN  PTR     worst.example.com.`
Note that the 192.168.57.3 ip is the worst server, we put worst.example.com instead of www.example.com, so it can relate to the name server that we have to specify above.
In order to do that we just add `@   IN  NS  worst.example.com.` to it.

## Changes to worst

### named.conf.local
We added to both the regular zone and the inverse zone the lines `type slave;` and `masters { 192.168.57.2; };`.
The first line is to specify that that machine has to act like the slave machine, the other line is to specify that the masters machine ip is 192.168.57.2.

### named.conf.options
Here i just mirrored the configuration that i had in the bad machine, is ussually good practice to mirror the configuration from machine to machine to be consistent with the security that was applied to the first one.

### vagrantfile
We added the files that we needed for the configuration.

## Lets see if it works
### bad machine
### bind9 status
vagrant@bad:~$ systemctl status bind9
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2025-01-05 21:14:59 UTC; 1min 45s ago
       Docs: man:named(8)
   Main PID: 2018 (named)
      Tasks: 10 (limit: 208)
     Memory: 10.1M
        CPU: 35ms
     CGroup: /system.slice/named.service
             └─2018 /usr/sbin/named -f -u bind
We can see that it is active and running.
### check files
vagrant@bad:~$ sudo named-checkconf
vagrant@bad:~$

vagrant@bad:~$ sudo named-checkzone example.com /var/lib/bind/example.com.dns
zone example.com/IN: loaded serial 1
OK

vagrant@bad:~$ sudo named-checkzone 57.168.192.in-addr.arpa /var/lib/bind/192.168.57.dns
zone 57.168.192.in-addr.arpa/IN: loaded serial 1
OK

### worst machine
### bind9 status
vagrant@worst:~$ systemctl status bind9
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2025-01-05 21:15:50 UTC; 7min ago
       Docs: man:named(8)
   Main PID: 2022 (named)
      Tasks: 10 (limit: 208)
     Memory: 10.0M
        CPU: 41ms
     CGroup: /system.slice/named.service
             └─2022 /usr/sbin/named -f -u bind
### check files
vagrant@worst:~$ sudo named-checkconf
vagrant@worst:~$
