# Java Web 网上商城后台

## 一、使用的技术

- Spring Boot
- MyBatis
- Redis
- MySQL
- Git

## 二、组成模块

- 用户模块
    - 登录
    - 登出
    - 注册
    - 忘记密码
    - 重置密码
    - Redis缓存用户登录信息,用于多服务器session共享
- 商品分类模块
    - 添加分类
    - 获取分类
    - 修改分类信息
- 商品模块
    - 商品详情
    - 商品列表
    - 管理员上架商品
    - 编辑商品信息
- 购物车模块
    - 获取购物车
    - 添加商品到购物车
    - 从购物车删除商品
    - 修改购买商品数量
    - 全选/全反选
- 订单模块
    - 创建订单
    - 取消订单
    - 获取订单详情
    - 获取订单列表
    - 集成支付宝的面对面支付模块
    - 查询订单支付状态
    - 管理员订单管理
- 收货地址模块
    - 添加收货地址
    - 更新收货地址
    - 删除收货地址
    - 获取所有收货地址
- 支付宝接入模块
    - 请求支付宝预支付获取支付二维码
    - 处理支付宝回调,判断支付是否成功
    - 查询订单的支付状态
    
## 三、重要部分实现细节

### 1. Redis实现单点登录

在单一的服务器中实现登录的常用方法是在用户登录后将用户的信息存入session中, 然后每次需要校验用户是否登录时将用户对象从session中取出来判断是否为空.

但是如果部署多台web服务器, 并使用Nginx的负载均衡将用户请求分配到指定的web服务器上时这种做法就会出现问题. 比如如果用户第一次请求时Nginx将此次请求分配到A服务器上, A服务器将用户信息存入session中. 用户第二次请求时Nginx将此次请求分配到B服务器上, 由于A服务器session中的用户登录信息B服务器并不能获取. 而且B服务器中并没有用户登录的session信息. 那么B服务器就会认为用户没有登录, 从而让用户再次登录, 所以用户的体验就不好.

使用Redis的目的是解决多服务器之间session共享问题. 当用户首次登录后, 将此次的登录的session ID存入cookie中, 并且使用这个session ID作为键, 将用户对象序列化为字符串作为值存入Redis中. 每当用户请求任一服务器时, 服务器将请求中的cookie中保存的session ID拿到, 然后根据这个session ID从Redis中获取对应的用户序列化之后的JSON字符串并反序列化成用户对象. 这样就解决了多服务器之间的单点登录问题.

### 2. 集成支付宝当面付

支付宝官方文档中的当面付执行流程如下:

![](https://github.com/yibo141/ssmcrud/raw/master/images/alipay_f2f.png)

首先向支付宝请求预下单, 支付宝会返回一个支付二维码的链接, 将此链接转换为二维码图片并展示给用户. 如果用户扫描二维码图片支付后, 服务器接受支付宝的回调, 校验并判断支付是否成功, 如果成功则更改订单的支付状态, 失败则向客户端返回错误信息.