Title: tomcat的异步servlet处理
Date: 2013-07-17 12:02:00
Category: tomcat
Tags: tomcat
Slug: tomcat-async
Author: djjsindy
Summary:  这个是以前看的，由于最近工作中用到异步servlet，看了一下tomcat在对servlet3.0中AsyncContext的实现过程，总结一下，使用异步servlet主要原因就是因为，在service方法中业务逻辑如果碰到io操作时间比较长的操作，这样这个service方法就会长时间占用tomcat容器线程池中的线程，这样是不利于其他请求的处理的，当线程池中的线程处理任务时，任务由于长时间io操作，肯定会阻塞线程处理其他任务，引入异步servlet的目的就是将容器线程池和业务线程池分离开。在处理大io的业务操作的时候，把这个操作移动到业务线程池中进行，释放容器线程，使得容器线程处理其他任务，在业务逻辑执行完毕之后，然后在通知tomcat容器线程池来继续后面的操作，这个操作应该是把处理结果commit到客户端或者是dispatch到其他servlet上。


这个是以前看的，由于最近工作中用到异步servlet，看了一下tomcat在对servlet3.0中`AsyncContext`的实现过程，总结一下，使用异步servlet主要原因就是因为，在service方法中业务逻辑如果碰到io操作时间比较长的操作，这样这个service方法就会长时间占用tomcat容器线程池中的线程，这样是不利于其他请求的处理的，当线程池中的线程处理任务时，任务由于长时间io操作，肯定会阻塞线程处理其他任务，引入异步servlet的目的就是将容器线程池和业务线程池分离开。在处理大io的业务操作的时候，把这个操作移动到业务线程池中进行，释放容器线程，使得容器线程处理其他任务，在业务逻辑执行完毕之后，然后在通知tomcat容器线程池来继续后面的操作，这个操作应该是把处理结果commit到客户端或者是dispatch到其他servlet上。
![Alt text](https://lh4.googleusercontent.com/-NNjNxSYLdCA/UhCB_tLhmPI/AAAAAAAAABk/GalwMQ-zXrM/s512/Screen%2520Shot%25202013-07-17%2520at%252011.40.30%2520AM.png)

上图是原始servlet和容器线程池的模型，下图是异步servlet，容器线程池和业务线程池的模型，从图中可以看出，原始模型在处理业务逻辑的过程中会一直占有容器线程池，而异步servlet模型，可以看出在业务线程池处理的过程中，有一段时间容器线程池中的那个线程是空闲的，这种设计大大提高了容器的处理请求的能力。

异步servlet的开启在service中开启，对于一般请求，在service方法之后，都会commit response结果到客户端，但是在异步servlet中这个commit是没有意义的，因为输出还没产生，在业务线程池中还未处理完毕，这时需要把当前处理环境保存起来，以便业务线程池处理完毕后，再次找到这个处理环境继续处理。
 
 参考下代码，重点看下普通servlet和异步servlet处理分支的逻辑
 
	// 当有请求过来的时候，或者是异步servlet，业务线程处理完毕，通知容器线程处理后续的逻辑，都会触发这个process函数
	public SocketState process(SocketWrapper<S> socket,
                SocketStatus status) {
	//可以看出来如果是异步servlet，会从connections中取回之前执行servlet时的上下文环境
            Processor<S> processor = connections.remove(socket.getSocket());

            if (status == SocketStatus.DISCONNECT && processor == null) {
                //nothing more to be done endpoint requested a close
                //and there are no object associated with this connection
                return SocketState.CLOSED;
            }

            socket.setAsync(false);
	// 如果没有取到processor，就意味着不是异步的servlet
            try {
                if (processor == null) {
                    processor = recycledProcessors.poll();
                }
                if (processor == null) {
                    processor = createProcessor();
                }

                initSsl(socket, processor);

                SocketState state = SocketState.CLOSED;
                do {
                    if (status == SocketStatus.DISCONNECT &&
                            !processor.isComet()) {
                        // Do nothing here, just wait for it to get recycled
                        // Don't do this for Comet we need to generate an end
                        // event (see BZ 54022)
                    } else if (processor.isAsync() ||
                            state == SocketState.ASYNC_END) { 
	//异步servlet的后续处理，这里做直接返回给客户端结果，或者请求其他servlet，这里为什么是判断isAsync和state呢，因为complete操作和dispatch操作，可以认为dispatch操作最后processor的状态还是异步，但是complete就不是了只是前面循环返回的state是ASYNC_END,所以这里要判断isAsync或者state，isAsync代表dispatch的后续操作，ASYNC_END代表complete的后续操作
                        state = processor.asyncDispatch(status);
                    } else if (processor.isComet()) { //tomcat 对comet请求的处理
                        state = processor.event(status);
                    } else if (processor.isUpgrade()) { //tomcat对 websocket的处理
                        state = processor.upgradeDispatch();
                    } else {
                        state = processor.process(socket); //普通servlet的处理
                    }
        //这里post操作很重要，目的是改变异步servlet的状态机，这里来判断是否业务线程池中的任务是否处理完毕，如果处理完毕，则state返回 ASYNC_END，如果未处理完毕，则表示当前循环中，等不到业务线程池处理任务完毕了，这样当前占用的容器线程池就需要被释放了，等待异步servlet通知容器线程池重新处理servlet
                    if (state != SocketState.CLOSED && processor.isAsync()) { 
                        state = processor.asyncPostProcess();
                    }

                    if (state == SocketState.UPGRADING) {
                        // Get the UpgradeInbound handler
                        UpgradeInbound inbound = processor.getUpgradeInbound();
                        // Release the Http11 processor to be re-used
                        release(socket, processor, false, false);
                        // Create the light-weight upgrade processor
                        processor = createUpgradeProcessor(socket, inbound);
                        inbound.onUpgradeComplete();
                    }
                } while (state == SocketState.ASYNC_END ||
                        state == SocketState.UPGRADING); 
	//刚才的post操作如果返回的状态是ASYNC_END，那么说明在当前容器线程处理任务之前，业务线程已经处理完任务了，那么就不需要释放当前容器线程池了，只需要再次在当前容器线程中继续处理servlet就行，否则表示容器线程不需要等待业务线程了，直接释放容器线程吧。
	//state是long表示当前处理的servlet是异步servlet，需要保存当前处理上下文，需要把这个processor保存起来，再业务线程处理完毕之后，再从connnections中取得这个processor，见代码的第一行。
                if (state == SocketState.LONG) {
                    // In the middle of processing a request/response. Keep the
                    // socket associated with the processor. Exact requirements
                    // depend on type of long poll
                    longPoll(socket, processor);
                } else if (state == SocketState.OPEN) { //open表示Http1.1的长连接
                    // In keep-alive but between requests. OK to recycle
                    // processor. Continue to poll for the next request.
                    release(socket, processor, false, true);
                } else if (state == SocketState.SENDFILE) {
                    // Sendfile in progress. If it fails, the socket will be
                    // closed. If it works, the socket will be re-added to the
                    // poller
                    release(socket, processor, false, false);
                } else if (state == SocketState.UPGRADED) {
                    // Need to keep the connection associated with the processor
                    longPoll(socket, processor);
                } else {
                    // Connection closed. OK to recycle the processor.
                    if (!(processor instanceof UpgradeProcessor)) {
                        release(socket, processor, true, false);
                    }
                }
                return state;
      
      
上面代码中很重要的一点就是每个processor，也就是每个servlet的处理过程，这个processor里面有个状态机，来记录异步servlet的过程。

1. `DISPATCHED`：普通servlet结束的状态
2. `STARTING`：servlet开始异步时的状态
3. `STARTED`：当前servlet已经开始异步，释放容器线程之前异步servlet并未结束的状态
4. `MUST_COMPLETE`：释放容器线程之前，异步servlet已经结束的状态（complete函数）
5. `COMPLETING`：异步servlet并未dispatch到其他servlet上，然后异步结束的状态
6. `TIMING_OUT`：当前异步servlet已经超时的状态
7. `MUST_DISPATCH`：释放容器线程之前，异步servlet dispatch到其他servlet上的状态
8. `DISPATCHING`：异步servlet结束，dispatch到其他servlet上的状态
9. `ERROR`：异步servlet异常的状态


异步servlet结束的操作有两种一种是在异步servlet中调用`complete`函数，另一种是调用`dispatch`函数，但是业务线程的结束时间是有说法的，如果在容器线程池释放线程之前业务线程就结束了，那么是不会释放容器线程的，状态机回出现 `MUST_COMPLETE`和 `MUST_DISPATCH`这两种状态，相反如果释放容器线程时，业务线程还没有处理完毕，那么异步servlet结束时就会出现 `COMPLETING`， `DISPATCHING`这两种状态，同时`dispatch`函数在业务线程池中只是在`AsyncContext`中设置需要`dispatch`到其他servlet，但是并不会直接`dispatch`，而是等到容器线程池，在后面处理的时候再去真正的dispatch。


状态迁移图来源于tomcat源码注解

![Alt text](https://lh5.googleusercontent.com/-ecPpiOiyEcU/UhCCJ6Al_lI/AAAAAAAAABs/3WXifccVdbU/s640/Screen%2520Shot%25202013-07-17%2520at%252011.48.33%2520AM.png)


对于dispatch操作，在业务线程中只是设置一下要dispatch，然后激发一下状态机。

代码在AsyncContextImpl中

	public void dispatch(ServletContext context, String path) {
        if (log.isDebugEnabled()) {
            logDebug("dispatch ");
        }
        check();
        if (request.getAttribute(ASYNC_REQUEST_URI)==null) {
            request.setAttribute(ASYNC_REQUEST_URI, request.getRequestURI());
            request.setAttribute(ASYNC_CONTEXT_PATH, request.getContextPath());
            request.setAttribute(ASYNC_SERVLET_PATH, request.getServletPath());
            request.setAttribute(ASYNC_PATH_INFO, request.getPathInfo());
            request.setAttribute(ASYNC_QUERY_STRING, request.getQueryString());
        }
        final RequestDispatcher requestDispatcher = context.getRequestDispatcher(path);
        if (!(requestDispatcher instanceof AsyncDispatcher)) {
            throw new UnsupportedOperationException(
                    sm.getString("asyncContextImpl.noAsyncDispatcher"));
        }
        final AsyncDispatcher applicationDispatcher =
                (AsyncDispatcher) requestDispatcher;
        final HttpServletRequest servletRequest =
                (HttpServletRequest) getRequest();
        final HttpServletResponse servletResponse =
                (HttpServletResponse) getResponse();
        //这个runnable暂时不执行，后面会invoke到
        Runnable run = new Runnable() {
            @Override
            public void run() {
                //状态迁移到DISPATCHED
                request.getCoyoteRequest().action(ActionCode.ASYNC_DISPATCHED, null);
                try {
                    applicationDispatcher.dispatch(servletRequest, servletResponse);//真正dispatch的函数
                }catch (Exception x) {
                    //log.error("Async.dispatch",x);
                    throw new RuntimeException(x);
                }
            }
        };
        
        this.dispatch = run;
        // 迁移到 MUST_DISPATCH或者 DISPATCHING状态
        this.request.getCoyoteRequest().action(ActionCode.ASYNC_DISPATCH, null); 
    }
    
最后在`standardWrapperValve`中可以看到上面那个runnable被执行，这个是地方是普通servlet请求，容器最终调用service函数的地方。

	if (request.isAsyncDispatching()) { //是DISPATCHING状态
                        //TODO SERVLET3 - async
                        ((AsyncContextImpl)request.getAsyncContext()).doInternalDispatch(); //执行runnable
                    } else if (comet) {
                        request.setComet(true);
                        filterChain.doFilterEvent(request.getEvent());
                    } else {
                        filterChain.doFilter
                            (request.getRequest(), response.getResponse());
                    }