---
title: DNS sinkhole
tags: ['Node.js', 'Typescript']
categories: ['實作']
date: 2023-02-09 20:00:00
description: DNS sinkhole是一種特殊的DNS伺服器，對黑名單內的域名查詢給出無效結果，以此達到阻擋特定網域的目的，本文將以Typescript進行實作。
---
## 前言
DNS沉洞(DNS sinkhole)是一種特殊的DNS伺服器，對黑名單內的域名查詢給出無效結果(通常為NXDOMAIN或0.0.0.0)，以此達到阻擋特定網域的目的，本文將以Typescript進行實作。

## 封包結構
DNS封包的結構可以分成五部分而這邊會用到的只有前三個區塊，分別為儲存查詢或回覆型態的**Header**(標頭區段)、儲存查詢內容的**Question**(問題區段)以及DNS回覆的**Answer**(答案區段)。

![](./dns-packet.png)

### 標頭區段
- ID (16 bits): 識別碼，客戶端將會檢視伺服器回傳的此號碼辨識是否為己方發出之訊息。
- Flag (16 bits): 
  - QR (1 bit): Question or Response 0表示查詢封包、1則為回應封包。
  - OPCODE (4 bits): Operation Code 運作模式，0表示標準查詢(standard query)、1表示反向查詢(reversed query)、2表示狀態查詢(server status request)，一般常用標準查詢。
  - AA (1 bit): Authoritative Answer 授權答案，在回應封包中表示查詢資料是否由管轄此域名之名稱伺服器發出。
  - TC (1 bit): Truncation 截短，若此位元為1代表回應因過長被截短。
  - RD (1 bit): Recursive Desired 是否期望遞迴查詢。
  - RA (1 bit): Recursive Available 回應此伺服器是否支援遞迴。
  - Z (3 bits): 保留位元，全0。
  - RCODE (4 bits): Response Code 回應碼。0 無錯誤、1 格式錯誤、2 伺服器錯誤、3 名稱錯誤(權威名稱伺服器回應域名不存在NXDOMAIN)、4 不支援OPCODE類型、5 因政策等原因拒絕查詢
- QDCOUNT (16 bits): 問題數，DNS伺服器軟體應該拒絕回應任何問題數不等於1的請求。
- ANCOUNT (16 bits): 回應封包的Resource Record數量。
- NSCOUNT (16 bits): 授權區段的Resource Record數量。
- ARCOUNT (16 bits): 額外的Resource Record數量。

```ts ./src/interfaces/Flag.ts
interface Flag {
  QR: boolean;
  OPCode: number;
  AA: boolean;
  TC: boolean;
  RD: boolean;
  RA: boolean;
  RCode: number;
};
```
```ts ./src/interfaces/DnsHeader.ts
interface DnsHeader {
  ID: number;
  Flag: Flag;
  QuestionCount: number;
  RRCount: number;
  AuthRRCount: number;
  AddRRCount: number;
};
```

對於DNS sinkhole來說解析DNS標頭主要有兩個值需要注意，回應時將RCODE設定為2(NXDOMAIN)以及由QDCOUNT判斷封包是否要進一步處理。
```ts ./src/modules/dnsParser.ts
const flag: Flag = {
  QR: Boolean(msg.readUintBE(2, 2) >> 15),
  OPCode: (msg.readUintBE(2, 2) >> 11) & ((1 << 4) - 1),
  AA: Boolean((msg.readUintBE(2, 2) >> 10) & 1),
  TC: Boolean((msg.readUintBE(2, 2) >> 9) & 1),
  RD: Boolean((msg.readUintBE(2, 2) >> 8) & 1),
  RA: Boolean((msg.readUintBE(2, 2) >> 7) & 1),
  RCode: msg.readUintBE(2, 2) & ((1 << 5) - 1)
};

const header: DnsHeader = {
  ID: msg.readUIntBE(0, 2),
  Flag: flag,
  QuestionCount: msg.readUintBE(4, 2),
  RRCount: msg.readUintBE(6, 2),
  AuthRRCount: msg.readUintBE(8, 2),
  AddRRCount: msg.readUintBE(10, 2)
};
```

![](./packet-header.png)

### 問題區段
- QNAME: Question Name 欲查詢的域名，將域名以句號做分割在每一段前面加上此段字串之長度，並以0作為結尾，整體內容不含句號字元。
以`google.com`為例
| 6 | g | o | o | g | l | e | 3 | c | o | m | 0 |
|---|---|---|---|---|---|---|---|---|---|---|---|
- QTYPE: Question Type 問題型態，常見的有A(IP位址)、AAAA(IPv6)、NS(名稱伺服器)、MX(郵件伺服器)、PTR(反向查詢網域名稱)等。
- QCLASS: 一般情況下均為1(IN)

```ts ./src/interfaces/DnsQuery.ts
interface DnsQuery {
  Name: string;
  Type: string;
  Class: number;
};
```

```ts ./src/modules/dnsParser.ts
let domain = "";
let index = 12;
let subStringLength = msg.readUint8(index++);

while(subStringLength != 0) {
  for(let i = 0; i < subStringLength; i++) {
    domain = domain.padEnd(domain.length + 1, String.fromCharCode(msg.readUInt8(index++)));
  }
  subStringLength = msg.readUInt8(index++);
  if(subStringLength != 0) {
    domain = domain.padEnd(domain.length + 1, ".");
  }
}

const query: DnsQuery = {
  Name: domain,
  Type: ResourceRecord[msg.readUIntBE(index, 2)],
  Class: msg.readUIntBE(index + 2, 2)
};
```

![](./packet-question.png)

## 黑/白名單&沉洞
在接收到DNS封包後，會先檢查Header，若Header中的問題數不為1，將會回傳Format error。

```ts ./src/modules/dnsHandler.ts
let domainIsSafe: boolean = true;
const Header = HeaderParser(msg);
let Query: DnsQuery;
if(Header.QuestionCount != 1) {
  const errorResBuff = DnsSinker(msg, 1);

  server.send(errorResBuff, rinfo.port, rinfo.address);
}
```

取得Query內的域名後，先對其與較短的白名單內的域名比對，若沒有符合的結果才會進行下一步黑名單的比對，以節省比對時間且較不會出現因為誤ban而導致部分網站服務無法使用的情形。

```ts ./src/modules/dnsHandler.ts
Query = QueryParser(msg);
if(!server.WhiteList.has(Query.Name)) {
  if(server.BlackList.has(Query.Name)) {
    domainIsSafe = false;
  }
}
```

通過黑名單檢查的域名將直接回傳dns.google的請求結果，未通過者則回傳NXDOMAIN。

```ts ./src/modules/dnsHandler.ts
if(domainIsSafe) {
  const dns = dgram.createSocket("udp4");
  dns.on("message", (recvMsg, remoteInfo) => {
    server.send(recvMsg, rinfo.port, rinfo.address);
  });
  dns.send(msg, 53, "8.8.8.8");
}
else {
  const sinkMsg = DnsSinker(msg, 3);
  server.send(sinkMsg, rinfo.port, rinfo.address);
}
```

nslookup的測試結果
![](./demo-block.png)

## Repo
[pierre0210/dns-sinkhole](https://github.com/pierre0210/dns-sinkhole)

## 參考資料
- [RFC1035](https://www.rfc-editor.org/rfc/rfc1035)
- [Fundamentals of Computer Networking  Project 1: Simple DNS Client](https://mislove.org/teaching/cs4700/spring11/handouts/project1-primer.pdf)
- [TCP/IP 與 Internet 網路：第十三章 網域名稱系統 13-6 DNS 訊息格式](http://www.tsnien.idv.tw/Internet_WebBook/chap13/13-6%20DNS%20%E8%A8%8A%E6%81%AF%E6%A0%BC%E5%BC%8F.html)
- [What does QD stand for in DNS RFC1035](https://stackoverflow.com/questions/32031349/what-does-qd-stand-for-in-dns-rfc1035)
