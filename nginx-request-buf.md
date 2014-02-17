Title: nginx request buf
Date: 2013-06-26 15:45:00
Category: nginx
Tags: nginx
Slug: nginx-header-buf
Author: djjsindy
Summary: nginx在接收到客户端得请求之后，就开始解析http请求，也就是解析http header，需要分配一段buf来接收这些数据，nginx并不知道这个http header的大小，在nginx配置中`client_header_buffer_size`和`large_client_header_buffers`这两个配置项起到了作用  

nginx在接收到客户端得请求之后，就开始解析http请求，也就是解析http header，需要分配一段buf来接收这些数据，nginx并不知道这个http header的大小，在nginx配置中`client_header_buffer_size`和`large_client_header_buffers`这两个配置项起到了作用 。

1.     client_header_buffer_size 1k
2.     large_client_header_buffers 4 8k 


 `client_header_buffer_size`默认是1024字节。`large_client_header_buffers`默认最大分配4组8192字节的buf，每次分配一个buf。nginx处理http header的过程是先处理request line（http 请求的第一行），然后在处理每一个header，那么处理request line的过程首先会分配`client_header_buffer_size`大小的空间，如果这个空间不够，那么再分配一个`large_client_header_buffers`的空间，然后把之前的`client_header_buffer_size copy`到大buffer的前半部分中。如果在不够，nginx就会返回给客户端400的错误。每个header也是和如上的request line一个处理步骤，所以对于request line 和每个header的大小应该不超过1个`large_client_header_buffers`。对于整个request line和所有header来讲，总大小不应该超过4*8192字节大小，否则也会产生400的错误。
 
 
 在解析request line的时候，回首先分配一个`client_header_buffer_size`来解析请求，当空间不足的时候，copy数据到第一个`large_client_header_buffers`中，如果这个buf仍然不能满足要求就返回400错误。函数`ngx_http_process_request_line`是解析request line的,在解析buf中的数据后，会判断是否解析完request line，如果buf的pos和end相等，就说明仍然未解析完成，需要再次从fd中读取数据，然后扩充buf到large buf。
 
 	/* NGX_AGAIN: a request line parsing is still incomplete */
        if (r->header_in->pos == r->header_in->end) {
            rv = ngx_http_alloc_large_header_buffer(r, 1);
            
  
  函数ngx_http_alloc_large_header_buffer是扩充buf的，这里面涉及到free和busy数组的使用，这个和pipeline模式的请求有关。对于http 1.1而言，浏览器可以开启pipeline模式请求，对于一般的keepalive请求，浏览器对于每一个http请求都是通过一条tcp连接发送出去，在这个请求发送出去到接收到完整的http响应这段时间是肯定不会在发送http请求的。因为如果在这个过程中再次发送其他的http请求，那么由于网络延迟可能这两个请求的响应不是按照先后顺序到达浏览器的，所以这样的请求模型肯定是不行的。所以一般的http1.1的请求都是在一条tcp通道中，发送完一个请求，接收到响应，在发送第二个请求。这种发送模型的请求效率比较低。pipeline模式是一个http包中包含有多个http请求，一次就发送多个http报文，然后对于服务器来说依次处理这些请求，产生响应报文，一次再发送回客户端。这种模式增加了客户端和服务器之间效率，所以再pipeline的这种模式下，如果同一个http包中的多个http报文会共享那些分配的large buf。对于第一个请求如果使用了large buf，二个请求就会使用之前的那些large buf，避免再次直接alloc内存，之前的那些large buf如果不够用的时候再去alloc。


	static ngx_int_t
	ngx_http_alloc_large_header_buffer(ngx_http_request_t *r,
    ngx_uint_t request_line)
	{
    u_char *old, *new;
    ngx_buf_t *b;
    ngx_http_connection_t *hc;
    ngx_http_core_srv_conf_t *cscf;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http alloc large header buffer");

    if (request_line && r->state == 0) {

        /* the client fills up the buffer with "\r\n" */

        r->header_in->pos = r->header_in->start;
        r->header_in->last = r->header_in->start;

        return NGX_OK;
    }
    //old表示是这个buf从哪开始转到large buf中，如果是request_line扩充buf，那么就肯定是从buf的开始，如果是header，那么就从当前未解析的那个header开始，因为之前的hea       der都已经解析完了，也就不需要那段空间了。
    old = request_line ? r->request_start : r->header_name_start;
    //如果迁移的这段空间和large buf一样大，那么就报错，那么这里就说明了只能一个request line和每一个header只能分配一次large buf
    cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
    //state !=0 表示这在解析过程中
    if (r->state != 0
        && (size_t) (r->header_in->pos - old)
                                     >= cscf->large_client_header_buffers.size)
    {
        return NGX_DECLINED;
    }
    //pipeline请求共享这个hc 
    hc = r->http_connection;
    //free就是pipeline请求共享的那个large buf，如果nfree大于0，表示之前的请求有large buf，那么这个请求直接用那个空间就可以了，不需要再次分配large buf 了
    if (hc->nfree) {
        b = hc->free[--hc->nfree];

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http large header free: %p %uz",
                       b->pos, b->end - b->last);
    //每个请求分配的large buf的数目不会超过那个配置中设置的数目
    } else if (hc->nbusy < cscf->large_client_header_buffers.num) {
         //busy表示当前请求已经用的large buf
        if (hc->busy == NULL) {
            //如果还没有用这个large buf，那么重新分配空间，
            hc->busy = ngx_palloc(r->connection->pool,
                  cscf->large_client_header_buffers.num * sizeof(ngx_buf_t *));
            if (hc->busy == NULL) {
                return NGX_ERROR;
            }
        }
        //首先建立一个large buf就可以了
        b = ngx_create_temp_buf(r->connection->pool,
                                cscf->large_client_header_buffers.size);
        if (b == NULL) {
            return NGX_ERROR;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "http large header alloc: %p %uz",
                       b->pos, b->end - b->last);

    } else {
        //如果超过了large buf的个数，那么直接400错误
        return NGX_DECLINED;
    }
    //把这个空间放到busy数组里，同时busy加1，这个是必须的，表示当前请求已经用的large buf，不管这个空间来源于哪里，是新分配的还是继承pipeline请求的，如果继承的pipelin       e请求的buf不计算如这次，那么如果这次请求过大，还是会超过large buf size＊num的，这样肯定是违背那个配置的
    hc->busy[hc->nbusy++] = b;
    //注释很清楚，如果state＝0表示已经解析完成了，不需要再分配large buf了
    if (r->state == 0) {
        /*
         * r->state == 0 means that a header line was parsed successfully
         * and we do not need to copy incomplete header line and
         * to relocate the parser header pointers
         */

        r->header_in = b;

        return NGX_OK;
    }

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "http large header copy: %d", r->header_in->pos - old);
   
    new = b->start;
    //把老buf的数据copy到large buf 中
    ngx_memcpy(new, old, r->header_in->pos - old);
    //重新设置指针
    b->pos = new + (r->header_in->pos - old);
    b->last = new + (r->header_in->pos - old);
    //如果是解析申请空间request line重新计算那些已经解析出的指针
    if (request_line) {
        r->request_start = new;
        if (r->request_end) {
            r->request_end = new + (r->request_end - old);
        }
        r->method_end = new + (r->method_end - old);
        r->uri_start = new + (r->uri_start - old);
        r->uri_end = new + (r->uri_end - old);
        if (r->schema_start) {
            r->schema_start = new + (r->schema_start - old);
            r->schema_end = new + (r->schema_end - old);
        }
        if (r->host_start) {
            r->host_start = new + (r->host_start - old);
            if (r->host_end) {
                r->host_end = new + (r->host_end - old);
            }
        }
        if (r->port_start) {
            r->port_start = new + (r->port_start - old);
            r->port_end = new + (r->port_end - old);
        }
        if (r->uri_ext) {
            r->uri_ext = new + (r->uri_ext - old);
        }
        if (r->args_start) {
            r->args_start = new + (r->args_start - old);
        }
        if (r->http_protocol.data) {
            r->http_protocol.data = new + (r->http_protocol.data - old);
        }

    } else {
    //解析header的指针
        r->header_name_start = new;
        r->header_name_end = new + (r->header_name_end - old);
        r->header_start = new + (r->header_start - old);
        r->header_end = new + (r->header_end - old);
    }
    r->header_in = b;
    return NGX_OK;
    
 从上面的代码中可以看出来，当`client_header_buffer_size`不够用的时候会分配large buf，然后把之前的buffer中的数据copy到large buf中，无论是request line还是某个header的长度都不会大于large buf，总体的长度也会由于nbusy的限制不会超过`large buf size＊num`，如果超过了就会发送给客户端400错误，这个large buf的空间的有可能是新分配的，也有可能是之前pipeline请求已经分配好的，这样对于pipeline请求来说，就极大的减少了重新分配large buf的过程。
   
 处理完http请求之后，需要finalize 请求的时候，就需要判断是否是pipeline请求，也就是那个buf的完整http报文后面是否还有其他的http报文，如果有的话就认为是pipeline请求，就回收那些已经分配的buf（busy数组）到free数组中，提供给下次请求使用。
    下面的代码是最后`ngx_http_set_keepalive`函数，迁移buf的过程

	//如果buf解析到的pos在buf的最后的位置之前，那么就意味着buf中有其他的数据，也就是pipeline请求
 	if (b->pos < b->last) { 
        /* the pipelined request */ 
     //c->buffer是client_header_buffer_size的那个最初始的buf，如果b不是那个buf，那么b肯定是large buf
        if (b != c->buffer) { 

            /* 
             * If the large header buffers were allocated while the previous
             * request processing then we do not use c->buffer for
             * the pipelined request (see ngx_http_create_request()).
             *
             * Now we would move the large header buffers to the free list.
             */

            cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
           //初始化那个free数组
            if (hc->free == NULL) { 
                hc->free = ngx_palloc(c->pool, 
                  cscf->large_client_header_buffers.num * sizeof(ngx_buf_t *));

                if (hc->free == NULL) { 
                    ngx_http_close_request(r, 0); 
                    return;
                }
            }
             //开始迁移buzy数组到free数组中
            for (i = 0; i < hc->nbusy - 1; i++) { 
                f = hc->busy[i]; 
                hc->free[hc->nfree++] = f; 
                f->pos = f->start; 
                f->last = f->start; 
            }
           //busy的第一个位置是上一个请求未处理完，还有数据的buf，所以新的request应该使用这个buf作为第一个buf，就不用那个client_header_buffer_size的buf了
            hc->busy[0] = b;
            hc->nbusy = 1;
        }
    }
    
    
上面的代码中，可以想到下一个请求就应该直接用这个busy[0]就可以了，不用重新分配那个`client_header_buffer_size`的buf了，因为重新分配的肯定丢掉了原来那个buf中的数据，所以在pipeline构建下一个request的时候，在初始化header_in的时候直接会用这个busy[0]。
    在`ngx_http_create_request`函数中有下面的代码，就说明了如果是pipeline的请求就直接用busy[0]就可以了
    
    r->header_in = hc->nbusy ? hc->busy[0] : c->buffer;     
>总结：

>  nginx在处理request的时候，会预先分配一个client_header_buffer_size的buf，如果不够就会分配large_client_header_buffers的buf，对于request line和每个header而言，每一个不应该超过large buf，所有的总和也不应该超过large buf size＊num。Http 1.1的pipeline请求，如果前面的请求分配的large buf，那么后面的请求会继承使用这个large buf分配的空间，当large buf 不够了再去主动分配large buf。


>>参考文章：
   1. [nginx中request buf的设计和实现](http://www.pagefault.info/?p=220)
   2. [nginx对keepalive和pipeline请求处理分析](http://www.pagefault.info/?p=225)  
   
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-43260693-1', 'djjsindy.github.io');
  ga('send', 'pageview');

</script>
