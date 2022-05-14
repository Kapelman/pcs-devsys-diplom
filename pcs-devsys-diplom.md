## Курсовая работа по итогам модуля "DevOps и системное администрирование" ##



1. Создайте виртуальную машину Linux. 

- Создаем виртуальную машину с параметрами CPU - 6 ядра, RAM - 6 Гб, HDD - 30 ГБ
![Созданный виртуальный хост](Virtual-host.jpg)

2. Установите ufw и разрешите к этой машине сессии на порты 22 и 443, при этом трафик на интерфейсе localhost (lo) 
должен ходить свободно на все порты.

- проверим, что статус неактивный.
```
vagrant@vagrant:~$ sudo ufw status
Status: inactive
```

- настроим отдельный сетевой интерфейс для будущего web сервера
```
vagrant@vagrant:~$ sudo ip link set eth1 up
vagrant@vagrant:~$ ip -c -br address
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             10.0.2.15/24 fe80::a00:27ff:feb1:285d/64
eth1             UP             192.168.34.12/24 fe80::a00:27ff:feb2:77e1/64
```
- присвоим ip адрес новому интерфейсу
```
vagrant@vagrant:~$ sudo ip addr add 192.168.34.12/255.255.255.0 broadcast 192.168.34.255 dev eth1
```
- итого получается 3 сетевых интерфейса. 
```
vagrant@vagrant:~$ ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:b1:28:5d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 84697sec preferred_lft 84697sec
    inet6 fe80::a00:27ff:feb1:285d/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:b2:77:e1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.34.12/24 brd 192.168.34.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feb2:77e1/64 scope link
       valid_lft forever preferred_lft forever

```

- включаем логирование для UFW
```
vagrant@vagrant:~$ sudo ufw logging on
Logging enabled
```
- Задаем правила для сетевых интерфейсов. Сначала разрешающие на eth1 и eth0, lo (для них разрешено все, т.к. это localhost)
```
vagrant@vagrant:~$ sudo ufw allow  in on eth1 proto tcp from any  to any port 22,443 comment 'eth1 in 22,443'
Rules updated
Rules updated (v6)

vagrant@vagrant:~$ sudo ufw allow out on eth1 proto tcp from any port 22,443 to any comment 'eth1 out 22,443'
Rules updated
Rules updated (v6)

vagrant@vagrant:~$ sudo ufw allow out on lo
Rules updated
Rules updated (v6)
vagrant@vagrant:~$ sudo ufw allow in on lo
Rules updated
Rules updated (v6)

vagrant@vagrant:~$ sudo ufw allow in on eth0
Rules updated
Rules updated (v6)
vagrant@vagrant:~$ sudo ufw allow out on eth0
Rules updated
Rules updated (v6)
```
- теперь запрещаем все прочие соединения на eth1
```
vagrant@vagrant:~$ sudo ufw deny out on eth1
vagrant@vagrant:~$ sudo ufw deny in on eth1
```
- активируем firewall
```
vagrant@vagrant:~$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

vagrant@vagrant:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22,443/tcp on eth1         ALLOW       Anywhere                   # eth1 in 22,443
Anywhere on lo             ALLOW       Anywhere
Anywhere on eth0           ALLOW       Anywhere
Anywhere on eth1           DENY        Anywhere
22,443/tcp (v6) on eth1    ALLOW       Anywhere (v6)              # eth1 in 22,443
Anywhere (v6) on lo        ALLOW       Anywhere (v6)
Anywhere (v6) on eth0      ALLOW       Anywhere (v6)
Anywhere (v6) on eth1      DENY        Anywhere (v6)

Anywhere                   ALLOW OUT   22,443/tcp on eth1         # eth1 out 22,443
Anywhere                   ALLOW OUT   Anywhere on lo
Anywhere                   ALLOW OUT   Anywhere on eth0
Anywhere                   DENY OUT    Anywhere on eth1
Anywhere (v6)              ALLOW OUT   22,443/tcp (v6) on eth1    # eth1 out 22,443
Anywhere (v6)              ALLOW OUT   Anywhere (v6) on lo
Anywhere (v6)              ALLOW OUT   Anywhere (v6) on eth0
Anywhere (v6)              DENY OUT    Anywhere (v6) on eth1
```
- Проверяем доступ с windows хоста для интерфейса eth1, доступ работает.
```
PS C:\Users\kapli> ssh vagrant@192.168.34.12 -p 22
vagrant@192.168.34.12's password:
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-109-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed 04 May 2022 10:18:08 PM UTC

  System load:  0.0                Processes:             140
  Usage of /:   13.1% of 30.88GB   Users logged in:       1
  Memory usage: 5%                 IPv4 address for eth0: 10.0.2.15
  Swap usage:   0%                 IPv4 address for eth1: 192.168.34.12


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Wed May  4 21:52:42 2022 from 192.168.34.1
```

3. Установите hashicorp vault (инструкция по ссылке).

- устанавливаем Vault по инструкции
```
vagrant@vagrant:~$ curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
OK
vagrant@vagrant:~$ sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
Get:1 https://apt.releases.hashicorp.com focal InRelease [16.3 kB]
Get:2 https://apt.releases.hashicorp.com focal/main amd64 Packages [52.8 kB]
Hit:3 http://us.archive.ubuntu.com/ubuntu focal InRelease
Get:4 http://us.archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
Get:5 http://us.archive.ubuntu.com/ubuntu focal-backports InRelease [108 kB]
Get:6 http://us.archive.ubuntu.com/ubuntu focal-security InRelease [114 kB]
Get:7 http://us.archive.ubuntu.com/ubuntu focal-updates/main amd64 Packages [1,750 kB]
Get:8 http://us.archive.ubuntu.com/ubuntu focal-updates/universe amd64 Packages [921 kB]
Get:9 http://us.archive.ubuntu.com/ubuntu focal-security/main amd64 Packages [1,422 kB]
Get:10 http://us.archive.ubuntu.com/ubuntu focal-security/universe amd64 Packages [700 kB]
Fetched 5,198 kB in 12s (417 kB/s)
Reading package lists... Done
vagrant@vagrant:~$ sudo apt-get update && sudo apt-get install vault
Hit:1 https://apt.releases.hashicorp.com focal InRelease
Hit:2 http://us.archive.ubuntu.com/ubuntu focal InRelease
Hit:3 http://us.archive.ubuntu.com/ubuntu focal-updates InRelease
Hit:4 http://us.archive.ubuntu.com/ubuntu focal-backports InRelease
Hit:5 http://us.archive.ubuntu.com/ubuntu focal-security InRelease
Reading package lists... Done
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following package was automatically installed and is no longer required:
  libfwupdplugin1
Use 'sudo apt autoremove' to remove it.
The following NEW packages will be installed:
  vault
0 upgraded, 1 newly installed, 0 to remove and 3 not upgraded.
Need to get 70.5 MB of archives.
After this operation, 191 MB of additional disk space will be used.
Get:1 https://apt.releases.hashicorp.com focal/main amd64 vault amd64 1.10.2-1 [70.5 MB]
Fetched 70.5 MB in 20s (3,513 kB/s)
Selecting previously unselected package vault.
(Reading database ... 47672 files and directories currently installed.)
Preparing to unpack .../vault_1.10.2-1_amd64.deb ...
Unpacking vault (1.10.2-1) ...
Setting up vault (1.10.2-1) ...
Generating Vault TLS key and self-signed certificate...
Generating a RSA private key
................................................++++
..............................................++++
writing new private key to 'tls.key'
-----
Vault TLS key and self-signed certificate have been generated in '/opt/vault/tls'.
```

- Проверим работу Vault, работает.

```
vagrant@vagrant:~$ vault
Usage: vault <command> [args]

Common commands:
    read        Read data and retrieves secrets
    write       Write data, configuration, and secrets
    delete      Delete secrets and configuration
    list        List data or secrets
    login       Authenticate locally
    agent       Start a Vault agent
    server      Start a Vault server
    status      Print seal and HA status
    unwrap      Unwrap a wrapped secret

Other commands:
    audit                Interact with audit devices
    auth                 Interact with auth methods
    debug                Runs the debug command
    kv                   Interact with Vault's Key-Value storage
    lease                Interact with leases
    monitor              Stream log messages from a Vault server
    namespace            Interact with namespaces
    operator             Perform operator-specific tasks
    path-help            Retrieve API help for paths
    plugin               Interact with Vault plugins and catalog
    policy               Interact with policies
    print                Prints runtime configurations
    secrets              Interact with secrets engines
    ssh                  Initiate an SSH session
    token                Interact with tokens
    version-history      Prints the version history of the target Vault server
```


4. Cоздайте центр сертификации по инструкции (ссылка) и выпустите сертификат для использования его в настройке веб-сервера nginx (срок жизни сертификата - месяц).

- установим приложение для парсинга json
```
vagrant@vagrant:~$ sudo apt-get install jq
```
- запустим сервер vault в dev режиме с токеном root 

```
vagrant@vagrant:~$ vault server -dev -dev-root-token-id=root

==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.17.9
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: true, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.10.2
             Version Sha: 94325865b12662cb72efa3003d6aaa4f5ae57f3a

==> Vault server started! Log data will stream in below:

2022-05-05T19:52:38.086Z [INFO]  proxy environment: http_proxy="" https_proxy="" no_proxy=""
2022-05-05T19:52:38.086Z [WARN]  no `api_addr` value specified in config or in VAULT_API_ADDR; falling back to detection if possible, but this value should be manually set
2022-05-05T19:52:38.087Z [INFO]  core: Initializing versionTimestamps for core
2022-05-05T19:52:38.088Z [INFO]  core: security barrier not initialized
2022-05-05T19:52:38.088Z [INFO]  core: security barrier initialized: stored=1 shares=1 threshold=1
2022-05-05T19:52:38.089Z [INFO]  core: post-unseal setup starting
2022-05-05T19:52:38.091Z [INFO]  core: loaded wrapping token key
2022-05-05T19:52:38.091Z [INFO]  core: Recorded vault version: vault version=1.10.2 upgrade time="2022-05-05 19:52:38.091808728 +0000 UTC"
2022-05-05T19:52:38.091Z [INFO]  core: successfully setup plugin catalog: plugin-directory=""
2022-05-05T19:52:38.091Z [INFO]  core: no mounts; adding default mount table
2022-05-05T19:52:38.098Z [INFO]  core: successfully mounted backend: type=cubbyhole path=cubbyhole/
2022-05-05T19:52:38.099Z [INFO]  core: successfully mounted backend: type=system path=sys/
2022-05-05T19:52:38.099Z [INFO]  core: successfully mounted backend: type=identity path=identity/
2022-05-05T19:52:38.100Z [INFO]  core: successfully enabled credential backend: type=token path=token/
2022-05-05T19:52:38.100Z [INFO]  rollback: starting rollback manager
2022-05-05T19:52:38.100Z [INFO]  core: restoring leases
2022-05-05T19:52:38.117Z [INFO]  expiration: lease restore complete
2022-05-05T19:52:38.118Z [INFO]  identity: entities restored
2022-05-05T19:52:38.118Z [INFO]  identity: groups restored
2022-05-05T19:52:38.483Z [INFO]  core: post-unseal setup complete
2022-05-05T19:52:38.484Z [INFO]  core: root token generated
2022-05-05T19:52:38.484Z [INFO]  core: pre-seal teardown starting
2022-05-05T19:52:38.484Z [INFO]  rollback: stopping rollback manager
2022-05-05T19:52:38.484Z [INFO]  core: pre-seal teardown complete
2022-05-05T19:52:38.484Z [INFO]  core.cluster-listener.tcp: starting listener: listener_address=127.0.0.1:8201
2022-05-05T19:52:38.484Z [INFO]  core.cluster-listener: serving cluster requests: cluster_listen_address=127.0.0.1:8201
2022-05-05T19:52:38.484Z [INFO]  core: post-unseal setup starting
2022-05-05T19:52:38.484Z [INFO]  core: loaded wrapping token key
2022-05-05T19:52:38.484Z [INFO]  core: successfully setup plugin catalog: plugin-directory=""
2022-05-05T19:52:38.484Z [INFO]  core: successfully mounted backend: type=system path=sys/
2022-05-05T19:52:38.485Z [INFO]  core: successfully mounted backend: type=identity path=identity/
2022-05-05T19:52:38.485Z [INFO]  core: successfully mounted backend: type=cubbyhole path=cubbyhole/
2022-05-05T19:52:38.485Z [INFO]  core: successfully enabled credential backend: type=token path=token/
2022-05-05T19:52:38.485Z [INFO]  core: restoring leases
2022-05-05T19:52:38.486Z [INFO]  identity: entities restored
2022-05-05T19:52:38.486Z [INFO]  identity: groups restored
2022-05-05T19:52:38.486Z [INFO]  core: post-unseal setup complete
2022-05-05T19:52:38.486Z [INFO]  core: vault is unsealed
2022-05-05T19:52:38.487Z [INFO]  expiration: revoked lease: lease_id=auth/token/root/hb43773c000f5a7fa91cf7d600cf5bc9baee3dd22fc2bd1bfef7577ec084a5fd9
2022-05-05T19:52:38.487Z [INFO]  rollback: starting rollback manager
2022-05-05T19:52:38.487Z [INFO]  expiration: lease restore complete
2022-05-05T19:52:38.635Z [INFO]  core: successful mount: namespace="" path=secret/ type=kv
2022-05-05T19:52:38.642Z [INFO]  secrets.kv.kv_0c94ec26: collecting keys to upgrade
2022-05-05T19:52:38.642Z [INFO]  secrets.kv.kv_0c94ec26: done collecting keys: num_keys=1
2022-05-05T19:52:38.642Z [INFO]  secrets.kv.kv_0c94ec26: upgrading keys finished
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: lClHy4BhdMEjsvS0f/PP9YsSlMEWrS728O63oPkEw9Q=
Root Token: root

Development mode should NOT be used in production installations!
```

- Пропишем переменные, необходимые для работы Vault
```
vagrant@vagrant:~$ export VAULT_ADDR='http://127.0.0.1:8200'
vagrant@vagrant:~$ export VAULT_TOKEN='root'
vagrant@vagrant:~$ vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.10.2
Storage Type    inmem
Cluster Name    vault-cluster-5cac7b9d
Cluster ID      ec684c39-fdb2-ce5f-f412-5fb86f55ca69
HA Enabled      false
```


- Step 1: Сделаем root CA

```
vagrant@vagrant:~$ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/

vagrant@vagrant:~$ vault write -field=certificate pki/root/generate/internal common_name="example.com" ttl=87600h > CA_cert.crt
vagrant@vagrant:~$ ls -al
-rw-rw-r-- 1 vagrant vagrant 1171 May  5 12:38 CA_cert.crt
```
- Создадим запрос на сертификат и промежуточный сертификат для example.com

```
vagrant@vagrant:~$ vault write pki/config/urls issuing_certificates="$VAULT_ADDR/v1/pki/ca" crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
Success! Data written to: pki/config/urls

vagrant@vagrant:~$ vault secrets enable -path=pki_int pki
Success! Enabled the pki secrets engine at: pki_int/

vagrant@vagrant:~$ vault secrets tune -max-lease-ttl=43800h pki_int
Success! Tuned the secrets engine at: pki_int/

vagrant@vagrant:~$ vault write -format=json pki_int/intermediate/generate/internal common_name="example.com Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr

vagrant@vagrant:~$ cat pki_intermediate.csr

-----BEGIN CERTIFICATE REQUEST-----
MIICcjCCAVoCAQAwLTErMCkGA1UEAxMiZXhhbXBsZS5jb20gSW50ZXJtZWRpYXRl
IEF1dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAMcZ5DrT
85fDtUC0SKubBC9fITUlCL7QKOPBKZvu3mkDHojbzXTazkiGyEP7rQnvW7Uos3YJ
DUT2M2VccMYaq+d9lF6Tb6KQKzs0PHeady68uzy7Aoi8/yJzN6Y5AvYqxeLZqNuF
9d3zxbInSc0Gp4BDSZG4zbY9Udgy/IzmdjJP4hTttOukq1+0mg4bC82934Mjwnmv
Z/VgeXFaxYGtiJqx3UBVnlR8pD2S3Asi+6hBxK4S7Gp2/4FxR5/fIKbhsy+Zj8Ko
uYznrHZmjMeENWB3pR90tKh4XryoKntowf3eeL/FYetVGjMcb60PNXJxIlMGHSca
FsbSwVMxe1ue0NkCAwEAAaAAMA0GCSqGSIb3DQEBCwUAA4IBAQC5MvO8wlHgckCq
oXlEQRm7Ze/Z7OHoxPZYElYhgskz2CiNI/KWD6rW72Mg6Xj82rBzoLMozXcOKPwV
byXPqH5/ZfL7nmf8+KCACkPPDxQZJxinXPrT6sxKF4dkntHaqng4aawcwOHpimVF
HOFuZ0R+p6GSC6T9TKqHKSbR2aVnE7vgX2IM0EtQum5jve6KbxSze16gzGx7EMPp
bI/ltFRS4B4L9b22gltQK204yx46A67za1Ls9//1EPAAmrhzwpbXtoPjlEJ9Q+qn
TCnMQ3Pcuq5M/MBR0QsjCgzDkZv3TkW9ZSuNsX6M4JVzbOPPcOjSzLWAfXRxJNrk
zFn1dAat
-----END CERTIFICATE REQUEST-----

vagrant@vagrant:~$ vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr format=pem_bundle ttl="744h" | jq -r '.data.certificate' > intermediate.cert.pem
vagrant@vagrant:~$ cat intermediate.cert.pem

-----BEGIN CERTIFICATE-----
MIIDpjCCAo6gAwIBAgIUT/yJfMnaL2XnwBSOBqxf+dZt0R8wDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTA2MTMzODAzWhcNMjIw
NjA2MTMzODMzWjAtMSswKQYDVQQDEyJleGFtcGxlLmNvbSBJbnRlcm1lZGlhdGUg
QXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxxnkOtPz
l8O1QLRIq5sEL18hNSUIvtAo48Epm+7eaQMeiNvNdNrOSIbIQ/utCe9btSizdgkN
RPYzZVxwxhqr532UXpNvopArOzQ8d5p3Lry7PLsCiLz/InM3pjkC9irF4tmo24X1
3fPFsidJzQangENJkbjNtj1R2DL8jOZ2Mk/iFO2066SrX7SaDhsLzb3fgyPCea9n
9WB5cVrFga2ImrHdQFWeVHykPZLcCyL7qEHErhLsanb/gXFHn98gpuGzL5mPwqi5
jOesdmaMx4Q1YHelH3S0qHhevKgqe2jB/d54v8Vh61UaMxxvrQ81cnEiUwYdJxoW
xtLBUzF7W57Q2QIDAQABo4HUMIHRMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
BTADAQH/MB0GA1UdDgQWBBTjeRiF7/fVlygLb2ee7q+x5TAPbTAfBgNVHSMEGDAW
gBTjZVkM+7/Fq+y+6QD4WhSLM3su/TA7BggrBgEFBQcBAQQvMC0wKwYIKwYBBQUH
MAKGH2h0dHA6Ly8xMjcuMC4wLjE6ODIwMC92MS9wa2kvY2EwMQYDVR0fBCowKDAm
oCSgIoYgaHR0cDovLzEyNy4wLjAuMTo4MjAwL3YxL3BraS9jcmwwDQYJKoZIhvcN
AQELBQADggEBAI4s0KTi3tWrQhudk8gIFQZIUv4cyBFOjdbi112aXNNNwu8SGSKy
m79qsdpnfCnoC6HZBjIvC+zlKZsBlWI7CIJzSl/xC1Lg0s2CzK6owGRw96Rd8Oxs
AzgV9aMd7qbBodqe+NRfq9bShFLtYsiR3YPXnyjzCaXbKCTBx6mvt4gehUDzyczs
Mm6XVy+aNg0I6P83RsS17iE2GIaLz8sgJRXwng0IiFWeWzyRFZPUFcYxVYhZ2vwG
Ds/wpGo745tH6uVTu9abu5f3sTUNi+kYp19l6ReC3xaNn6BsVEjODdhSGloUbl9J
o7OEqO5palck5TtGz+kuePjXxkesaxmb9ig=
-----END CERTIFICATE-----

vagrant@vagrant:~$ vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
Success! Data written to: pki_int/intermediate/set-signed
```


- Создадим роль 
```
    vagrant@vagrant:~$ vault write pki_int/roles/example-dot-com  allowed_domains="example.com" allow_subdomains=true  max_ttl="720h"
Success! Data written to: pki_int/roles/example-dot-com
```
- сгенерируем сертификат для test.example.com на 740 часов.

```
vagrant@vagrant:~$ vault write pki_int/issue/example-dot-com common_name="test.example.com" ttl="740h" >cert.txt
vagrant@vagrant:~$ ls -al
total 166632
drwxr-xr-x 4 vagrant vagrant      4096 May  6 16:35 .
drwxr-xr-x 3 root    root         4096 Dec 19 22:42 ..
-rw------- 1 vagrant vagrant      4629 May  6 16:12 .bash_history
-rw-r--r-- 1 vagrant vagrant       220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 vagrant vagrant      3771 Feb 25  2020 .bashrc
-rw-rw-r-- 1 vagrant vagrant      1171 May  6 16:36 CA_cert.crt
drwx------ 3 vagrant vagrant      4096 May  5 22:48 .cache
-rw-rw-r-- 1 vagrant vagrant      5841 May  6 16:39 cert.txt
-rw-rw-r-- 1 vagrant vagrant      1326 May  6 16:38 intermediate.cert.pem
-rw-rw-r-- 1 vagrant vagrant       924 May  6 16:38 pki_intermediate.csr
-rw-r--r-- 1 vagrant vagrant       807 Feb 25  2020 .profile
drwx------ 2 vagrant root         4096 May  5 22:20 .ssh
-rw-r--r-- 1 vagrant vagrant         0 Dec 19 22:42 .sudo_as_admin_successful
-rw-rw-r-- 1 vagrant vagrant 170553892 May  6 16:08 vault
-rw------- 1 vagrant vagrant         4 May  6 16:35 .vault-token
-rw-r--r-- 1 vagrant vagrant         6 Dec 19 22:42 .vbox_version
-rw-r--r-- 1 root    root          180 Dec 19 22:44 .wget-hsts
```

- просмотрим содержимое сгенерированных сертификатов.
```
vagrant@vagrant:~$ cat cert.txt
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDpjCCAo6gAwIBAgIUT/yJfMnaL2XnwBSOBqxf+dZt0R8wDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTA2MTMzODAzWhcNMjIw
NjA2MTMzODMzWjAtMSswKQYDVQQDEyJleGFtcGxlLmNvbSBJbnRlcm1lZGlhdGUg
QXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxxnkOtPz
l8O1QLRIq5sEL18hNSUIvtAo48Epm+7eaQMeiNvNdNrOSIbIQ/utCe9btSizdgkN
RPYzZVxwxhqr532UXpNvopArOzQ8d5p3Lry7PLsCiLz/InM3pjkC9irF4tmo24X1
3fPFsidJzQangENJkbjNtj1R2DL8jOZ2Mk/iFO2066SrX7SaDhsLzb3fgyPCea9n
9WB5cVrFga2ImrHdQFWeVHykPZLcCyL7qEHErhLsanb/gXFHn98gpuGzL5mPwqi5
jOesdmaMx4Q1YHelH3S0qHhevKgqe2jB/d54v8Vh61UaMxxvrQ81cnEiUwYdJxoW
xtLBUzF7W57Q2QIDAQABo4HUMIHRMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
BTADAQH/MB0GA1UdDgQWBBTjeRiF7/fVlygLb2ee7q+x5TAPbTAfBgNVHSMEGDAW
gBTjZVkM+7/Fq+y+6QD4WhSLM3su/TA7BggrBgEFBQcBAQQvMC0wKwYIKwYBBQUH
MAKGH2h0dHA6Ly8xMjcuMC4wLjE6ODIwMC92MS9wa2kvY2EwMQYDVR0fBCowKDAm
oCSgIoYgaHR0cDovLzEyNy4wLjAuMTo4MjAwL3YxL3BraS9jcmwwDQYJKoZIhvcN
AQELBQADggEBAI4s0KTi3tWrQhudk8gIFQZIUv4cyBFOjdbi112aXNNNwu8SGSKy
m79qsdpnfCnoC6HZBjIvC+zlKZsBlWI7CIJzSl/xC1Lg0s2CzK6owGRw96Rd8Oxs
AzgV9aMd7qbBodqe+NRfq9bShFLtYsiR3YPXnyjzCaXbKCTBx6mvt4gehUDzyczs
Mm6XVy+aNg0I6P83RsS17iE2GIaLz8sgJRXwng0IiFWeWzyRFZPUFcYxVYhZ2vwG
Ds/wpGo745tH6uVTu9abu5f3sTUNi+kYp19l6ReC3xaNn6BsVEjODdhSGloUbl9J
o7OEqO5palck5TtGz+kuePjXxkesaxmb9ig=
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZjCCAk6gAwIBAgIUa4izq3St2J0G5gJvKI0WjR+5WpIwDQYJKoZIhvcNAQEL
BQAwLTErMCkGA1UEAxMiZXhhbXBsZS5jb20gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTAeFw0yMjA1MDYxMzM5MjJaFw0yMjA2MDUxMzM5NTJaMBsxGTAXBgNVBAMTEHRl
c3QuZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC+
yYOIbQAl7xLprPeyAlrVzdJErSUjlm8Shipb2+2LDT4Q+WXQ7g6rSLVFetOo/kwc
jM9ZMkIUrzNqPR+QN57gvJCw+csZXkSAGSzfwioRoWaZ4ky1KpGWlg+t1iM6L4jp
kaourUXS+6tePAk+++wZVuF19QMap3J80DzmGB8eeQtDK2W9ME/0gycwbeTMQikv
gKVpZz0kyE22pMbrOqWMRGWh6csZaMrr8xWDMeyskYrb7AwDLejJ/03NXfP1lxc0
pA8TLko6+r5WxmTv8igoIreVNbTxFruKmaijpZWEmen4Vjw6POZMBuLkrKDv07b7
WRABh3VFlnjX2WoO50F3AgMBAAGjgY8wgYwwDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUicIX+p+3LET6FXM+
VLXBWeSnW0MwHwYDVR0jBBgwFoAU43kYhe/31ZcoC29nnu6vseUwD20wGwYDVR0R
BBQwEoIQdGVzdC5leGFtcGxlLmNvbTANBgkqhkiG9w0BAQsFAAOCAQEAAjkdbtf5
SirseDUem8tBUGIjNy7OdSPSeMMLq3wdOUpUXT3+EQ+vap9P/6NAwqrRC7BssBI5
xR2vGk5xC4LmEisim0zR14PABlVsNqz+nBHbd8bohNt0itBZy2dIl8hRSbssBVnm
2d6vsdVtfuoGIlUzro3s8jA7p9kzm/tkwF8dXEajY5bh1/z5cxlMCSmeS2XahRlp
HbbvkNNR2bG1/XAtdixs2GNYzTQ0B8U0BBo6mWx8xCcs1mw4PXdEK16qCbcrGhG8
K1ebSufiBIrDb0rYks6NBgoF9rTVfMjVjZ2F7jd8opzJ9PDN3BVQoH3u3gk9nk8T
BFwaCTqiTtJw0A==
-----END CERTIFICATE-----
expiration          1654436392
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDpjCCAo6gAwIBAgIUT/yJfMnaL2XnwBSOBqxf+dZt0R8wDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTA2MTMzODAzWhcNMjIw
NjA2MTMzODMzWjAtMSswKQYDVQQDEyJleGFtcGxlLmNvbSBJbnRlcm1lZGlhdGUg
QXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxxnkOtPz
l8O1QLRIq5sEL18hNSUIvtAo48Epm+7eaQMeiNvNdNrOSIbIQ/utCe9btSizdgkN
RPYzZVxwxhqr532UXpNvopArOzQ8d5p3Lry7PLsCiLz/InM3pjkC9irF4tmo24X1
3fPFsidJzQangENJkbjNtj1R2DL8jOZ2Mk/iFO2066SrX7SaDhsLzb3fgyPCea9n
9WB5cVrFga2ImrHdQFWeVHykPZLcCyL7qEHErhLsanb/gXFHn98gpuGzL5mPwqi5
jOesdmaMx4Q1YHelH3S0qHhevKgqe2jB/d54v8Vh61UaMxxvrQ81cnEiUwYdJxoW
xtLBUzF7W57Q2QIDAQABo4HUMIHRMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
BTADAQH/MB0GA1UdDgQWBBTjeRiF7/fVlygLb2ee7q+x5TAPbTAfBgNVHSMEGDAW
gBTjZVkM+7/Fq+y+6QD4WhSLM3su/TA7BggrBgEFBQcBAQQvMC0wKwYIKwYBBQUH
MAKGH2h0dHA6Ly8xMjcuMC4wLjE6ODIwMC92MS9wa2kvY2EwMQYDVR0fBCowKDAm
oCSgIoYgaHR0cDovLzEyNy4wLjAuMTo4MjAwL3YxL3BraS9jcmwwDQYJKoZIhvcN
AQELBQADggEBAI4s0KTi3tWrQhudk8gIFQZIUv4cyBFOjdbi112aXNNNwu8SGSKy
m79qsdpnfCnoC6HZBjIvC+zlKZsBlWI7CIJzSl/xC1Lg0s2CzK6owGRw96Rd8Oxs
AzgV9aMd7qbBodqe+NRfq9bShFLtYsiR3YPXnyjzCaXbKCTBx6mvt4gehUDzyczs
Mm6XVy+aNg0I6P83RsS17iE2GIaLz8sgJRXwng0IiFWeWzyRFZPUFcYxVYhZ2vwG
Ds/wpGo745tH6uVTu9abu5f3sTUNi+kYp19l6ReC3xaNn6BsVEjODdhSGloUbl9J
o7OEqO5palck5TtGz+kuePjXxkesaxmb9ig=
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEpgIBAAKCAQEAvsmDiG0AJe8S6az3sgJa1c3SRK0lI5ZvEoYqW9vtiw0+EPll
0O4Oq0i1RXrTqP5MHIzPWTJCFK8zaj0fkDee4LyQsPnLGV5EgBks38IqEaFmmeJM
tSqRlpYPrdYjOi+I6ZGqLq1F0vurXjwJPvvsGVbhdfUDGqdyfNA85hgfHnkLQytl
vTBP9IMnMG3kzEIpL4ClaWc9JMhNtqTG6zqljERloenLGWjK6/MVgzHsrJGK2+wM
Ay3oyf9NzV3z9ZcXNKQPEy5KOvq+VsZk7/IoKCK3lTW08Ra7ipmoo6WVhJnp+FY8
OjzmTAbi5Kyg79O2+1kQAYd1RZZ419lqDudBdwIDAQABAoIBAQCDhnCDEIHVpOco
6PEpaq4sTuPyLFs6GW8hGhUBlrA/+DUdhpFGY5cBRfKRn4djLUxTMk+PcjJUCCQt
zU2BIpcI33XwS4RUTMR0euaQq05NMTOGSpgPHPWZFoJgmQtYBYUhYCYuX42PUahm
IoVNQmcVYOu1Yd9t9JncqcIR7fcmDaNDDrBRX7elya2YzNEi1oow8tcXv9zaP9h0
OfvQ+PNHxVw275lusnrP+3btCzov6G4EMJLpZK8XsfhoIae9gCwXfSn/LDJ+9aiX
y/OE+YrW3s+3yuLjmBx/8KRvzm61CTqRubmW+uEmooWYCvOWZJGTeYRKoMV4YXPk
t6y++vJhAoGBAOfHMZFj6Vvjx4D1VJciPnaOSyuXMgBc/o4vUbnnK8OtAHmmFf3Q
aTikF+2dYTKd43u062PWD/orGTapgJnlg1K6Py2sbahhTvTSVJG/ZSgtNR8Ahyab
aOq07DB8TsW/upUKLn9k4L5mAdSlpeYp6yGmlQssBF2AtguQx1AH9d1lAoGBANK5
rm1KhalO2ZfY5p0bRdP+zx73Limd2qQO7/WtSuXEMyZd5j/ApZVNyAhpzbGtHYly
2OhlvHKYoyNBAAHVSPP2hcgPY7goRaOXOxIIvsXpDw6yCnJrjmbUpraMLn/JoeOz
0GnmCed57sZWr3L8WUX7J2ITm52fE+W04nOm2HOrAoGBAKYp4BdjkYEXI4gIWcSg
ADfue2Wc13P7N2On+OdWG8IE3TpZH5r/NvAsdBsdoPkwyE1/vuGsLXgkPcc6bb/n
EubUL0cHPvZgDMLnp0l19YeAIX1AW6QJ9oWNf2ajN7X9Zj5DNmKOZ7EYYGea+fyw
QKm3S1LfE6IVDG3uNXXivxkhAoGBAI8kPo5S/vSBcWGqnwy9h/N3tW5NbfgrGGg1
FV+HdccaPbykMFHCJTiEe4rrhCRbeiLx9tH2ucoGUVsTPlQAc+v02++INK7V/Ggh
xKh9dEmwBOBwpJQQ58yGSzDZjISVjPjGHp6w+q1YsI/eNpQfQu08rAX9V8n8U2HI
O8Nf5My9AoGBAJP7j9lnYxKkBjdS6noLDZcLeRA44npA3pkWsRv9Z1zwOunytndK
+K0f0vHHA4lNbr29wf1KzCbBv5XDPZJ7DhHjwdt46xh/LSummMKvEI47t1pNysT5
Uoq41JWC/W7VVGPs2mjyBaR6FYGrHii9AnQokZ0yximGDyWysyonptR+
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       6b:88:b3:ab:74:ad:d8:9d:06:e6:02:6f:28:8d:16:8d:1f:b9:5a:92
```

- создадим сертификат для будущего сайта (certificate+issuing_ca+CA_certificate+private_key) в файл cert.crt
```
vagrant@vagrant:/etc/nginx$ cat cert.crt
-----BEGIN CERTIFICATE-----
MIIDZjCCAk6gAwIBAgIUa4izq3St2J0G5gJvKI0WjR+5WpIwDQYJKoZIhvcNAQEL
BQAwLTErMCkGA1UEAxMiZXhhbXBsZS5jb20gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTAeFw0yMjA1MDYxMzM5MjJaFw0yMjA2MDUxMzM5NTJaMBsxGTAXBgNVBAMTEHRl
c3QuZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC+
yYOIbQAl7xLprPeyAlrVzdJErSUjlm8Shipb2+2LDT4Q+WXQ7g6rSLVFetOo/kwc
jM9ZMkIUrzNqPR+QN57gvJCw+csZXkSAGSzfwioRoWaZ4ky1KpGWlg+t1iM6L4jp
kaourUXS+6tePAk+++wZVuF19QMap3J80DzmGB8eeQtDK2W9ME/0gycwbeTMQikv
gKVpZz0kyE22pMbrOqWMRGWh6csZaMrr8xWDMeyskYrb7AwDLejJ/03NXfP1lxc0
pA8TLko6+r5WxmTv8igoIreVNbTxFruKmaijpZWEmen4Vjw6POZMBuLkrKDv07b7
WRABh3VFlnjX2WoO50F3AgMBAAGjgY8wgYwwDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUicIX+p+3LET6FXM+
VLXBWeSnW0MwHwYDVR0jBBgwFoAU43kYhe/31ZcoC29nnu6vseUwD20wGwYDVR0R
BBQwEoIQdGVzdC5leGFtcGxlLmNvbTANBgkqhkiG9w0BAQsFAAOCAQEAAjkdbtf5
SirseDUem8tBUGIjNy7OdSPSeMMLq3wdOUpUXT3+EQ+vap9P/6NAwqrRC7BssBI5
xR2vGk5xC4LmEisim0zR14PABlVsNqz+nBHbd8bohNt0itBZy2dIl8hRSbssBVnm
2d6vsdVtfuoGIlUzro3s8jA7p9kzm/tkwF8dXEajY5bh1/z5cxlMCSmeS2XahRlp
HbbvkNNR2bG1/XAtdixs2GNYzTQ0B8U0BBo6mWx8xCcs1mw4PXdEK16qCbcrGhG8
K1ebSufiBIrDb0rYks6NBgoF9rTVfMjVjZ2F7jd8opzJ9PDN3BVQoH3u3gk9nk8T
BFwaCTqiTtJw0A==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDpjCCAo6gAwIBAgIUT/yJfMnaL2XnwBSOBqxf+dZt0R8wDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTA2MTMzODAzWhcNMjIw
NjA2MTMzODMzWjAtMSswKQYDVQQDEyJleGFtcGxlLmNvbSBJbnRlcm1lZGlhdGUg
QXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxxnkOtPz
l8O1QLRIq5sEL18hNSUIvtAo48Epm+7eaQMeiNvNdNrOSIbIQ/utCe9btSizdgkN
RPYzZVxwxhqr532UXpNvopArOzQ8d5p3Lry7PLsCiLz/InM3pjkC9irF4tmo24X1
3fPFsidJzQangENJkbjNtj1R2DL8jOZ2Mk/iFO2066SrX7SaDhsLzb3fgyPCea9n
9WB5cVrFga2ImrHdQFWeVHykPZLcCyL7qEHErhLsanb/gXFHn98gpuGzL5mPwqi5
jOesdmaMx4Q1YHelH3S0qHhevKgqe2jB/d54v8Vh61UaMxxvrQ81cnEiUwYdJxoW
xtLBUzF7W57Q2QIDAQABo4HUMIHRMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
BTADAQH/MB0GA1UdDgQWBBTjeRiF7/fVlygLb2ee7q+x5TAPbTAfBgNVHSMEGDAW
gBTjZVkM+7/Fq+y+6QD4WhSLM3su/TA7BggrBgEFBQcBAQQvMC0wKwYIKwYBBQUH
MAKGH2h0dHA6Ly8xMjcuMC4wLjE6ODIwMC92MS9wa2kvY2EwMQYDVR0fBCowKDAm
oCSgIoYgaHR0cDovLzEyNy4wLjAuMTo4MjAwL3YxL3BraS9jcmwwDQYJKoZIhvcN
AQELBQADggEBAI4s0KTi3tWrQhudk8gIFQZIUv4cyBFOjdbi112aXNNNwu8SGSKy
m79qsdpnfCnoC6HZBjIvC+zlKZsBlWI7CIJzSl/xC1Lg0s2CzK6owGRw96Rd8Oxs
AzgV9aMd7qbBodqe+NRfq9bShFLtYsiR3YPXnyjzCaXbKCTBx6mvt4gehUDzyczs
Mm6XVy+aNg0I6P83RsS17iE2GIaLz8sgJRXwng0IiFWeWzyRFZPUFcYxVYhZ2vwG
Ds/wpGo745tH6uVTu9abu5f3sTUNi+kYp19l6ReC3xaNn6BsVEjODdhSGloUbl9J
o7OEqO5palck5TtGz+kuePjXxkesaxmb9ig=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDNTCCAh2gAwIBAgIUJ1kCd9813PF9FYG4oL1PS1oEscswDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTA2MTMzNjIwWhcNMjIw
NjA3MTMzNjQ5WjAWMRQwEgYDVQQDEwtleGFtcGxlLmNvbTCCASIwDQYJKoZIhvcN
AQEBBQADggEPADCCAQoCggEBAL6jwGMyA0WH/z2FKLshfqc602ColkRCeR2Ks8VH
F+2zA8MZVD3ddjVUtztScPhluT3+JlINSWZq4Z9IcM5FS/B1HxMI/01+mVOv5ySB
PSdjNc/HyZ3hPNIC8oVbtopczGMPzBeNSRCQd0kytxlqRR1ccB4Ezv5lUY8aRwrc
Tf0hL0iTBJqU2M6Of+j68E/JyuMtDr2ML2+q6tj1+2O2c+NMxdLhz4KsuIM5YeCK
F7rWeziaID1Dkt2gU5eBR0r/rHKAuhV7PrSGTZMPWIajG0Dwi2P7MCWkOlmuZBm/
m0zEEKY/0TQC1S0tPjPucGjltI15RIv+3T5dGurCh09jV9sCAwEAAaN7MHkwDgYD
VR0PAQH/BAQDAgEGMA8GA1UdEwEB/wQFMAMBAf8wHQYDVR0OBBYEFONlWQz7v8Wr
7L7pAPhaFIszey79MB8GA1UdIwQYMBaAFONlWQz7v8Wr7L7pAPhaFIszey79MBYG
A1UdEQQPMA2CC2V4YW1wbGUuY29tMA0GCSqGSIb3DQEBCwUAA4IBAQCLLZxjdt/t
aBkj99Pj+OCxKyVjkdS3I9R5wDfj9qM+kvtYliRSCeVk9ER+eeUK3syxpTqZR5fm
4je9FdIcvOutE+efjtUzMESXF8cuIm5yqdourc0Iw4VKSBXrVqDZLi+oWIO0jax3
Zd6X/hNU1ZPW74pCpXOatzapWbp2frOoqji71SlrOpNE63y0cyF5eRUkGAIWgaAx
OdyQsZqs/yLlyAcnjBh1U5lEpYEPGev9OVTDzoHekxPOTLJc4nQjT0F3l1R7XG35
UYfOV8XdwirtfgIejXZ08sldo/9tyIPNXXamfFZSnh5a0W4ZXZxPDJfXDcoEQx11
xuc0+np9M+ky
-----END CERTIFICATE-----
-----BEGIN RSA PRIVATE KEY-----
MIIEpgIBAAKCAQEAvsmDiG0AJe8S6az3sgJa1c3SRK0lI5ZvEoYqW9vtiw0+EPll
0O4Oq0i1RXrTqP5MHIzPWTJCFK8zaj0fkDee4LyQsPnLGV5EgBks38IqEaFmmeJM
tSqRlpYPrdYjOi+I6ZGqLq1F0vurXjwJPvvsGVbhdfUDGqdyfNA85hgfHnkLQytl
vTBP9IMnMG3kzEIpL4ClaWc9JMhNtqTG6zqljERloenLGWjK6/MVgzHsrJGK2+wM
Ay3oyf9NzV3z9ZcXNKQPEy5KOvq+VsZk7/IoKCK3lTW08Ra7ipmoo6WVhJnp+FY8
OjzmTAbi5Kyg79O2+1kQAYd1RZZ419lqDudBdwIDAQABAoIBAQCDhnCDEIHVpOco
6PEpaq4sTuPyLFs6GW8hGhUBlrA/+DUdhpFGY5cBRfKRn4djLUxTMk+PcjJUCCQt
zU2BIpcI33XwS4RUTMR0euaQq05NMTOGSpgPHPWZFoJgmQtYBYUhYCYuX42PUahm
IoVNQmcVYOu1Yd9t9JncqcIR7fcmDaNDDrBRX7elya2YzNEi1oow8tcXv9zaP9h0
OfvQ+PNHxVw275lusnrP+3btCzov6G4EMJLpZK8XsfhoIae9gCwXfSn/LDJ+9aiX
y/OE+YrW3s+3yuLjmBx/8KRvzm61CTqRubmW+uEmooWYCvOWZJGTeYRKoMV4YXPk
t6y++vJhAoGBAOfHMZFj6Vvjx4D1VJciPnaOSyuXMgBc/o4vUbnnK8OtAHmmFf3Q
aTikF+2dYTKd43u062PWD/orGTapgJnlg1K6Py2sbahhTvTSVJG/ZSgtNR8Ahyab
aOq07DB8TsW/upUKLn9k4L5mAdSlpeYp6yGmlQssBF2AtguQx1AH9d1lAoGBANK5
rm1KhalO2ZfY5p0bRdP+zx73Limd2qQO7/WtSuXEMyZd5j/ApZVNyAhpzbGtHYly
2OhlvHKYoyNBAAHVSPP2hcgPY7goRaOXOxIIvsXpDw6yCnJrjmbUpraMLn/JoeOz
0GnmCed57sZWr3L8WUX7J2ITm52fE+W04nOm2HOrAoGBAKYp4BdjkYEXI4gIWcSg
ADfue2Wc13P7N2On+OdWG8IE3TpZH5r/NvAsdBsdoPkwyE1/vuGsLXgkPcc6bb/n
EubUL0cHPvZgDMLnp0l19YeAIX1AW6QJ9oWNf2ajN7X9Zj5DNmKOZ7EYYGea+fyw
QKm3S1LfE6IVDG3uNXXivxkhAoGBAI8kPo5S/vSBcWGqnwy9h/N3tW5NbfgrGGg1
FV+HdccaPbykMFHCJTiEe4rrhCRbeiLx9tH2ucoGUVsTPlQAc+v02++INK7V/Ggh
xKh9dEmwBOBwpJQQ58yGSzDZjISVjPjGHp6w+q1YsI/eNpQfQu08rAX9V8n8U2HI
O8Nf5My9AoGBAJP7j9lnYxKkBjdS6noLDZcLeRA44npA3pkWsRv9Z1zwOunytndK
+K0f0vHHA4lNbr29wf1KzCbBv5XDPZJ7DhHjwdt46xh/LSummMKvEI47t1pNysT5
Uoq41JWC/W7VVGPs2mjyBaR6FYGrHii9AnQokZ0yximGDyWysyonptR+
-----END RSA PRIVATE KEY-----
```

5. Установите корневой сертификат созданного центра сертификации в доверенные в хостовой системе.

- добавление корневого сертификата для linux машины
```
vagrant@vagrant:~$ sudo cp CA_cert.crt /usr/local/share/ca-certificates
vagrant@vagrant:~$ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
```
- добавление корневого сертифика на window хост:


![Windows Host](CA-test-example-com.png)

6. Установите nginx.
- Установим nginx
```
vagrant@vagrant:~$ sudo apt-add-repository ppa:nginx/stable
 This PPA contains the latest Stable Release version of the nginx web server software.

**Only Non-End-of-Life Ubuntu Releases are supported in this PPA**

**Development releases of Ubuntu are not officially supported by this PPA, and uploads for those will not be available until actual final releases for those versions**
 More info: https://launchpad.net/~nginx/+archive/ubuntu/stable
Press [ENTER] to continue or Ctrl-c to cancel adding it.

Hit:1 http://us.archive.ubuntu.com/ubuntu focal InRelease
Get:2 http://ppa.launchpad.net/nginx/stable/ubuntu focal InRelease [17.5 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
Get:4 http://ppa.launchpad.net/nginx/stable/ubuntu focal/main amd64 Packages [4,944 B]
Get:5 http://ppa.launchpad.net/nginx/stable/ubuntu focal/main Translation-en [4,572 B]
Get:6 http://us.archive.ubuntu.com/ubuntu focal-backports InRelease [108 kB]
Get:7 http://us.archive.ubuntu.com/ubuntu focal-security InRelease [114 kB]
Unpacking libnginx-mod-mail (1.18.0-3ubuntu1+focal2) ...
Selecting previously unselected package libnginx-mod-stream.
Preparing to unpack .../20-libnginx-mod-stream_1.18.0-3ubuntu1+focal2_amd64.deb ...
Unpacking libnginx-mod-stream (1.18.0-3ubuntu1+focal2) ...
Selecting previously unselected package libnginx-mod-stream-geoip.
Preparing to unpack .../21-libnginx-mod-stream-geoip_1.18.0-3ubuntu1+focal2_amd64.deb ...
Unpacking libnginx-mod-stream-geoip (1.18.0-3ubuntu1+focal2) ...
Selecting previously unselected package nginx-core.
Preparing to unpack .../22-nginx-core_1.18.0-3ubuntu1+focal2_amd64.deb ...
Unpacking nginx-core (1.18.0-3ubuntu1+focal2) ...
Selecting previously unselected package libnginx-mod-http-image-filter.
Preparing to unpack .../23-libnginx-mod-http-image-filter_1.18.0-3ubuntu1+focal2_amd64.deb ...
Unpacking libnginx-mod-http-image-filter (1.18.0-3ubuntu1+focal2) ...
Selecting previously unselected package nginx.
Preparing to unpack .../24-nginx_1.18.0-3ubuntu1+focal2_all.deb ...
Unpacking nginx (1.18.0-3ubuntu1+focal2) ...
Setting up libxau6:amd64 (1:1.0.9-0ubuntu1) ...
Setting up libxdmcp6:amd64 (1:1.1.3-0ubuntu1) ...
Setting up libxcb1:amd64 (1.14-2) ...
Setting up nginx-common (1.18.0-3ubuntu1+focal2) ...
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /lib/systemd/system/nginx.service.
Setting up libjbig0:amd64 (2.1-3.1build1) ...
Setting up libx11-data (2:1.6.9-2ubuntu1.2) ...
Setting up libwebp6:amd64 (0.6.1-2ubuntu0.20.04.1) ...
Setting up fonts-dejavu-core (2.37-1) ...
Setting up libjpeg-turbo8:amd64 (2.0.3-0ubuntu1.20.04.1) ...
Setting up libgeoip1:amd64 (1.6.12-6build1) ...
Setting up libx11-6:amd64 (2:1.6.9-2ubuntu1.2) ...
Setting up geoip-database (20191224-2) ...
Setting up libjpeg8:amd64 (8c-2ubuntu8) ...
Setting up libxpm4:amd64 (1:3.5.12-1) ...
Setting up fontconfig-config (2.13.1-2ubuntu3) ...
Setting up libtiff5:amd64 (4.1.0+git191117-2ubuntu0.20.04.2) ...
Setting up libfontconfig1:amd64 (2.13.1-2ubuntu3) ...
Setting up libgd3:amd64 (2.2.5-5.2ubuntu2.1) ...
Setting up libnginx-mod-stream (1.18.0-3ubuntu1+focal2) ...
Setting up nginx-core (1.18.0-3ubuntu1+focal2) ...
 * Upgrading binary nginx                                                                                                                                                                                 [ OK ]
Setting up libnginx-mod-mail (1.18.0-3ubuntu1+focal2) ...
Setting up nginx (1.18.0-3ubuntu1+focal2) ...
Setting up libnginx-mod-http-image-filter (1.18.0-3ubuntu1+focal2) ...
Setting up libnginx-mod-stream-geoip (1.18.0-3ubuntu1+focal2) ...
Setting up libnginx-mod-http-xslt-filter (1.18.0-3ubuntu1+focal2) ...
Processing triggers for libc-bin (2.31-0ubuntu9.2) ...
Processing triggers for ufw (0.36-6ubuntu1) ...
Processing triggers for systemd (245.4-4ubuntu3.13) ...
Processing triggers for man-db (2.9.1-1) ...
Processing triggers for nginx-core (1.18.0-3ubuntu1+focal2) ...
Triggering nginx reload ...

```
- запустим службу и проверим, что она работает.
```
vagrant@vagrant:~$ sudo systemctl enable nginx
Synchronizing state of nginx.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable nginx

vagrant@vagrant:/etc/nginx/sites-available$ sudo systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2022-05-06 16:53:53 MSK; 6h ago
       Docs: man:nginx(8)
   Main PID: 3847 (nginx)
      Tasks: 7 (limit: 7038)
     Memory: 9.3M
     CGroup: /system.slice/nginx.service
             ├─3847 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             ├─3998 nginx: worker process
             ├─3999 nginx: worker process
             ├─4000 nginx: worker process
             ├─4001 nginx: worker process
             ├─4002 nginx: worker process
             └─4003 nginx: worker process

May 06 16:53:53 vagrant systemd[1]: Starting A high performance web server and a reverse proxy server...
May 06 16:53:53 vagrant systemd[1]: Started A high performance web server and a reverse proxy server.
May 06 16:53:56 vagrant systemd[1]: Reloading A high performance web server and a reverse proxy server.
May 06 16:53:56 vagrant systemd[1]: Reloaded A high performance web server and a reverse proxy server.
```
7.По инструкции (ссылка) настройте nginx на https, используя ранее подготовленный сертификат:
можно использовать стандартную стартовую страницу nginx для демонстрации работы сервера;
можно использовать и другой html файл, сделанный вами;

- создадим следующую конфигурацию для сайта example.com.conf, будем использовать ранее созданный сертификат cert.crt
```
vagrant@vagrant:~$ sudo cp cert.crt /etc/nginx/
vagrant@vagrant:/etc/nginx/sites-available$ cat example.com.conf
server {
    listen              443 ssl;
    root /var/www/html;
    server_name         test.example.com;
    ssl_certificate     cert.crt;
    ssl_certificate_key cert.crt;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
}

```
- проверим, что конфигурация проходит проверку
```
vagrant@vagrant:/etc/nginx/sites-enabled$ sudo ln -s /etc/nginx/sites-available/example.com.conf test
```

- пропишем dns  имя в файле host
```
vagrant@vagrant:~$ cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 vagrant
192.168.34.12 test.example.com
192.168.34.12 www.test.example.com

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
- в windows
```
# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# This file contains the mappings of IP addresses to host names. Each
# entry should be kept on an individual line. The IP address should
# be placed in the first column followed by the corresponding host name.
# The IP address and the host name should be separated by at least one
# space.
#
# Additionally, comments (such as these) may be inserted on individual
# lines or following the machine name denoted by a '#' symbol.
#
# For example:
#
#      102.54.94.97     rhino.acme.com          # source server
#       38.25.63.10     x.acme.com              # x client host

# localhost name resolution is handled within DNS itself.
#	127.0.0.1       localhost
#	::1             localhost

192.168.33.10 www.test.site
192.168.33.10 test.site
192.168.34.12 test.example.com
192.168.34.12 www.test.example.com

82.102.22.2 no-004.whiskergalaxy.com   #added by Windscribe, do not modify.
91.219.215.98 no-007.whiskergalaxy.com   #added by Windscribe, do not modify.
146.70.103.2 no-011.whiskergalaxy.com   #added by Windscribe, do not modify.
37.120.203.226 no-012.whiskergalaxy.com   #added by Windscribe, do not modify.
146.70.17.242 no-009.whiskergalaxy.com   #added by Windscribe, do not modify.
```

8. Откройте в браузере на хосте https адрес страницы, которую обслуживает сервер nginx.
- открытие в Opera
![opera](https-opera.jpg)

- открытие в EDGE
![edge](https-edge.jpg)

9. Создайте скрипт, который будет генерировать новый сертификат в vault:
генерируем новый сертификат так, чтобы не переписывать конфиг nginx;
перезапускаем nginx для применения нового сертификата.

Скрипт указан ниже. Генерируется сертификат для доменного имени test.example.com, происходит перезагрузка сервера NGINX.
```buildoutcfg
import os
import socket
import datetime
import time
import json
import argparse
import requests
import subprocess

from requests.exceptions import HTTPError

# get site name

#parser = argparse.ArgumentParser(description='Input site name')
#parser.add_argument('site', help='input site name. It should ends with example.com')

#args = parser.parse_args()
#print(args)
#site = args.site

site = 'test.example.com'

# perform http post request

if 'VAULT_ADDR' in os.environ:
    VAULT_ADDR = os.environ.get('VAULT_ADDR')
else:
    VAULT_ADDR = 'http://127.0.0.1:8200'

if 'VAULT_TOKEN' in os.environ:
    VAULT_TOKEN = os.environ.get('VAULT_TOKEN')
else:
    VAULT_TOKEN = 'root'


url = VAULT_ADDR + '/v1/pki_int/issue/example-dot-com'
headers = {'X-Vault-Token': VAULT_TOKEN}
data = {'common_name': site, 'ttl': '1h', 'format': 'pem_bundle'}

#url2 = 'http://127.0.0.1:8200/pki_int/roles/example-dot-com'

# request

try:
    response = requests.post(url, data=json.dumps(data), headers=headers)
#    response = requests.get(url2, headers=headers)
# если ответ успешен, исключения задействованы не будут
    response.raise_for_status()
except HTTPError as http_err:
    print(f'HTTP error occurred: {http_err}')
except Exception as err:
    print(f'Other error occurred: {err}')
else:
    print('Success!')

print(response.status_code)

cert_str=response.text

cert_json = json.loads(cert_str)

crt_mass = cert_json ['data']
cert_str = crt_mass['certificate']

print(cert_str)
file_crt = open('/etc/nginx/cert.crt', 'w')
file_crt.write(cert_str)
file_crt.close()

#sudo systemctl reload nginx
start_nginx = subprocess.run(['systemctl', 'reload','nginx'])

if start_nginx.returncode == 0:
    print(start_nginx.returncode)
    print('Обновление сертификата завершено успешно')
else:
    print('Обновление сертификата прошло некорректно, nginx не запущен')
```
Скрипт имеет следующий вывод на экран:
```
vagrant@vagrant:~/py-scripts$ sudo python3.9 my_script.py
Success!
200
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA9a5Dwj8i1ukLd/VHLqTvWxTmFR4ZlElPFKrckYuluEGDpSK3
mxvfUqdl9XE7qKC6xjVMBvycbeGNB4/p+zJU4Gz6Jt16CNGx/P6Uw0Bl7/CQ5AZE
xTgMVGhYPwwx52bZ2Oi5d0Oo7BK6qnCfa5MP5NcqZyT5g3prr91l6/o9wwFLzJfU
72wOd3iDH7p2Ws+/jV/qz4KM6RVPn43xjsoyj+gzkyU0eNANp+7opDro1/NVSGSj
ch89mHa/C79vFD9Ks5a+UsqoD/kbTePrFbvvWexuMA9fcxv5PzIlf7uMnJeo/lkB
jT/Wy73hIK9qWBl5BfS5qjhexQOuYtCYeKGh7wIDAQABAoIBAFuLR6xFolG6HI/i
UVfMFXQ24VTVFmrutgzZbb697k4t+0O5H/kn9ZryzUcbnkNDQNVCXsXrss4u0UIU
GfeD1aSadsuskyBeIphGtF/cyx8Q4KxWuaFTNSTujJNRk/bJmDr+3LzX9UWh5PRF
auHGtpFqq14rpN2c7Ctnr+ayI/lY813hZbscjPaBVtYKWkNSAArATO6AQi/PsBso
VVpaQbb2Px2U40bP4AiZdio7mIo+1Jmzcd8jnQAG1K0WRo2vPXQFKqJQnlwx1siY
pEaczZQnP7+P+iPY5QYYogO9GsdwLrVkwwZRJ+NMpcQXfiMmfMo2yBQ5Tjl3gq15
oOAjfiECgYEA9sk2JSSGb7U9gdi8++j9nEECsy3xoa5vO6efUAgvKzElblRjnNj8
CLmNs/pC0R2WUF2iWIf/Kf64RI83liAoleO2dYYt4aaykbMz4PGUlywsQHYuzs8S
dsKV8L0RI25embGEaMfnH+Zjz49xBQJwBvAzkj6b1hjjoIlqrryP6uUCgYEA/tp9
Mt6KHzV60A39r03sb7vm4tsAaTyO2RtcXSWpM436RI4X1tH9ICP/Aw+DEixtYpbk
jmY7Hy2EZORTTzRaWl7hrqjZ/WspEcbOmXm6MPYjmjgDml7731YurSVNCr36xFl/
7hJqzmpbcPTyY+GV0lr9eKeaU+RMCkZLYwcgCEMCgYEAqPdU2hNq9DrsFOYXZoa8
dWpRdeqkAOYUhIyc2EfyO6lXjnW7Ch3KWhLpQWu8qb6PNxEU4XajrtYWxeIk1eXI
w+938nlfGptusbE3ePEOon59Admo7LH8W1GZJY3dmRutK5/OH4uW5FFyzx7P6/Ui
TK3TMXMqcWjzGOxdHaw/AT0CgYAUJ7VlE4J662DWGXUpqxD62TJuWIGU0Qw/EVHh
r/vMrvrO9yndfGf3ZcFiPZTRGCBifqp+tIkScygYTpJkHjDTGtPgW0w6JYJu/HkU
XDMkciLorGtkhwSGaitw2BVatyGg668ZZe16e6DVsxmEWq/dngKTyzWCUvjMizil
EUgwPwKBgQDe90AN8XTeC77zQa/3w5sOaqKzyfI6qgFexIr9VVgeEwxGu3q3LM8A
NE3F1QVkN5hvZPuUw04QNoAJONG+p/rpBkvHjI2zRZnLxWJtJ0+oIEy9TuF9bQF4
05sltZt/33XDMwFD+IfyWED3B36zb5f2lemf5X/LKImakt7Nk3GUcw==
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIDZjCCAk6gAwIBAgIUO0nYAhrYVmV28kR2ZqW8Z2s/yVQwDQYJKoZIhvcNAQEL
BQAwLTErMCkGA1UEAxMiZXhhbXBsZS5jb20gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTAeFw0yMjA1MTQyMTA3MjJaFw0yMjA1MTQyMjA3NTJaMBsxGTAXBgNVBAMTEHRl

c3QuZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQD1
rkPCPyLW6Qt39UcupO9bFOYVHhmUSU8UqtyRi6W4QYOlIrebG99Sp2X1cTuooLrG
NUwG/Jxt4Y0Hj+n7MlTgbPom3XoI0bH8/pTDQGXv8JDkBkTFOAxUaFg/DDHnZtnY
6Ll3Q6jsErqqcJ9rkw/k1ypnJPmDemuv3WXr+j3DAUvMl9TvbA53eIMfunZaz7+N
X+rPgozpFU+fjfGOyjKP6DOTJTR40A2n7uikOujX81VIZKNyHz2Ydr8Lv28UP0qz
lr5SyqgP+RtN4+sVu+9Z7G4wD19zG/k/MiV/u4ycl6j+WQGNP9bLveEgr2pYGXkF
9LmqOF7FA65i0Jh4oaHvAgMBAAGjgY8wgYwwDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQU71bBwPD6R7F1K2Qv
mULtlq/O+ZgwHwYDVR0jBBgwFoAUXWk0XByfFVg5ySIrZ4p1ZdtdqfMwGwYDVR0R
BBQwEoIQdGVzdC5leGFtcGxlLmNvbTANBgkqhkiG9w0BAQsFAAOCAQEAfnI3j4UH
HPNONYPdnizfr/0h3qjixgd3bARTWRI7jR6XujQnCAJcNsr/vHhdlx8RPDNWJJlW
PxF4KHDu1kP8aFOapenTXcg9I7A57r/Kl7oz0r062ozJcWG8ytHh+7/+fLDeuscw
IhPT+PPYcdlvR+CmHRLLq3N2d+XF0AzouSwdbWIHWuzLvH82rIsYuOYd1Di0I6Ht
mT8JMhF10W7r4ZNnRzDuwXhhNw9peEgD78jAeCR7sFgHIkUC13cFqal88ynOOufb
VlB73Q+is6ZFSAcB1C9ReIT+fvk6iHjeLTglD0mqJ6yo311SiVa/ROihudHfrGKk
JF4hRfziuWQyZg==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDpjCCAo6gAwIBAgIUTRsKAHigjwfh9xxnzxSUmC43MHIwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTE0MTk1NzA0WhcNMjIw
NjE0MTk1NzM0WjAtMSswKQYDVQQDEyJleGFtcGxlLmNvbSBJbnRlcm1lZGlhdGUg
QXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAx9xEmkYA
nomIIFKAEmaOiBb4DcxwpixD+9T2ECvjzEHNPlTet3mca6dHtKJqLUxqTlsFr1YU
Qzharfv1dlcJmZue0Hd2250If+dl5nZoAOyDvs3JJbIidFDpUOZTgImqAgkxCeFR
9+NgqZwU6KiVXD08rFYkbRNK0kFu7l+un38fd4UsTX3/Vk1uyaou4y/Y9XDjM/YS
Xo0+xJtfV2JaS8pSccLyZE0yb0/iVTpD9GdHhur0vouz7fAw2tb9s+fWiSBehQ8y
UJoz+a15qDP8eowdrgy2QrKge2CNM+vbZqvHKNqM/elB+A67AF4a5YCLoOZTElCu
sX+EeMMdvUwWwwIDAQABo4HUMIHRMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
BTADAQH/MB0GA1UdDgQWBBRdaTRcHJ8VWDnJIitninVl212p8zAfBgNVHSMEGDAW
gBRn6TqWHjzSYCOrdnFq1DajUwP0ozA7BggrBgEFBQcBAQQvMC0wKwYIKwYBBQUH
MAKGH2h0dHA6Ly8xMjcuMC4wLjE6ODIwMC92MS9wa2kvY2EwMQYDVR0fBCowKDAm
oCSgIoYgaHR0cDovLzEyNy4wLjAuMTo4MjAwL3YxL3BraS9jcmwwDQYJKoZIhvcN
AQELBQADggEBACp44DEZZSyMkTTKYXBzjlkVb4veqhi+mfsxwQm3UrrIlsDZjgWG
bxJPiY6SrKTeyF6hRjoQ00Q0ufHSH40APEzaOE6ix+SDIg9Jyev/hwURhbCSIFwf
sn2ozYXcE2y2xOm78NRYQjwn5QYNYTeZayYX1hIsbdrOroWdi/ZSVtk4v6OZKH4v
0tGo8TaRRFF+Tno5/BjAxmZPFq0IH0gqC8uLk9h7Copn2AwC4xEUcWvlrC7gA3y7
T/tiQ6Lbu2oC3vi62gc8N8BEKSPSvPLfl9xCQHCz7g9J+djVO/i88NpZPLy+VUC2
fK7fc4k/VTafIhflCvPD4RdluYHQJa/vnXU=
-----END CERTIFICATE-----
0
Обновление сертификата завершено успешно
```
10. Поместите скрипт в crontab, чтобы сертификат обновлялся какого-то числа каждого месяца в удобное для вас время.
- планируем задание (по Universal time: Sat 2022-05-14 22:15:00 UTC)

```
vagrant@vagrant:~$ sudo crontab -u root -e
15 22 14 * * python3.9 /home/vagrant/py-scripts/my_script.py >> /home/vagrant/script.log
crontab: installing new crontab
```
- ниже видим лог отработки задания
```
vagrant@vagrant:/etc/cron.monthly$ cat /var/log/syslog | grep CRON
May 14 10:04:20 vagrant cron[686]: (CRON) INFO (pidfile fd = 3)
May 14 10:04:20 vagrant cron[686]: (CRON) INFO (Running @reboot jobs)
May 14 10:17:01 vagrant CRON[4111]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 10:55:31 vagrant cron[693]: (CRON) INFO (pidfile fd = 3)
May 14 10:55:31 vagrant cron[693]: (CRON) INFO (Running @reboot jobs)
May 14 11:17:01 vagrant CRON[5952]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 12:17:01 vagrant CRON[14364]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 13:17:01 vagrant CRON[25267]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 19:17:01 vagrant CRON[34912]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 20:17:01 vagrant CRON[50992]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 21:17:01 vagrant CRON[61934]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
May 14 22:15:01 vagrant CRON[70046]: (root) CMD (python3.9 /home/vagrant/py-scripts/my_script.py >> /home/vagrant/script.log)
```
- сгенерированный сертификат (время изменения по localtime)
```
vagrant@vagrant:/etc/nginx$ ls -al
total 80
drwxr-xr-x   8 root root 4096 May 14 23:34 .
drwxr-xr-x 100 root root 4096 May 15 01:06 ..
-rw-r--r--   1 root root 4245 May 15 01:15 cert.crt
drwxr-xr-x   2 root root 4096 Jul  6  2020 conf.d
-rw-r--r--   1 root root 1125 Jul  6  2020 fastcgi.conf
-rw-r--r--   1 root root 1055 Jul  6  2020 fastcgi_params
-rw-r--r--   1 root root 2837 Jul  6  2020 koi-utf
-rw-r--r--   1 root root 2223 Jul  6  2020 koi-win
-rw-r--r--   1 root root 3957 Jul  6  2020 mime.types
drwxr-xr-x   2 root root 4096 Jul  6  2020 modules-available
drwxr-xr-x   2 root root 4096 May 12 13:59 modules-enabled
-rw-r--r--   1 root root 1447 Jul  6  2020 nginx.conf
-rw-r--r--   1 root root  180 Jul  6  2020 proxy_params
-rw-r--r--   1 root root  636 Jul  6  2020 scgi_params
drwxr-xr-x   2 root root 4096 May 14 22:48 sites-available
drwxr-xr-x   2 root root 4096 May 12 22:53 sites-enabled
drwxr-xr-x   2 root root 4096 May 12 13:59 snippets
-rw-r--r--   1 root root  664 Jul  6  2020 uwsgi_params
-rw-r--r--   1 root root 3071 Jul  6  2020 win-utf
```
- лог работы скрипта (время создания по localtime)
```
vagrant@vagrant:~$ ls -al
total 168520
-rw-r--r-- 1 root    root         4339 May 15 01:15 script.log

```

```
vagrant@vagrant:~$ cat script.log
Success!
200
-----BEGIN RSA PRIVATE KEY-----
MIIEpgIBAAKCAQEAy+q4WkOvSx4hCWpXTSKGac2X8JNUXh6Z7poWJIoquD8DI/km
FWVqZSS3rtn7mD2JsC8qXm8A5zg9jkExvN9DoRkRVLR8EMCAEz8HVA3VxRtkKQPN
YKIjqVp19EsEeER74EACzzib+O4eCEmKBwFosHE6NVjh4Y/zlRK5K1h1y0fMSi7a
cg/DOQwFpnnExpp3DWrLoxmmCxAA8BJ5S4iCP5yq1T1FqPL9ptMXwJNsVujbontT
3rcodZVH939b5x9lt1N6JQ/InoCfZOgsSszi5RZqo4ObVG13mMZl0fiZzUUsJMdh
4Pp/lkclDFZPV7um0BqxOi8fwN2iYNps99WkmQIDAQABAoIBAQC4UuthbF0/oXRr
dlyDtPmbGeSF6icRDMhIDbCGRv5mnU7RbZ0nKg0KcC45N4FdFAMp5vh9dAl0+Dls
JTI4b/hJ3sbzslGTMu6Sdx7As+DP0vquuGklOGrT4plMwaLAs+blxfbEKX+TfGpZ
Q2ApBPUEyNS89CL6KY0c61fF/qg7noXMciMJPCrExnbGK9RCqw+040MuHPXQKCWg
l0UdLE+vJrusruCWfRs02+I0OG3MqyXTbHj7hX2b3knGpaiIzhU3kIfmkBUNlj5L
urzC73RpHvLKeuFzK/5EkTaNokhoOvbFPBtBIY2PvNStvguQAARAmlZFoYQuazBE
M3l0LaUhAoGBAOGxdDj6irBTPt4HbQVFJkIJ5m54Bjm0m9a/zTngbFKhvCvxUklv
tBcW1H4YctzlbLkabid4O4+aYeUmHdB+8fkouCGdssUF7eebiIsB53Tx3Y4YJqTS
9GjwD+fg0BdQ/slNa6DzGxnF7sOKcue8Lr0xNe+VXKyjYe7fqQBZIKAvAoGBAOdM
rC6kJ5xHmRwWyxY3gk1IcHNbZ+X4xkfzgfb48yEDPyO2EL+zZVYmYiKGGQPbcQ+y
QMTtu2kkXwZsFvpaSSiifrkf7SA7JbuL3eqnrBz7VPN1fVbUYsCp9ay4zbMU7Uzj
PQ1gzR18ORwCERFThekrJ7nL2pnEHomSiBrBu023AoGBALQo7ojJXspwOUANl3Zc
g7S1qG9Iele4F6pug0mPH4k6uDYvtisO8yh3PfrGU3KH5alBMuto68qvRzQfoC8e
FtR5L7sN4nWqTQXUP0QPqg7NP7m9B6QNDbNy5o+IAzSJoEla9u6NAXYqbXVEYIMh
0gGOdeqhRVA+P9/qREMV4phbAoGBAIY56HHosVPv+NpkH9b0v5k5wIgxM0ox5F88
4N8B+ZXe3HVM4stcwCIOhESkrT+fySzitNOTYn7RrJN9hpgibPI32nJ0JJffuzXr
JQ1X4HmxIMtoEySge+5ysy9Yz4ZtToigFn38/wttW3b1fBIh9kQp1xPrSSCSv2X2
syVT5+5hAoGBAKKoHufQzYLLi27WioODmfwVrSkOAnze36WP+MuscUJyUw6pjBNa
+S3Zeh8EVN3Wxjvxx1pTMP2byISYTsAKHELYPjdOkcKSjnSJjPImllK4w7timkQS
CCUrlvinzPlkejz+ZbWFFvluSZIa42gjSgku2rDZpHn4mWXlgazLKSv+
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIDZjCCAk6gAwIBAgIUbiNlVJ6+OJv+d5WoyMvDK9tTRPEwDQYJKoZIhvcNAQEL
BQAwLTErMCkGA1UEAxMiZXhhbXBsZS5jb20gSW50ZXJtZWRpYXRlIEF1dGhvcml0
eTAeFw0yMjA1MTQyMjE0MzJaFw0yMjA1MTQyMzE1MDFaMBsxGTAXBgNVBAMTEHRl
c3QuZXhhbXBsZS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDL
6rhaQ69LHiEJaldNIoZpzZfwk1ReHpnumhYkiiq4PwMj+SYVZWplJLeu2fuYPYmw
LypebwDnOD2OQTG830OhGRFUtHwQwIATPwdUDdXFG2QpA81goiOpWnX0SwR4RHvg
QALPOJv47h4ISYoHAWiwcTo1WOHhj/OVErkrWHXLR8xKLtpyD8M5DAWmecTGmncN
asujGaYLEADwEnlLiII/nKrVPUWo8v2m0xfAk2xW6Nuie1Petyh1lUf3f1vnH2W3
U3olD8iegJ9k6CxKzOLlFmqjg5tUbXeYxmXR+JnNRSwkx2Hg+n+WRyUMVk9Xu6bQ
GrE6Lx/A3aJg2mz31aSZAgMBAAGjgY8wgYwwDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUOuL6lBD3Gn1pwYUU
Mc+Clcl4F3gwHwYDVR0jBBgwFoAUXWk0XByfFVg5ySIrZ4p1ZdtdqfMwGwYDVR0R
BBQwEoIQdGVzdC5leGFtcGxlLmNvbTANBgkqhkiG9w0BAQsFAAOCAQEAIN+l904S
m6rsO2CbOKuMZzTEpO5BgE4NKieOQQ+FA74xV36RQdo9K6eg5nCOmpGyBr7bCoOy
438a0CLYShnA+Y1KavJMQE/AHc2u0wnVPMTG8iFldwO8jcme93dVev7Bbn0nXjjA
0GPI3fwCt3t2tE5PKU9eJPf7Ce179D9L8AzXn6evhZUtkaPct36oE7mqpCTL47vy
tvIxh7F4GIH9xGX25Ubh87/m+iwSyej3o07NJYl37J+6P18Py/wxltdf6n2lcj5m
Fw2PqzIYh0XnsaG1y34OQsgabbTwO1eQgqr0bzarbpn1grfuKWl+XAdo6kSnms+q
L+m1KIH2P+XAOg==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDpjCCAo6gAwIBAgIUTRsKAHigjwfh9xxnzxSUmC43MHIwDQYJKoZIhvcNAQEL
BQAwFjEUMBIGA1UEAxMLZXhhbXBsZS5jb20wHhcNMjIwNTE0MTk1NzA0WhcNMjIw
NjE0MTk1NzM0WjAtMSswKQYDVQQDEyJleGFtcGxlLmNvbSBJbnRlcm1lZGlhdGUg
QXV0aG9yaXR5MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAx9xEmkYA
nomIIFKAEmaOiBb4DcxwpixD+9T2ECvjzEHNPlTet3mca6dHtKJqLUxqTlsFr1YU
Qzharfv1dlcJmZue0Hd2250If+dl5nZoAOyDvs3JJbIidFDpUOZTgImqAgkxCeFR
9+NgqZwU6KiVXD08rFYkbRNK0kFu7l+un38fd4UsTX3/Vk1uyaou4y/Y9XDjM/YS
Xo0+xJtfV2JaS8pSccLyZE0yb0/iVTpD9GdHhur0vouz7fAw2tb9s+fWiSBehQ8y
UJoz+a15qDP8eowdrgy2QrKge2CNM+vbZqvHKNqM/elB+A67AF4a5YCLoOZTElCu
sX+EeMMdvUwWwwIDAQABo4HUMIHRMA4GA1UdDwEB/wQEAwIBBjAPBgNVHRMBAf8E
BTADAQH/MB0GA1UdDgQWBBRdaTRcHJ8VWDnJIitninVl212p8zAfBgNVHSMEGDAW
gBRn6TqWHjzSYCOrdnFq1DajUwP0ozA7BggrBgEFBQcBAQQvMC0wKwYIKwYBBQUH
MAKGH2h0dHA6Ly8xMjcuMC4wLjE6ODIwMC92MS9wa2kvY2EwMQYDVR0fBCowKDAm
oCSgIoYgaHR0cDovLzEyNy4wLjAuMTo4MjAwL3YxL3BraS9jcmwwDQYJKoZIhvcN
AQELBQADggEBACp44DEZZSyMkTTKYXBzjlkVb4veqhi+mfsxwQm3UrrIlsDZjgWG
bxJPiY6SrKTeyF6hRjoQ00Q0ufHSH40APEzaOE6ix+SDIg9Jyev/hwURhbCSIFwf
sn2ozYXcE2y2xOm78NRYQjwn5QYNYTeZayYX1hIsbdrOroWdi/ZSVtk4v6OZKH4v
0tGo8TaRRFF+Tno5/BjAxmZPFq0IH0gqC8uLk9h7Copn2AwC4xEUcWvlrC7gA3y7
T/tiQ6Lbu2oC3vi62gc8N8BEKSPSvPLfl9xCQHCz7g9J+djVO/i88NpZPLy+VUC2
fK7fc4k/VTafIhflCvPD4RdluYHQJa/vnXU=
-----END CERTIFICATE-----
0
Обновление сертификата завершено успешно
```