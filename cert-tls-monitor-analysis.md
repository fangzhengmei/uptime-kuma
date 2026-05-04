# Uptime Kuma HTTPS 证书、域名过期和 TLS 元数据监控流程分析

## 目录

1. [概述](#1-概述)
2. [证书采集与解析](#2-证书采集与解析)
3. [证书信息保存](#3-证书信息保存)
4. [域名过期监控](#4-域名过期监控)
5. [告警通知机制](#5-告警通知机制)
6. [前端展示](#6-前端展示)
7. [告警状态边界关系分析](#7-告警状态边界关系分析)
8. [端到端时序说明](#8-端到端时序说明)
9. [关键代码位置汇总](#9-关键代码位置汇总)
10. [关键技术点](#10-关键技术点)
11. [配置与扩展](#11-配置与扩展)

---

## 1. 概述

Uptime Kuma 对 HTTPS 证书、域名过期和 TLS 元数据的监控是一个完整的闭环流程，包括：

- **采集**：通过 TLS 连接获取证书信息
- **解析**：提取证书有效期、指纹、颁发者等元数据
- **保存**：将解析后的信息存储到数据库
- **告警**：根据剩余天数触发通知
- **展示**：在前端页面实时显示剩余天数和告警状态

> **重要区分**：通知发送、页面展示、徽章展示是三套**独立的机制**，使用不同的阈值和判断逻辑。详见 [第7章](#7-告警状态边界关系分析)。

---

## 2. 证书采集与解析

### 2.1 核心工具函数

证书采集和解析的核心逻辑位于 `server/util-server.js` 中。

#### 2.1.1 `checkCertificate(socket)` 函数

**位置**：`server/util-server.js:498-520`

**功能**：从 TLS Socket 中提取证书信息

```javascript
exports.checkCertificate = function (socket) {
    let certInfoStartTime = dayjs().valueOf();

    if (socket === undefined || socket == null) {
        return null;
    }

    // 获取对端证书（包含完整证书链）
    const info = socket.getPeerCertificate(true);
    const valid = socket.authorized || false;

    // 解析证书信息
    const parsedInfo = parseCertificateInfo(info);

    return {
        valid: valid,
        certInfo: parsedInfo,
    };
};
```

**关键步骤**：
1. 使用 `socket.getPeerCertificate(true)` 获取完整证书链
2. 通过 `socket.authorized` 判断证书是否有效
3. 调用 `parseCertificateInfo` 解析证书元数据

#### 2.1.2 `parseCertificateInfo(info)` 函数

**位置**：`server/util-server.js:450-491`

**功能**：解析证书原始数据，计算剩余天数，识别证书类型

```javascript
const parseCertificateInfo = function (info) {
    let link = info;
    let i = 0;
    const existingList = {};

    while (link) {
        if (!link.valid_from || !link.valid_to) {
            break;
        }
        
        // 转换有效期格式
        link.validTo = new Date(link.valid_to);
        
        // 提取可选域名（SAN）
        link.validFor = link.subjectaltname?.replace(/DNS:|IP Address:/g, "").split(", ");
        
        // 计算剩余天数（UTC时间）
        link.daysRemaining = dayjs.utc(link.validTo).diff(dayjs.utc(), "day");

        existingList[link.fingerprint] = true;

        // 识别证书类型
        if (link.issuerCertificate == null) {
            link.certType = i === 0 ? "self-signed" : "root CA";
            break;
        } else if (link.issuerCertificate.fingerprint in existingList) {
            // 根CA证书通常自签名
            link.certType = i === 0 ? "self-signed" : "root CA";
            link.issuerCertificate = null;
            break;
        } else {
            link.certType = i === 0 ? "server" : "intermediate CA";
            link = link.issuerCertificate;
        }
        
        // 防止死循环
        if (i > 500) {
            throw new Error("Dead loop occurred in parseCertificateInfo");
        }
        i++;
    }

    return info;
};
```

**解析的证书类型**：
| 类型 | 说明 |
|------|------|
| `server` | 服务器证书（链中的第一个证书） |
| `intermediate CA` | 中间 CA 证书 |
| `root CA` | 根 CA 证书 |
| `self-signed` | 自签名证书 |

#### 2.1.3 `getDaysRemaining(validFrom, validTo)` 函数

**位置**：`server/util-server.js:435-441`

**功能**：计算两个日期之间的剩余天数，支持负数表示已过期

```javascript
const getDaysRemaining = (validFrom, validTo) => {
    const daysRemaining = getDaysBetween(validFrom, validTo);
    if (new Date(validTo).getTime() < new Date().getTime()) {
        return -daysRemaining;  // 已过期返回负数
    }
    return daysRemaining;
};
```

### 2.2 不同监控类型的证书采集方式

#### 2.2.1 HTTP/HTTPS 监控

**位置**：`server/model/monitor.js:615-659`

**采集方式**：通过 Axios 的 HTTPS Agent 监听 TLS 事件

```javascript
// 方式1：通过 keylog 事件监听（主要方式）
options.httpsAgent.once("keylog", async (line, tlsSocket) => {
    tlsSocket.once("secureConnect", async () => {
        tlsInfo = checkCertificate(tlsSocket);
        tlsInfo.valid = tlsSocket.authorized || false;
        // 检查证书主机名匹配
        tlsInfo.hostnameMatchMonitorUrl = checkCertificateHostname(
            tlsInfo.certInfo.raw,
            this.getUrl()?.hostname
        );
        await this.handleTlsInfo(tlsInfo);
    });
});

// 方式2：备用方式（通过响应 socket）
if (this.getUrl()?.protocol === "https:" && tlsInfo.valid === undefined) {
    const tlsSocket = res.request.res.socket;
    if (tlsSocket) {
        tlsInfo = checkCertificate(tlsSocket);
        // ... 同样的处理逻辑
        await this.handleTlsInfo(tlsInfo);
    }
}
```

#### 2.2.2 TCP 端口监控（支持 TLS）

**位置**：`server/monitor-types/tcp.js:236-279`

**采集方式**：直接使用 Node.js 的 `tls.connect()` 建立 TLS 连接

```javascript
async checkTlsCertificate(monitor, reuseSocket) {
    let socket = null;
    try {
        const options = {
            host: monitor.hostname,
            port: monitor.port,
            servername: monitor.hostname,  // SNI 支持
            ...reuseSocket,
        };

        const tlsInfoObject = await new Promise((resolve, reject) => {
            socket = tls.connect(options);

            socket.on("secureConnect", () => {
                const info = checkCertificate(socket);
                resolve(info);
            });

            socket.on("error", (error) => {
                reject(error);
            });
        });

        await monitor.handleTlsInfo(tlsInfoObject);
        
        if (!tlsInfoObject.valid) {
            throw new Error("Certificate is invalid");
        }
    } finally {
        if (socket && !socket.destroyed) {
            socket.end();
        }
    }
}
```

#### 2.2.3 STARTTLS 协议支持

**位置**：`server/monitor-types/tcp.js:154-228`

**支持的协议**：SMTP、IMAP、XMPP

**工作流程**：
1. 先建立普通 TCP 连接
2. 发送协议特定的 STARTTLS 命令
3. 等待服务器响应
4. 升级到 TLS 连接
5. 采集证书信息

```javascript
performStartTls(monitor) {
    return new Promise((resolve, reject) => {
        const socket_ = net.connect(monitor.port, monitor.hostname);

        socket_.on("data", (data) => {
            const response = data.toString();
            const response_ = response.toLowerCase();
            
            switch (true) {
                // SMTP: 250-STARTTLS -> STARTTLS
                case response.startsWith("220") || response.includes("ESMTP"):
                    socket_.write(`EHLO ${monitor.hostname}\r\n`);
                    break;
                case response.includes("250-STARTTLS"):
                    socket_.write("STARTTLS\r\n");
                    break;
                case response_.includes("start tls") || response_.includes("begin tls"):
                    doResolve();  // 可以升级了
                    break;
                // IMAP: CAPABILITY STARTTLS -> a001 STARTTLS
                case response.startsWith("* OK") || response.match(/CAPABILITY.+STARTTLS/):
                    socket_.write("a001 STARTTLS\r\n");
                    break;
                // XMPP: <starttls> -> <proceed>
                case response_.includes("<starttls"):
                    socket_.write('<starttls xmlns="urn:ietf:params:xml:ns:xmpp-tls"/>');
                    break;
                case response_.includes("<proceed"):
                    doResolve();
                    break;
            }
        });
    });
}
```

### 2.3 证书主机名匹配检查

**位置**：`server/util-server.js:529-544`

**功能**：验证证书的域名是否与监控目标匹配

```javascript
exports.checkCertificateHostname = function (certBuffer, hostname) {
    let X509Certificate;
    try {
        X509Certificate = require("node:crypto").X509Certificate;
    } catch (_) {
        // Node.js 版本不支持，默认通过
        return true;
    }

    if (!X509Certificate || !certBuffer || !hostname) {
        return true;
    }

    let certObject = new X509Certificate(certBuffer);
    // checkHost 返回匹配的名称，undefined 表示不匹配
    return certObject.checkHost(hostname) !== undefined;
};
```

---

## 3. 证书信息保存

### 3.1 数据库表结构

**位置**：`db/knex_init_db.js:358-370`

```javascript
await knex.schema.createTable("monitor_tls_info", (table) => {
    table.increments("id");
    table
        .integer("monitor_id")
        .unsigned()
        .notNullable()
        .references("id")
        .inTable("monitor")
        .onDelete("CASCADE")
        .onUpdate("CASCADE");
    table.text("info_json");  // JSON 格式存储完整证书信息
});
```

### 3.2 保存逻辑

#### 3.2.1 `handleTlsInfo(tlsInfo)` 函数

**位置**：`server/model/monitor.js:2096-2104`

**功能**：统一处理 TLS 信息的入口函数

```javascript
async handleTlsInfo(tlsInfo) {
    // 1. 保存到数据库
    await this.updateTlsInfo(tlsInfo);
    
    // 2. 更新 Prometheus 指标
    this.prometheus?.update(null, tlsInfo, null);

    // 3. 检查是否需要发送过期告警
    if (!this.getIgnoreTls() && this.isEnabledExpiryNotification()) {
        await checkCertExpiryNotifications(this, tlsInfo);
    }
}
```

#### 3.2.2 `updateTlsInfo(checkCertificateResult)` 函数

**位置**：`server/model/monitor.js:1292-1328`

**功能**：将证书信息保存到数据库，处理证书变更逻辑

```javascript
async updateTlsInfo(checkCertificateResult) {
    let tlsInfoBean = await R.findOne("monitor_tls_info", "monitor_id = ?", [this.id]);

    if (tlsInfoBean == null) {
        // 新建记录
        tlsInfoBean = R.dispense("monitor_tls_info");
        tlsInfoBean.monitor_id = this.id;
    } else {
        // 检查证书是否变更（通过指纹对比）
        try {
            let oldCertInfo = JSON.parse(tlsInfoBean.info_json);
            let isValidObjects =
                oldCertInfo && oldCertInfo.certInfo && 
                checkCertificateResult && checkCertificateResult.certInfo;

            if (isValidObjects) {
                if (oldCertInfo.certInfo.fingerprint256 !== 
                    checkCertificateResult.certInfo.fingerprint256) {
                    // 证书已变更，清除之前的通知发送记录
                    log.debug("monitor", "Resetting sent_history");
                    await R.exec(
                        "DELETE FROM notification_sent_history WHERE type = 'certificate' AND monitor_id = ?",
                        [this.id]
                    );
                }
            }
        } catch (e) {}
    }

    // 保存 JSON 格式的证书信息
    tlsInfoBean.info_json = JSON.stringify(checkCertificateResult);
    await R.store(tlsInfoBean);

    return checkCertificateResult;
}
```

### 3.3 证书信息 JSON 结构

保存到 `info_json` 字段的数据结构示例：

```json
{
    "valid": true,
    "hostnameMatchMonitorUrl": true,
    "certInfo": {
        "subject": {
            "CN": "example.com",
            "O": "Example Inc",
            "C": "US"
        },
        "issuer": {
            "CN": "Let's Encrypt Authority X3",
            "O": "Let's Encrypt"
        },
        "valid_from": "Jan 1 00:00:00 2024 GMT",
        "valid_to": "Dec 31 23:59:59 2024 GMT",
        "validTo": "2024-12-31T23:59:59.000Z",
        "daysRemaining": 180,
        "certType": "server",
        "fingerprint": "AA:BB:CC:...",
        "fingerprint256": "XX:YY:ZZ:...",
        "serialNumber": "1234567890abcdef",
        "subjectaltname": "DNS:example.com, DNS:www.example.com",
        "validFor": ["example.com", "www.example.com"],
        "raw": "<Buffer ...>",
        "issuerCertificate": {
            "certType": "intermediate CA",
            "daysRemaining": 365,
            "issuerCertificate": {
                "certType": "root CA",
                "daysRemaining": 1000,
                "issuerCertificate": null
            }
        }
    }
}
```

---

## 4. 域名过期监控

### 4.1 概述

域名过期监控使用 **RDAP (Registration Data Access Protocol)** 从域名注册商获取过期信息。相比传统的 WHOIS 协议，RDAP 提供了标准化的 JSON API。

### 4.2 核心类：DomainExpiry

**位置**：`server/model/domain_expiry.js`

#### 4.2.1 数据库表结构

```javascript
// 隐式使用 redbean-node 的 Bean 模式
// 表: domain_expiry
// 字段: id, domain, expiry, lastCheck, lastExpiryNotificationSent
```

#### 4.2.2 RDAP 服务器发现机制

**位置**：`server/model/domain_expiry.js:20-33`

```javascript
async function getRdapServer(tld) {
    const rdapDnsData = await getRdapDnsData();
    const services = rdapDnsData["services"] ?? [];
    const rootTld = tld?.split(".").pop();
    
    if (rootTld) {
        for (const [tlds, urls] of services) {
            if (tlds.includes(rootTld)) {
                return urls[0];  // 返回第一个可用的 RDAP 服务器
            }
        }
    }
    return null;
}
```

#### 4.2.3 IANA RDAP DNS 数据获取

**位置**：`server/model/domain_expiry.js:39-82`

```javascript
async function getRdapDnsData() {
    // 缓存一周
    if (cacheRdapDnsData && Date.now() < nextChecking) {
        return cacheRdapDnsData;
    }

    try {
        running = true;
        log.info("rdap", "Updating RDAP DNS data from IANA...");
        
        // 从 IANA 获取 RDAP 服务器列表
        const response = await fetch("https://data.iana.org/rdap/dns.json");
        const data = await response.json();

        cacheRdapDnsData = data;
        nextChecking = Date.now() + 7 * 24 * 60 * 60 * 1000;  // 一周后更新
        await Settings.set("rdapDnsData", data);
    } catch (error) {
        // 失败时使用本地缓存或内置数据
        cacheRdapDnsData = await getOfflineRdapDnsData();
        nextChecking = Date.now() + 24 * 60 * 60 * 1000;  // 一天后重试
    }

    running = false;
    return cacheRdapDnsData;
}
```

#### 4.2.4 查询域名过期日期

**位置**：`server/model/domain_expiry.js:110-140`

```javascript
async function getRdapDomainExpiryDate(domain) {
    const tld = DomainExpiry.parseTld(domain).publicSuffix;
    const rdapServer = await getRdapServer(tld);
    
    if (rdapServer === null) {
        log.warn("rdap", `No RDAP server found, TLD ${tld} not supported.`);
        return null;
    }
    
    const url = `${rdapServer}domain/${domain}`;

    try {
        const res = await fetch(url);
        if (res.status !== 200) {
            return null;
        }
        const rdapInfos = await res.json();

        // 在 events 数组中查找过期事件
        if (rdapInfos["events"] === undefined) {
            return null;
        }
        for (const event of rdapInfos["events"]) {
            if (event["eventAction"] === "expiration") {
                return new Date(event["eventDate"]);
            }
        }
        return null;
    } catch {
        log.warn("rdap", "Not able to get expiry date from RDAP");
        return null;
    }
}
```

#### 4.2.5 检查域名过期（带缓存）

**位置**：`server/model/domain_expiry.js:278-302`

```javascript
static async checkExpiry(domainName) {
    let bean = await DomainExpiry.findByDomainNameOrCreate(domainName);
    let expiryDate;

    // 缓存检查：24小时内不重复查询
    if (bean?.lastCheck && 
        dayjs.utc().diff(dayjs.utc(bean.lastCheck), "day") < 1) {
        log.debug("domain_expiry", 
            `Domain expiry already checked recently for ${bean.domain}, won't re-check.`);
        return bean.expiry;
    } else if (bean) {
        // 查询 RDAP 获取新的过期日期
        expiryDate = await bean.getExpiryDate();

        // 如果过期日期延后了，说明域名已续费，清除通知记录
        if (dayjs.utc(expiryDate).isAfter(dayjs.utc(bean.expiry))) {
            bean.lastExpiryNotificationSent = null;
        }

        // 更新数据库
        bean.expiry = R.isoDateTimeMillis(expiryDate);
        bean.lastCheck = R.isoDateTimeMillis(dayjs.utc());
        await R.store(bean);
    }

    if (expiryDate === null) {
        return;
    }
    return expiryDate;
}
```

#### 4.2.6 计算剩余天数

**位置**：`server/model/domain_expiry.js:262-264`

```javascript
get daysRemaining() {
    return dayjs.utc(this.expiry).diff(dayjs.utc(), "day");
}
```

### 4.3 支持的监控类型

**位置**：`src/util.ts:776-794` 中定义的 `TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD`

```javascript
export const TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD = {
    http: "url",
    keyword: "url",
    "json-query": "url",
    "real-browser": "url",
    "websocket-upgrade": "url",
    port: "hostname",
    ping: "hostname",
    "grpc-keyword": "grpcUrl",
    dns: "hostname",
    smtp: "hostname",
    snmp: "hostname",
    gamedig: "hostname",
    steam: "hostname",
    mqtt: "hostname",
    radius: "hostname",
    "tailscale-ping": "hostname",
    "sip-options": "hostname",
} as const;
```

**支持的类型和对应字段**：

| 字段 | 监控类型 |
|------|----------|
| `url` | http, keyword, json-query, real-browser, websocket-upgrade |
| `hostname` | port, ping, dns, smtp, snmp, gamedig, steam, mqtt, radius, tailscale-ping, sip-options |
| `grpcUrl` | grpc-keyword |

### 4.3.1 `checkSupport` 完整逻辑

**位置**：`server/model/domain_expiry.js:211-245`

```javascript
static async checkSupport(monitor) {
    // 检查 1: 监控类型是否在支持列表中
    if (!(monitor.type in TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD)) {
        throw new TranslatableError("domain_expiry_unsupported_monitor_type");
    }
    
    // 检查 2: 目标字段是否有值
    const targetField = TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD[monitor.type];
    const target = monitor[targetField];
    if (typeof target !== "string" || target.length === 0) {
        throw new TranslatableError("domain_expiry_unsupported_missing_target");
    }
    
    // 检查 3: 是否是 ICANN 域名
    const tld = parseTld(target);
    if (!tld.isIcann) {
        throw new TranslatableError("domain_expiry_unsupported_is_icann", {...});
    }
    
    // 检查 4: 是否有 RDAP 服务器
    const publicSuffix = tld.publicSuffix;
    const rdap = await getRdapServer(publicSuffix);
    if (!rdap) {
        throw new TranslatableError("domain_expiry_unsupported_unsupported_tld_no_rdap_endpoint", {...});
    }
    
    return {
        domain: tld.domain,
        tld: rootTld,
    };
}
```

**注意**：`sendDomainInfo` 会静默捕获 `checkSupport` 抛出的异常，所以即使类型不支持，也不会报错，只是不会推送数据。

### 4.4 监控触发时机

**位置**：`server/model/monitor.js:1040-1063`

域名过期检查在每次心跳成功后触发（如果启用了域名过期通知）：

```javascript
if (bean.status !== MAINTENANCE && Boolean(this.domainExpiryNotification)) {
    try {
        // 检查是否支持域名过期监控
        const supportInfo = await DomainExpiry.checkSupport(this);
        
        // 获取/更新过期日期
        const domainExpiryDate = await DomainExpiry.checkExpiry(supportInfo.domain);
        
        if (domainExpiryDate) {
            // 检查是否需要发送通知
            DomainExpiry.sendNotifications(
                supportInfo.domain,
                (await Monitor.getNotificationList(this)) || []
            );
        }
    } catch (error) {
        // 处理不支持的 TLD 等情况
        if (error.message === "domain_expiry_unsupported_unsupported_tld_no_rdap_endpoint") {
            log.warn("domain_expiry", 
                `Domain expiry unsupported for '.${error.meta.publicSuffix}' because its RDAP endpoint is not listed in the IANA database.`);
        }
    }
}
```

> **重要**：`domainExpiryNotification` 只影响**是否发送通知**和**是否执行 RDAP 查询**，不影响数据推送到前端（见 [第7章](#7-告警状态边界关系分析)）。

---

## 5. 告警通知机制

### 5.1 证书过期告警

#### 5.1.1 核心函数：`checkCertExpiryNotifications`

**位置**：`server/util-server.js:917-972`

```javascript
async function checkCertExpiryNotifications(monitor, tlsInfoObject) {
    // 基本校验
    if (!tlsInfoObject || !tlsInfoObject.certInfo || !tlsInfoObject.certInfo.daysRemaining) {
        return;
    }

    // 获取该监控配置的通知方式
    let notificationList = await R.getAll(
        "SELECT notification.* FROM notification, monitor_notification WHERE monitor_id = ? AND monitor_notification.notification_id = notification.id ",
        [monitor.id]
    );

    if (!notificationList.length > 0) {
        log.debug("monitor", "No notification, no need to send cert notification");
        return;
    }

    // 获取告警天数配置（默认: 7, 14, 21 天）
    let notifyDays = await Settings.get("tlsExpiryNotifyDays");
    if (notifyDays == null || !Array.isArray(notifyDays)) {
        await Settings.set("tlsExpiryNotifyDays", [7, 14, 21], "general");
        notifyDays = [7, 14, 21];
    }

    // 遍历每个告警天数
    // 注意：证书过期检查没有对 notifyDays 进行排序！
    // 对比：域名过期检查明确进行了升序排序
    for (const targetDays of notifyDays) {
        let certInfo = tlsInfoObject.certInfo;
        
        // 遍历证书链中的每个证书（服务器证书、中间CA、根CA）
        while (certInfo) {
            let subjectCN = certInfo.subject["CN"];
            
            // 跳过已知的根证书（内置根证书库）
            if (monitor.rootCertificates.has(certInfo.fingerprint256)) {
                log.debug("monitor", 
                    `Known root cert: ${certInfo.certType} certificate "${subjectCN}" (${certInfo.daysRemaining} days valid) on ${targetDays} deadline.`);
                break;
            } 
            // 剩余天数大于目标天数，无需告警
            else if (certInfo.daysRemaining > targetDays) {
                log.debug("monitor", 
                    `No need to send cert notification for ${certInfo.certType} certificate "${subjectCN}" (${certInfo.daysRemaining} days valid) on ${targetDays} deadline.`);
            } 
            // 需要发送告警
            else {
                log.debug("monitor", 
                    `call sendCertNotificationByTargetDays for ${targetDays} deadline on certificate ${subjectCN}.`);
                await monitor.sendCertNotificationByTargetDays(
                    subjectCN,
                    certInfo.certType,
                    certInfo.daysRemaining,
                    targetDays,
                    notificationList
                );
            }
            
            // 检查证书链中的下一个证书
            certInfo = certInfo.issuerCertificate;
        }
    }
}
```

#### 5.1.2 根证书指纹库

**位置**：`server/util-server.js:798-823`

系统内置了 Node.js 信任的根证书指纹库，用于跳过对这些已知根证书的告警：

```javascript
module.exports.rootCertificatesFingerprints = () => {
    let fingerprints = tls.rootCertificates.map((cert) => {
        // 解析 PEM 格式证书，计算 SHA256 指纹
        let certLines = cert.split("\n");
        certLines.shift();  // 移除 -----BEGIN CERTIFICATE-----
        certLines.pop();    // 移除 -----END CERTIFICATE-----
        let certBody = certLines.join("");
        let buf = Buffer.from(certBody, "base64");

        const shasum = crypto.createHash("sha256");
        shasum.update(buf);

        return shasum
            .digest("hex")
            .toUpperCase()
            .replace(/(.{2})(?!$)/g, "$1:");  // 格式化为 AA:BB:CC:...
    });

    // 添加额外的已知根证书
    fingerprints.push(
        "6D:99:FB:26:5E:B1:C5:B3:74:47:65:FC:BC:64:8F:3C:D8:E1:BF:FA:FD:C4:C2:F9:9B:9D:47:CF:7F:F1:C2:4F"
    ); // ISRG X1 cross-signed with DST X3
    fingerprints.push(
        "8B:05:B6:8C:C6:59:E5:ED:0F:CB:38:F2:C9:42:FB:FD:20:0E:6F:2F:F9:F8:5D:63:C6:99:4E:F5:E0:B0:27:01"
    ); // ISRG X2 cross-signed with ISRG X1

    return new Set(fingerprints);
};
```

#### 5.1.3 发送证书过期通知

**位置**：`server/model/monitor.js:1578-1614`

```javascript
async sendCertNotificationByTargetDays(certCN, certType, daysRemaining, targetDays, notificationList) {
    // 关键：查询条件是 days <= targetDays，不是 days = targetDays
    // 这意味着：如果已经发送过 7 天的通知（days=7），
    // 当检查 14 天的阈值时，7 <= 14 成立，所以不会发送 14 天的通知
    let row = await R.getRow(
        "SELECT * FROM notification_sent_history WHERE type = ? AND monitor_id = ? AND days <= ?",
        ["certificate", this.id, targetDays]
    );

    // Sent already, no need to send again
    if (row) {
        log.debug("monitor", "Sent already, no need to send again");
        return;
    }

    let sent = false;
    log.debug("monitor", "Send certificate notification");

    for (let notification of notificationList) {
        try {
            log.debug("monitor", "Sending to " + notification.name);
            await Notification.send(
                JSON.parse(notification.config),
                `[${this.name}][${this.url}] ${certType} certificate ${certCN} will expire in ${daysRemaining} days`
            );
            sent = true;
        } catch (e) {
            log.error("monitor", "Cannot send cert notification to " + notification.name);
            log.error("monitor", e);
        }
    }

    if (sent) {
        // 记录发送的是 targetDays，不是 daysRemaining
        await R.exec("INSERT INTO notification_sent_history (type, monitor_id, days) VALUES(?, ?, ?)", [
            "certificate",
            this.id,
            targetDays,
        ]);
    }
}
```

> **关键点**：
> 1. 查询条件是 `days <= targetDays`，这是一个**累积防重**逻辑
> 2. 记录的是 `targetDays`（目标天数），不是 `daysRemaining`（实际剩余天数）
> 3. 证书过期检查**没有对 `notifyDays` 进行排序**（对比域名过期有排序）

### 5.2 域名过期告警

#### 5.2.1 核心函数：`DomainExpiry.sendNotifications`

**位置**：`server/model/domain_expiry.js:309-365`

```javascript
static async sendNotifications(domainName, notificationList) {
    const domain = await DomainExpiry.findByDomainNameOrCreate(domainName);
    
    if (!notificationList.length > 0) {
        log.debug("domain_expiry", "No notification, no need to send domain notification");
        return;
    }

    // 校验过期日期
    if (!domain.expiry || isNaN(new Date(domain.expiry).getTime())) {
        log.warn("domain_expiry", 
            `No valid expiry date passed to sendNotifications for ${domainName} (expiry: ${domain.expiry}), skipping notification`);
        return;
    }

    const daysRemaining = domain.daysRemaining;
    const lastSent = domain.lastExpiryNotificationSent;
    
    log.debug("domain_expiry", `${domainName} expires in ${daysRemaining} days`);

    // 获取告警天数配置
    let notifyDays = await setting("domainExpiryNotifyDays");
    if (notifyDays == null || !Array.isArray(notifyDays)) {
        await setSetting("domainExpiryNotifyDays", [7, 14, 21], "general");
        notifyDays = [7, 14, 21];
    }

    if (Array.isArray(notifyDays)) {
        // 升序排列，确保只发送最紧急的一次通知
        // 注释说明：Asc sort to avoid sending multiple notifications if daysRemaining is below multiple targetDays
        notifyDays.sort((a, b) => a - b);
        
        for (const targetDays of notifyDays) {
            // 剩余天数大于目标天数，跳过
            if (daysRemaining > targetDays) {
                log.debug("domain_expiry", 
                    `No need to send domain notification for ${domainName} (${daysRemaining} days valid) on ${targetDays} deadline.`);
                continue;
            } 
            // 已经发送过该天数的通知，跳过
            // 关键：条件是 lastSent <= targetDays
            // 这意味着：如果已经发送过 7 天的通知，14 天和 21 天的都不会发送
            else if (lastSent && lastSent <= targetDays) {
                log.debug("domain_expiry", 
                    `Notification for ${domainName} on ${targetDays} deadline sent already, no need to send again.`);
                continue;
            }
            
            // 发送通知
            const sent = await sendDomainNotificationByTargetDays(
                domainName,
                daysRemaining,
                targetDays,
                notificationList
            );
            
            // 记录发送状态
            if (sent) {
                domain.lastExpiryNotificationSent = targetDays;
                await R.store(domain);
                return targetDays;  // 只发送一次最紧急的，直接返回
            }
        }
    }
}
```

#### 5.2.2 发送域名过期通知

**位置**：`server/model/domain_expiry.js:150-168`

```javascript
async function sendDomainNotificationByTargetDays(domain, daysRemaining, targetDays, notificationList) {
    let sent = false;
    log.debug("domain_expiry", `Send domain expiry notification for ${targetDays} deadline.`);

    for (let notification of notificationList) {
        try {
            log.debug("domain_expiry", `Sending to ${notification.name}`);
            await Notification.send(
                JSON.parse(notification.config),
                `Domain name ${domain} will expire in ${daysRemaining} days`
            );
            sent = true;
        } catch (e) {
            log.error("domain_expiry", `Cannot send domain notification to ${notification.name}:`, e);
        }
    }

    return sent;
}
```

### 5.3 通知发送历史表

**位置**：`db/knex_init_db.js:373-380`

```javascript
await knex.schema.createTable("notification_sent_history", (table) => {
    table.increments("id");
    table.string("type", 50).notNullable();  // 'certificate' 或 'domain'
    table.integer("monitor_id").unsigned().notNullable();
    table.integer("days").notNullable();      // 目标天数（7, 14, 21）
    table.unique(["type", "monitor_id", "days"]);  // 防止重复
    table.index(["type", "monitor_id", "days"], "good_index");
});
```

### 5.4 证书 vs 域名：通知防重逻辑对比

| 特性 | 证书过期 | 域名过期 |
|------|----------|----------|
| **存储位置** | `notification_sent_history` 表 | `domain_expiry.lastExpiryNotificationSent` 字段 |
| **查询条件** | `days <= targetDays` | `lastSent <= targetDays` |
| **排序** | ❌ 没有排序 | ✅ 升序排序 |
| **发送后返回** | 继续遍历 | 立即 `return` |
| **记录值** | `targetDays` | `targetDays` |

> **潜在问题**：证书过期检查没有对 `notifyDays` 进行排序。如果用户自定义顺序为 `[21, 14, 7]`，当剩余天数为 5 天时：
> 1. 检查 21 天：`5 <= 21`，发送，记录 `days=21`
> 2. 检查 14 天：查询 `days <= 14`，`21 <= 14` 不成立，发送，记录 `days=14`
> 3. 检查 7 天：查询 `days <= 7`，`14 <= 7` 不成立，发送，记录 `days=7`
> 
> 这样会发送 3 次通知！而域名过期由于有排序和 `return`，只会发送 1 次。

---

## 6. 前端展示

### 6.1 数据传输机制

前端通过 Socket.io 实时接收后端推送的证书和域名信息。

#### 6.1.1 后端推送

**位置**：`server/model/monitor.js:1349-1410`

```javascript
// 在 sendStats 中调用
static async sendStats(io, monitorID, userID) {
    // ... 其他统计信息
    
    // 发送证书信息
    await Monitor.sendCertInfo(io, monitorID, userID);
    
    // 发送域名信息
    await Monitor.sendDomainInfo(io, monitorID, userID);
}

// 发送证书信息
static async sendCertInfo(io, monitorID, userID) {
    let tlsInfo = await R.findOne("monitor_tls_info", "monitor_id = ?", [monitorID]);
    if (tlsInfo != null) {
        io.to(userID).emit("certInfo", monitorID, tlsInfo.info_json);
    }
}

// 发送域名信息
static async sendDomainInfo(io, monitorID, userID) {
    const monitor = await R.findOne("monitor", "id = ?", [monitorID]);
    try {
        // 注意：这里只检查监控类型是否支持，不检查 domainExpiryNotification
        const supportInfo = await DomainExpiry.checkSupport(monitor);
        const domain = await DomainExpiry.findByDomainNameOrCreate(supportInfo.domain);
        if (domain?.expiry) {
            io.to(userID).emit("domainInfo", monitorID, domain.daysRemaining, new Date(domain.expiry));
        }
    } catch (e) {}
}
```

> **关键发现**：`sendDomainInfo` **不检查** `domainExpiryNotification`，只检查监控类型是否支持。只要数据库中有过期日期，就会推送到前端。

#### 6.1.2 前端接收

**位置**：`src/mixins/socket.js:52,252-258`

```javascript
// 初始化
domainInfoList: {},

// 接收证书信息
socket.on("certInfo", (monitorID, data) => {
    this.tlsInfoList[monitorID] = JSON.parse(data);
});

// 接收域名信息
socket.on("domainInfo", (monitorID, daysRemaining, expiresOn) => {
    this.domainInfoList[monitorID] = { daysRemaining: daysRemaining, expiresOn: expiresOn };
});
```

### 6.2 前端组件

#### 6.2.1 主展示组件：CertificateInfo.vue

**位置**：`src/components/CertificateInfo.vue`

```vue
<template>
    <div>
        <h4>{{ $t("Certificate Info") }}</h4>
        {{ $t("Certificate Chain:") }}
        
        <!-- 证书有效性标签 -->
        <div v-if="valid" class="rounded d-inline-flex ms-2 text-white tag-valid">
            {{ $t("Valid") }}
        </div>
        <div v-if="!valid" class="rounded d-inline-flex ms-2 text-white tag-invalid">
            {{ $t("Invalid") }}
        </div>
        
        <!-- 递归显示证书链 -->
        <certificate-info-row :cert="certInfo" />
    </div>
</template>

<script>
import CertificateInfoRow from "./CertificateInfoRow.vue";
export default {
    components: {
        CertificateInfoRow,
    },
    props: {
        certInfo: { type: Object, required: true },
        valid: { type: Boolean, required: true },
    },
};
</script>
```

#### 6.2.2 证书链行组件：CertificateInfoRow.vue

**位置**：`src/components/CertificateInfoRow.vue`

```vue
<template>
    <div>
        <div class="d-flex flex-row align-items-center p-1 overflow-hidden">
            <div class="m-3 ps-3">
                <div class="cert-icon">
                    <font-awesome-icon icon="file" />
                    <font-awesome-icon class="award-icon" icon="award" />
                </div>
            </div>
            <div class="m-3">
                <table class="text-start">
                    <tbody>
                        <tr>
                            <td class="px-3">{{ $t("Subject:") }}</td>
                            <td>{{ formatSubject(cert.subject) }}</td>
                        </tr>
                        <tr>
                            <td class="px-3">{{ $t("Valid To:") }}</td>
                            <td><Datetime :value="cert.validTo" /></td>
                        </tr>
                        <tr>
                            <td class="px-3">{{ $t("Days Remaining:") }}</td>
                            <td>{{ cert.daysRemaining }}</td>
                        </tr>
                        <tr>
                            <td class="px-3">{{ $t("Issuer:") }}</td>
                            <td>{{ formatSubject(cert.issuer) }}</td>
                        </tr>
                        <tr>
                            <td class="px-3">{{ $t("Fingerprint:") }}</td>
                            <td>{{ cert.fingerprint }}</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
        
        <!-- 递归显示证书链 -->
        <div class="d-flex">
            <font-awesome-icon v-if="cert.issuerCertificate" class="m-2 ps-6 link-icon" icon="link" />
        </div>
        <certificate-info-row v-if="cert.issuerCertificate" :cert="cert.issuerCertificate" />
    </div>
</template>
```

### 6.3 详情页面展示

**位置**：`src/pages/Details.vue:253-295`

```vue
<!-- 证书过期信息摘要 -->
<div v-if="tlsInfo" class="col-12 col-sm col row d-flex align-items-center d-sm-block">
    <h4 class="col-4 col-sm-12">{{ $t("Cert Exp.") }}</h4>
    <p class="col-4 col-sm-12 mb-0 mb-sm-2">
        (<Datetime :value="tlsInfo.certInfo.validTo" date-only />)
    </p>
    <span class="col-4 col-sm-12 num">
        <!-- 点击展开详细信息 -->
        <a href="#" @click.prevent="toggleCertInfoBox = !toggleCertInfoBox">
            {{ $t("days", tlsInfo.certInfo.daysRemaining) }}
        </a>
        <!-- 主机名不匹配警告 -->
        <font-awesome-icon
            v-if="tlsInfo.hostnameMatchMonitorUrl === false"
            class="cert-info-warn"
            icon="exclamation-triangle"
            :title="$t('certHostnameMismatch')"
        />
    </span>
</div>

<!-- 域名过期信息摘要 -->
<!-- 注意：只检查 domainInfo 是否存在，不检查 domainExpiryNotification -->
<div v-if="domainInfo" class="col-12 col-sm col row d-flex align-items-center d-sm-block">
    <h4 class="col-4 col-sm-12">{{ $t("labelDomainExpiry") }}</h4>
    <p class="col-4 col-sm-12 mb-0 mb-sm-2">
        (<Datetime :value="domainInfo.expiresOn" date-only />)
    </p>
    <span class="col-4 col-sm-12 num">
        {{ $t("days", domainInfo.daysRemaining) }}
    </span>
</div>

<!-- 证书详细信息弹窗 -->
<transition name="slide-fade" appear>
    <div v-if="showCertInfoBox" class="shadow-box big-padding text-center">
        <div class="row">
            <div class="col">
                <certificate-info :certInfo="tlsInfo.certInfo" :valid="tlsInfo.valid" />
            </div>
        </div>
    </div>
</transition>
```

### 6.4 计算属性

**位置**：`src/pages/Details.vue:560-580`

```javascript
// 证书信息
tlsInfo() {
    // 检查 tlsInfoList 中是否有数据，且有 certInfo 字段
    if (this.$root.tlsInfoList[this.monitor.id] && this.$root.tlsInfoList[this.monitor.id].certInfo) {
        return this.$root.tlsInfoList[this.monitor.id];
    }
    return null;
},

// 域名信息
// 修正：实际代码不检查 domainExpiryNotification，只检查是否有数据
domainInfo() {
    return this.$root.domainInfoList[this.monitor.id] || null;
},

// 是否显示证书详情框
showCertInfoBox() {
    return this.toggleCertInfoBox && this.tlsInfo;
},
```

> **关键修正**：
> - 原文档描述有误，`domainInfo()` 计算属性**不检查** `monitor.domainExpiryNotification`
> - 只检查 `domainInfoList` 中是否有数据
> - 数据推送和展示与 `domainExpiryNotification` 无关

### 6.5 Badge API（公开徽章）

**位置**：`server/routers/api-router.js:424-505`

系统还提供了公开的 Badge API，用于在外部网站显示证书状态：

```javascript
router.get("/api/badge/:id/cert-exp", cache("5 minutes"), async (request, response) => {
    allowAllOrigin(response);

    // 徽章颜色阈值（可通过 URL 参数自定义）
    const {
        upColor = badgeConstants.defaultUpColor,        // 绿色 #66c20a
        warnColor = badgeConstants.defaultWarnColor,      // 黄色 #eed202
        downColor = badgeConstants.defaultDownColor,      // 红色 #c2290a
        warnDays = badgeConstants.defaultCertExpireWarnDays,  // 默认 14 天
        downDays = badgeConstants.defaultCertExpireDownDays,  // 默认 7 天
        // ...
    } = request.query;

    try {
        const requestedMonitorId = parseInt(request.params.id, 10);
        const publicMonitor = await isMonitorPublic(requestedMonitorId);

        if (!publicMonitor) {
            // 监控不是公开的，显示 N/A
            badgeValues.message = "N/A";
            badgeValues.color = badgeConstants.naColor;
        } else {
            const tlsInfoBean = await R.findOne("monitor_tls_info", "monitor_id = ?", [requestedMonitorId]);

            if (!tlsInfoBean) {
                // 没有保存的证书信息
                badgeValues.message = "No/Bad Cert";
                badgeValues.color = badgeConstants.naColor;
            } else {
                const tlsInfo = JSON.parse(tlsInfoBean.info_json);

                if (!tlsInfo.valid) {
                    // 证书无效
                    badgeValues.message = "Bad Cert";
                    badgeValues.color = downColor;
                } else {
                    const daysRemaining = parseInt(overrideValue ?? tlsInfo.certInfo.daysRemaining);

                    // 关键：徽章使用区间判断
                    if (daysRemaining > warnDays) {
                        badgeValues.color = upColor;      // 绿色：>14天
                    } else if (daysRemaining > downDays) {
                        badgeValues.color = warnColor;    // 黄色：7-14天
                    } else {
                        badgeValues.color = downColor;    // 红色：<=7天
                    }
                    
                    badgeValues.message = `${daysRemaining} days`;
                }
            }
        }

        const svg = makeBadge(badgeValues);
        response.type("image/svg+xml");
        response.send(svg);
    } catch (error) {
        sendHttpError(response, error.message);
    }
});
```

---

## 7. 告警状态边界关系分析

### 7.1 三套独立机制

**重要发现**：通知发送、页面展示、徽章展示是三套**完全独立**的机制，使用不同的阈值和判断逻辑。

| 机制 | 配置项 | 默认值 | 判断逻辑 | 用途 |
|------|--------|--------|----------|------|
| **通知发送** | `tlsExpiryNotifyDays` | `[7, 14, 21]` | `daysRemaining <= targetDays` | 触发邮件/Slack/Telegram 等通知 |
| **通知发送** | `domainExpiryNotifyDays` | `[7, 14, 21]` | `daysRemaining <= targetDays` | 触发邮件/Slack/Telegram 等通知 |
| **徽章警告** | `defaultCertExpireWarnDays` | `14` | `daysRemaining > 7 && daysRemaining <= 14` | 显示黄色徽章 |
| **徽章危险** | `defaultCertExpireDownDays` | `7` | `daysRemaining <= 7` | 显示红色徽章 |
| **页面展示** | 无专用配置 | 无 | `v-if="tlsInfo"` / `v-if="domainInfo"` | 只要有数据就显示 |

### 7.2 判断逻辑对比

#### 7.2.1 通知发送逻辑

```
检查条件：daysRemaining <= targetDays

默认 targetDays: [7, 14, 21]

示例：daysRemaining = 5
├── 检查 21 天：5 <= 21 ✅ 可发送
├── 检查 14 天：5 <= 14 ✅ 可发送
└── 检查 7 天：  5 <= 7  ✅ 可发送

但由于防重机制（days <= targetDays），只会发送一次（最紧急的那个）
```

#### 7.2.2 徽章展示逻辑

```
检查条件：区间判断

默认阈值：warnDays=14, downDays=7

示例：daysRemaining = 5
├── daysRemaining > 14?  5 > 14?  ❌
├── daysRemaining > 7?   5 > 7?   ❌
└── 否则显示红色（downColor）

示例：daysRemaining = 10
├── daysRemaining > 14?  10 > 14? ❌
├── daysRemaining > 7?   10 > 7?  ✅ 显示黄色（warnColor）
└── ...

示例：daysRemaining = 30
├── daysRemaining > 14?  30 > 14? ✅ 显示绿色（upColor）
└── ...
```

#### 7.2.3 页面展示逻辑

```
检查条件：是否有数据

证书：v-if="tlsInfo"
域名：v-if="domainInfo"

只要 domainInfoList / tlsInfoList 中有数据，就显示
不根据剩余天数改变显示样式（除了主机名不匹配的警告图标）
```

### 7.3 阈值对应关系（默认配置）

| 剩余天数 | 证书通知 | 域名通知 | 徽章颜色 | 页面展示 |
|----------|----------|----------|----------|----------|
| > 21 天 | ❌ 不触发 | ❌ 不触发 | 🟢 绿色 | ✅ 显示 |
| 15-21 天 | ✅ 21 天阈值 | ✅ 21 天阈值 | 🟢 绿色 | ✅ 显示 |
| 8-14 天 | ✅ 14 天阈值 | ✅ 14 天阈值 | 🟡 黄色 | ✅ 显示 |
| 1-7 天 | ✅ 7 天阈值 | ✅ 7 天阈值 | 🔴 红色 | ✅ 显示 |
| **0 天** | ⚠️ **跳过!** | ✅ 7 天阈值 | 🔴 红色 | ✅ 显示 |
| < 0 天 (已过期) | ✅ 7 天阈值 | ✅ 7 天阈值 | 🔴 红色 | ✅ 显示 |

**注意**：证书通知在 `daysRemaining = 0` 时会被跳过，详见 7.4.4 节分析。

### 7.4 关键边界情况

#### 7.4.1 数据推送 vs 通知发送

**场景**：用户关闭了 `domainExpiryNotification`

**影响**：
| 功能 | 是否执行 |
|------|----------|
| RDAP 查询过期日期 | ❌ 不执行（心跳成功后的检查被跳过） |
| 数据推送到前端 | ⚠️ 取决于数据库是否已有数据 |
| 页面展示 | ⚠️ 取决于数据库是否已有数据 |
| 发送通知 | ❌ 不发送 |

**注意**：如果数据库中已有过期日期（比如之前启用过），`sendDomainInfo` 仍然会推送数据到前端。

#### 7.4.2 证书更新后的行为

**场景**：证书更新，`fingerprint256` 改变

**行为**：
1. `updateTlsInfo` 检测到指纹变化
2. 清除 `notification_sent_history` 中该监控的证书通知记录
3. 重新开始告警周期

#### 7.4.3 域名续费后的行为

**场景**：域名续费，`expiry` 日期延后

**行为**：
1. `checkExpiry` 检测到新日期 > 旧日期
2. 清除 `lastExpiryNotificationSent`（设为 null）
3. 重新开始告警周期

### 7.5 防重机制深度解析

#### 7.5.1 证书通知防重

```
数据库表：notification_sent_history
查询条件：type = 'certificate' AND monitor_id = ? AND days <= ?

场景：默认配置 [7, 14, 21]，按升序遍历

第一次检查（daysRemaining = 5）：
├── 检查 7 天：
│   ├── 查询：days <= 7
│   ├── 无记录，发送通知
│   └── 插入记录：days = 7
├── 检查 14 天：
│   ├── 查询：days <= 14
│   ├── 有记录（7 <= 14），跳过
│   └── 不发送
└── 检查 21 天：
    ├── 查询：days <= 21
    ├── 有记录（7 <= 21），跳过
    └── 不发送

结果：只发送 1 次通知（7 天阈值）
```

#### 7.5.2 域名通知防重

```
存储位置：domain_expiry.lastExpiryNotificationSent
查询条件：lastSent <= targetDays

场景：默认配置 [7, 14, 21]，升序排序后遍历

第一次检查（daysRemaining = 5）：
├── 检查 7 天：
│   ├── lastSent 为 null，条件不成立
│   ├── 发送通知
│   └── 记录 lastSent = 7，立即 return
├── 检查 14 天：（不会执行，因为已 return）
└── 检查 21 天：（不会执行，因为已 return）

结果：只发送 1 次通知（7 天阈值）
```

#### 7.5.3 潜在问题：证书过期检查没有排序

```
场景：用户自定义 notifyDays = [21, 14, 7]，daysRemaining = 5

检查顺序：21 → 14 → 7

├── 检查 21 天：
│   ├── 查询：days <= 21
│   ├── 无记录，发送通知
│   └── 插入记录：days = 21
├── 检查 14 天：
│   ├── 查询：days <= 14
│   ├── 21 <= 14? 不成立 ❌
│   ├── 发送通知
│   └── 插入记录：days = 14
└── 检查 7 天：
    ├── 查询：days <= 7
    ├── 14 <= 7? 不成立 ❌
    ├── 发送通知
    └── 插入记录：days = 7

结果：发送 3 次通知！（这是潜在问题）
```

> **对比**：域名过期检查明确进行了升序排序，并有注释说明原因。证书过期检查没有排序，可能导致多次通知。

---

## 8. 端到端时序说明

### 8.1 证书监控完整时序

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           HTTPS 证书监控端到端时序                                │
└─────────────────────────────────────────────────────────────────────────────────┘

时间轴 →
│
├── T1: 心跳检测开始
│   │
│   ├── HTTP 监控类型
│   │   ├── 建立 HTTPS 连接
│   │   ├── 监听 keylog / secureConnect 事件
│   │   └── 触发 checkCertificate(socket)
│   │
│   ├── TCP/TLS 监控类型
│   │   ├── tls.connect() 建立连接
│   │   ├── secureConnect 事件触发
│   │   └── 触发 checkCertificate(socket)
│   │
│   └── STARTTLS 类型
│       ├── TCP 连接 → 发送 STARTTLS 命令
│       ├── 升级到 TLS 连接
│       └── 触发 checkCertificate(socket)
│
├── T2: 证书解析
│   │
│   ├── checkCertificate(socket)
│   │   ├── socket.getPeerCertificate(true) 获取证书链
│   │   └── 调用 parseCertificateInfo(info)
│   │
│   └── parseCertificateInfo(info)
│       ├── 转换 valid_to → validTo (Date)
│       ├── 计算 daysRemaining (UTC)
│       ├── 识别 certType (server/intermediate/root)
│       └── 提取 validFor (SAN 域名)
│
├── T3: 数据保存
│   │
│   └── handleTlsInfo(tlsInfo)
│       ├── updateTlsInfo(tlsInfo)
│       │   ├── 检查 fingerprint256 是否变化
│       │   ├── 变化则清除 notification_sent_history
│       │   └── JSON.stringify → monitor_tls_info 表
│       │
│       ├── prometheus.update() 更新指标
│       │
│       └── 条件检查：!getIgnoreTls() && isEnabledExpiryNotification()
│           └── ✅ 满足 → 调用 checkCertExpiryNotifications()
│
├── T4: 告警检查（可选）
│   │
│   └── checkCertExpiryNotifications(monitor, tlsInfoObject)
│       ├── 获取 notificationList
│       ├── 获取 notifyDays（默认 [7, 14, 21]）
│       │
│       ├── 遍历每个 targetDays
│       │   └── 遍历证书链
│       │       ├── 跳过已知根证书
│       │       ├── 检查 daysRemaining <= targetDays
│       │       └── ✅ 满足 → 调用 sendCertNotificationByTargetDays()
│       │
│       └── sendCertNotificationByTargetDays()
│           ├── 查询：days <= targetDays（防重）
│           ├── ❌ 有记录 → 跳过
│           └── ✅ 无记录 → 发送通知 + 插入 notification_sent_history
│
├── T5: 前端数据推送
│   │
│   └── sendStats(io, monitorID, userID) 触发
│       │
│       ├── sendCertInfo()
│       │   ├── 查询 monitor_tls_info 表
│       │   └── ✅ 有数据 → socket.emit("certInfo", monitorID, info_json)
│       │
│       └── sendDomainInfo()
│           ├── 检查监控类型是否支持
│           ├── 查询 domain_expiry 表
│           └── ✅ 有数据 → socket.emit("domainInfo", monitorID, daysRemaining, expiresOn)
│
└── T6: 前端展示
    │
    ├── Socket 事件监听
    │   ├── "certInfo" → tlsInfoList[monitorID] = JSON.parse(data)
    │   └── "domainInfo" → domainInfoList[monitorID] = { daysRemaining, expiresOn }
    │
    ├── 计算属性
    │   ├── tlsInfo() → 检查 tlsInfoList[monitor.id]?.certInfo
    │   └── domainInfo() → 返回 domainInfoList[monitor.id] || null
    │
    └── 模板渲染
        ├── v-if="tlsInfo" → 显示证书摘要
        ├── v-if="domainInfo" → 显示域名摘要
        └── 点击 → 显示 CertificateInfo 组件（证书链详情）
```

### 8.2 域名过期监控完整时序

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           域名过期监控端到端时序                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

时间轴 →
│
├── T1: 心跳检测成功
│   │
│   └── 条件检查：bean.status !== MAINTENANCE && Boolean(domainExpiryNotification)
│       │
│       └── ✅ 满足 → 执行域名过期检查
│           │
│           ├── DomainExpiry.checkSupport(monitor)
│           │   └── 检查 monitor.type 是否在 TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD
│           │
│           └── DomainExpiry.checkExpiry(domainName)
│
├── T2: RDAP 查询（可选，有缓存）
│   │
│   └── DomainExpiry.checkExpiry(domainName)
│       │
│       ├── 检查 lastCheck（24 小时缓存）
│       │   ├── ✅ 24 小时内 → 直接返回 bean.expiry
│       │   └── ❌ 超过 24 小时 → 执行 RDAP 查询
│       │
│       ├── getRdapDnsData()
│       │   ├── 检查缓存（7 天有效）
│       │   ├── 无缓存 → fetch https://data.iana.org/rdap/dns.json
│       │   └── 失败 → 使用内置数据 (extra/rdap-dns.json)
│       │
│       ├── getRdapServer(tld)
│       │   └── 根据 TLD 查找 RDAP 服务器 URL
│       │
│       └── getRdapDomainExpiryDate(domain)
│           ├── 请求 {rdapServer}/domain/{domain}
│           └── 查找 events 数组中 eventAction="expiration" 的日期
│
├── T3: 数据更新
│   │
│   └── checkExpiry() 继续
│       │
│       ├── 检查：新日期 > 旧日期？（域名是否续费）
│       │   └── ✅ 是 → lastExpiryNotificationSent = null
│       │
│       └── 更新 domain_expiry 表
│           ├── expiry = 新日期
│           └── lastCheck = 当前时间
│
├── T4: 告警检查
│   │
│   └── DomainExpiry.sendNotifications(domainName, notificationList)
│       │
│       ├── 获取 notifyDays（默认 [7, 14, 21]）
│       ├── 升序排序：notifyDays.sort((a, b) => a - b)
│       │
│       ├── 遍历每个 targetDays
│       │   ├── 检查 daysRemaining > targetDays
│       │   │   └── ✅ 是 → 跳过
│       │   │
│       │   ├── 检查 lastSent && lastSent <= targetDays（防重）
│       │   │   └── ✅ 是 → 跳过
│       │   │
│       │   └── ✅ 都不满足 → 发送通知
│       │
│       └── sendDomainNotificationByTargetDays()
│           ├── 遍历 notificationList
│           ├── 调用 Notification.send()
│           └── ✅ 发送成功 → 记录 lastExpiryNotificationSent = targetDays
│           └── 立即 return（只发送一次最紧急的）
│
├── T5: 前端数据推送
│   │
│   └── sendStats() 中的 sendDomainInfo()
│       │
│       ├── DomainExpiry.checkSupport(monitor)
│       │   └── 不检查 domainExpiryNotification！
│       │
│       └── DomainExpiry.findByDomainNameOrCreate(supportInfo.domain)
│           └── ✅ 有 expiry 数据 → socket.emit("domainInfo", ...)
│
└── T6: 前端展示
    │
    └── 同证书监控流程
        ├── socket.on("domainInfo") → domainInfoList[monitorID] = { ... }
        ├── domainInfo() 计算属性
        └── v-if="domainInfo" → 显示域名过期摘要
```

### 8.3 徽章 API 时序

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Badge API 端到端时序                                   │
└─────────────────────────────────────────────────────────────────────────────────┘

时间轴 →
│
├── T1: HTTP 请求到达
│   │
│   └── GET /api/badge/:id/cert-exp
│       │
│       ├── cache("5 minutes") → 检查缓存
│       │   ├── ✅ 有缓存 → 直接返回
│       │   └── ❌ 无缓存 → 继续处理
│       │
│       ├── 解析参数
│       │   ├── warnDays（默认 14）
│       │   ├── downDays（默认 7）
│       │   ├── upColor / warnColor / downColor
│       │   └── ...
│       │
│       └── 检查监控是否公开
│
├── T2: 数据查询
│   │
│   ├── ❌ 监控不公开
│   │   ├── message = "N/A"
│   │   └── color = naColor (#999)
│   │
│   └── ✅ 监控公开
│       │
│       ├── 查询 monitor_tls_info 表
│       │   │
│       │   ├── ❌ 无记录
│       │   │   ├── message = "No/Bad Cert"
│       │   │   └── color = naColor
│       │   │
│       │   └── ✅ 有记录
│       │       │
│       │       ├── 检查 tlsInfo.valid
│       │       │   └── ❌ 无效
│       │       │       ├── message = "Bad Cert"
│       │       │       └── color = downColor
│       │       │
│       │       └── ✅ 有效 → 区间判断颜色
│       │           │
│       │           ├── daysRemaining > warnDays?
│       │           │   └── ✅ 是 → color = upColor（绿色）
│       │           │
│       │           ├── daysRemaining > downDays?
│       │           │   └── ✅ 是 → color = warnColor（黄色）
│       │           │
│       │           └── ❌ 否则 → color = downColor（红色）
│       │
│       └── message = `${daysRemaining} days`
│
├── T3: 生成 SVG 徽章
│   │
│   └── makeBadge(badgeValues) → 返回 SVG 图像
│
└── T4: 缓存响应
    │
    └── cache("5 minutes") → 缓存 5 分钟内的相同请求
```

---

## 9. 关键代码位置汇总

### 9.1 后端代码

| 功能模块 | 文件路径 | 关键函数/类 |
|---------|---------|------------|
| 证书解析 | `server/util-server.js:450-491` | `parseCertificateInfo()` |
| 证书采集 | `server/util-server.js:498-520` | `checkCertificate()` |
| 主机名匹配检查 | `server/util-server.js:529-544` | `checkCertificateHostname()` |
| 证书过期告警检查 | `server/util-server.js:917-972` | `checkCertExpiryNotifications()` |
| 根证书指纹库 | `server/util-server.js:798-823` | `rootCertificatesFingerprints()` |
| 监控主模型 | `server/model/monitor.js` | `class Monitor` |
| TLS 信息处理 | `server/model/monitor.js:2096-2104` | `handleTlsInfo()` |
| TLS 信息保存 | `server/model/monitor.js:1292-1328` | `updateTlsInfo()` |
| 证书通知发送 | `server/model/monitor.js:1578-1614` | `sendCertNotificationByTargetDays()` |
| 证书信息推送 | `server/model/monitor.js:1386-1391` | `sendCertInfo()` |
| 域名信息推送 | `server/model/monitor.js:1400-1410` | `sendDomainInfo()` |
| TCP/TLS 监控 | `server/monitor-types/tcp.js` | `class TCPMonitorType` |
| STARTTLS 支持 | `server/monitor-types/tcp.js:154-228` | `performStartTls()` |
| 域名过期监控 | `server/model/domain_expiry.js` | `class DomainExpiry` |
| RDAP 服务器发现 | `server/model/domain_expiry.js:20-33` | `getRdapServer()` |
| RDAP 数据获取 | `server/model/domain_expiry.js:39-82` | `getRdapDnsData()` |
| 域名过期查询 | `server/model/domain_expiry.js:110-140` | `getRdapDomainExpiryDate()` |
| 域名过期检查 | `server/model/domain_expiry.js:278-302` | `checkExpiry()` |
| 域名过期通知 | `server/model/domain_expiry.js:309-365` | `sendNotifications()` |
| 徽章 API | `server/routers/api-router.js:424-505` | `/api/badge/:id/cert-exp` |

### 9.2 前端代码

| 功能模块 | 文件路径 | 关键组件/函数 |
|---------|---------|--------------|
| 证书信息展示 | `src/components/CertificateInfo.vue` | `CertificateInfo` 组件 |
| 证书链展示 | `src/components/CertificateInfoRow.vue` | `CertificateInfoRow` 组件 |
| Socket 事件监听 | `src/mixins/socket.js:52,252-258` | `certInfo`, `domainInfo` 事件 |
| 详情页计算属性 | `src/pages/Details.vue:560-580` | `tlsInfo`, `domainInfo` 计算属性 |
| 详情页模板 | `src/pages/Details.vue:253-295` | `v-if="tlsInfo"`, `v-if="domainInfo"` |
| 支持域名过期的监控类型 | `src/util.ts:776-781` | `TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD` |
| 徽章常量 | `src/util.ts:143-161` | `badgeConstants` |

### 9.3 数据库

| 表名 | 用途 | 关键字段 |
|------|------|----------|
| `monitor_tls_info` | 存储 TLS 证书信息 | `monitor_id`, `info_json` |
| `domain_expiry` | 存储域名过期信息 | `domain`, `expiry`, `lastCheck`, `lastExpiryNotificationSent` |
| `notification_sent_history` | 通知发送历史 | `type`, `monitor_id`, `days` |

### 9.4 配置项

| 配置项 | 默认值 | 用途 |
|--------|--------|------|
| `tlsExpiryNotifyDays` | `[7, 14, 21]` | 证书过期告警天数（通知发送） |
| `domainExpiryNotifyDays` | `[7, 14, 21]` | 域名过期告警天数（通知发送） |
| `rdapDnsData` | 内置数据 | IANA RDAP 服务器列表 |
| `defaultCertExpireWarnDays` | `14` | 徽章黄色警告阈值 |
| `defaultCertExpireDownDays` | `7` | 徽章红色危险阈值 |
| `defaultUpColor` | `#66c20a` | 徽章绿色（正常） |
| `defaultWarnColor` | `#eed202` | 徽章黄色（警告） |
| `defaultDownColor` | `#c2290a` | 徽章红色（危险） |

---

## 10. 关键技术点

### 10.1 时间处理

- **统一使用 UTC 时间**：所有日期计算都使用 `dayjs.utc()` 避免时区问题
- **证书有效期**：`validTo` 使用 Date 对象存储，`daysRemaining` 实时计算
- **缓存策略**：
  - RDAP DNS 数据：缓存 7 天
  - 域名过期查询：缓存 24 小时
  - 徽章 API：缓存 5 分钟

### 10.2 证书链处理

- **递归解析**：通过 `issuerCertificate` 字段遍历完整证书链
- **类型识别**：
  - 链中第一个 = `server`
  - 有上级且非自签名 = `intermediate CA`
  - 自签名或无上级 = `root CA` 或 `self-signed`
- **指纹对比**：使用 `fingerprint256` 检测证书是否更新

### 10.3 防重复告警机制

#### 10.3.1 证书通知防重

- **存储位置**：`notification_sent_history` 表
- **查询条件**：`days <= targetDays`（累积防重）
- **记录值**：`targetDays`（目标天数），不是 `daysRemaining`

#### 10.3.2 域名通知防重

- **存储位置**：`domain_expiry.lastExpiryNotificationSent` 字段
- **查询条件**：`lastSent <= targetDays`（累积防重）
- **记录值**：`targetDays`（目标天数）

#### 10.3.3 关键差异

| 特性 | 证书过期 | 域名过期 |
|------|----------|----------|
| 排序 | ❌ 无排序 | ✅ 升序排序 |
| 发送后返回 | 继续遍历 | 立即 `return` |

> **潜在问题**：证书过期检查没有排序。如果用户自定义 `notifyDays = [21, 14, 7]`，可能导致发送 3 次通知。

### 10.4 三套独立机制

| 机制 | 判断逻辑 | 阈值来源 |
|------|----------|----------|
| **通知发送** | `daysRemaining <= targetDays` | `tlsExpiryNotifyDays` / `domainExpiryNotifyDays` |
| **徽章展示** | 区间判断 | `defaultCertExpireWarnDays` / `defaultCertExpireDownDays` |
| **页面展示** | 数据存在性 | 无专用阈值 |

### 10.5 性能优化

- **缓存机制**：RDAP 数据、域名查询结果、徽章响应都有缓存
- **异步处理**：通知发送不阻塞主监控流程
- **按需查询**：只有启用了过期通知的监控才会执行相关检查
- **快速失败**：无通知配置时直接跳过告警检查

---

## 11. 配置与扩展

### 11.1 告警天数配置

通过 Settings 表配置，默认值：
- 证书通知：`[7, 14, 21]` 天
- 域名通知：`[7, 14, 21]` 天

### 11.2 徽章阈值配置

徽章阈值是硬编码常量（可通过 URL 参数覆盖）：
- `warnDays`（黄色）：默认 14 天
- `downDays`（红色）：默认 7 天

### 11.3 支持的域名后缀

通过 IANA 的 RDAP DNS 数据动态支持，常见 TLD 如 `.com`, `.net`, `.org`, `.io` 等都支持。

### 11.4 支持的 STARTTLS 协议

- SMTP (邮件)
- IMAP (邮件)
- XMPP (即时通讯)

### 11.5 支持域名过期监控的类型

```javascript
const TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD = {
    http: "url",
    keyword: "url",
    "json-query": "url",
    "real-browser": "url",
    "websocket-upgrade": "url",
};
```

---

*文档生成时间：2026-05-04*
*基于 Uptime Kuma 代码库分析*

---

## 修订记录

### v2.0（2026-05-04）

**修正内容**：

1. **前端 domainInfo 展示条件**
   - 修正：原文档错误描述 `domainInfo()` 检查 `monitor.domainExpiryNotification`
   - 实际：`domainInfo()` 只检查 `domainInfoList` 中是否有数据
   - `domainExpiryNotification` 只影响通知发送和 RDAP 查询，不影响数据推送和展示

2. **证书通知防重逻辑**
   - 强调查询条件是 `days <= targetDays`，不是 `days = targetDays`
   - 这是一个"累积防重"逻辑
   - 记录的是 `targetDays`，不是 `daysRemaining`

3. **增加告警状态边界关系分析（第7章）**
   - 明确通知发送、页面展示、徽章展示是三套独立机制
   - 对比三套机制的阈值、判断逻辑、用途
   - 分析关键边界情况

4. **增加端到端时序说明（第8章）**
   - 证书监控完整时序（T1-T6）
   - 域名过期监控完整时序（T1-T6）
   - 徽章 API 完整时序（T1-T4）

5. **发现潜在问题**
   - 证书过期检查没有对 `notifyDays` 进行排序
   - 域名过期检查明确进行了升序排序，并有注释说明
   - 如果用户自定义 `notifyDays = [21, 14, 7]`，证书可能发送 3 次通知

---

### v2.1（2026-05-04）

**深入校正内容**：

1. **`TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD` 完整列表**
   - 原文档只记录了 5 种类型（http, keyword, json-query, real-browser, websocket-upgrade）
   - 实际支持 18 种类型，使用 3 种不同字段：
     - `url` 字段：http, keyword, json-query, real-browser, websocket-upgrade
     - `hostname` 字段：port, ping, dns, smtp, snmp, gamedig, steam, mqtt, radius, tailscale-ping, sip-options
     - `grpcUrl` 字段：grpc-keyword

2. **`checkSupport` 完整逻辑**
   - 检查 1：`monitor.type` 是否在支持列表中
   - 检查 2：目标字段是否有值
   - 检查 3：是否是 ICANN 域名（`tld.isIcann`）
   - 检查 4：是否有 RDAP 服务器
   - 任一检查失败都会抛出异常，`sendDomainInfo` 会静默忽略

3. **`domainExpiryNotification` 影响范围校正**
   - ❌ **不影响**：数据推送 (`sendDomainInfo`)、页面展示 (`v-if="domainInfo"`)
   - ✅ **影响**：RDAP 查询 (`checkExpiry`)、通知发送 (`sendNotifications`)

4. **关键边界场景**
   - **场景 A**：新监控 + `domainExpiryNotification = false`
     - `findByDomainNameOrCreate` 创建新记录，但 `expiry` 为 null
     - `domain?.expiry` 为 null，不推送数据
     - 前端不显示
   - **场景 B**：曾经启用过，现在关闭
     - 数据库中已有 `expiry` 数据
     - `sendDomainInfo` 检查 `domain?.expiry` 为 true
     - 继续推送数据，前端继续显示
   - **场景 C**：TCP 端口监控 + `domainExpiryNotification = false`
     - 类型支持（`port` 在列表中）
     - 但不会执行 RDAP 查询
     - 除非数据库已有数据，否则不显示

5. **`findByDomainNameOrCreate` 逻辑**
   - 先查找现有记录
   - 没有记录且 `domainName` 存在时，创建新记录
   - 新创建的记录 `expiry` 为 null，不会触发推送