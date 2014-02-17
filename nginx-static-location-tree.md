Title:  nginx static location tree 
Date: 2013-06-07 23:56:00
Category: nginx
Tags: nginx
Slug: nginx-static-location-tree
Author: djjsindy
Summary:  nginx在处理location的配置的时候，用到了一种三叉排序树，加速了通过request的url和location的映射速度。

nginx在处理location的配置的时候，用到了一种三叉排序树，加速了通过request的url和location的映射速度。

![Alt](https://lh4.googleusercontent.com/-ImJc2DVKOG4/UhxB0gMpFwI/AAAAAAAAALY/EJejCGD4uh8/s1024/Screen%2520Shot%25202013-06-07%2520at%252011.32.58%2520PM.png)

可以从图中看到，static location tree的结构中，left指向的node是比parent的节点小的，right指向的node是比parent节点大的，tree指向拥有parent前缀的节点。初始的static location的类型包括extact_match，前缀匹配。看一下nginx是如何uri和location之间快速做映射的。

	static ngx_int_t
	ngx_http_core_find_static_location(ngx_http_request_t *r,
    ngx_http_location_tree_node_t *node)
	{
    u_char *uri;
    size_t len, n;
    ngx_int_t rc, rv;

    len = r->uri.len;  //request的请求路径长度
    uri = r->uri.data; //request请求的地址

    rv = NGX_DECLINED; //默认精准匹配和前缀匹配 匹配不到，需要匹配后面的正则

    for ( ;; ) {

        if (node == NULL) {
            return rv;
        }

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "test location: "%*s"", node->len, node->name);
        //n是uri的长度和node name长度的最小值，好比较他们的交集
        n = (len <= (size_t) node->len) ? len : node->len; 
        //比较uri和node 的name交集
        rc = ngx_filename_cmp(uri, node->name, n); 
        //不得0表示uri和node的name不相等，这时候三叉树就能加速查找的效率，选择node的左节点或者右节点
        if (rc != 0) {     
            node = (rc < 0) ? node->left : node->right;

            continue; //更新节点后重新开始比较匹配
        }
         //如果交集相等，如果uri的长度比node的长度还要长
        if (len > (size_t) node->len) {

            if (node->inclusive) {//如果这个节点是前缀匹配的那种需要递归tree节点，因为tree节点后面的子节点拥有相同的前缀。
        //因为前缀已经匹配到了，所以这里先暂且把loc_conf作为target，但是不保证后面的tree节点的子节点是否有和uri完全匹配或者更多前缀匹配的。例如如果uri是/abc,当前node节点是/a,虽然匹配到了location /a,先把/a的location配置作为target，但是有可能在/a的tree节点有/abc的location，所以需要递归tree节点看一下。

                r->loc_conf = node->inclusive->loc_conf; 
        //设置成again表示需要递归嵌套location，为什么要嵌套递归呢，因为location的嵌套配置虽然官方不推荐，但是配置的话，父子location需要有相同的前缀。所以需要递归嵌套location
                rv = NGX_AGAIN;

                node = node->tree; //node重新变为tree节点
                uri += n;
                len -= n;
       continue;
            }

            /* exact only */
        //对于精确匹配的location不会放在公共前缀节点的tree节点中，会单拉出来一个node和前缀节点平行。也就是说对于精确匹配 ＝/abcd 和前缀匹配的/abc两个location配置，=/abcd不会是/abc节点的tree节点。=/abcd 只能是／abc的right节点
            node = node->right; 

            continue;
        }

        if (len == (size_t) node->len) { //如果是uri和node的name是完全相等的

            if (node->exact) {           //如果是精确匹配，那么就是直接返回ok了
                r->loc_conf = node->exact->loc_conf;
                return NGX_OK;

            } else {                 //如果还是前缀模式的location，那么需要递归嵌套location了，需要提前设置loc_conf，如果嵌套有匹配的再覆盖
                r->loc_conf = node->inclusive->loc_conf;
                return NGX_AGAIN;
            }
        }

        /* len < node->len */

        if (len + 1 == (size_t) node->len && node->auto_redirect) {

            r->loc_conf = (node->exact) ? node->exact->loc_conf:
                                          node->inclusive->loc_conf;
            rv = NGX_DONE;
        }
        //如果前缀相等，uri的长度比node的长度还要小，比如node的name是/abc ，uri是/ab,这种情况是/abc 一定是精确匹配，因为如果是前缀匹配那么／abc 肯定会再／ab的tree 指针里面。
        node = node->left; 
    }
    
 可以从上面的代码看出，三叉树优化了static location的查找过程，防止了O(n)的复杂度来匹配location。location tree的建立过程比较复杂，首先在解析完所有location后，`ngx_http_core_loc_conf_t`的locations保存了所有的location配置，形成了一个链表，在merge完server和main的配置之后，就开始建立这个static location tree了。

 首先从locations链表中去掉那些正则匹配，还有named和nonamed的location节点。那么location链表中只剩下精准匹配和前缀匹配的那些location节点了，从这些节点中产生static location tree。
 
生成location tree的必经一步是生成location list，也是一种前缀list，大概的产生步骤是这样，从第一个location节点开始，找到一个与第一个location节点前缀不相同的节点，然后把这个节点之前的list到location，全部作为第一个location的location list，然后递归这个location list，同时继续递归后面剩下的location，说下上图中原始location 的分布。

![alt](https://lh3.googleusercontent.com/-QFJd7ugmHs4/UhxB0hOMgTI/AAAAAAAAALY/xfI4pPpKChM/s1024/Screen%2520Shot%25202013-06-07%2520at%252011.40.29%2520PM.png)

从a1开始寻找和a1前缀相同的location，表示没有，所以a1就没有前缀list，继续aa节点，从aa节点到aad节点都是以aa为前缀的，所以location变为了下图。

![alt](https://lh5.googleusercontent.com/-VWl-Bg0Cfrg/UhxB0oByBEI/AAAAAAAAALY/UGg7PKjd59E/s1048/Screen%2520Shot%25202013-06-07%2520at%252011.41.26%2520PM.png)

然后递归分离的aa节点的list ，aac和aad节点，看aac的节点的后继节点有没有是aac前缀的。然后主location继续递归ab节点的后继节点。最后形成如下的location list。

![alt](https://lh3.googleusercontent.com/-KU2hvc_C-ok/UhxB1W1t1bI/AAAAAAAAALY/epVtU9fIDUA/s986/Screen%2520Shot%25202013-06-07%2520at%252011.43.07%2520PM.png)

最终location list的节点分布如上图，list指针的链表后面的元素都是拥有相同前缀的。
   
  再说下构建list元素的时候经常用到的一个函数：`ngx_queue_split(locations, q, &tail)`,作用是把location切割成两个双向循环队列，location队列和tail队列，location队列从原始的头元素到q元素之前的元素，tail队列从q元素开始到原location队列的最后一个元素。split操作节点示意图如下：
  
  ![alt](https://lh5.googleusercontent.com/-sOXSwTyhyuU/UhxB2fFTsKI/AAAAAAAAALY/x__n3521DlE/s1024/Screen%2520Shot%25202013-06-07%2520at%252011.44.08%2520PM.png)
  
  看下location list 的建立过程
  
  	static void 
	ngx_http_create_locations_list(ngx_queue_t *locations, ngx_queue_t *q)
	{
    u_char *name;
    size_t len; 
    ngx_queue_t *x, tail;
    ngx_http_location_queue_t *lq, *lx; 

    if (q == ngx_queue_last(locations)) { //如果location为空就没有必要继续走下面的流程了，尤其是递归到嵌套location
        return;
    } 

    lq = (ngx_http_location_queue_t *) q;

    if (lq->inclusive == NULL) { 
        ngx_http_create_locations_list(locations, ngx_queue_next(q)); //如果这个节点是精准匹配那么这个节点，就不会作为某些节点的前缀，不用拥有tree节点
        return;
    } 

    len = lq->name->len; 
    name = lq->name->data;

    for (x = ngx_queue_next(q);
         x != ngx_queue_sentinel(locations);
         x = ngx_queue_next(x))
    { 
        lx = (ngx_http_location_queue_t *) x;
         //由于所有location已经按照顺序排列好，递归q节点的后继节点，如果后继节点的长度小于后缀节点的长度，那么可以断定，这个后继节点肯定和后缀节点不一样，并且不可能有共同的后缀；如果后继节点和q节点的交集做比较，如果不同，就表示不是同一个前缀，所以可以看出，从q节点的location list应该是从q.next到x.prev节点
        if (len > lx->name->len 
            || (ngx_strncmp(name, lx->name->data, len) != 0))
        { 
            break;
        } 
    } 

    q = ngx_queue_next(q); 

    if (q == x) { //如果q和x节点直接没有节点，那么就没有必要递归后面了产生q节点的location list，直接递归q的后继节点x，产生x节点location list
        ngx_http_create_locations_list(locations, x);
        return;
    } 
    ngx_queue_split(locations, q, &tail); //location从q节点开始分割，那么现在location就是q节点之前的一段list
    ngx_queue_add(&lq->list, &tail); //q节点的list初始为从q节点开始到最后的一段list

    //原则上因为需要递归两段list，一个为p的location list（从p.next到x.prev），另一段为x.next到location的最后一个元素，这里如果x已经是location的最后一个了，那么就没有必要递归x.next到location的这一段了，因为这一段都是空的。

    if (x == ngx_queue_sentinel(locations)) { 
        ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list));
        return;
    }
    //到了这里可以知道需要递归两段location list了
    ngx_queue_split(&lq->list, x, &tail);//再次分割，lq->list剩下p.next到x.prev的一段了
    ngx_queue_add(locations, &tail); // 放到location 中去

    ngx_http_create_locations_list(&lq->list, ngx_queue_head(&lq->list)); //递归p.next到x.prev

    ngx_http_create_locations_list(locations, x); //递归x.next到location 最后了
	}
	
 最终static location tree的生成是从这个static location list中得到的也就是上图中的list。
 
 ![alt](https://lh4.googleusercontent.com/-fPu-X_ex1zE/UhxB1zd0X6I/AAAAAAAAALY/exWiLb00N5g/s984/Screen%2520Shot%25202013-06-07%2520at%252011.46.56%2520PM.png)
 
 location list的结构中，原始的那个location 的队列中直升下了a1，aa，ab，ac，ad，ae这几个location节点，tree的构建是个递归的过程，首先从location队列中取中间节点，就认为是tree的root节点，它的list指针认为是tree节点，中间节点之前的那段list ，a1 ,aa认为是ab节点的左节点，ac,ad,ae认为是ab节点的右节点。形成了如下形式：
 
 ![alt](https://lh4.googleusercontent.com/--Km7TVRp_-I/UhxB2UUSg7I/AAAAAAAAALY/_CPZbbFLchY/s1024/Screen%2520Shot%25202013-06-07%2520at%252011.48.35%2520PM.png)
 
 然后递归每个container再进行刚才的操作，最终就能成为文章中最一开始的那个图的样子，对于一个tree的生成最重要的就是，把当前的location list折中，中间的节点的前驱list作为左节点，后继list作为右节点，list指针作为tree节点，然后递归每个节点。

看下代码：

	static ngx_http_location_tree_node_t *
	ngx_http_create_locations_tree(ngx_conf_t *cf, ngx_queue_t *locations,
    size_t prefix)
	{
    size_t len;
    ngx_queue_t *q, tail;
    ngx_http_location_queue_t *lq;
    ngx_http_location_tree_node_t *node;

    q = ngx_queue_middle(locations);         //取中间节点

    lq = (ngx_http_location_queue_t *) q;
    len = lq->name->len – prefix;            //len是name减去prefix的长度

    node = ngx_palloc(cf->pool,
                      offsetof(ngx_http_location_tree_node_t, name) + len);
    if (node == NULL) {
        return NULL;
    }

    node->left = NULL;
    node->right = NULL;
    node->tree = NULL;
    node->exact = lq->exact;
    node->inclusive = lq->inclusive;

    node->auto_redirect = (u_char) ((lq->exact && lq->exact->auto_redirect)
                           || (lq->inclusive && lq->inclusive->auto_redirect));

    node->len = (u_char) len;
    ngx_memcpy(node->name, &lq->name->data[prefix], len);      //可以看到实际node的name是父节点的增量（不存储公共前缀，也许这是为了节省空间）

    ngx_queue_split(locations, q, &tail);                  //location队列是从头节点开始到q节点之前的节点，tail是q节点到location左右节点的队列

    if (ngx_queue_empty(locations)) { 
        /*
         * ngx_queue_split() insures that if left part is empty,
         * then right one is empty too
         */
        goto inclusive;
    }

    node->left = ngx_http_create_locations_tree(cf, locations, prefix); //递归构建node的左节点
	if (node->left == NULL) {
        return NULL;
    }

    ngx_queue_remove(q);

    if (ngx_queue_empty(&tail)) {
        goto inclusive;
    }

    node->right = ngx_http_create_locations_tree(cf, &tail, prefix);//递归构建node的右节点
    if (node->right == NULL) {
        return NULL;
    }

	inclusive:

    if (ngx_queue_empty(&lq->list)) {
        return node;
    }

    node->tree = ngx_http_create_locations_tree(cf, &lq->list, prefix + len);      //根据list指针构造node的tree指针
    if (node->tree == NULL) {
        return NULL;
    }

    return node;
	}
	
>总结：
     
> static location tree大大优化了精准匹配和前缀匹配的location的查找过程，线性递归查找效率低下，三叉树的左节点代表当前比node节点的name小的节点，右节点代表比当前node节点name大的节点，tree节点表示拥有相同前缀的节点。
