# tc-openvpn

參見http://serverfault.com/a/780233/209089

**OpenVPN調用`tc`腳本通過（流量控制）對單個客戶端進行流量整形**.

在`tc.sh`具有以下功能的腳本中處理流量控制設置：

- 使用指令由OpenVPN呼叫：`up`，`down`，`client-connect`和`client-disconnect`
- 所有設置都通過環境變量傳遞
- 理論上最多支持`/16`子網（最多65534個客戶端）
- 使用散列過濾器進行過濾以進行非常快速的大規模過濾
- 過濾器和類僅當前連接的客戶端設置，並分別添加和刪除而不會影響其他`tc`使用唯一標識符的設置（`hashtables`，`handles`，`classids`）。這些標識符是從客戶端的遠程VPN IP的後16位生成的
- 根據CN名稱（客戶端證書通用名稱）對客戶端進行單獨的限制/限制
- 客戶端設置存儲在包含他們的"訂閱類"文件（`bronze`，`silver`和`gold`），使用其他類只需編輯腳本，並根據需要進行修改
- 連接客戶端時，可以從外部應用程序即時修改"訂閱類"和相應的數據速率（"頻寬"）
----------

Configuration
---

**OpenVPN服務器配置[`/etc/openvpn/tc/conf`](https://github.com/ThanatosDi/tc-openvpn/blob/master/server.conf):**

用正確的IP地址替換最後兩行中的DNS服務器

**頻寬控制腳本 [`/etc/openvpn/tc/tc.sh`](https://github.com/rda0/tc-openvpn/blob/master/tc.sh):**

使它可執行：

    chmod +x /etc/openvpn/tc/tc.sh

**訂閱數據庫目錄 `/etc/openvpn/tc/db/`:**

該目錄包含每個客戶端文件，該文件以其**CN名稱**（包含"預訂類"字符串）命名，其配置如下：:

<!-- language: bash -->

    mkdir -p /etc/openvpn/tc/db
    echo bronze > /etc/openvpn/tc/db/client1
    echo silver > /etc/openvpn/tc/db/client2
    echo gold > /etc/openvpn/tc/db/client3

**IP數據庫目錄 `/etc/openvpn/tc/ip/`:**

此目錄將包含`CN-name <-> IP-address`關係和`tun interface`運行時，`tc`在客戶端連接時，外部應用程序必須提供該關係並更新設置

<!-- language: bash -->

    mkdir -p /etc/openvpn/tc/ip

它看起來如下：

    root@ubuntu:/etc/openvpn/tc/ip# ls -l
    -rw-r--r-- 1 root root    9 Jun  1 08:31 client1.ip
    -rw-r--r-- 1 root root    9 Jun  1 08:30 client2.ip
    -rw-r--r-- 1 root root    9 Jun  1 08:30 client3.ip
    -rw-r--r-- 1 root root    5 Jun  1 08:25 dev
    root@ubuntu:/etc/openvpn/tc/ip# cat *
    10.8.0.2
    10.8.1.0
    10.8.2.123
    tun0

**啟用IP轉發:**

<!-- language: bash -->

    echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
    sysctl -p

**配置NAT（網路地址轉換）：**

如果您有靜態外部IP地址，請使用`SNAT`：

<!-- language: bash -->

    iptables -t nat -A POSTROUTING -s 10.8.0.0/16 -o <if> -j SNAT --to <ip>

或者，如果您有動態分配的IP地址，請使用`MASQUERADE`（慢一些）：

<!-- language: bash -->

    iptables -t nat -A POSTROUTING -s 10.8.0.0/16 -o <if> -j MASQUERADE

而

 - **`<if>`** 是外部接口的名稱（即`eth0`）
 - **`<ip>`** 是外部接口的IP地址

----------

腳本用法和顯示tc配置
---
**`tc`從外部應用程序更新"訂閱類"和設置：**

當OpenVPN服務器啟動且客戶端連接時，發出以下命令（例如，升級`client1`到`"gold"`訂閱的範例）：

<!-- language: bash -->

    echo gold > /etc/openvpn/tc/db/client1
    /etc/openvpn/tc/tc.sh update client1

**`tc` 顯示設置的命令:**

    tc -s qdisc show dev tun0
    tc class show dev tun0
    tc filter show dev tun0

----------

附加信息
---

**注意事項和可能的優化：**
- 腳本和`tc`設置僅在少數客戶端上進行了測試
- 必須同時進行大量客戶端流量的大規模測試，並且可能`tc`必須優化設置
- 我不完全了解入口設置的工作方式。ifb如本[答案所述][2]，可能應該使用接口對它們進行優化。


**Related documentation for a deeper understanding:**

- [Traffic Control HOWTO][3]
- [Linux Advanced Routing & Traffic Control HOWTO][4] (especially chapter 9-12)
- [HTB Linux queuing discipline manual - user guide][5] (very good explanation of `htb` qdisc)
- [TC manpage][6]
- [Identifying tc filters for `add` and `del` operations][7]
- [OpenVPN 2.3 manpage][8]


  [1]: http://lartc.org/howto/lartc.adv-filter.hashing.html
  [2]: http://serverfault.com/a/386791/209089
  [3]: http://linux-ip.net/articles/Traffic-Control-HOWTO/index.html
  [4]: http://lartc.org/howto/index.html
  [5]: http://luxik.cdi.cz/~devik/qos/htb/manual/userg.htm
  [6]: http://lartc.org/manpages/tc.txt
  [7]: https://bugzilla.kernel.org/show_bug.cgi?id=14875
  [8]: https://community.openvpn.net/openvpn/wiki/Openvpn23ManPage
