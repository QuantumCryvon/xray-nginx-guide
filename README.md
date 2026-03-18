# xray nginx all in 443 配置示例

## 📌 使用说明

- `your.domain.com` → 你的主域名(建议开启CDN)
  
- `reality.a.com` → 你的 Reality 伪装域名
  
  ## 🧩 Xray 配置
  
  > ⚠️ **重要：复制使用前，请删除所有 `//` 后面的注释，否则配置会报错！**
  

```
{
  "log": {
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log",
    "loglevel": "warning"
  },
  "dns": {
    "queryStrategy": "UseIPv4",
    "servers": [
      "1.1.1.1",
      "8.8.8.8",
      {
        "address": "https+local://1.1.1.1/dns-query",
        "skipFallback": false
      }
    ]
  },
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "port": 10001,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "email": "user@example.com",
            "id": "uuid",      //执行"xray uuid"生成
            "level": 0
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "xhttp",
        "xhttpSettings": {
          "host": "",
          "mode": "auto",    //这里建议设置为auto兼容3种模式
          "path": "/xhttp-path"
        }
      }
    },
    {
      "listen": "127.0.0.1",
      "port": 10002,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "flow": "xtls-rprx-vision",
            "id": "uuid"     //执行"xray uuid"生成，也可使用上面的uuid
          }
        ],
        "decryption": "none"
      },
      "sniffing": {
        "destOverride": [
          "http",
          "tls",
          "quic"
        ],
        "enabled": true,
        "routeOnly": true
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "dest": "reality.a.com:443",//请填入你的伪装域名
          "privateKey": "PrivateKey",//执行"xray x25519"生成
          "serverNames": [
            "reality.a.com"
          ],
          "shortIds": [
            "yourShortIds"//执行"openssl rand -hex 8"生成
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ],
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "block",
        "type": "field"
      },
      {
        "ip": [
          "geoip:cn"
        ],
        "outboundTag": "block",
        "type": "field"
      },
      {
        "domain": [
          "geosite:category-ads-all"
        ],
        "outboundTag": "block",
        "type": "field"
      }
    ]
  }
}
```

### **🛠 参数生成**

```
在服务器执行

生成 UUID：  
xray uuid  

生成 Reality 密钥：  
xray x25519  

生成 shortId：  
openssl rand -hex 8
```

## 🌐 Nginx 配置

> ✅ nginx 支持 `#` 注释，不需要删除注释内容**

```
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
    # multi_accept on;
}

#基于SNI分流
stream {
    map $ssl_preread_server_name $backend {
        reality.a.com reality; #reality伪装域名
        your.domain.com web_xray; #填入你的域名
        default drop;
    }

    upstream reality {
        server 127.0.0.1:10002; #reality端口
    }

    upstream web_xray {
        server 127.0.0.1:10000; #web_xray端口
    }

    upstream drop {
        server 0.0.0.0:1; #丢弃数据包
    }

    server {
        listen 443 reuseport;#监听443tcp
        proxy_pass $backend;
        ssl_preread on;
    }
}
###########################

#HTTP 及 HTTPS 主体配置

###########################
http {
    ## 基本设置
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    server_tokens off;

    ## 日志设置
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    ## Gzip 设置（可选，根据需求打开）
    gzip on;
    # gzip_vary on;
    # gzip_proxied any;
    # gzip_comp_level 6;
    # gzip_buffers 16 8k;
    # gzip_http_version 1.1;
    # gzip_types
    #     text/plain
    #     text/css
    #     application/json
    #     application/javascript
    #     text/xml
    #     application/xml
    #     application/xml+rss
    #     text/javascript;

    #################################################################
    # 1. HTTP 默认站
    #################################################################
    server {
        listen 80;
        listen [::]:80;
        server_name your.domain.com;#填入你的域名

        # 强制跳转到 HTTPS
        return 301 https://$host$request_uri;
    }

    #################################################################
    # 2. 域名 HTTPS 配置：
    #################################################################
    server {
        listen 10000 ssl http2;
        listen [::]:10000 ssl http2;
        server_name your.domain.com;#填入你的域名
        #填入你证书路径
        ssl_certificate /home/admin/cert/domain.crt;
        #填入你证书私钥路径
        ssl_certificate_key /home/admin/cert/domain.key;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers off;

        add_header Strict-Transport-Security "max-age=63072000" always;

        root /home/admin/webpage;#填入你的网页文件路径
        index index.html;
        
        #################################################################
        #  3. xray配置部分                                               #
        #################################################################
        #下方有两种写法，第一种只可以使用xhttp packet-up模式
        #第二种可以使用xhttp所有模式
        #如果你决定使用其中一种配置，请删除另一种配置示例
        #vless-xhttp 示例配置1
        location /xhttp-path {
            proxy_pass http://127.0.0.1:10001;# 填入你xray监听的地址和端口
            proxy_http_version 1.1;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_redirect off;
        }
        #vless-xhttp 示例配置2
        location /xhttp-path {
            grpc_buffer_size 16k;
            grpc_connect_timeout 60s;
            grpc_read_timeout 3600s;
            grpc_send_timeout 3600s;
            grpc_socket_keepalive on;
            grpc_pass grpc://127.0.0.1:10001; # 填入你xray监听的地址和端口
            grpc_set_header Host $host;
            grpc_set_header X-Real-IP $remote_addr;
            grpc_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            client_max_body_size 0;
            proxy_redirect off;
        }

       ###########################
       # 其他所有路径 → 尝试找静态文件，找不到返回 主页面
       ###########################
       location / {
        try_files $uri $uri/ /index.html;
       }
}
```

##### 🧠 原理解析

```
数据包发往服务器443端口----->nginx监听443端口
                               |
                               |
                   sni是否为reality伪装域名or你的域名-----否---->丢弃数据包
                        |                 |
                        |                 |                   
 转发本地10002端口<-reality伪装域名       你的域名
 交由xray处理流量                         |
                                          |
                                   转发本地10000端口
                                          |
                                          |
   返回index.html<---否------是否为xhttp设定路径"/xhttp-path"
                                          |
                                          是
                                          |
                                          |
                                转发至本地10001端口
                                交由xray处理流量
```

**“如有错误或改进建议，欢迎指正，本配置已在个人环境中测试可用。”** ✅

💬 **发现问题或想讨论？请到：  
https://github.com/QuantumCryvon/xray-nginx-guide/issues**
