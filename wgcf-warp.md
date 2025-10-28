# WARP 双栈出口配置手册 (适用于纯IPv4和纯IPv6服务器)

**目标：** 提供两套经过优化的WireGuard配置文件模板，可以为任何纯IPv4或纯IPv6的Debian/Ubuntu服务器，快速添加稳定可靠的WARP双栈出口，并确保SSH连接的绝对安全。

**特性**：本手册将引导您从官方发布页面获取**最新版本**的 `wgcf` 程序，确保您始终使用最新的功能和修复。

---

### **第一部分：通用准备步骤 (所有服务器通用)**

在选择具体配置模板之前，请先在您的服务器上完成以下通用准备工作。

#### **1. 安装依赖**

```bash
sudo apt update
sudo apt install -y wireguard-tools curl
```

#### **2. 手动获取并安装最新版 wgcf**

为了确保您总是使用最新版本的`wgcf`，直接从GitHub的官方发布页面获取下载链接。

1.  **打开最新版发布页面**
    [**https://github.com/ViRb3/wgcf/releases/latest**](https://github.com/ViRb3/wgcf/releases/latest)

    ```bash
    # 示例: curl -L "https://github.com/ViRb3/wgcf/releases/download/v2.2.29/wgcf_2.2.29_linux_amd64" -o wgcf
    curl -L "[在此处粘贴您复制的最新版链接]" -o wgcf

    # 授予执行权限并安装
    chmod +x wgcf
    sudo mv wgcf /usr/local/bin/
    ```

#### **3. 注册并生成初始配置**

```bash
# 这将在您的当前目录下生成 wgcf-account.toml 和 wgcf-profile.conf
echo | wgcf register
wgcf generate
```

#### **4. 获取必要的WARP配置信息**

执行以下命令，并**复制** `PrivateKey` 和 `Address` (IPv6那一行) 的值。

```bash
cat wgcf-profile.conf
```

---

### **第二部分：选择适合您服务器的配置模板**

#### **模板 A：适用于【纯IPv6】服务器**

**目标**：为一台只有原生IPv6地址的服务器，添加IPv4和IPv6双栈出口。

1.  创建并打开最终的配置文件：
    ```bash
    sudo mkdir -p /etc/wireguard
    sudo nano /etc/wireguard/wgcf.conf
    ```
2.  将以下模板完整地粘贴到编辑器中，并替换必要的占位符：

    ```ini
    [Interface]
    PrivateKey = [请粘贴您复制的PrivateKey]
    Address = 172.16.0.2/32
    Address = [请粘贴您复制的WARP IPv6地址]/128

    DNS = 1.1.1.1, 2606:4700:4700::1111
    MTU = 1280
    
    # 自动查找并保护服务器的原生IPv6地址，确保SSH稳定
    PostUp = ip -6 rule add from $(ip -6 addr show scope global | grep inet6 | head -n 1 | awk '{print $2}' | cut -d'/' -f1) lookup main
    PostDown = ip -6 rule delete from $(ip -6 addr show scope global | grep inet6 | head -n 1 | awk '{print $2}' | cut -d'/' -f1) lookup main

    [Peer]
    PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
    AllowedIPs = 0.0.0.0/0
    AllowedIPs = ::/0
    
    # 纯IPv6服务器必须使用IPv6 Endpoint进行引导连接
    Endpoint = [2606:4700:d0::a29f:c001]:2408
    
    PersistentKeepalive = 25
    ```

---

#### **模板 B：适用于【纯IPv4】服务器**

**目标**：为一台只有原生IPv4地址的服务器，添加IPv4和IPv6双栈出口。

1.  创建并打开最终的配置文件：
    ```bash
    sudo mkdir -p /etc/wireguard
    sudo nano /etc/wireguard/wgcf.conf
    ```
2.  将以下模板完整地粘贴到编辑器中，并替换必要的占位符：

    ```ini
    [Interface]
    PrivateKey = [请粘贴您复制的PrivateKey]
    Address = 172.16.0.2/32
    Address = [请粘贴您复制的WARP IPv6地址]/128

    DNS = 1.1.1.1, 2606:4700:4700::1111
    MTU = 1280
    
    # 自动查找并保护服务器的原生IPv4地址，确保SSH稳定
    PostUp = ip -4 rule add from $(ip -4 addr show scope global | grep inet | head -n 1 | awk '{print $2}' | cut -d'/' -f1) lookup main
    PostDown = ip -4 rule delete from $(ip -4 addr show scope global | grep inet | head -n 1 | awk '{print $2}' | cut -d'/' -f1) lookup main

    [Peer]
    PublicKey = bmXOC+F1FxEMF9dyiK2H5/1SUtzH0JuVo51h2wPfgyo=
    AllowedIPs = 0.0.0.0/0
    AllowedIPs = ::/0
    
    # 纯IPv4服务器必须使用IPv4 Endpoint进行引导连接
    Endpoint = 162.159.192.1:2408
    
    PersistentKeepalive = 25
    ```

---

### **第三部分：【高级】自定义`AllowedIPs`以控制出口协议**

默认情况下，以上两个模板都会为您提供**双栈出口**。您可以根据自己的特定需求，通过注释掉 `AllowedIPs` 的某一行来精确控制出口行为。

#### **场景 1：我只需要IPv4出口，完全禁用IPv6出口**

*   **方法**：编辑您的 `wgcf.conf` 文件，在 `AllowedIPs = ::/0` 这一行的行首加上 `#` 号将其注释掉。

    ```ini
    [Peer]
    ...
    AllowedIPs = 0.0.0.0/0  # 这一行【启用】了所有IPv4流量通过隧道
    # AllowedIPs = ::/0      # 这一行被【禁用】，所有IPv6流量将无法通过隧道
    ...
    ```

#### **场景 2：我只需要IPv6出口，完全禁用IPv4出口**

*   **方法**：编辑您的 `wgcf.conf` 文件，在 `AllowedIPs = 0.0.0.0/0` 这一行的行首加上 `#` 号将其注释掉。

    ```ini
    [Peer]
    ...
    # AllowedIPs = 0.0.0.0/0  # 这一行被【禁用】，所有IPv4流量将无法通过隧道
    AllowedIPs = ::/0      # 这一行【启用】了所有IPv6流量通过隧道
    ...
    ```

---

### **第四部分：最终步骤 (所有服务器通用)**

#### **1. 设置系统网络优先级 (选择一个执行)**

在这里，您可以决定服务器默认优先使用哪个协议访问网络。

*   **选项 A：优先使用 IPv4 (推荐，兼容性最好)**
    执行此命令后，系统将优先通过WARP的IPv4地址访问网络。
    ```bash
    echo "precedence ::ffff:0:0/96  100" | sudo tee -a /etc/gai.conf
    ```

*   **选项 B：优先使用 IPv6**
    如果您之前设置了优先IPv4，执行此命令可以撤销该设置，恢复到Debian默认的“IPv6优先”策略。
    ```bash
    # 这条命令会安全地从配置文件中删除优先IPv4的规则（如果存在的话）
    sudo sed -i '/precedence ::ffff:0:0\/96  100/d' /etc/gai.conf
    ```

#### **2. 启动、验证并设置开机自启**
```bash
# 启动WireGuard服务
sudo wg-quick up wgcf

# 验证出口IP (根据您在第三部分的设置，部分命令可能不通)
curl -6 ifconfig.me
curl -4 ifconfig.me

# 确认无误后，设置开机自启
sudo systemctl enable wg-quick@wgcf
```