# Wonderland
---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.130.182.151
```

![](images/Pasted%20image%2020260620183758.png)

One of the web directories contains an interesting image, so let's download it.

```bash
wget http://MACHINE_IP/img/white_rabbit_1.jpg
```

![](images/Pasted%20image%2020260620184148.png)

JPGs are always a little suspicious in CTFs, so it's worth checking whether anything is hidden inside.

We can use **steghide** to extract embedded files, assuming they aren't password protected.

```bash
steghide extract -sf white_rabbit_1.png
```

The extraction gives us the following clue:

> **follow the r a b b i t**

Browsing to:

`http://MACHINE_IP/r/a/b/b/i/t`

reveals another interesting page.

After viewing the page source (**Ctrl + U**), we discover what appears to be a set of credentials.

- **User:** alice
    
- **Password:** HowDothTheLittleCrocodileImproveHisShiningTail
    

![](images/Pasted%20image%2020260620185109.png)

Let's see if they work over SSH.

```bash
ssh alice@MACHINE_IP
```

They do.

---

## Foothold

Running `sudo -l` immediately reveals something interesting.

Alice can execute a Python script without providing a password, but the script runs as **rabbit**.

![](images/Pasted%20image%2020260620185624.png)

Looking through the script, we notice that it imports the **random** module without using an absolute path.

That means Python will first search the current directory before loading the standard library module.

We can abuse this by creating our own **random.py** in the same directory.

Instead of implementing the actual module, our file simply executes **/bin/bash**.

Since the script itself runs as **rabbit**, our payload is executed with Rabbit's privileges, giving us a shell as that user.

---

## Privilege Escalation

After getting access as **rabbit**, we continue enumerating the system.

Inside Rabbit's home directory, we find an executable called **teaParty**.

After downloading it for a quick look, we inspect the strings inside the binary.

```bash
strings teaParty | awk 'length($0) > 10'
```

![](images/Pasted%20image%2020260620190424.png)

One thing immediately stands out.

The program executes **date**, but once again it doesn't use an absolute path.

Just like before, we can hijack the command by creating our own executable named **date** earlier in the `PATH`.

When the binary calls `date`, it executes our payload instead, giving us a shell as **hatter**.

While enumerating Hatter's environment, we discover that **perl 5.26.1** is running with the **SUID** bit set.

A quick search on **GTFOBins** shows exactly what we need.

[https://gtfobins.org/gtfobins/perl/](https://gtfobins.org/gtfobins/perl/)

![](images/Pasted%20image%2020260620190821.png)

Using the GTFOBins technique, we escalate to **root**.

All that's left is retrieving the root flag.

---

## Conclusion

A really enjoyable room with a nice progression. The privilege escalation path is built around abusing executables that don't use absolute paths, making it a great reminder of why that's considered a bad practice. The final SUID Perl escalation is straightforward if you're familiar with GTFOBins, making for a satisfying finish.

**— 0xchnk**