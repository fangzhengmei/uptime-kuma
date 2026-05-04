# Uptime Kuma HTTPS 证书、域名过期和 TLS 元数据监控流程分析

## 目录

1. [概述](#1-概述)
2. [证书采集与解析](#2-证书采集与解析)
3. [证书信息保存](#3-证书信息保存)
4. [域名过期监控](#4-域名过期监控)
5. [告警通知机制](#5-告警通知机制)
6. [前端展示](#6-前端展示)
7. [关键代码位置汇总](#7-关键代码位置汇总)

---

## 1. 概述

Uptime Kuma 对 HTTPS 证书、域名过期和 TLS 元数据的监控是一个完整的闭环流程，包括：

- **采集**：通过 TLS 连接获取证书信息
- **解析**：提取证书有效期、指纹、颁发者等元数据
- **保存**：将解析后的信息存储到数据库
- **告警**：根据剩余天数触发通知
- **展示**：在前端页面实时显示剩余天数和告警状态

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

**位置**：`src/util.js` 中定义的 `TYPES_WITH_DOMAIN_EXPIRY_SUPPORT_VIA_FIELD`

只有特定类型的监控支持域名过期检查，需要从监控配置中提取域名。

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

**位置**：`server/model/monitor.js` 中的 `sendCertNotificationByTargetDays` 方法

该方法会：
1. 检查 `notification_sent_history` 表，避免重复发送
2. 构造通知消息
3. 通过配置的通知渠道发送（Email、Slack、Telegram 等）
4. 记录发送历史

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
        notifyDays.sort((a, b) => a - b);
        
        for (const targetDays of notifyDays) {
            // 剩余天数大于目标天数，跳过
            if (daysRemaining > targetDays) {
                log.debug("domain_expiry", 
                    `No need to send domain notification for ${domainName} (${daysRemaining} days valid) on ${targetDays} deadline.`);
                continue;
            } 
            // 已经发送过该天数的通知，跳过
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
                return targetDays;  // 只发送一次最紧急的
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
        const supportInfo = await DomainExpiry.checkSupport(monitor);
        const domain = await DomainExpiry.findByDomainNameOrCreate(supportInfo.domain);
        if (domain?.expiry) {
            io.to(userID).emit("domainInfo", monitorID, domain.daysRemaining, new Date(domain.expiry));
        }
    } catch (e) {}
}
```

#### 6.1.2 前端接收

**位置**：`src/mixins/socket.js:252-258`

```javascript
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
    if (this.$root.tlsInfoList[this.monitor.id] && this.$root.tlsInfoList[this.monitor.id].certInfo) {
        return this.$root.tlsInfoList[this.monitor.id];
    }
    return null;
},

// 域名信息
domainInfo() {
    if (this.monitor.domainExpiryNotification && 
        this.$root.domainInfoList[this.monitor.id]) {
        return this.$root.domainInfoList[this.monitor.id];
    }
    return null;
},

// 是否显示证书详情框
showCertInfoBox() {
    return this.toggleCertInfoBox && this.tlsInfo;
},
```

### 6.5 Badge API（公开徽章）

**位置**：`server/routers/api-router.js:461-494`

系统还提供了公开的 Badge API，用于在外部网站显示证书状态：

```javascript
router.get("/api/badge/:id/cert", cache("5 minutes"), async (request, response) => {
    // ...
    const tlsInfoBean = await R.findOne("monitor_tls_info", "monitor_id = ?", [requestedMonitorId]);

    if (!tlsInfoBean) {
        badgeValues.message = "No/Bad Cert";
        badgeValues.color = badgeConstants.naColor;
    } else {
        const tlsInfo = JSON.parse(tlsInfoBean.info_json);

        if (!tlsInfo.valid) {
            badgeValues.message = "Bad Cert";
            badgeValues.color = downColor;
        } else {
            const daysRemaining = tlsInfo.certInfo.daysRemaining;
            
            // 根据剩余天数设置颜色
            if (daysRemaining > warnDays) {
                badgeValues.color = upColor;      // 绿色
            } else if (daysRemaining > downDays) {
                badgeValues.color = warnColor;    // 黄色
            } else {
                badgeValues.color = downColor;    // 红色
            }
            
            badgeValues.message = `${daysRemaining} days`;
        }
    }
    // 生成 SVG 徽章
    const svg = makeBadge(badgeValues);
    response.type("image/svg+xml");
    response.send(svg);
});
```

---

## 7. 关键代码位置汇总

### 7.1 后端代码

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

### 7.2 前端代码

| 功能模块 | 文件路径 | 关键组件/函数 |
|---------|---------|--------------|
| 证书信息展示 | `src/components/CertificateInfo.vue` | `CertificateInfo` 组件 |
| 证书链展示 | `src/components/CertificateInfoRow.vue` | `CertificateInfoRow` 组件 |
| Socket 事件监听 | `src/mixins/socket.js:252-258` | `certInfo`, `domainInfo` 事件 |
| 详情页展示 | `src/pages/Details.vue` | `tlsInfo`, `domainInfo` 计算属性 |

### 7.3 数据库

| 表名 | 用途 | 关键字段 |
|------|------|----------|
| `monitor_tls_info` | 存储 TLS 证书信息 | `monitor_id`, `info_json` |
| `domain_expiry` | 存储域名过期信息 | `domain`, `expiry`, `lastCheck`, `lastExpiryNotificationSent` |
| `notification_sent_history` | 通知发送历史 | `type`, `monitor_id`, `days` |

### 7.4 配置项

| 配置项 | 默认值 | 用途 |
|--------|--------|------|
| `tlsExpiryNotifyDays` | `[7, 14, 21]` | 证书过期告警天数 |
| `domainExpiryNotifyDays` | `[7, 14, 21]` | 域名过期告警天数 |
| `rdapDnsData` | 内置数据 | IANA RDAP 服务器列表 |

---

## 8. 流程图总结

### 8.1 证书监控完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HTTPS 证书监控流程                                   │
└─────────────────────────────────────────────────────────────────────────────┘

1. 采集阶段
   ┌──────────────┐
   │  HTTP 监控   │───── keylog/secureConnect 事件 ─────┐
   └──────────────┘                                        │
   ┌──────────────┐                                        ▼
   │  TCP/TLS 监控│───── tls.connect() ──────────────▶ TLS Socket
   └──────────────┘                                        │
   ┌──────────────┐                                        │
   │ STARTTLS    │───── 协议命令 → 升级 TLS ────────────┘
   └──────────────┘
                           │
                           ▼
2. 解析阶段
   ┌─────────────────────────────────────────────────────┐
   │ checkCertificate(socket)                             │
   │   ├── socket.getPeerCertificate(true)  // 获取证书链 │
   │   └── parseCertificateInfo(info)       // 解析元数据 │
   │       ├── 转换 valid_to → Date 对象                  │
   │       ├── 计算 daysRemaining (UTC)                  │
   │       ├── 识别 certType (server/intermediate/root)  │
   │       └── 提取 validFor (SAN 域名)                   │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
3. 保存阶段
   ┌─────────────────────────────────────────────────────┐
   │ handleTlsInfo(tlsInfo)                               │
   │   ├── updateTlsInfo()                                │
   │   │   ├── 检查证书指纹是否变更                         │
   │   │   ├── 变更则清除 notification_sent_history        │
   │   │   └── JSON.stringify → monitor_tls_info 表      │
   │   ├── prometheus.update()  // 更新指标               │
   │   └── checkCertExpiryNotifications() // 检查告警     │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
4. 告警阶段
   ┌─────────────────────────────────────────────────────┐
   │ checkCertExpiryNotifications()                      │
   │   ├── 获取配置的通知天数 [7, 14, 21]                │
   │   ├── 遍历证书链中的每个证书                          │
   │   │   ├── 跳过已知的根证书 (rootCertificates)        │
   │   │   ├── 检查 daysRemaining <= targetDays          │
   │   │   └── 检查 notification_sent_history 防重复      │
   │   └── 发送通知 (Email/Slack/Telegram 等)            │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
5. 展示阶段
   ┌─────────────────────────────────────────────────────┐
   │ 前端展示                                             │
   │   ├── Socket.io 接收 certInfo 事件                   │
   │   │   └── 存储到 this.tlsInfoList[monitorID]        │
   │   ├── Details.vue 显示摘要                           │
   │   │   ├── 剩余天数 (可点击展开)                      │
   │   │   ├── 过期日期                                   │
   │   │   └── 主机名不匹配警告图标                       │
   │   └── CertificateInfo.vue 显示详情                   │
   │       ├── 证书链 (递归显示)                          │
   │       ├── 主题、颁发者、指纹、有效期                  │
   │       └── 证书类型标签                               │
   └─────────────────────────────────────────────────────┘
```

### 8.2 域名过期监控完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           域名过期监控流程                                     │
└─────────────────────────────────────────────────────────────────────────────┘

1. 触发时机
   ┌─────────────────────────────────────────────────────┐
   │ 每次心跳成功后，检查 monitor.domainExpiryNotification │
   │ 是否已启用                                           │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
2. RDAP 服务器发现
   ┌─────────────────────────────────────────────────────┐
   │ getRdapDnsData()                                    │
   │   ├── 检查本地缓存 (一周有效)                        │
   │   ├── 无缓存则从 IANA 获取:                          │
   │   │   https://data.iana.org/rdap/dns.json          │
   │   └── 失败则使用内置数据 (extra/rdap-dns.json)      │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
3. 查询过期日期
   ┌─────────────────────────────────────────────────────┐
   │ DomainExpiry.checkExpiry(domainName)                │
   │   ├── 检查上次查询时间 (24小时缓存)                   │
   │   ├── getRdapDomainExpiryDate()                     │
   │   │   ├── 根据 TLD 查找 RDAP 服务器                  │
   │   │   ├── 请求: {rdapServer}/domain/{domain}        │
   │   │   └── 在 events 数组中查找 eventAction=expiration │
   │   ├── 更新 domain_expiry 表:                         │
   │   │   ├── expiry (过期日期)                          │
   │   │   └── lastCheck (上次查询时间)                   │
   │   └── 日期延后则清除 lastExpiryNotificationSent      │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
4. 告警检查
   ┌─────────────────────────────────────────────────────┐
   │ DomainExpiry.sendNotifications()                    │
   │   ├── 获取配置的通知天数 [7, 14, 21]                │
   │   ├── 升序排列，确保只发送最紧急的一次                │
   │   ├── 检查 daysRemaining <= targetDays              │
   │   ├── 检查 lastExpiryNotificationSent 防重复         │
   │   └── 发送通知                                       │
   └─────────────────────────────────────────────────────┘
                           │
                           ▼
5. 前端展示
   ┌─────────────────────────────────────────────────────┐
   │ 前端展示                                             │
   │   ├── Socket.io 接收 domainInfo 事件                 │
   │   │   └── 存储到 this.domainInfoList[monitorID]      │
   │   └── Details.vue 显示:                              │
   │       ├── 剩余天数                                   │
   │       └── 过期日期                                   │
   └─────────────────────────────────────────────────────┘
```

---

## 9. 关键技术点

### 9.1 时间处理

- **统一使用 UTC 时间**：所有日期计算都使用 `dayjs.utc()` 避免时区问题
- **证书有效期**：`validTo` 使用 Date 对象存储，`daysRemaining` 实时计算
- **缓存策略**：
  - RDAP DNS 数据：缓存 7 天
  - 域名过期查询：缓存 24 小时

### 9.2 证书链处理

- **递归解析**：通过 `issuerCertificate` 字段遍历完整证书链
- **类型识别**：
  - 链中第一个 = `server`
  - 有上级且非自签名 = `intermediate CA`
  - 自签名或无上级 = `root CA` 或 `self-signed`
- **指纹对比**：使用 `fingerprint256` 检测证书是否更新

### 9.3 防重复告警

- **证书**：使用 `notification_sent_history` 表，唯一键 `(type, monitor_id, days)`
- **域名**：使用 `domain_expiry.lastExpiryNotificationSent` 字段
- **证书更新**：指纹变更时自动清除历史记录

### 9.4 性能优化

- **缓存机制**：RDAP 数据、域名查询结果都有缓存
- **异步处理**：通知发送不阻塞主监控流程
- **按需查询**：只有启用了过期通知的监控才会执行相关检查

---

## 10. 配置与扩展

### 10.1 告警天数配置

通过 Settings 表配置，默认值：
- 证书：`[7, 14, 21]` 天
- 域名：`[7, 14, 21]` 天

### 10.2 支持的域名后缀

通过 IANA 的 RDAP DNS 数据动态支持，常见 TLD 如 `.com`, `.net`, `.org`, `.io` 等都支持。

### 10.3 支持的 STARTTLS 协议

- SMTP (邮件)
- IMAP (邮件)
- XMPP (即时通讯)

---

*文档生成时间：2026-05-04*
*基于 Uptime Kuma 代码库分析*
