1.首先下载脚本：
wget https://install.direct/go.sh
2.然后执行安装脚本
sudo bash go.sh
3.安装完之后，使用以下命令启动 V2Ray
systemctl start v2ray
4.v2ray 的配置文件位于 /etc/v2ray/config.json。v2ray 相对于 shadowsocks 更复杂的地方就在于其有众多的配置选项，针对不同的应用场景有不同的配置方案，从而实现不同的功能，而 shadowsocks 则相对傻瓜一些。要详细讲述 v2ray 的所有配置选项是需要很长的内容的，本文针对被 TCP 阻断的 VPS 来实现 vpn 的场景。下面简单介绍一些配置内容：

VMess

VMess 协议是由 V2Ray 原创并使用于 V2Ray 的加密传输协议，如同 Shadowsocks 一样为了对抗墙的深度包检测而研发的。在 V2Ray 上客户端与服务器的通信主要是通过 VMess 协议通信。

V2Ray 使用 inbound(传入) 和 outbound(传出) 的结构，这样的结构非常清晰地体现了数据包的流动方向，同时也使得 V2Ray 功能强大复杂的同时而不混乱，清晰明了。形象地说，我们可以把 V2Ray 当作一个盒子，这个盒子有入口和出口(即 inbound 和 outbound)，我们将数据包通过某个入口放进这个盒子里，然后这个盒子以某种机制（这个机制其实就是路由，后面会讲到）决定这个数据包从哪个出口吐出来。以这样的角度理解的话，V2Ray 做客户端，则 inbound 接收来自浏览器数据，由 outbound 发出去(通常是发到 V2Ray 服务器)；V2Ray 做服务器，则 inbound 接收来自 V2Ray 客户端的数据，由 outbound 发出去(通常是如 Google 等想要访问的目标网站)。

mKCP

V2Ray 引入了 KCP 传输协议，并且做了一些不同的优化，称为 mKCP。 mKCP 使用 UDP 来模拟 TCP 连接，这样即使 vps 被 TCP 阻断了，还是能够通过 UDP 来连接。mKCP 牺牲带宽来降低延迟，传输同样的内容，mKCP 一般比 TCP 消耗更多的流量，但是对于我购买的 vps 流量一般都用的很少，每个月用不完 十分之一，所以采用 mKCP 对流量消耗也没有太大的问题。

服务端采用 vmess + mKCP 的配置时，配置文件 /etc/v2ray/config.json 的内容如下：

{
  "inbounds": [
    {
      "port": 40827,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "505f001d-4aa8-4519-9c54-6b65749ee3fb",
            "alterId": 64
          }
        ]
      },
      "streamSettings": {
        "network": "mkcp",
        "kcpSettings": {
          "uplinkCapacity": 5,
          "downlinkCapacity": 100,
          "congestion": true,
          "header": {
            "type": "none"
          }
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
修改完配置文件后，需要重新启动 v2ray：

# systemctl restart v2ray
最后，还需要修改 vps 的防火墙设置，将对应的 udp 端口放行：

# firewall-cmd --zone=public --add-port=40827/udp --permanent
# firewall-cmd --reload
再设置 v2ray 开机自启动，修改 /etc/rc.local 文件，添加：

systemctl restart v2ray
服务端配置完毕。
