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

**打开最新版发布页面**  
<https://github.com/ViRb3/wgcf/releases/latest>

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

```bash
cat wgcf-profile.conf
```

---

### **第二部分：选择适合您服务器的配置模板**

---

#### **模板 A：适用于【纯IPv6】服务器**

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
Endpoint = [2606:4700:d0::a29f:c001]:2408
PersistentKeepalive = 25
```

---

#### **模板 B：适用于【纯IPv4】服务器**

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
Endpoint = 162.159.192.1:2408
PersistentKeepalive = 25
```

---

### **第三部分：【高级】自定义 AllowedIPs**

#### ✅ 只开启 IPv4 出口
```ini
AllowedIPs = 0.0.0.0/0
# AllowedIPs = ::/0
```

#### ✅ 只开启 IPv6 出口
```ini
# AllowedIPs = 0.0.0.0/0
AllowedIPs = ::/0
```

---

### **第四部分：系统网络优先级设置**

#### **选项 A：优先使用 IPv4**
```bash
echo "precedence ::ffff:0:0/96  100" | sudo tee -a /etc/gai.conf
```

#### **选项 B：恢复 IPv6 默认优先**
```bash
sudo sed -i '/precedence ::ffff:0:0\/96  100/d' /etc/gai.conf
```

---

### **第五部分：启动与验证**

```bash
sudo wg-quick up wgcf

# 验证出口
curl -6 ifconfig.me
curl -4 ifconfig.me

# 设置开机启动
sudo systemctl enable wg-quick@wgcf
```

---
