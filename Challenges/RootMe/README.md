We start with basic enumeration:

```bash
nmap -sV -sC 10.128.169.170
```

![](images/Pasted%20image%2020260626181428.png)

busting directories:

```bash
gobuster dir -u 10.128.169.170 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](images/Pasted%20image%2020260626181534.png)

Since php files are not allowed, then we upload a php file as an phtml (**the method will work as long as the server checks extensions only and not content!**)

![](images/Pasted%20image%2020260626182455.png)


![](images/Pasted%20image%2020260626182510.png)

Once we open the file or curl it in case we do not have UI access, we get a shell

![](images/Pasted%20image%2020260626182624.png)

Now we check directories and files and what permissions we have on them, cronjobs, suid, listening ports,...etc

```bash 
python -m SimpleHTTPServer 8010
```
![](images/Pasted%20image%2020260626183643.png)

Nothing here, we move on;
![](images/Pasted%20image%2020260626183918.png)

First flag retrieved!
Now we try escalating our privileges to root or to a super user with root permissions if possible:

When looking for files with SUID, **python** stands out and we directly check the GTFObins website

https://gtfobins.org/gtfobins/python/

![](images/Pasted%20image%2020260626184647.png)

And it is this simple:

![](images/Pasted%20image%2020260626190158.png)

See ya!

**— 0xchnk**