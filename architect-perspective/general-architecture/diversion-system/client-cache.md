# 客户端缓存

::: tip 客户端缓存（Client Cache）

HTTP协议的无状态性决定了它必须依靠客户端缓存来解决网络传输效率上的缺陷。

:::

浏览器的缓存机制几乎是在万维网刚刚出现就已经存在，在HTTP协议设计之初，便确定了服务端与客户端之间“无状态”（Stateless）的交互原则，即要求每次请求是独立的，每次请求无法感知和依赖另一个请求的存在，这既简化了HTTP服务器的设计，也为其水平扩展能力留下了广袤的空间。但无状态并不只有好的一面，由于每次请求都是独立的，服务端不保存此前请求的状态和资源，所以也不可避免地导致其携带有重复的数据，造成网络性能降低。HTTP协议对此的解决方案就是客户端缓存，在HTTP从1.0到最新2.0版本的每次演进中，都提出过现在被称为“状态缓存”、“强制缓存”（许多资料中简称为“强缓存”）和“协商缓存”的缓存机制。

其中，状态缓存是指不经过服务器，客户端直接根据缓存信息对目标网站的状态判断，以前只有301/Moved Permanently（永久重定向）这一个；后来在[RFC6797](https://tools.ietf.org/html/rfc6797)中增加了[HSTS](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security)（HTTP Strict Transport Security）机制，用于避免依赖301/302跳转HTTPS时可能产生的降级中间人劫持（详细可见安全架构中的“[传输](../system-security/transport-security.html)”），这也属于另一种状态缓存。由于状态缓存所涉内容就只有这一点，后续我们就只聚焦于强制缓存与协商缓存两种机制。

## 强制缓存

只要是缓存，几乎都不可避免地会遇到一致性的问题。强制缓存对一致性处理就如它的名字一样，显得十分的直接粗暴，假设在某个时间点（譬如10分钟）之内，资源的内容和状态一定不会被改变，因此客户端可以无需经过任何浏览器请求，在该时间点来临前一直持有和使用该资源的本地缓存副本。

根据约定，强制缓存在用户在浏览器输入地址、页面链接跳转、新开窗口、前进/后退中均可生效，但在使用F5刷新页面时应当失效。有以下两类HTTP Header可以实现强缓存：

- **Expires**：Expires是HTTP/1.0协议中提供的Header（当然，在HTTP/1.1中同样存在），后面跟随一个截至时间参数。当服务器返回某个资源时带有该Header的话，意味着服务器承诺截止时间之前资源不会发生变动，浏览器可直接缓存该数据，不再重新发请求，示例：

  ```http
  HTTP/1.1 200 OK
  Expires: Wed, 8 Apr 2020 07:28:00 GMT
  ```

  Expires是HTTP协议最初版本的缓存机制，设计非常直观易懂，但考虑得并不够周全，它至少存在以下显而易见的问题：

  - 受限于客户端的本地时间。譬如，客户端修改了本地时间，可能会造成缓存提前失效或超期持有。
  - 无法处理涉及到用户身份的私有资源，譬如，某些资源被登录用户缓存在自己的浏览器上是合理的，但如果被CDN服务器缓存起来，则可能被其他未认证的用户所获取。
  - 无法描述“**不**缓存”的语义。譬如，浏览器为了提高性能，往往会自动在当次会话中缓存某些MINE类型的资源，在HTTP/1.0的服务器中就缺乏手段强制浏览器不允许缓存某个资源。以前为了实现这类功能，通常不得不使用Script脚本，在资源后面增加时间戳（如“xx.js?t=1586359920”）来保证每次资源都会重新获取。<br/>关于“不缓存”的语义，在HTTP/1.0中其实设计了“Pragma: no-cache”来实现，但Pragma在HTTP响应中的行为没有确切描述，随后就被HTTP/1.1中出现过的Cache-Control所替代，现在，尽管主流浏览器通常都会支持Pragma，但实际并没有什么使用价值了。

- **Cache-Control**：Cache-Control是HTTP/1.1协议中定义的强制缓存Header，它的语义比起Expires来说就丰富了很多，如果Cache-Control和Expires同时存在，并且语义存在冲突（Expires与max-age / s-maxage冲突）的话，必须以Cache-Control为准。Cache-Control的示例如下：

  ```http
  HTTP/1.1 200 OK
  Cache-Control: max-age=600
  ```

  Cache-Control在客户端的请求头或服务器的响应头中都可以使用，它定义了一系列的参数，且允许扩展（不在标准RFC协议中，由浏览器自行支持），其标准的参数主要包括有：

  - max-age / s-maxage：max-age后面跟随一个以秒为单位的数字，表明相对于请求时间（Date Header中也会注明请求时间），多少秒以内缓存是有效的，资源不需要重新从服务器中获取。相对时间避免了Expires中采用的绝对时间可能受客户端时钟影响的尴尬。s-maxage中的s是“Share”的缩写，意味“共享缓存”（即被CDN、代理等持有的缓存）有效时间，用于提示CDN这类服务器如何对缓存进行失效。
  - public / private：指明是否涉及到用户身份的私有资源，如果是public，着可以被代理、CDN等缓存，如果是private，着只能由客户端进行私有缓存。
  - no-cache / no-store：no-cache指明该资源不应该被缓存，哪怕是同一个会话中对同一个URL地址的请求，也必须从服务端获取（但协商缓存机制依然是生效的）；no-store不强制会话中相同URL资源的重复获取，但禁止浏览器、CDN等以任何形式保存该资源。
  - no-transform：禁止资源被任何形式地修改。譬如，某些CDN、透明代理支持自动GZIP压缩图片或文本，以提升网络性能，而no-transform就禁止了这样的行为，它要求Content-Encoding、Content-Range、Content-Type均不允许进行任何形式的修改。
  - min-fresh / only-if-cached：这两个参数是仅用于客户端的请求Header。min-fresh后续跟随一个以秒为单位的数字，用于建议服务器能返回一个不少于该时间的缓存资源（即包含max-age且不少于min-fresh的数字）。only-if-cached表示要求客户端要求不发送网络请求，只使用缓存来进行响应，若缓存不能命中，就直接返回503/Service Unavailable错误。
  - must-revalidate / proxy-revalidate：must-revalidate表示在资源过期后，一定需要从服务器中进行验证（即超过了max-age的时间，就等同于no-cache的行为），proxy-revalidate用于提示代理、CDN等缓存服务，语义与must-revalidate一致。


## 协商缓存

强制缓存是基于时效性的，但无论是人还是服务器，其实多数情况下都并没有什么把握去承诺某项资源多久不会发生变化。另外一种基于变化检测的缓存机制，在一致性上会有比强制缓存更好的表现，但需要一次变化检测的交互开销，性能上就会略差一些，这种基于检测的缓存机制，通常被称为“协商缓存”。另外，应注意在HTTP中协商缓存与强制缓存并没有排他性，这两套机制是并行工作的，譬如，当强制缓存存在时，直接从强制缓存中返回资源，无需进行变动检查；而当强制缓存超过时效，或者被禁止（no-cache / must-revalidate），协商缓存仍可以正常地工作。协商缓存主要有根据资源的修改时间或根据资源唯一标识是否发生变化来进行变动检查的机制，这都是靠一组成对出现的请求、响应Header来实现的：

- **Last-Modified和If-Modified-Since**：Last-Modified是服务器的响应Header，用于告诉客户端这个资源的最后修改时间。对于带有这个Header的资源，当客户端需要在此请求时，会通过If-Modified-Since把之前收到的资源最后修改时间发送回服务端。<br/>如果此时服务端发现资源在该时间后没有被修改过，就只要返回一个304/Not Modified的响应即可，无需附带消息体，如下所示：

  ```http
  HTTP/1.1 304 Not Modified
  Cache-Control: public, max-age=600
  Last-Modified: Wed, 8 Apr 2020 15:31:30 GMT
  ```

  如果此时服务端发现资源在该时间之后有变动，就会返回200/OK的完整响应，在消息体中包含最新的资源，如下所示：

  ```http
  HTTP/1.1 200 OK
  Cache-Control: public, max-age=600
  Last-Modified: Wed, 8 Apr 2020 15:31:30 GMT
  
  Content
  ```

- **Etag和If-None-Match**：Etag是服务器的响应Header，用于告诉客户端这个资源的唯一标识（HTTP服务器可以根据自己的意愿来选择如何生成这个标识，譬如Apache服务器的Etag值，默认是对文件的索引节点（INode），大小（Size）和最后修改时间（MTime）进行哈希计算后得到的），对于带有这个Header的资源，当客户端需要在此请求时，会通过If-None-Match把之前收到的资源唯一标识发送回服务端。<br/>如果此时服务端计算后发现资源的唯一标识与上传回来的一致，说明资源没有被修改过，就只要返回一个304/Not Modified的响应即可，无需附带消息体，如下所示：

  ```http
  HTTP/1.1 304 Not Modified
  Cache-Control: public, max-age=600
  Last-Modified: Wed, 8 Apr 2020 15:31:30 GMT
  ```

  如果此时服务端发现资源的唯一标识有变动，就会返回200/OK的完整响应，在消息体中包含最新的资源，如下所示：

  ``` http
  HTTP/1.1 200 OK
  Cache-Control: public, max-age=600
  Last-Modified: Wed, 8 Apr 2020 15:31:30 GMT
  
  Content
  ```

Etag是HTTP中一致性最强的缓存机制，譬如，Last-Modified标注的最后修改只能精确到秒级，如果某些文件在1秒钟以内，被修改多次的话，它将不能准确标注文件的修改时间；又或者如果某些文件会被定期生成，可能内容并没有任何变化，但Last-Modified却改变了，导致文件无法有效使用缓存，这些情况Last-Modified都有可能产生一致性问题，只能使用Etag解决。

Etag却又是HTTP中性能最差的缓存机制，体现在每次请求时，服务端都必须对资源进行哈希计算，这比起简单获取一下修改时间，开销要大了很多。Etag和Last-Modified是允许一起使用的，服务器会优先验证Etag，在Etag一致的情况下，再去对比Last-Modified，这是为了防止有一些HTTP服务器未将文件修改日期纳入哈希范围内。

到这里为止，HTTP的协商缓存机制已经能很好地处理通过URL获取**单个资源**的场景，“单个资源”是什么意思？在HTTP协议的设计中，一个URL地址有可能能够提供多份不同版本的资源，譬如，一段文字的不同语言版本，一个文件的不同编码格式版本，一份数据的不同压缩方式版本，等等。HTTP协议设计了Accept\*（Accept、Accept-Language、Accept-Charset、Accept-Encoding）的一套请求Header和对应的Content-\*（Content-Language、Content-Type、Content-Encoding）的响应Header，这被称为HTTP的内容协商机制。与之对应的，对于一个URL能够获取多个资源的场景中，缓存也同样也需要有明确的标识来获知根据什么内容来对同一个URL返回给用户正确的资源。这个就是Vary Header的作用，Vary后面可以跟随其他Header的名字，譬如：

```http
HTTP/1.1 200 OK
Vary: Accept, User-Agent
```

以上说明应该根据MINE类型和浏览器类型来缓存资源，获取资源时也需要根据请求头中对应的字段来筛选出适合的资源版本。

根据约定，协商缓存不仅在用户在浏览器输入地址、页面链接跳转、新开窗口、前进/后退中生效，而且在使用F5刷新页面时也同样是生效的，只有用户强制刷新（Ctrl+F5）或者禁用缓存（譬如在DevTools中设定）时才会失效，此时客户端向服务端发出的请求会自动带有“Cache-Control: no-cache”。