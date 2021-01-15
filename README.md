

# HTTPS防抓包机制

[TOC]

## 一、常用的抓包工具

### 1.1 抓包工具

* `Charles`
* `Fiddler`

#### 1.1.1 `charles`

官网：[www.charlesproxy.com](www.charlesproxy.com)

安装步骤：

> 1. `PC`端安装`Charles`根证书：`help --> SSLProxying --> Install Charles Root Ceriticate`;
> 2. 安装`Charles`根证书到手机：`help --> SSLProxying --> Install Charles Root Ceriticate on a Mobile Device or Remote Browser`.

 **注意**：安装证书过程需要手机`wifi`设置电脑`IP`地址代理，否则不会下载证书。

* 在手机浏览器中访问地址`chls.pro/ssl`，下载并安装`Charles`根证书；
* `PC`端口设置代理`https`端口（域名地址）

`Proxy --> SSL Proxying Settings`

`HTTP`抓包示例：

![image](https://github.com/tianyalu/NePreventHttpsPacketCapture/raw/master/show/http_packet_capture.png)

#### 1.1.2 `Fiddler`

官网：[https://www.telerik.com/download/fiddler](https://www.telerik.com/download/fiddler)

安装步骤：

>1. 确保`Android`设备和安装`Fiddler`的电脑连接到同一个`Wifi AP`上；
>2. 配置`Fiddler`抓取并解密`HTTPS`包：`Tools --> Fiddler Option --> HTTPS`选项卡勾选`Capture HTTPS CONNECTs` 和`Decrypt HTTPS traffic`；由于通过`Wifi`远程连过来，所以在下面的选项中选择`..from remote clients only`；切换到`Connections`选项卡修改监听端口，勾选上`Allow remote computer to connect`；
>3. 设置`Android`设备，添加上代理服务器；
>4. 导证书到`Android`设备；
>5. 打开设备自带的浏览器，在地址栏输入代理服务器的`IP`和端口导入`FiddlerRoot certificate`。

`HTTPS`抓包示例：

![image](https://github.com/tianyalu/NePreventHttpsPacketCapture/raw/master/show/https_packet_capture.png)

## 二、中间人攻击原理

### 2.1 基础概念

#### 2.1.1 `TCP/IP`分层

`TCP/IP`的分层共分为四层：应用层、传输层、网络层、数据链路层。

> 1. 应用层：想用户提供营养服务时的通讯活动（`ftp`、`dns`、`http`）；
> 2. 传输层：网络连接中两台计算机的数据传输（`tcp`、`udp`）;
> 3. 网络层：处理网络上流动的数据包，通过怎样的传输路径把数据包传送给对方（`ip`）;
> 4. 数据链路层：与硬件相关的网卡、设备驱动等等。

#### 2.2.2 `HTTP/HTTPS`

`HTTP`：`HyperText Transfer Protocol`(超文本传输协议)，被用于在`web`浏览器和网站服务器之间传递信息，在`TCP/IP`中处于应用层。

> 1. 通信使用明文，内容可能被窃听；
> 2. 不验证通信方的身份，因此可能遭遇伪装；
> 3. 无法证明报文的完整性，所以有可能遭到篡改。

`HTTPS`：`HTTPS`中的`S`表示`SSL`或者`TLS`，就是在原`HTTP`的基础上加上一层用于数据加密、界面、身份认证的安全层。

> `HTTP`+加密+认证+完整性保护 = `HTTPS`

#### 2.1.3 `HTTPS`单向认证

![image](https://github.com/tianyalu/NePreventHttpsPacketCapture/raw/master/show/https_one_way_authentication.png)

#### 2.1.4 `HTTPS`双向认证

![image](https://github.com/tianyalu/NePreventHttpsPacketCapture/raw/master/show/https_two_way_authentication.png)

#### 2.1.5 抓包原理

![image](https://github.com/tianyalu/NePreventHttpsPacketCapture/raw/master/show/packet_capture_theory.png)

## 三、`https`防抓包手段

### 3.1 网络代理

#### 3.1.1 代理检测

* 检测是否使用网络代理
* 将网络库（如`OKHttp`库）设置为无代理模式，不走系统代理

```java
// HttpURLConnection:
URL url = new URL(urlStr);
urlConnection = (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);

// OkHttp:
OkHttpClient client = new OkHttpClient().newBuilder().proxy(Proxy.NO_PROXY).build();
```

#### 3.1.2 防御破解

```java
try {
  var URL = Java.use("java.net.URL");
  URL.openConnection.overload('java.net.Proxy').implementation = function() {
    return this.openConnection();
  }
} catch(e) {
  console.log("" + e);
}

try {
  var Builder = Java.use("okhttp3.OkHttpClient$Builder");
  var mybuilder = Builder.$new();
  Builder.proxy.overload('java.net.Proxy').implementation = function(arg1) {
    return mybuilder;
  }
} catch(e) {
  console.log("" + e);
}
```

### 3.2 证书固定

#### 3.2.1 `SSL-Pinning`

* 证书锁定（`Certificate Pinning`）: 在客户端代码内置仅接受指定域名的证书，而不接受操作系统或浏览器内置的`CA`根证书对应的任何证书；-->弊端：证书有效期问题
* 公钥锁定（`Public key Pinning`）：提取证书中的公钥并内置到客户端中，通过与服务器对比公钥值来验证链接的正确性。

#### 3.2.2 破解`SSL-Pinning`

`Xposed`框架 + `justTrustMe`插件

* `Xposed`框架：`Android`上应用广泛的`HOOK`框架，基于`Xposed`框架制作的外挂模块可以`hook`任意应用层的`java`函数，修改函数实现；
* `justTrustMe`插件：`justTrustMe`插件是一个用来禁用、绕过`SSL`证书检查的基于`Xposed`模块，将`Android`系统中所有用于校验`SSL`证书的`API`证书的`API`都进行了`Hook`，从而绕过证书检查。

### 3.3 对抗`HOOK`

* 检测`HOOK`：检测`Xposed`、`Frida`、`Substrate`等`HOOK`框架；
* 使用`Socket`连接：使用`Socket`走`TCP/UDP`，防止被应用层抓包；
* 传输数据加密：协议字段加密传输，并因此秘钥，应用层加固；
* `native`层传输：将网络传输逻辑写到`jni`层实现，提高反编译门槛。