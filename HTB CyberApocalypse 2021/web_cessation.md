# Web - Cessation
This challenge was pretty straightforward. It had a single downloadable file which was called `remap.config` and had the following contents: 
```
regex_map http://.*/shutdown http://127.0.0.1/403
regex_map http://.*/ http://127.0.0.1/
```
The homepage didn't have anything interesting on it, so it seems like the only path to exploitation was to somehow access the shutdown URL. 

Accessing it directly didn't work due to the regex, but accessing `http://<instance ip>//shutdown` (with two slashes) showed a page with the flag.