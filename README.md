# vaycat
一款Web存活+PoC探测的扫描器
## 简介
重复造轮子的思路参考自B哥的 [**bscan**](https://github.com/broken5/bscan) 项目，体验了几个月的**bscan**之后觉得这样的设计思路有很大的提升空间，所以尝试重新写了个（也是为了学习一下Go语言），扫描器也保留了**bscan**的常用使用方式（使用方式相对来说几乎一致），由于刚学Go语言，扫描器使用过程中可能会有超多BUG
## 使用方式
使用 `-help` 或 `-h` 查看帮助信息
![image.png](https://cdn.nlark.com/yuque/0/2021/png/12501780/1625890605609-1ca76851-27fb-49ec-85fa-0abd907e9e5d.png#align=left&display=inline&height=295&margin=%5Bobject%20Object%5D&name=image.png&originHeight=590&originWidth=750&size=499775&status=done&style=stroke&width=375)
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
### 配置文件说明
配置文件相比**bscan**来说，增删了些配置
```yaml
threads: 50
# 每秒请求数
max_qps: 99999
# 是否启用poc扫描
poc_enabled: true
# 相关路径配置，支持相对路径或者绝对路径
fingerprint_path: /path/to/fingerprint.yml
blacklist_path: /path/to/blacklist.yml
# PoC配置，会自动扫描该目录下的所有文件及子目录下的所有.yml文件
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
  # 配置代理，格式：IP:PORT
  proxy: ""
  headers:
    Accept: '*/*'
    Accept-Language: en
    User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.66 Safari/537.36
    Cookie: rememberMe=test;

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
