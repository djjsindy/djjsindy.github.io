Title: 图解Nginx 中的4级指针 
Date: 2013-08-08 22:45:00
Category: nginx
Tags: nginx
Slug: nginx-pointer
Author: djjsindy
Summary:nginx的所有配置结构体全部放在一个cycle的四级指针中，本文就具体分析一下每级指针究竟指向的是什么

 
 
   
   nginx的所有配置结构体全部放在一个cycle的四级指针中，本文就具体分析一下每级指针究竟指向的是什么,下图表示了这个四级指针每一级的指向,图中虚拟地址模拟了真实地址，ctx指针存的指向虚拟地址为1的数据，图中只列举出core，event，http模块最基础的配置结构。
   				
   				
   				   
       
![Alt text](https://lh6.googleusercontent.com/-EvDQj2yRuCU/UhCCwPNtrFI/AAAAAAAAAJw/QPY2xNu2zzM/s576/Screen%2520Shot%25202013-08-08%2520at%252010.33.57%2520PM.png)

 需要注意得是，从图中不难发现，对于常用的模块，core模块，event模块，http模块，core模块只需要2级指针就能搞定，event模块按照逻辑只需要3级指针，而http模块需要4级指针。event按照常理缺少了http模块中间那个虚拟地址1000的结构体（区分main conf，serv conf，loc conf），需要3级指针就行。但是nginx为了方便配置，也在event模块中间也加入了类似的一个结构体指针（虚拟地址75存储100 的指针），不过这个指针直接指向了event模块的指针。这样event模块和http模块一样也形成了4级指针，这样设计event模块我认为是为了兼容http模块配置才形成了4级指针。

   从代码切入，看一下ctx的初始化过程。
   
	cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
 这段代码初始化了cycle的context，conf_ctx指向一段空间，每个地址存储void ＊的指针，这个指针能指向任何的一段空间，也就是各种conf结构体。
     以下代码为第一层指针赋值，也就是每个conf_ctx每个空间存储第一层conf的结构体的首地址，这类conf的结构体不会包含event，http模块的，因为nginx并不确定配置一定有event和http模块的配置，所以在conf文件中没有碰到event和http的时候，是不会建立对应的结构体的。
     
     
	for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->type != NGX_CORE_MODULE) {     //加载core module,log module等
            continue;
        } 

        module = ngx_modules[i]->ctx;

        if (module->create_conf) {
            rv = module->create_conf(cycle);              //rv 是conf结构体首地址
            if (rv == NULL) {
                ngx_destroy_pool(pool);
                return NULL;
            } 
            cycle->conf_ctx[ngx_modules[i]->index] = rv; //conf_ctx每个空间存储conf的首地址，构造了两层指针
        }
    }    
    
 下面的代码是解析command，获得这个command的conf结构体
 
 1.    NGX_DIRECT_CONF表示ngx_core_module,也就是配置中最外边配置的那些参数，因为 ngx_core_module,所对应的conf结构体，在初始化的时候已经创建好了，所以再ctx[index]的位 置是有conf的首地址的指针的，相当于图中虚拟地址50，后面在set command到结构体的时候，只需要把这个指针传过去就行了，这里ctx 2级指针是说明ctx一级指针指向的存储conf的首地址虚拟地址1，ctx二级指针指向conf的首地址，相当于图中虚拟地址50。
 2.    NGX_MAIN_CONF表示events http的配置，也就是说在cycle初始化的时候并不会初始化这些conf，前面说了因为我们根本不确定这些配置项是否真的有。所以在碰到events 或者http那种配置项的时候，那么就是初始化这一类配置项的时候，首先需要在ctx的相应位置赋值他们core conf module的结构体。因为之前ctx在这个位置并没有任何指向结构体的指针。所以把ctx位置的指针（虚拟地址2）赋值给conf，让后面的set函数，为这个位置赋值配置项结构体的首地址。
 
    
 ![Alt text](https://lh6.googleusercontent.com/-gDhkSPbxjig/UhCCxLCholI/AAAAAAAAACk/lMNtzNIT0iY/s1024/Screen%2520Shot%25202013-08-08%2520at%252010.38.53%2520PM.png)
 
  对于第三个位置注释的代码说明，应该还是先说说event模块的加载，下面的代码是nginx events模块set函数
	ctx = ngx_pcalloc(cf->pool, sizeof(void *)); //建立一个指针，虚拟地址75 （为了兼容http模块的4级指针的指针，图中红色的位置）
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    } 

    *ctx = ngx_pcalloc(cf->pool, ngx_event_max_module * sizeof(void *)); //可以看到 event模块位置被赋值了一段空间，相当于ctx指向虚拟地址100，也就是虚拟地址75存储了100，相当于图中标黄色的过程1
    if (*ctx == NULL) {
        return NGX_CONF_ERROR;
    } 

    *(void **) conf = ctx; //所以这里conf相当于虚拟地址2的地方存储是首地址为75的指针，相当于图中标红的过程2

    for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->type != NGX_EVENT_MODULE) {
            continue;
        } 

        m = ngx_modules[i]->ctx;

        if (m->create_conf) {
            (*ctx)[ngx_modules[i]->ctx_index] = m->create_conf(cf->cycle); //相当于图中标绿的过程3
            if ((*ctx)[ngx_modules[i]->ctx_index] == NULL) {               //相当于地址为100的位置存储首地址600，600是event core module的位置，虚拟地址101的地方存储首地址700，700是event epoll模块的结构体的首地址
                return NGX_CONF_ERROR;
            } 
        } 
    } 

    pcf = *cf; 
    cf->ctx = ctx; //重新赋值虚拟地址75的指针，继续解析配置。
    cf->module_type = NGX_EVENT_MODULE;
    cf->cmd_type = NGX_EVENT_CONF;

    rv = ngx_conf_parse(cf, NULL);
    
    
 从上面的代码可以看出event模块的四级指针确实有些牵强，完全可以省略虚拟地址75位置的指针，虚拟地址2的位置直接存储100，不省略的原因正是前面说的原因，为了兼容http模块的配置，再看下http的代码
 
 ![Alt text](https://lh5.googleusercontent.com/-9I2BjhwFt3U/UhCCzZtL9bI/AAAAAAAAACs/ISULGqMKK-Y/s1024/Screen%2520Shot%25202013-08-08%2520at%252010.40.34%2520PM.png)
 
 	ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    } 

    *(ngx_http_conf_ctx_t **) conf = ctx; //相当于图中标红的过程1

    ngx_http_max_module = 0; 
    for (m = 0; ngx_modules[m]; m++) {
        if (ngx_modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        } 

        ngx_modules[m]->ctx_index = ngx_http_max_module++;
    } 

    ctx->main_conf = ngx_pcalloc(cf->pool,
                                 sizeof(void *) * ngx_http_max_module); //相当于图中标黄的过程2
    if (ctx->main_conf == NULL) {
        return NGX_CONF_ERROR;
    }
    ctx->srv_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->srv_conf == NULL) {
        return NGX_CONF_ERROR;
    }

    ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
    if (ctx->loc_conf == NULL) {
        return NGX_CONF_ERROR;
    }

    for (m = 0; ngx_modules[m]; m++) {
        if (ngx_modules[m]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = ngx_modules[m]->ctx;
        mi = ngx_modules[m]->ctx_index;

        if (module->create_main_conf) {
            ctx->main_conf[mi] = module->create_main_conf(cf); //相当于图中标绿色的过程3
            if (ctx->main_conf[mi] == NULL) {
                return NGX_CONF_ERROR;
            }
        }
        
        
 看完了上面http模块的初始化，应该就了解前面代码的用意：
	 
	    confp = *(void **) ((char *) cf->ctx + cmd->conf); 
  		if (confp) {
         conf = confp[ngx_modules[i]->ctx_index];
             }
       
  在初始化完event模块之后，cf->ctx存储的就是75,event模块的cmd->conf都是0，因为这个cf->ctx一个兼容指针，所以confp指向的就是虚拟地址100，那么后面的代码根据模块的ctx_index就可以得到conf 了。
     对于http模块cf->ctx存储的是1000，那么cmd->conf对于http模块不是0，在结构体ngx_http_conf_ctx_t中main，srv，loc有不同的位移，后面的逻辑和event一致，所以event模块的四级指针只是为了兼容http模块而多设了一层指针，那层指针充当了http指向ngx_http_conf_ctx_t结构体作用，这层对event模块是没有什么用的，对于http模块来说可以区分是main，srv，loc的配置。


<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-43260693-1', 'djjsindy.github.io');
  ga('send', 'pageview');

</script>



