# vaycat

一款Web存活+PoC探测的扫描器

## 简介

重复造轮子的思路参考自B哥的 [**bscan**](https://github.com/broken5/bscan) 项目，体验了几个月的**bscan**之后觉得这样的设计思路有很大的提升空间，所以尝试重新写了个（也是为了学习一下Go语言），扫描器也保留了**bscan**的常用使用方式（使用方式相对来说几乎一致），由于刚学Go语言，扫描器使用过程中可能会有超多BUG

## 使用方式

查看帮助信息

```bash
vaycat -help
```

执行如下命令生成默认配置文件(如果相应配置文件已存在，不会覆盖已有文件)
```bash
vaycat -init
```
使用 `-config` 指定配置文件的相对路径或绝对路径(如当前目录下存在 `config.yml` 文件，可忽略此参数)，例如：
```bash
vaycat -config /path/to/config.yml
```
开始一次扫描
```bash
vaycat -target targets.txt
```
或者（0.4.0版本后添加了命令行-host参数）
```bash
vaycat -host 119.11.22.1/24
```
### 配置文件说明
配置文件相比**bscan**来说，增删了些配置
```yaml
threads: 50
# 每秒请求数
max_qps: 99999
# 是否启用poc扫描
poc_enabled: true
# 指纹库路径配置，支持相对路径或者绝对路径
fingerprint_path: /path/to/fingerprint.yml
# 黑名单路径配置，支持相对路径或者绝对路径
blacklist_path: /path/to/blacklist.yml
# PoC路径配置，会自动扫描该目录下的所有文件及子目录下的所有.yml文件
pocs_path: /path/to/pocs/
# 端口配置，支持范围配置
ports: [80-90, 300, 443, 591, 593, 832, 888, 981, 1010, 1311, 2082,
        2087, 2095, 2096, 2480, 3000, 3128, 3333, 4243, 4567, 4711,
        4712, 4993, 5000, 5104, 5108, 5800, 6543, 7000, 7396, 7443,
        7474, 8000, 8001, 8008, 8014, 8016, 8042, 8069, 8080-8091,
        8118, 8123, 8172, 8222, 8243, 8280, 8281, 8333, 8443,
        8500, 8834, 8880, 8888, 8983, 9000, 9043, 9060, 9080,
        9090, 9091, 9200, 9443, 9800, 9981, 12443, 16080, 18080,
        18092, 20720, 28017]

# 请求参数
request:
  timeout: 15
  allow_redirects: true
  # 最大重定向次数
  max_redirects_count: 10
  # 0.3.0 版本新增重试次数
  max_retry_count: 3
  # 配置代理，格式：IP:PORT(或者[http|https]://IP:PORT)
  proxy: ""
  headers:
    Accept: '*/*'
    Accept-Language: en
    User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36
    Cookie: rememberMe=test
  # 0.4.0 版本新增不允许访问的 hostname 列表配置
  hostname_disallowed:
    - '*google*'
    - '*github*'

# ceye.io反连平台配置
reverse:
  api_key: 21232f297a57a5a743894a0e4a801fc3
  domain: xxx.ceye.io
```
### 指纹识别
在 `response` 下新增了 `icon_hash` 变量，用于匹配网页 `favicon.ico` 的hash值，例如匹配 `SpringBoot` 指纹：
```yaml
- application: spring-boot
  lang: java
  expression: response.icon_hash.contains("116323821")
```
在 `response` 下新增了 `title` 变量，用于匹配网页的标题属性
```yaml
- application: 用友NC
  expression: >
    response.body.bcontains(b"uclient.yonyou.com") ||
    response.body.bcontains(b"UClient") ||
    response.title.icontains("YonYou NC") ||
    response.icon_hash.contains("1085941792")
```
### 黑名单机制
与指纹识别的 `expression` 一致，例如排除Aliyun的云存储
```yaml
- name: Aliyun-Bucket
  expression: response.body.bcontains(b".aliyuncs.com</HostId>")
```
从 0.2.0 版本起，命令行新增了 `-filter` 参数，使用时如果包含引号需要注意， `-filter` 命令行使用方式如下
```bash
vaycat -target target.txt -filter response.body.bcontains(b'.aliyuncs.com</HostId>')
```

### PoC模块
兼容[xray](https://github.com/chaitin/xray)项目的**PoC**语法（兼容性未全面测试，特别是反连平台效果），只需要在原有的**PoC**配置中添加 `filter_expr` 配置，即可使用**xray**的**PoC**，这里使用如上的**用友NC**指纹作为例子：
```yaml
name: poc-yaml-yonyou-nc-rce-2021-0602
rules:
  - method: GET
    path: "/servlet/~ic/bsh.servlet.BshServlet"
    expression: >
      response.status==200 &&
      response.body.bcontains(b'name="bsh.script"') &&
      response.body.bcontains(b'</TEXTAREA>')
detail:
  author: vaycore
  Affected Version: "Yonyou NC 6.5"
  fofa: icon_hash=1085941792

# 匹配到的指纹 application 变量中包含"用友NC"时，加载该PoC配置
filter_expr: fp.application.contains("用友NC")
```
无图无真相，PoC扫描结果如下

![image.png](https://cdn.nlark.com/yuque/0/2021/png/12501780/1625919672796-9d803819-9d00-4969-8dca-9dae34e47fa2.png#align=left&display=inline&height=299&margin=%5Bobject%20Object%5D&name=image.png&originHeight=398&originWidth=806&size=182515&status=done&style=stroke&width=605)

#### 新增Paths配置

0.3.0版本后，在rules配置中新增了paths配置，典型的例子是**用于爆破备份文件**

```yaml
name: backup
set:
  domain: request.url.domain
rules:
  - method: GET
    paths:
      - /web.zip
      - /{{domain}}.zip
      - /www.zip
      - /bin.zip
      - /ROOT.zip
      - /ROOT.war
      - /tmp.zip
    headers:
      User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.121 Safari/537.36
      Referer: https://www.baidu.com/
      Accept-Language: en
      Accept-Encoding: gzip, deflate
      Connection: Keep-Alive
      Range: bytes=0-10240
    expression: >
      (response.status == 200 || response.status == 206) &&
      int(response.headers["Content-Length"]) > 1024 &&
      (response.headers["Content-Type"].icontains("application/zip") ||
      response.headers["Content-Type"].icontains("application/octet-stream"))

```

0.3.0修改了-poc命令行参数使用方式，对于如上不指定 `filter_expr` 的PoC，可使用参数指定运行当前PoC，例如：

```bash
vaycat -target target.txt -poc /path/to/bak.yml
```

#### 新增host命令行参数

0.4.0版本后，在命令行新增了 `-host` 参数，可直接在命令行指定扫描的目标，支持的格式如下

```
http(s)://x.com
http(s)://x.com:8080
x.com
x.com:8080
192.168.1.1,192.168.1.2
192.168.1.1-255、192.168.1.1-192.168.1.255
192.168.1.1/24
```

## 辅助功能
目前只有一个小功能，后续会慢慢添加更多有意思的辅助功能
### hash
获取网站 `favicon.ico` 的hash值
```bash
vaycat -hash https://www.baidu.com/favicon.ico
```
执行结果

![image.png](https://cdn.nlark.com/yuque/0/2021/png/12501780/1625921208012-d329365f-0464-44d7-a59a-9e6a0e936768.png#align=left&display=inline&height=161&margin=%5Bobject%20Object%5D&name=image.png&originHeight=215&originWidth=634&size=52126&status=done&style=stroke&width=476)
## 后续版本

- 修复BUG、优化性能
- 增加分布式扫描
- 由于代码是瞎写的，写的无法直视，暂不考虑开源
