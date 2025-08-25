# Cookie/Session/Token

---



## 什么是Cookie

---

​	前端请求的时候，如果后端中间件有设置Cookie，那么响应的HTTP Header会携带Set-Cookie，将cookie返回前端并缓存在本地；后续请求前端可以通过携带该cookie快速进行用户认证等操作，从而获取相关信息。

​	而Cookie存储的值是不固定的，类型也不一致，同时也有长度限制；除此外还可能存在CSRF攻击，安全性存在问题。cookie存储在本地，并不在服务端。

## 什么是Session

---

​	Session是存储用户信息的另一种方式，存储在服务端，或者redis中，在前端请求时，后端服务生成随机的sessionID，并将相关信息存储到session中，session在代码中的存储是结构体的类型或其他；sessionID绑定该session，后端将session保存在服务内存中或者缓存redis，将sessionID复制存在cookie中保存，返回给前端，下次前端请求的时候携带cookie，后端从cookie中取出sessionID并进行映射，取出对应的sessin，即可获取到相关的信息。

​	session的存储也是存在问题的，如果使用单机的redis存储数据，那需要保证其高可用，因此必须设置redis集群，朱从/哨兵等方式；增加了资源的消耗。同时也存在CSRF攻击以及Session重放攻击。

## 什么是Token

​	Token是利用服务端通过签名/时间戳/用户数据等信息进行加密，生成一串密文，返回给前端，前端下次请求时，通过设置头部携带该Token，客户端使用对应的签名进行解密，即可获取到用户数据等信息

​	这种方式减少了资源消耗，避免csrf攻击。

​	设计时可以通过返回两个token，一个用于验证用户，另一个是refresh_token，用于前面的token过期时，根据该token快速刷新用户数据生成token存储，refresh_token的过期时间会比正常的验证token的过期时间长。

## 什么是JWT

​	JWT，即`json web token`，

​	JWT由三部分组成，Header、Payload、Signature，

