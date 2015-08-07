`` 对NGX_HTTP_UPSTREAM_DYNAMIC_MODULE的分析：
``

# 内容： 

## 一：配置说明

## 二：代码解析

## 三：总结注意点




### 一：配置说明

```nginx
upstream backend {
        dynamic_resolve fallback=stale fail_timeout=30s;   
        server www.baidu.com;        
    }
    server {
        location / {
            proxy_pass http://backend;
        }
    }
```

指定在某个upstream中启用动态域名解析功能。


解析:
其中有一个结构体类型:
```c 
ngx_http_upstream_dynamic_srv_t  {
    ngx_int_t     enabled;
    ngx_int_t     fallback;
    time_t        fail_timeout;
    time_t        fail_check;

    ngx_http_upstream_init_pt  original_init_get_peer;
        ngx_http_upstream_init_peer_t  original_init_peer;
}
```
---     

属性含义：
    fallback参数指定了当域名无法解析时采取的动作:

    stale, 使用tengine启动的时候获取的旧地址
    next, 选择upstream中的下一个server
    shutdown, 结束当前请求
    fail_timeout参数指定了将DNS服务当做无法使用的时间，也就是当某次DNS请求失败后，假定后续多长的时间内DNS服务依然不可用，以减少对无效DNS的查询。


### 二：代码解析

* 发现dynamic_resolve指令的时候调用该函数进行处理
```c
static char *
ngx_http_upstream_dynamic(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_upstream_srv_conf_t            *uscf;
    ngx_http_upstream_dynamic_srv_conf_t    *dcf;
    ngx_str_t   *value, s;
    ngx_uint_t   i;
    time_t       fail_timeout;
    ngx_int_t    fallback;

    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

    dcf = ngx_http_conf_upstream_srv_conf(uscf,
                                          ngx_http_upstream_dynamic_module);

    if (dcf->original_init_upstream) {
        return "is duplicate";
    }
    //设置原有的回调函数（可以叫做记录，本来应该赋给uscf->peer.init_upstream的）
    dcf->original_init_upstream = uscf->peer.init_upstream
                                  ? uscf->peer.init_upstream
                                  : ngx_http_upstream_init_round_robin;
   
    uscf->peer.init_upstream = ngx_http_upstream_init_dynamic;

    /* read options */

    return NGX_CONF_OK;
}
```

* 上面设置的uscf->peer.init_upstream函数在ngx_http_upstream_init_main_conf中调用，其uscf->peer.init_upstream
指向原型如下：
```c
static ngx_int_t
ngx_http_upstream_init_dynamic(ngx_conf_t *cf,
    ngx_http_upstream_srv_conf_t *us)
{

    //此处执行一个后端的负载均衡算法，默认是round_robin
    if (dcf->original_init_upstream(cf, us) != NGX_OK) {
        return NGX_ERROR;
    }

    if (us->servers) {
        server = us->servers->elts;

        for (i = 0; i < us->servers->nelts; i++) {
            host = server[i].host;
            //将ip地址转换为长整形数字
            if (ngx_inet_addr(host.data, host.len) == INADDR_NONE) {
                break;
            }
        }

        if (i == us->servers->nelts) {
            dcf->enabled = 0;

            return NGX_OK;
        }
    }
    // us->peer.init 默认是init_round_robin_peer函数  
    dcf->original_init_peer = us->peer.init;
    //下面是当一个请求来到的时候进行调用该函数
    us->peer.init = ngx_http_upstream_init_dynamic_peer;
    //把其设置为激活状态
    dcf->enabled = 1;

    return NGX_OK;
}
```

* 其中在ngx_http_upstream_init_request中调用us->peer.init，其中在上面函数中其指向
ngx_http_upstream_init_dynamic_peer如下：
```c
static ngx_int_t
ngx_http_upstream_init_dynamic_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
    //默认是init_round_robin_peer
    if (dcf->original_init_peer(r, us) != NGX_OK) {
        return NGX_ERROR;
    }
    //下面就是一点小的技巧，修改原有回调指向，其实就想在原来的函数基础上增加自己的功能，自己的函数中再次调用原来的函数
    //这个思想值得学习
    dp->conf = dcf;
    dp->upstream = r->upstream;
    dp->data = r->upstream->peer.data;
    dp->original_get_peer = r->upstream->peer.get;
    dp->original_free_peer = r->upstream->peer.free;
    dp->request = r;

    r->upstream->peer.data = dp;
    r->upstream->peer.get = ngx_http_upstream_get_dynamic_peer;
    r->upstream->peer.free = ngx_http_upstream_free_dynamic_peer;

    return NGX_OK;
}

最后在ngx_http_upstream_init_request函数中调用ngx_http_upstream_connect(r, u)在其又调用ngx_event_connect_peer(&u->peer)
 rc = pc->get(pc, pc->data);其中pc->get就是指向ngx_http_upstream_get_dynamic_peer;此时就完成了在upstream模块选择使用动态
``` 
 
 
* dns查询后端peer了。其中ngx_http_upstream_get_dynamic_peer如下：

```c
static ngx_int_t
 ngx_http_upstream_get_dynamic_peer(ngx_peer_connection_t *pc, void *data)
{
    .....

    if (pc->resolved == NGX_HTTP_UPSTREAM_DR_OK) {
        return NGX_OK;
    }

    dscf = bp->conf;
    r = bp->request;
    u = r->upstream;
   //当dns无法解析的时候  按照配置文件所选择的方式进行执行   
    if (pc->resolved == NGX_HTTP_UPSTREAM_DR_FAILED) {

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "resolve failed! fallback: %ui", dscf->fallback);

        switch (dscf->fallback) {
        //当无法访问的时候还是使用旧的解析结果
        case NGX_HTTP_UPSTREAM_DYN_RESOLVE_STALE:
            return NGX_OK;
          //当无法访问的时候把当其关闭
        case NGX_HTTP_UPSTREAM_DYN_RESOLVE_SHUTDOWN:
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_BAD_GATEWAY);
            return NGX_YIELD;
          //当无法访问的时候默认使用下一个server
        default:
            /* default fallback action: check next upstream */
            return NGX_DECLINED;
        }

        return NGX_DECLINED;
    }
    //当没有失败的时候则进行超时时间的判断，在timeout时间内dns是无效的
    if (dscf->fail_check
        && (ngx_time() - dscf->fail_check < dscf->fail_timeout))
    {
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "in fail timeout period, fallback: %ui", dscf->fallback);

        switch (dscf->fallback) {

        case NGX_HTTP_UPSTREAM_DYN_RESOLVE_STALE:
            return bp->original_get_peer(pc, bp->data);

        case NGX_HTTP_UPSTREAM_DYN_RESOLVE_SHUTDOWN:
            ngx_http_upstream_finalize_request(r, u, NGX_HTTP_BAD_GATEWAY);
            return NGX_YIELD;

        default:
            /* default fallback action: check next upstream, still need
             * to get peer in fail timeout period
             */
            return bp->original_get_peer(pc, bp->data);
        }

        return NGX_DECLINED;
    }

    //bp->original_get_peer 其指向各种负载均衡算法的(ip_hash、round_robin、consistent_hash)get_peer

    rc = bp->original_get_peer(pc, bp->data);

    if (rc != NGX_OK) {
        return rc;
    }

    /* resolve name */

    if (pc->host == NULL) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "load balancer doesn't support dyn resolve!");
        return NGX_OK;
    }
    //判断该host是不是一个IP  若是一个IP则直接返回
    if (ngx_inet_addr(pc->host->data, pc->host->len) != INADDR_NONE) {
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, pc->log, 0,
                       "host is an IP address, connect directly!");
        return NGX_OK;
    }

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);
    if (clcf->resolver == NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "resolver has not been configured!");
        return NGX_OK;
    }

    temp.name = *pc->host;
    //开始准备DNS解析 
    ctx = ngx_resolve_start(clcf->resolver, &temp);
    if (ctx == NULL) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "resolver start failed!");
        return NGX_OK;
    }
    //当没有配置dns服务器的时候
    if (ctx == NGX_NO_RESOLVER) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "resolver started but no resolver!");
        return NGX_OK;
    }

    ctx->name = *pc->host;
    /* TODO remove */
    // ctx->type = NGX_RESOLVE_A;
    /* END */
    //设置其回调函数
    //此函数主要是做一些后续处理，如当dns查询成功或是失败修改一些状态  
    //以及函数ngx_resolve_name_done的调用，用来释放一些空间以及调用ngx_resolver_expire
    //上面函数做一些rn结点的过期检查 
    ctx->handler = ngx_http_upstream_dynamic_handler;
    ctx->data = r;
    ctx->timeout = clcf->resolver_timeout;

    u->dyn_resolve_ctx = ctx;
    //根据名字进行dns查询
    if (ngx_resolve_name(ctx) != NGX_OK) {
        ngx_log_error(NGX_LOG_ERR, pc->log, 0,
                      "resolver name failed!\n");

        u->dyn_resolve_ctx = NULL;

        return NGX_OK;
    }

    return NGX_YIELD;
}
```


### 三：总结注意点


### 注意点:
  * 1.当一个域名下对应有多个ip的时候:
      但是在调用ngx_http_upstream_get_dynamic_peer的时候调用round_robin_get_peer选择了一个server,但是其判断该peer的hos
t->name是一个域名,此时nginx会做域名解析最后只会随机的进行选择一个server进行连接.其nginx是这样做的当一个域名对应多个ip
的时候，对其获得的ip进行随机排序，最后是选择第一个。       
      
     当有多个addr的时候，函数是给 rn->naddrs 进行随机排序: 
     addrs = ngx_resolver_export(r, rn, 1);
     每次只是选择域名下的第一个addr:  
     csin = (struct sockaddr_in *) ctx->addrs[0].sockaddr;
       
   
  * 2.配置了dynamic_resolve指令可能不会生效:
      当在使用ngx_http_upstream_dynamic(upstream动态域名解析模块)模块的时候切记点:只有配置了resolver的时候才会去动态DN
S查询(此处的配置resolver有两处:第一处是在nginx的配置文件中locationserverhttp配置块都可以第二处是nginx会在该路径下读取(
/etc/resolve.conf下寻找)当#define NGX_RESOLVER_FILE  "/etc/resolv.conf"
))

  * 3.当其upstream模块动态dns生效了如何进行dns解析:
      首先判断该host->name是一个ip还是一个域名,若是一个ip直接returnok,否则会在本地catche进行查询域名对应的ip,对其域名
进行crc哈希得到一个hash值,然后在name_rbtree中进行查找得到resolvernode节点(rn),进行判断rn?=NULL,当其不为空则表明在catch
e中找到了,然后判断其是否过期,没有过期的话就直接进行选择,执行回调函数ctx->handlr(ctx)设置pc->sockaddr进行后端的连接处理.
     当其所访问到的catche过期了则把该节点从队列中删除,然后重新发起dns请求,把得到的域名对应的ip copy到 name_rbtree 
rn中(rn->u.addrs=addr).(注意在前面只是删除了队列中数据  并没有删除name_rbtree中的节点).

  * 4.当查询一个catch中不存在的域名的时候:
      这个和第3点就多一步工作分配一个rn节点,并且插入name_rbtree中,得到该值后在插入队列.

  * 5.缓存中的rn节点失效时间是如何管理的:
      其中在resolver指令有一个参数vailed值(r->vailed = vailed),其中在进行节点失效与否时有这样一句: rn->vailed >= ngx_time();
      其中rn->vailed = ngx_time() + (r->vailed ? r->vailed : rn->ttl):
      其中rn->ttl = 0xff ff ff ff default  rn->ttl = min(rn->ttl,ttl)  ttl 是每次dns返回来的ttl.
      所以其rn节点的有效时间用户可以自己配置,若没有配置则使用dns返回来的ttl作为有效时间.
 
  * 6.当本地catch的某一个rn节点长期没有访问时如何处理:
      每次在get_dynamic_peer查询结束的时候（调用ngx_resolve_addr_done），都会检查有没有缓存过期，如果有，就会进行释放。
      其原理就是调用:ngx_resolver_expire  其中:
           q = ngx_queue_last(queue);  
           rn = ngx_queue_data(q,ngx_resolver_node_t,queue);
           now=ngx_now  if now > rn->expire  则把该rn节点分别从queue以及name_rbtree中删除.
           其中每次缓存命中的时候都会 update expire: rn->expire = r->expire + ngx_now(); r->expire=30

  * 7.区分resolver指令中的vailed与dynamic_resolve中的fail_timeout的区别:
      其中resolver指令中的vailed是catche中rn节点的有效期时间,dynamic_resolve中的fail_timeout表示上次dns解析失败与本次是否可以使用dns解析最大时间间隔(即DNS服务无法使用的时间).
   
  * 8.当其resolver后配置了多个server的时候(即有多个dns的时候)：
      nginx 会采用Round Robin的方式轮流查询各个dns server。在方法ngx_resolver_send_query中通过在每次调用时改变last_connection来轮流使用不同的dns server进行查询.
  
  * 9.在域名后面是否可以指定weight等参数：
      在upstream模块的server模块若后面给得是一个域名，是可以指定其weight的值，其实可以这样理解一个域名对应有ip（可能有多个ip），其weight值就是其ip对应的weight.其相应的参数是否生效只和相应的负载均衡算法有关，如在使用ip_hash算法的时候其中
backup参数对其无效.

  * 10.dns本地缓存数据结构以及架构是如何设计的:
       函数ngx_resolver_create()对其进行创建:
       对查询结果的缓存，采用RedBlackTree的数据结构，以要查询名字的Hash作为Key,其中有可能不同域名的hash的值相同,此时则
比较域名的字符串大小插入rbtree中,当然最后查询的时候也是按照同样的规则进行查找,节点信息存放在 struct 
ngx_resolver_node_t中。
       因为resolver是全局的，与任何一个connection都无关，所有需要放在一个随时都可以取到的地方，如ngx_mail_core_srv_conf_t结构体上，在使用时从当前session找到ngx_mail_core_srv_conf_t，然后找到resolver。

---     
       为不阻塞当前线程，Nginx采用了异步的方式进行域名查询。整个查询过程主要分为三个步骤，这点在各种异步处理时都是一样
       的：

         (1).准备函数调用需要的信息，并设置回调方法.
         (2).调用函数.
         (3).处理结束后回调方法被调用.


## 有问题反馈
在使用中有任何问题，欢迎反馈给我，可以用以下联系方式跟我交流

* 邮件(1031379296#qq.com, 把#换成@)
* QQ: 1031379296
* weibo: [@王发康](http://weibo.com/u/2786211992/home)


## 感激

### chunshengsterATgmail.com


## 关于作者

### Linux\nginx\golang\c\c++爱好者
### 欢迎一起交流  一起学习# 
