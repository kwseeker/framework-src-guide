# JWT 规范

JWT（Json Web Token）是一种无状态的跨域的认证规范[RFC7519](https://www.rfc-editor.org/info/rfc7519) 。



## 规范主要内容

这部分资料很多，不重复写了，只记录比较重要的东西。

### JWT原理

### JWT数据结构

- **Header**（头部，JSON对象）

  下面是一些约定的可选字段，也可以添加自定义字段。

  - alg

    算法类型

  - typ

    令牌类型

  - jku		

    报头参数（JWK Set URL）

  - jwk		

    JSON Web Key

  - kid		

    Key ID

  + x5u		

    X.509 URL

  + x5c		

    X.509 Certificate Chain

  + x5t		

    X.509 Certificate SHA-1 Thumbprint

  + x5t#S256		

    X.509 Certificate SHA-256 Thumbprint

  + cty		

    Content Type

  + crit	

    Critical

- **Payload**（负载，JSON对象）

  下面是一些约定的可选字段，也可以添加自定义字段。

  - iss

    签发人信息，大小写敏感，StringOrURI，可选

  - sub 

    主题，要么iss下本地惟一，要么全局惟一，大小写敏感，StringOrURI，可选

  - aud

    收件人，有区分大小写的字符串构成的数组， StringOrURI，可选

  - exp 

    失效时间，（可能有时差，可以留几分钟缓和余地）必须是number类型包含NumericDate值，可选

  - nbf 

    生效时间，（可能有时差，可以留几分钟缓和余地）必须是number类型包含NumericDate值，可选

  - iat 

    JWT 所颁发的时间，用于计算这个 JWT 已经使用的期限，必须是number类型包含NumericDate值，可选

  - jti 

    JWT 的唯一标识符，用于避免 JWT 被重复发送，大小写敏感，可选

- **Signature**（签名）

  ```javascript
  HMACSHA256(
    base64UrlEncode(header) + "." +
    base64UrlEncode(payload),
    secret)
  ```



## 基础知识

下面列举使用JWT必须掌握的知识点：

- **加密算法**

  - **消息认证码**

    - **[HMAC](https://zh.wikipedia.org/wiki/HMAC)**

      类似MD5摘要算法，

      常用实现：**HS256**(HMAC-SHA256)

  - **非对称加密**（公开密钥加密）

    - **[RSA](https://zh.wikipedia.org/wiki/RSA加密演算法)** (使用最广泛,三个人的名字首字母)

      利用极大整数难以因数分解的原理。

      NIST建议的RSA[密钥长度](https://zh.wikipedia.org/wiki/密钥长度)为至少2048位。

      常用实现：**RS256** (RSA-SHA256)

    - **[ECDSA](https://zh.wikipedia.org/wiki/ECDSA)**

      常用实现：**ES256**(ECDSA-SHA256)

    - [ElGamal](https://zh.wikipedia.org/wiki/ElGamal)

    - Rabin

    - [DSA](https://zh.wikipedia.org/wiki/数字签名算法)

  - **对称加密**（对称密钥加密）

    - **[AES](https://zh.wikipedia.org/wiki/高级加密标准)**
    - [ChaCha20](https://zh.wikipedia.org/wiki/Salsa20)
    - **[3DES](https://zh.wikipedia.org/wiki/3DES)**
    - [Salsa20](https://zh.wikipedia.org/wiki/Salsa20)
    - **[DES](https://zh.wikipedia.org/wiki/資料加密標準)**
    - [Blowfish](https://zh.wikipedia.org/wiki/Blowfish)
    - [IDEA](https://zh.wikipedia.org/wiki/國際資料加密演算法)
    - [RC5](https://zh.wikipedia.org/wiki/RC5)
    - [RC6](https://zh.wikipedia.org/wiki/RC6)
    - [Camellia](https://zh.wikipedia.org/wiki/Camellia)

- **JWS & JWE**

  是JWT的两种实现方式（具有不同的格式规范）。

  `JWS (JSON Web Signature)`的Token由`Header`、`Payload`、`Signature`三部分组成;

  分别用于存储基本信息（如类型typ、签名算法alg等）、业务相关信息（如JWT的签发者iss、面向用户sub、接收者aud、过期时间exp、签发时间戳iat、以及拓展的业务数据等）、签名字符串。

  JWS的主要目的是保证了数据在传输过程中不被修改，验证数据的完整性。

  `JWE (JSON Web Encryption)`的Token由`JOSE Header`、`JWE Encypted Key`、`Initialization Vector`、`Ciphertext`、`Authentication Tag`五部分组成。

  JWE相对于JWS将业务数据等信息也进行了加密，兼顾数据的安全性与完整性。



## 使用场景&优缺点

### 使用场景

- **一次性验证**

  如：邮件中的激活链接。

- **HTTP无状态身份认证**

### 优缺点

**优点**

- **服务端无状态**

  可以节省服务端内存占用（Token信息一般都是存内存数据库的）。也不需要连接数据库。

- **跨服务调用**

**缺点**

- **增加流量消耗、可能降低接口性能**

- **默认对负载信息不隐藏，不适合存敏感信息**

  如果包含敏感信息可以使用`JWE`，或者后端在生成签名前自行对负载信息加密。

**使用注意**

- **JWT token 泄漏怎么办？**

  前端选择`Cookie`存储而不是`LocalStorage`, 相对应的后端服务器设置`HttpOnly`属性；禁止客户端脚本访问Cookie，但是需要客户端浏览器支持。

  关于`HttpOnly`参考附录`Using HTTP cookies`。

- **用户注销或修改密码如何让JWT token立即失效？**

  比较好的实现有两个：

  1）**黑名单**（推荐）

  用户注销或修改密码后，将原来的token加入黑名单。因为注销和修改密码行为还是很少的，空间、性能损失也较很低。

  2）生成签名时不使用统一的密钥，而是使用用户相关的信息生成密钥

  比如使用用户ID、用户状态（是否注销）和用户密码等通过摘要算法等生成密钥。

- **JWT token 难续签问题**

  JWT的特性天然不支持续签，不可能每次请求重新生成token（性能问题）, 也不可能由前端在快过期时刷新token（前端操作时间具有不确定性）, 也不适合在后台存token过期时间（违背JWT设计初衷）。

  解决方案：

  1）**每次前端请求，后台都判断下过期时间是否临近是否需要刷新，需要刷新则生成新的token，放到Response的header中**（推荐）

  2）前端设置定时任务，过期前重新刷新token, 以旧换新（可行）

  3）借鉴OAuth2的RefreshToken额外颁发一个长期有效的RefreshToken（比如几天）

   只要RefreshToken不过期就可以刷新用户token，不过Refresh Token刷新必须判定用户的合法性，例如 UA、IP 等特征资料。这样的话还不如直接使用OAuth2的无状态身份校验。

  4）直接将token改为长期有效，增加像RefreshToken一样的校验规则，降低token泄漏带来的风险

+ **为了降低被盗用的可能性，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输**



## 定制内容

### 加密算法选择

参考附录，有人对比了一些常用算法的优缺点。

总结：

如果只是在可信任的服务器上做token生成和校验，只需要使用HS256就行了；如果校验过程发生在第三方服务器或客户端，则需要选择非对称加密，根据是否有签名数据量限制选择RSxxx还是Esxxx。



## 参考资料

- [加密](https://zh.wikipedia.org/wiki/加密)
- 《密码学原理与Java实现》
- [RSA算法原理(二)](https://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)
- [RFC7519 (JWT)](https://www.rfc-editor.org/info/rfc7519) [RFC7515 (JWS)](https://www.rfc-editor.org/info/rfc7515) [RFC7516 (JWE)](https://www.rfc-editor.org/info/rfc7516)
- [JWT的签名算法选择研究](http://www.bewindoweb.com/301.html)
- [Using HTTP cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)

