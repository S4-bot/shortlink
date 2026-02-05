# tj
自学java微服务项目历程
https://b11et3un53m.feishu.cn/wiki/space/7210409107579469825?ccm_open_type=lark_wiki_spaceLink&open_tab_from=wiki_home

day01

一、项目的搭建模拟企业开发环境
1.配置了虚拟机的网络为192.168.150.101

2.由于访问naocs和mq需要输入端口和域名，不容易记忆，所以通过在Windows上配置。
2.1需要借助工具switchhosts来修改，这样只能方便域名，端口还是要输入。(C:\Windows\System32\drivers\etc\hosts)
2.2所以要通过ngix反向代理的特点去配置。（/usr/local/src/nginx/conf/nginx.conf）
2.3现在就可以输入配置的域名不需要端口就可以访问。

二、持续集成环境
<img width="1290" height="496" alt="image" src="https://github.com/user-attachments/assets/e71d3f35-21f6-499e-aadb-5cae241972d5" />
需要三个角色jenkins（获取，编译，构建，部署代码）、gogs（git私服）、docker(部署)
<img width="1300" height="587" alt="image" src="https://github.com/user-attachments/assets/590a7030-4a55-45f5-a30c-46a03ca1213b" />

遇到个问题，gogs进不去。。。。。
研究了快两个小时，原来是我用记事本添加配置他没生效，最后下载的Switchhosts管理员运行才生效。有点无语。

三、测试
一般都是用swagger测试
又是打不开的问题，希望这次能快点解决。这次是对应的微服务没打开。

四、本地部署
并没有什么问题，就是在git上拉取代码

五、修复BUG
来熟悉项目，熟悉了nacos的共享配置
但是其中提到了mq,去学习一下。
查看源码的时候，先向前端网络中查看请求，再去网管里找到对应的路由。
作比较时用equals。如果要比较的值明确是-128>=l && l<=127。如果超出这个范围，会创建一个新的值，再用==比较就是比较的地址。
如果要让开发环境网关路由到本地，只需要在本地启动服务，然后将开发环境的服务器权重设置为0
学习了如何用github去管理项目

