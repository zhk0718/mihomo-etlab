# 0. 简介

由于ETlab计算集群的dns存在未配置的情况，在需要联网的服务（例如安装python环境等）会出现问题，体现为对域名的访问会报错“未知的名称或服务”，而对IP的访问不受影响。具体体现为。

```shell
nslookup www.baidu.com
;; connection timed out; no servers could be reached


nslookup www.baidu.com 223.5.5.5
Server:         223.5.5.5
Address:        223.5.5.5#53

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com.
Name:   www.a.shifen.com
Address: 183.2.172.185
Name:   www.a.shifen.com
Address: 183.2.172.42
Name:   www.a.shifen.com
Address: 240e:ff:e020:9ae:0:ff:b014:8e8b
Name:   www.a.shifen.com
Address: 240e:ff:e020:966:0:ff:b042:f296
```

而利用代理方法是解决该问题一个方便快捷的方式，本文旨在介绍基于mihomo(clash.meta)的解决方案。

# 1. 在Linux中运行mihomo

该教程不仅适用于ETlab集群，也可运用于任意linux系统。本文仅介绍mihomo的实现方式，sing-box、*ray系用户自行解决。

## 1.1 mihomo下载

下载：[mihomo](https://github.com/MetaCubeX/mihomo/releases/download/Prerelease-Alpha/mihomo-linux-amd64-alpha-de19f92.gz)、[GeoSite.dat](https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat)、[GeoIP.dat](https://fastly.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat)文件。其中，可以自行在[Release](https://github.com/MetaCubeX/mihomo/releases)中下载mihomo的最新版本。下载完成后将这些文件置于同一文件夹中。

```shell
mkdir mihomo
cd mihomo
gunzip mihomo-*.gz
chmod +x mihomo*
```

![](C:\Users\zhk\AppData\Roaming\marktext\images\2024-11-14-16-11-56-image.png)

## 1.2 mihomo配置文件

编辑 ```config.yaml```文件，内容如下：

```yaml
# url 里填写自己的订阅,名称不能重复
proxy-providers:
  provider1:
    url: "你的订阅url"
    type: http
    interval: 86400
    health-check: {enable: true,url: "https://www.gstatic.com/generate_204", interval: 300}

proxies: 
  - name: "直连"
    type: direct
    udp: true

mixed-port: 7890 #此处自行修改一个未被占用的端口
ipv6: false
allow-lan: true
unified-delay: false
tcp-concurrent: true
#external-controller: 127.0.0.1:9090 #此处自行修改一个未被占用的端口

geodata-mode: true
geox-url:
  geoip: "https://mirror.ghproxy.com/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geoip-lite.dat"
  geosite: "https://mirror.ghproxy.com/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/geosite.dat"
  mmdb: "https://mirror.ghproxy.com/https://github.com/MetaCubeX/meta-rules-dat/releases/download/latest/country-lite.mmdb"
  asn: "https://mirror.ghproxy.com/https://github.com/xishang0128/geoip/releases/download/latest/GeoLite2-ASN.mmdb"

find-process-mode: strict
global-client-fingerprint: firefox

profile:
  store-selected: true
  store-fake-ip: true

sniffer:
  enable: true
  sniff:
    HTTP:
      ports: [80, 8080-8880]
      override-destination: true
    TLS:
      ports: [443, 8443]
    QUIC:
      ports: [443, 8443]
  skip-domain:
    - "Mijia Cloud"
    - "+.push.apple.com"
dns:
  enable: true
  ipv6: false
  respect-rules: true
  enhanced-mode: fake-ip
  fake-ip-filter:
    - "*"
    - "+.lan"
    - "+.local"
    - "+.market.xiaomi.com"
  nameserver:
    - https://120.53.53.53/dns-query
    - https://223.5.5.5/dns-query
  proxy-server-nameserver:
    - https://120.53.53.53/dns-query
    - https://223.5.5.5/dns-query
  nameserver-policy:
    "geosite:cn,private":
      - https://120.53.53.53/dns-query
      - https://223.5.5.5/dns-query
    "geosite:geolocation-!cn":
      - "https://dns.cloudflare.com/dns-query#默认"
      - "https://dns.google/dns-query#默认"

proxy-groups:

  - name: 默认
    type: select
    proxies: [自动选择,直连,香港,台湾,日本,新加坡,美国]

  #分隔,下面是地区分组
  - name: 香港
    type: url-test
    include-all: true
    tolerance: 10
    filter: "(?i)港|hk|hongkong|hong kong"

  - name: 台湾
    type: url-test
    include-all: true
    tolerance: 10
    filter: "(?i)台|tw|taiwan"

  - name: 日本
    type: url-test
    include-all: true
    tolerance: 10
    filter: "(?i)日|jp|japan"

  - name: 美国
    type: url-test
    include-all: true
    tolerance: 10
    filter: "(?i)美|us|unitedstates|united states"

  - name: 新加坡
    type: url-test
    include-all: true
    tolerance: 10
    filter: "(?i)(新|sg|singapore)"

  - name: 自动选择
    type: url-test
    include-all: true
    tolerance: 10

rules:
  #- GEOIP,lan,直连,no-resolve
  #- GEOSITE,CN,直连
  #- GEOIP,CN,直连
  #- MATCH,默认
  - MATCH,直连
```

其中，```mixed-port```的端口号需要自行更改。在拥有sudo权限的linux系统中，可以通过配置```tun```或```tproxy```实现透明代理，在防检测能力上更胜一筹。更多配置选项请参考[Wiki](https://wiki.metacubex.one)。

## 1.3 mihomo运行

准备好二进制文件```mihomo```、数据库文件```geosite.dat```、`geoip.dat`以及配置文件```config.yaml```后，在mihomo文件夹中运行二进制文件，成功启动的日志如下。

```shell
./mihomo* -f config.yaml -d ./
```

![](C:\Users\zhk\AppData\Roaming\marktext\images\2024-11-14-16-32-12-image.png)

而当出现端口占用情况时，日志通常会出现报错，这时候就需要修改配置文件中的```mixed-port```了。

```textile
ERRO[] Start Mixed(http+socks) server error: listen tcp :7890: bind: address already in use
```

为了让mihomo持续运行，需要将其挂在后台启动，可以通过```nohup```、```screen```或```pm2```等途径将其运行于后台，这里以nohup为例。

```shell
nohup ./mihomo* -f config.yaml -d ./ >> clash.log 2>&1 &
```

通过浏览clash.log文件判断是否成功在后台启动，或通过```ps```。

```shell
ps -ef | grep mihomo
```

![](C:\Users\zhk\AppData\Roaming\marktext\images\2024-11-14-16-37-35-image.png)

成功启动后，可以通过配置环境变量的方式应用代理，具体配置方法如下。

```shell
export http_proxy=http://10.0.0.1:7890 #将端口改为配置文件中设置的端口号
export https_proxy=http://10.0.0.1:7890
```

完成上述所有工作后，你应该可以在主节点访问所有国内域名了，也可以在任意节点```node0*```中访问互联网。可以通过```curl```进行测试对域名的访问情况，示例如下。

```shell
curl https://www.baidu.com
```

返回结果：

```html
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=https://ss1.bdstatic.com/5eN1bjq8AAUYm2zgoY3K/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus=autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn" autofocus></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=https://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');
                </script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

通过上述方法应该已经可以解决集群中的网络问题，现在可以将前面提到的环境变量写到```.bashrc```中来实现每次登录集群时自动应用代理了。

需要注意的是，mihomo并不会随着集群的重启而自动重启，因而当集群因断电重启后，需要重新运行。而在拥有sudo权限的linux系统中，可以通过```init.d```、```systemd```等服务实现mihomo的开机自启动，这里以```systemd```为例。

```shell
sudo nano /etc/systemd/system/mihomo.service
```

```textile
[Unit]
Description=mihomo
After=network.target

[Service]
Type=simple
Restart=always
ExecStart=/home/khz/mihomo/mohomo -f /home/khz/mihomo/config.yaml -d /home/khz/mihomo

[Install]
WantedBy=multi-user.target
```

## 1.4 在linux中访问外网（额外）

此节并非本文的主要探讨内容，有需要的请自行修改配置文件，需要修改的地方包括：

```yaml
# url 里填写自己的订阅,名称不能重复
proxy-providers:
  provider1:
    url: "你的订阅url"
    type: http
    interval: 86400
    health-check: {enable: true,url: "https://www.gstatic.com/generate_204", interval: 300}
```

在```url```选项中填写你的机场clash订阅地址；

```yaml
external-controller: 127.0.0.1:9090 #此处自行修改一个未被占用的端口
```

取消对```external-controller```的注释并修改端口号为一个未占用的；

```yaml
rules:
  - GEOIP,lan,直连,no-resolve
  - GEOSITE,CN,直连
  - GEOIP,CN,直连
  - MATCH,默认
  #- MATCH,直连
```

对rules的前面几行取消注释，并注释原有的最后一行。

完成上述工作后，重新启动mihomo。此时，mihomo应该会自动下载你的机场的订阅文件。根据配置的规则，启动后对国内域名会通过国内网络访问，国外域名会通过```默认-自动选择```节点，从机场配置文件的所有节点中按```url-test```得到的延迟选择最小的进行连接。可以通过```curl```检查对外网的连通性，示例如下。

```shell
curl https://www.google.com
```

```html
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage" lang="en-SG"><head><meta content="text/html; charset=UTF-8" http-equiv="Content-Type"><meta content="/images/branding/googleg/1x/googleg_standard_color_128dp.png" itemprop="image"><title>Google</title><script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){var _g={kEI:'rrw1Z96HAYSVseMP-sWG0Qk',kEXPI:'0,1303875,2396422,1087,448529,90132,2872,2891,8348,64702,94323,258423,8155,23351,22435,9779,8213,54444,76209,15816,1804,7734,27535,11813,1635,9708,19568,27083,5214194,8833731,5,38,13,3,1,35,7,1,6,1,6,1,19,16,2,14,6,4,1,6,1,6,1,7,8,1,7,6,1,55,26796958,1182080,16672,2169859,23029351,8163,4636,16436,2728,81317,11640,10983,15164,8182,17876,10969,20584,21671,6753,1914,5,21960,9138,3078,1522,328,546,3913,1766,23407,6,4577,5633,687,7852,24,21269,709,1347,13702,9104,6528,8139,6241,144,1380,1668,242,12565,797,378,11521,3172,1799,10667,1843,7691,1369,1888,597,1801,688,2,2625,3,2,2834,2943,3359,367,1672,41,2950,2645,3215,3072,864,314,102,477,1,1616,2742,525,2583,4,639,1,31,2706,567,1,1186,446,1,416,1837,3306,584,3789,939,1450,4779,888,260,4653,1114,458,392,1136,49,60,17,3,112,895,3,7,1154,529,38,558,1359,283,61,353,1401,1368,7,480,1651,1510,268,813,2,1398,1781,615,4,2,127,97,1321,8,529,2,1707,553,1208,1,687,3,383,1943,1317,372,514,95,619,370,3,1010,109,332,206,91,41,161,1234,392,1342,1381,1995,208,151,1123,49,392,53,423,776,452,1307,693,2,30,125,83,111,67,181,153,275,348,86,474,1422,998,730,213,43,1209,284,2,791,159,38,129,113,247,42,873,450,22,246,332,78,651,265,2,229,148,49,14,3,10,288,2067,13,288,830,10,529,442,1027,154,836,15,127,21402898,3,16997,18,2004,1478,868,3209,18,2766,1181',kBL:'HPw0',kOPI:89978449};(function(){var a;((a=window.google)==null?0:a.stvsc)?google.kEI=_g.kEI:window.google=_g;}).call(this);})();(function(){google.sn='webhp';google.kHL='en-SG';})();(function(){
var h=this||self;function l(){return window.google!==void 0&&window.google.kOPI!==void 0&&window.google.kOPI!==0?window.google.kOPI:null};var m,n=[];function p(a){for(var b;a&&(!a.getAttribute||!(b=a.getAttribute("eid")));)a=a.parentNode;return b||m}function q(a){for(var b=null;a&&(!a.getAttribute||!(b=a.getAttribute("leid")));)a=a.parentNode;return b}function r(a){/^http:/i.test(a)&&window.location.protocol==="https:"&&(google.ml&&google.ml(Error("a"),!1,{src:a,glmm:1}),a="");return a}
function t(a,b,c,d,k){var e="";b.search("&ei=")===-1&&(e="&ei="+p(d),b.search("&lei=")===-1&&(d=q(d))&&(e+="&lei="+d));d="";var g=b.search("&cshid=")===-1&&a!=="slh",f=[];f.push(["zx",Date.now().toString()]);h._cshid&&g&&f.push(["cshid",h._cshid]);c=c();c!=null&&f.push(["opi",c.toString()]);for(c=0;c<f.length;c++){if(c===0||c>0)d+="&";d+=f[c][0]+"="+f[c][1]}return"/"+(k||"gen_204")+"?atyp=i&ct="+String(a)+"&cad="+(b+e+d)};m=google.kEI;google.getEI=p;google.getLEI=q;google.ml=function(){return null};google.log=function(a,b,c,d,k,e){e=e===void 0?l:e;c||(c=t(a,b,e,d,k));if(c=r(c)){a=new Image;var g=n.length;n[g]=a;a.onerror=a.onload=a.onabort=function(){delete n[g]};a.src=c}};google.logUrl=function(a,b){b=b===void 0?l:b;return t("",a,b)};}).call(this);(function(){google.y={};google.sy=[];var d;(d=google).x||(d.x=function(a,b){if(a)var c=a.id;else{do c=Math.random();while(google.y[c])}google.y[c]=[a,b];return!1});var e;(e=google).sx||(e.sx=function(a){google.sy.push(a)});google.lm=[];var f;(f=google).plm||(f.plm=function(a){google.lm.push.apply(google.lm,a)});google.lq=[];var g;(g=google).load||(g.load=function(a,b,c){google.lq.push([[a],b,c])});var h;(h=google).loadAll||(h.loadAll=function(a,b){google.lq.push([a,b])});google.bx=!1;var k;(k=google).lx||(k.lx=function(){});var l=[],m;(m=google).fce||(m.fce=function(a,b,c,n){l.push([a,b,c,n])});google.qce=l;}).call(this);google.f={};(function(){
document.documentElement.addEventListener("submit",function(b){var a;if(a=b.target){var c=a.getAttribute("data-submitfalse");a=c==="1"||c==="q"&&!a.elements.q.value?!0:!1}else a=!1;a&&(b.preventDefault(),b.stopPropagation())},!0);document.documentElement.addEventListener("click",function(b){var a;a:{for(a=b.target;a&&a!==document.documentElement;a=a.parentElement)if(a.tagName==="A"){a=a.getAttribute("data-nohref")==="1";break a}a=!1}a&&b.preventDefault()},!0);}).call(this);</script><style>#gbar,#guser{font-size:13px;padding-top:1px !important;}#gbar{height:22px}#guser{padding-bottom:7px !important;text-align:right}.gbh,.gbd{border-top:1px solid #c9d7f1;font-size:1px}.gbh{height:0;position:absolute;top:24px;width:100%}@media all{.gb1{height:22px;margin-right:.5em;vertical-align:top}#gbar{float:left}}a.gb1,a.gb4{text-decoration:underline !important}a.gb1,a.gb4{color:#00c !important}.gbi .gb4{color:#dd8e27 !important}.gbf .gb4{color:#900 !important}
</style><style>body,td,a,p,.h{font-family:arial,sans-serif}body{margin:0;overflow-y:scroll}#gog{padding:3px 8px 0}td{line-height:.8em}.gac_m td{line-height:17px}form{margin-bottom:20px}.h{color:#1967d2}em{font-weight:bold;font-style:normal}.lst{height:25px;width:496px}.gsfi,.lst{font:18px arial,sans-serif}.gsfs{font:17px arial,sans-serif}.ds{display:inline-box;display:inline-block;margin:3px 0 4px;margin-left:4px}input{font-family:inherit}body{background:#fff;color:#000}a{color:#681da8;text-decoration:none}a:hover,a:active{text-decoration:underline}.fl a{color:#1967d2}a:visited{color:#681da8}.sblc{padding-top:5px}.sblc a{display:block;margin:2px 0;margin-left:13px;font-size:11px}.lsbb{background:#f8f9fa;border:solid 1px;border-color:#dadce0 #70757a #70757a #dadce0;height:30px}.lsbb{display:block}#WqQANb a{display:inline-block;margin:0 12px}.lsb{background:url(/images/nav_logo229.png) 0 -261px repeat-x;color:#000;border:none;cursor:pointer;height:30px;margin:0;outline:0;font:15px arial,sans-serif;vertical-align:top}.lsb:active{background:#dadce0}.lst:focus{outline:none}</style><script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){window.google.erd={jsr:1,bv:2116,de:true,dpf:'TxzeVgCgkdplkl9RXhNaZ9gnI7GpsEKiPUZPl6fFfsU'};
var g=this||self;var k,l=(k=g.mei)!=null?k:1,n,p=(n=g.sdo)!=null?n:!0,q=0,r,t=google.erd,v=t.jsr;google.ml=function(a,b,d,m,e){e=e===void 0?2:e;b&&(r=a&&a.message);d===void 0&&(d={});d.cad="ple_"+google.ple+".aple_"+google.aple;if(google.dl)return google.dl(a,e,d,!0),null;b=d;if(v<0){window.console&&console.error(a,b);if(v===-2)throw a;b=!1}else b=!a||!a.message||a.message==="Error loading script"||q>=l&&!m?!1:!0;if(!b)return null;q++;d=d||{};b=encodeURIComponent;var c="/gen_204?atyp=i&ei="+b(google.kEI);google.kEXPI&&(c+="&jexpid="+b(google.kEXPI));c+="&srcpg="+b(google.sn)+"&jsr="+b(t.jsr)+
"&bver="+b(t.bv);t.dpf&&(c+="&dpf="+b(t.dpf));var f=a.lineNumber;f!==void 0&&(c+="&line="+f);var h=a.fileName;h&&(h.indexOf("-extension:/")>0&&(e=3),c+="&script="+b(h),f&&h===window.location.href&&(f=document.documentElement.outerHTML.split("\n")[f],c+="&cad="+b(f?f.substring(0,300):"No script found.")));google.ple&&google.ple===1&&(e=2);c+="&jsel="+e;for(var u in d)c+="&",c+=b(u),c+="=",c+=b(d[u]);c=c+"&emsg="+b(a.name+": "+a.message);c=c+"&jsst="+b(a.stack||"N/A");c.length>=12288&&(c=c.substr(0,12288));a=c;m||google.log(0,"",a);return a};window.onerror=function(a,b,d,m,e){r!==a&&(a=e instanceof Error?e:Error(a),d===void 0||"lineNumber"in a||(a.lineNumber=d),b===void 0||"fileName"in a||(a.fileName=b),google.ml(a,!1,void 0,!1,a.name==="SyntaxError"||a.message.substring(0,11)==="SyntaxError"||a.message.indexOf("Script error")!==-1?3:0));r=null;p&&q>=l&&(window.onerror=null)};})();</script></head><body bgcolor="#fff"><script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){var src='/images/nav_logo229.png';var iesg=false;document.body.onload = function(){window.n && window.n();if (document.images){new Image().src=src;}
if (!iesg){document.f&&document.f.q.focus();document.gbqf&&document.gbqf.q.focus();}
}
})();</script><div id="mngb"><div id=gbar><nobr><b class=gb1>Search</b> <a class=gb1 href="https://www.google.com/imghp?hl=en&tab=wi">Images</a> <a class=gb1 href="https://maps.google.com.sg/maps?hl=en&tab=wl">Maps</a> <a class=gb1 href="https://play.google.com/?hl=en&tab=w8">Play</a> <a class=gb1 href="https://www.youtube.com/?tab=w1">YouTube</a> <a class=gb1 href="https://news.google.com/?tab=wn">News</a> <a class=gb1 href="https://mail.google.com/mail/?tab=wm">Gmail</a> <a class=gb1 href="https://drive.google.com/?tab=wo">Drive</a> <a class=gb1 style="text-decoration:none" href="https://www.google.com.sg/intl/en/about/products?tab=wh"><u>More</u> &raquo;</a></nobr></div><div id=guser width=100%><nobr><span id=gbn class=gbi></span><span id=gbf class=gbf></span><span id=gbe></span><a href="http://www.google.com.sg/history/optout?hl=en" class=gb4>Web History</a> | <a  href="/preferences?hl=en" class=gb4>Settings</a> | <a target=_top id=gb_70 href="https://accounts.google.com/ServiceLogin?hl=en&passive=true&continue=https://www.google.com/&ec=GAZAAQ" class=gb4>Sign in</a></nobr></div><div class=gbh style=left:0></div><div class=gbh style=right:0></div></div><center><br clear="all" id="lgpd"><div id="XjhHGf"><img alt="Google" height="92" src="/images/branding/googlelogo/1x/googlelogo_white_background_color_272x92dp.png" style="padding:28px 0 14px" width="272" id="hplogo"><br><br></div><form action="/search" name="f"><table cellpadding="0" cellspacing="0"><tr valign="top"><td width="25%">&nbsp;</td><td align="center" nowrap=""><input name="ie" value="ISO-8859-1" type="hidden"><input value="en-SG" name="hl" type="hidden"><input name="source" type="hidden" value="hp"><input name="biw" type="hidden"><input name="bih" type="hidden"><div class="ds" style="height:32px;margin:4px 0"><input class="lst" style="margin:0;padding:5px 8px 0 6px;vertical-align:top;color:#000" autocomplete="off" value="" title="Google Search" maxlength="2048" name="q" size="57"></div><br style="line-height:0"><span class="ds"><span class="lsbb"><input class="lsb" value="Google Search" name="btnG" type="submit"></span></span><span class="ds"><span class="lsbb"><input class="lsb" id="tsuid_1" value="I'm Feeling Lucky" name="btnI" type="submit"><script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){var id='tsuid_1';document.getElementById(id).onclick = function(){if (this.form.q.value){this.checked = 1;if (this.form.iflsig)this.form.iflsig.disabled = false;}
else top.location='/doodles/';};})();</script><input value="AL9hbdgAAAAAZzXKvj14tGb4OLmHi522CDY3dfrIcEvO" name="iflsig" type="hidden"></span></span></td><td class="fl sblc" align="left" nowrap="" width="25%"><a href="/advanced_search?hl=en-SG&amp;authuser=0">Advanced search</a></td></tr></table><input id="gbv" name="gbv" type="hidden" value="1"><script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){var a,b="1";if(document&&document.getElementById)if(typeof XMLHttpRequest!="undefined")b="2";else if(typeof ActiveXObject!="undefined"){var c,d,e=["MSXML2.XMLHTTP.6.0","MSXML2.XMLHTTP.3.0","MSXML2.XMLHTTP","Microsoft.XMLHTTP"];for(c=0;d=e[c++];)try{new ActiveXObject(d),b="2"}catch(h){}}a=b;if(a=="2"&&location.search.indexOf("&gbv=2")==-1){var f=google.gbvu,g=document.getElementById("gbv");g&&(g.value=a);f&&window.setTimeout(function(){location.href=f},0)};}).call(this);</script></form><div style="font-size:83%;min-height:3.5em"><br><div id="gws-output-pages-elements-homepage_additional_languages__als"><style>#gws-output-pages-elements-homepage_additional_languages__als{font-size:small;margin-bottom:24px}#SIvCob{color:#474747;display:inline-block;line-height:28px;}#SIvCob a{padding:0 3px;}.H6sW5{display:inline-block;margin:0 2px;white-space:nowrap}.z4hgWe{display:inline-block;margin:0 2px}</style><div id="SIvCob">Google offered in:  <a href="https://www.google.com/setprefs?sig=0_wa8f0LP4L92pVkhtwjaOipgmwCc%3D&amp;hl=zh-CN&amp;source=homepage&amp;sa=X&amp;ved=0ahUKEwjelpmyu9uJAxWESmwGHfqiIZoQ2ZgBCAY">&#20013;&#25991;(&#31616;&#20307;)</a>    <a href="https://www.google.com/setprefs?sig=0_wa8f0LP4L92pVkhtwjaOipgmwCc%3D&amp;hl=ms&amp;source=homepage&amp;sa=X&amp;ved=0ahUKEwjelpmyu9uJAxWESmwGHfqiIZoQ2ZgBCAc">Bahasa Melayu</a>    <a href="https://www.google.com/setprefs?sig=0_wa8f0LP4L92pVkhtwjaOipgmwCc%3D&amp;hl=ta&amp;source=homepage&amp;sa=X&amp;ved=0ahUKEwjelpmyu9uJAxWESmwGHfqiIZoQ2ZgBCAg">&#2980;&#2990;&#3007;&#2996;&#3021;</a>  </div></div></div><span id="footer"><div style="font-size:10pt"><div style="margin:19px auto;text-align:center" id="WqQANb"><a href="/intl/en/ads/">Advertising</a><a href="http://www.google.com.sg/intl/en/services/">Business Solutions</a><a href="/intl/en/about.html">About Google</a><a href="https://www.google.com/setprefdomain?prefdom=SG&amp;prev=https://www.google.com.sg/&amp;sig=K_nCtSTIKCWjixn5t-ZdD17TYSf70%3D">Google.com.sg</a></div></div><p style="font-size:8pt;color:#70757a">&copy; 2024 - <a href="/intl/en/policies/privacy/">Privacy</a> - <a href="/intl/en/policies/terms/">Terms</a></p></span></center><script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){window.google.cdo={height:757,width:1440};(function(){var a=window.innerWidth,b=window.innerHeight;if(!a||!b){var c=window.document,d=c.compatMode=="CSS1Compat"?c.documentElement:c.body;a=d.clientWidth;b=d.clientHeight}
if(a&&b&&(a!=google.cdo.width||b!=google.cdo.height)){var e=google,f=e.log,g="/client_204?&atyp=i&biw="+a+"&bih="+b+"&ei="+google.kEI,h="",k=[],l=window.google!==void 0&&window.google.kOPI!==void 0&&window.google.kOPI!==0?window.google.kOPI:null;l!=null&&k.push(["opi",l.toString()]);for(var m=0;m<k.length;m++){if(m===0||m>0)h+="&";h+=k[m][0]+"="+k[m][1]}f.call(e,"","",g+h)};}).call(this);})();</script>  <script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){google.xjs={basecomb:'/xjs/_/js/k\x3dxjs.hp.en.0OTD4tJ8n0M.es5.O/ck\x3dxjs.hp.CGnO94bKjEo.L.X.O/am\x3dAgAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQQAAAAAAAAAAgAEAwDAAAAAEAAQIAAAAAAAAAAAAAABEAAAKAMIEAAAgviMAEACLAADwAg/d\x3d1/ed\x3d1/dg\x3d0/ujg\x3d1/rs\x3dACT90oGMieAQwUsLDRSM-RJLIaUO3op7TQ',basecss:'/xjs/_/ss/k\x3dxjs.hp.CGnO94bKjEo.L.X.O/am\x3dAgAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAAAAAAwAAAAAAEAAQIAAAAAAAAAAAAAABEAAAKAMIE/rs\x3dACT90oECsDFHPd6YovTRozPj0Sy9crOc1g',basejs:'/xjs/_/js/k\x3dxjs.hp.en.0OTD4tJ8n0M.es5.O/am\x3dAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAgAEAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgviMAEACLAADwAg/dg\x3d0/rs\x3dACT90oFI78IKXK26Gq3C7fjWUtmwoF7udA',excm:[]};})();</script>       <script nonce="V8bVZVZ8ck1if7PvTAR5gw">(function(){var u='/xjs/_/js/k\x3dxjs.hp.en.0OTD4tJ8n0M.es5.O/am\x3dAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAAAAAAAAgAEAADAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgviMAEACLAADwAg/d\x3d1/ed\x3d1/dg\x3d3/rs\x3dACT90oFI78IKXK26Gq3C7fjWUtmwoF7udA/m\x3dsb_he,d';var st=1;var amd=1000;var mmd=0;var pod=true;var e=typeof Object.defineProperties=="function"?Object.defineProperty:function(a,b,c){if(a==Array.prototype||a==Object.prototype)return a;a[b]=c.value;return a},f=function(a){a=["object"==typeof globalThis&&globalThis,a,"object"==typeof window&&window,"object"==typeof self&&self,"object"==typeof global&&global];for(var b=0;b<a.length;++b){var c=a[b];if(c&&c.Math==Math)return c}throw Error("a");},h=f(this),k=function(a,b){if(b)a:{var c=h;a=a.split(".");for(var d=0;d<a.length-1;d++){var g=a[d];if(!(g in
c))break a;c=c[g]}a=a[a.length-1];d=c[a];b=b(d);b!=d&&b!=null&&e(c,a,{configurable:!0,writable:!0,value:b})}};k("globalThis",function(a){return a||h});
var l=this||self;function m(){var a,b,c;if(b=a=(b=window.google)==null?void 0:(c=b.ia)==null?void 0:c.r.B2Jtyd)b=a.m,b=b===1||b===5;return b&&a.cbfd!=null&&a.cbvi!=null?a:void 0};function n(){var a=[u];if(!google.dp){for(var b=0;b<a.length;b++){var c=a[b],d=document.createElement("link");d.setAttribute("as","script");d.setAttribute("href",c);d.setAttribute("rel","preload");document.body.appendChild(d)}google.dp=!0}};
var p=globalThis.trustedTypes,q;function r(){var a=null;if(!p)return a;try{var b=function(c){return c};a=p.createPolicy("goog#html",{createHTML:b,createScript:b,createScriptURL:b})}catch(c){}return a};var t=function(a){this.g=a};t.prototype.toString=function(){return this.g+""};function w(a,b){if(b instanceof t)b=b.g;else throw Error("b");a.src=b;var c;b=a.ownerDocument&&a.ownerDocument.defaultView||window;b=b===void 0?document:b;var d;b=(d=(c="document"in b?b.document:b).querySelector)==null?void 0:d.call(c,"script[nonce]");(c=b==null?"":b.nonce||b.getAttribute("nonce")||"")&&a.setAttribute("nonce",c)};var x=function(){var a=document;var b="SCRIPT";a.contentType==="application/xhtml+xml"&&(b=b.toLowerCase());return a.createElement(b)};function y(a){a=a===null?"null":a===void 0?"undefined":a;q===void 0&&(q=r());var b=q;return new t(b?b.createScriptURL(a):a)};google.ps===void 0&&(google.ps=[]);function z(){var a=u,b=function(){};google.lx=google.stvsc?b:function(){A(a);google.lx=b};google.bx||google.lx()}function B(a,b){b&&w(a,y(b));var c=a.onload;a.onload=function(d){c&&c(d);google.ps=google.ps.filter(function(g){return a!==g})};google.ps.push(a);document.body.appendChild(a)}google.as=B;function A(a){google.timers&&google.timers.load&&google.tick&&google.tick("load","xjsls");var b=x();b.onerror=function(){google.ple=1};b.onload=function(){google.ple=0};google.xjsus=void 0;B(b,a);google.aple=-1;google.dp=!0};function C(a){var b=a.getAttribute("jscontroller");return(b==="UBXHI"||b==="R3fhkb"||b==="TSZEqd")&&a.hasAttribute("data-src")}function D(){for(var a=document.getElementsByTagName("img"),b=0,c=a.length;b<c;b++){var d=a[b];if(d.hasAttribute("data-lzy_")&&Number(d.getAttribute("data-atf"))&1&&!C(d))return!0}return!1}for(var E=document.getElementsByTagName("img"),F=0,G=E.length;F<G;++F){var H=E[F];Number(H.getAttribute("data-atf"))&1&&C(H)&&(H.src=H.getAttribute("data-src"))};var I,J,K,L,M,N;function O(){google.xjsu=u;l._F_jsUrl=u;M=function(){z()};I=!1;J=(st===1||st===3)&&!!google.caft&&!D();K=m();L=(st===2||st===3)&&!!K&&!D();N=pod}function P(){I||J||L||(M(),I=!0)}setTimeout(function(){google&&google.tick&&google.timers&&google.timers.load&&google.tick("load","xjspls");O();if(J||L){if(J){var a=function(){J=!1;P()};google.caft(a);window.setTimeout(a,amd)}L&&(a=function(){L=!1;P()},K.cbvi.push(a),window.setTimeout(a,mmd));N&&(I||n())}else M()},0);})();window._ = window._ || {};window._DumpException = _._DumpException = function(e){throw e;};window._s = window._s || {};_s._DumpException = _._DumpException;window._qs = window._qs || {};_qs._DumpException = _._DumpException;(function(){var t=[2,16384,0,0,0,0,0,33554432,65,0,1638401,671887456,1343488,8208,1664,0,4456448,168306688,76,585608,9111568,770048];window._F_toggles = window._xjs_toggles = t;})();window._F_installCss = window._F_installCss || function(css){};(function(){google.jl={bfl:0,dw:false,ine:false,ubm:false,uwp:true,vs:false};})();(function(){var pmc='{\x22d\x22:{},\x22sb_he\x22:{\x22agen\x22:false,\x22cgen\x22:false,\x22client\x22:\x22heirloom-hp\x22,\x22dh\x22:true,\x22ds\x22:\x22\x22,\x22fl\x22:true,\x22host\x22:\x22google.com\x22,\x22jsonp\x22:true,\x22msgs\x22:{\x22cibl\x22:\x22Clear Search\x22,\x22dym\x22:\x22Did you mean:\x22,\x22lcky\x22:\x22I\\u0026#39;m Feeling Lucky\x22,\x22lml\x22:\x22Learn more\x22,\x22psrc\x22:\x22This search was removed from your \\u003Ca href\x3d\\\x22/history\\\x22\\u003EWeb History\\u003C/a\\u003E\x22,\x22psrl\x22:\x22Remove\x22,\x22sbit\x22:\x22Search by image\x22,\x22srch\x22:\x22Google Search\x22},\x22ovr\x22:{},\x22pq\x22:\x22\x22,\x22rfs\x22:[],\x22stok\x22:\x22I-YY9m5GhqZDjaqNaOK-rQgQdp0\x22}}';google.pmc=JSON.parse(pmc);})();(function(){var b=function(a){var c=0;return function(){return c<a.length?{done:!1,value:a[c++]}:{done:!0}}};
var e=this||self;var g,h;a:{for(var k=["CLOSURE_FLAGS"],l=e,n=0;n<k.length;n++)if(l=l[k[n]],l==null){h=null;break a}h=l}var p=h&&h[610401301];g=p!=null?p:!1;var q,r=e.navigator;q=r?r.userAgentData||null:null;function t(a){return g?q?q.brands.some(function(c){return(c=c.brand)&&c.indexOf(a)!=-1}):!1:!1}function u(a){var c;a:{if(c=e.navigator)if(c=c.userAgent)break a;c=""}return c.indexOf(a)!=-1};function v(){return g?!!q&&q.brands.length>0:!1}function w(){return u("Safari")&&!(x()||(v()?0:u("Coast"))||(v()?0:u("Opera"))||(v()?0:u("Edge"))||(v()?t("Microsoft Edge"):u("Edg/"))||(v()?t("Opera"):u("OPR"))||u("Firefox")||u("FxiOS")||u("Silk")||u("Android"))}function x(){return v()?t("Chromium"):(u("Chrome")||u("CriOS"))&&!(v()?0:u("Edge"))||u("Silk")}function y(){return u("Android")&&!(x()||u("Firefox")||u("FxiOS")||(v()?0:u("Opera"))||u("Silk"))};var z=v()?!1:u("Trident")||u("MSIE");y();x();w();var A=!z&&!w(),D=function(a){if(/-[a-z]/.test("ved"))return null;if(A&&a.dataset){if(y()&&!("ved"in a.dataset))return null;a=a.dataset.ved;return a===void 0?null:a}return a.getAttribute("data-"+"ved".replace(/([A-Z])/g,"-$1").toLowerCase())};var E=[],F=null;function G(a){a=a.target;var c=performance.now(),f=[],H=f.concat,d=E;if(!(d instanceof Array)){var m=typeof Symbol!="undefined"&&Symbol.iterator&&d[Symbol.iterator];if(m)d=m.call(d);else if(typeof d.length=="number")d={next:b(d)};else throw Error("b`"+String(d));for(var B=[];!(m=d.next()).done;)B.push(m.value);d=B}E=H.call(f,d,[c]);if(a&&a instanceof HTMLElement)if(a===F){if(c=E.length>=4)c=(E[E.length-1]-E[E.length-4])/1E3<5;if(c){c=google.getEI(a);a.hasAttribute("data-ved")?f=a?D(a)||"":"":f=(f=
a.closest("[data-ved]"))?D(f)||"":"";f=f||"";if(a.hasAttribute("jsname"))a=a.getAttribute("jsname");else{var C;a=(C=a.closest("[jsname]"))==null?void 0:C.getAttribute("jsname")}google.log("rcm","&ei="+c+"&tgtved="+f+"&jsname="+(a||""))}}else F=a,E=[c]}window.document.addEventListener("DOMContentLoaded",function(){document.body.addEventListener("click",G)});}).call(this);</script></body></html>
```

如果想要自行选择节点，可以参考Wiki的[api部分](https://wiki.metacubex.one/api/#_10)进行选择，示例如下。

```shell
curl -X PUT 'http://127.0.0.1:9095/proxies/默认' -d '{"name": "新加坡"}'
```

更详细的请参照[Wiki](https://wiki.metacubex.one/config)来修改配置文件。完成上述所有操作后，应该就可以恢复集群中的网络访问，并在linux中访问外网了。


