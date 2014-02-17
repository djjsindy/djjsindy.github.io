Title:  nginx hash结构存储 
Date: 2013-06-18 17:34:00
Category: nginx
Tags: nginx
Slug: nginx-hash
Author: djjsindy
Summary:  nginx在存储server_name和ngx_http_core_srv_conf_t的映射的时候用到了hash结构


nginx在存储server_name和`ngx_http_core_srv_conf_t`的映射的时候用到了hash结构，nginx中的非通配符server_name存储hash结构类似如下形式

![ALT](https://lh4.googleusercontent.com/-I6JcsqZXX9I/UhCBkJl_L9I/AAAAAAAAABE/GbXHXtRWWsA/s1024/Screen%2520Shot%25202013-06-18%2520at%25205.18.34%2520PM.png)

 配置`server_names_hash_max_size`控制bucket的最大数量，`server_names_hash_bucket_size`控制每个bucket的大小，每个bucket可以盛放一定数量的`ngx_hash_elt_t`，每个bucket存放的`ngx_hash_elt_t`都拥有相同的hash％size。这里就要求拥有相同hash％size的元素全都放在一个bucket中，因为每个元素的hash值是确定的，size是不确定的。所以有必要从一定的size开始测试，看能不能保证拥有相同hash％size的元素放在同一个bucket中，如果不能，那么就继续加大size，因为加大size有利于元素分布的更分散一些。所以测试size的大小直到`server_names_hash_max_size`，如果到达了max size也不能使得相同的hash％size的元素同时分布在一个bucket中，那么就会报错了，提示`server_names_hash_max_size`和`server_names_hash_bucket_size`设置的不合理。
 
   每个bucket的布局如图
   
![ALT](https://lh5.googleusercontent.com/-7BGRzD0PfX8/UhCBnS-7E4I/AAAAAAAAABM/e1YfL00u0dk/s800/Screen%2520Shot%25202013-06-18%2520at%25205.21.25%2520PM.png)


 看下代码的实现:
   
 构建hash的过程分为4步
  
 1. 第一步，检验bucket是否能存储每一个元素，如果设置的bucket_size过小，server_name又比较长，那么bucket是不能存下一个元素的。
 
 
 
     	ngx_int_t
		ngx_hash_init(ngx_hash_init_t *hinit, ngx_hash_key_t *names, ngx_uint_t nelts)
		{
    		u_char *elts; 
    	    size_t len; 
    		u_short *test; 
    		ngx_uint_t i, n, key, size, start, bucket_size; 
    		ngx_hash_elt_t *elt, **buckets;
    		
    		//遍历每一个元素，看下bucket是否能存储下每一个元素,这里 NGX_HASH_ELT_SIZE是每一个元素的大小
    		//#define NGX_HASH_ELT_SIZE(name) 
    		//(sizeof(void *) + ngx_align((name)->key.len + 2, sizeof(void *)))
    		//这里面每个name的value指向ngx_http_core_srv_conf_t，后面的key的大小就是key.len,再存储这个len，是个short类型，需要两个2字节的空间
    		
    		for (n = 0; n < nelts; n++) { 
        		if (hinit->bucket_size < NGX_HASH_ELT_SIZE(&names[n]) + sizeof(void *)) 
        		//每个bucket的size 再加上sizeof(void *)的目的是每个bucket有个结束元素，这个元素的value是个指针，指向null。
        		{
            		ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                          "could not build the %s, you should "
                          "increase %s_bucket_size: %i", 
                          hinit->name, hinit->name, hinit->bucket_size);
            		return NGX_ERROR; 
        		}
    	}
   
2.  第二步 要确定hash 的size，size尽可能的小，同时要保证具有相同hash％size的元素都再一个bucket中。
		
		//test数组中存放每个bucket的当前容量，如果某一个key的容量大于了bucket size就意味着需要加大hash桶的个数了
    	test = ngx_alloc(hinit->max_size * sizeof(u_short), hinit->pool->log);
    	if (test == NULL) { 
        return NGX_ERROR;
    	} 

   		//每个bucket的size需要去掉最后一个元素所占的空间，这个元素是个哑元素，用来判断当前bucket是否还有元素
    	bucket_size = hinit->bucket_size - sizeof(void *);
   		//start是最少的bucket的数目，因为每个元素占有的空间是value的指针和后面len和name按照字节对齐的总和
    	start = nelts / (bucket_size / (2 * sizeof(void *)));
    	start = start ? start : 1;
   		//经验值
    	if (hinit->max_size > 10000 && nelts && hinit->max_size / nelts < 100) {
        	start = hinit->max_size - 1000;
    	} 
   		//size从start开始，逐渐加大bucket的个数，直到恰好满足所有具有相同hash％size的元素都在同一个bucket，这样hash的size就能确定了
    	for (size = start; size < hinit->max_size; size++) {
      	//每次递归新的size的时候需要将旧test的数据清空
        	ngx_memzero(test, size * sizeof(u_short));
         //新size的开始，遍历每一个元素
        	for (n = 0; n < nelts; n++) { 
            	if (names[n].key.data == NULL) {
                	continue; 
            	}
            	key = names[n].key_hash % size;
            	//开始叠加每个bucket的size
            	test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
            	//如果某个bucket的size超过了bucket_size，那么加大bucket的个数，使得元素分布更分散一些
            	if (test[key] > (u_short) bucket_size) {
                	goto next;
            	}
        	}

        goto found;

    	next:

        continue;
    	}
    	 //最终如果size超过了max size，就报错
    	ngx_log_error(NGX_LOG_EMERG, hinit->pool->log, 0,
                  "could not build the %s, you should increase "
                  "either %s_max_size: %i or %s_bucket_size: %i",
                  hinit->name, hinit->name, hinit->max_size,
                  hinit->name, hinit->bucket_size);

    	ngx_free(test);

    	return NGX_ERROR;
    	
3. 第三步：表示找到了合适的bucket个数，使得所有具有相同的hash％size的元素都在同一个bucket中，需要根据元素具体数量，分配地址空间

		found:
    	//每个bucket都有一个哑元素
    	for (i = 0; i < size; i++) {
        	test[i] = sizeof(void *);
    	}
   		//遍历每个bucket计算每个bucket的大小
    	for (n = 0; n < nelts; n++) {
        	if (names[n].key.data == NULL) {
            	continue;
        	}

        	key = names[n].key_hash % size;
        	test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    	}

    	len = 0;
    	//叠加每个bucket 的大小，sum是len
    	for (i = 0; i < size; i++) {
        	if (test[i] == sizeof(void *)) {
            	continue;
        	}

        	test[i] = (u_short) (ngx_align(test[i], ngx_cacheline_size));

        	len += test[i];
    	}
    	//初始化hinit的hash结构空间
    	if (hinit->hash == NULL) {
        	hinit->hash = ngx_pcalloc(hinit->pool, sizeof(ngx_hash_wildcard_t)
                                             + size * sizeof(ngx_hash_elt_t *));
        	if (hinit->hash == NULL) {
            	ngx_free(test);
            	return NGX_ERROR;
        	}

        	buckets = (ngx_hash_elt_t **)
                      ((u_char *) hinit->hash + sizeof(ngx_hash_wildcard_t));

    	} else {
        	buckets = ngx_pcalloc(hinit->pool, size * sizeof(ngx_hash_elt_t *));
        	if (buckets == NULL) {
            	ngx_free(test);
            	return NGX_ERROR;
        	}
    	}
    	//malloc bucket的空间，根据那个上面计算出来的len
    	elts = ngx_palloc(hinit->pool, len + ngx_cacheline_size);
    	if (elts == NULL) {
        	ngx_free(test);
        	return NGX_ERROR;
    	}

    	elts = ngx_align_ptr(elts, ngx_cacheline_size);
    	//把elts地址空间分发到每个bucket中
    	for (i = 0; i < size; i++) {
        	if (test[i] == sizeof(void *)) {
            	continue;
        	}

        	buckets[i] = (ngx_hash_elt_t *) elts;
        	elts += test[i];

    	}

4. 第四步，hash结构已经有了，开始填充数据，填充每一个bucket

		for (i = 0; i < size; i++) {
        	test[i] = 0;
    	}

    	for (n = 0; n < nelts; n++) {
        	if (names[n].key.data == NULL) {
            	continue;
        	}
        	//这里test数组纪录每个bucket当前组后一个元素的位移，为了计算下一个元素的位置
        	key = names[n].key_hash % size;
        	elt = (ngx_hash_elt_t *) ((u_char *) buckets[key] + test[key]);

        	elt->value = names[n].value;
        	elt->len = (u_short) names[n].key.len;

        	ngx_strlow(elt->name, names[n].key.data, names[n].key.len);
        	//更新bucket中下一个元素的位置
        	test[key] = (u_short) (test[key] + NGX_HASH_ELT_SIZE(&names[n]));
    	}
    	//最后填充结束节点
    	for (i = 0; i < size; i++) {
        	if (buckets[i] == NULL) {
            	continue;
        	}
        	//强转不会出现问题，虽然只有一个指针的空间
        	elt = (ngx_hash_elt_t *) ((u_char *) buckets[i] + test[i]);
        
        	elt->value = NULL;
    	}

    	ngx_free(test);

   		hinit->hash->buckets = buckets;
    	hinit->hash->size = size;
    	
  
>
总结

>nginx的hash结构是静态的，也就是不需要插入删除元素，这里选择合适的bucket个数是这个init过程的关键，要保证bucket个数最小，同时要保证具有相同hash％size的元素都能存放在一个bucket当中。
		

