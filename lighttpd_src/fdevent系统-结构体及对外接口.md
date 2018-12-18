fdevent系统主要是处理各种IO事件，在web服务器中，主要就是向socket写数据和从socket读数据。

web服务器是IO密集型程序，因此，大部分的web服务器都采用非阻塞IO进行数据的读写。

lighttpd通过fdevent系统，采用类似OO中面向对象的方式将对IO事件的处理进行封装，对于不同的IO系统，提供一个统一的接口。

lighttpd采用了所谓的Reactor模式，也就是非阻塞IO加多路复用（non-blocking IO + IO multiplexing）。在多路复用上，lighttpd通过fdevent将各种不同的实现进行封装。

fdevent.h中fdevents结构体相当于一个虚基类，其中的函数指针是纯虚函数。对于每种实现，则相当于继承了这个基类并实现了其中的纯虚函数，也就是给函数指针赋一个函数地址值：
/*
 * 重置和释放fdevent系统。
 */
int fdevent_reset(fdevents * ev);
void fdevent_free(fdevents * ev);
/*
 * 将fd增加到fd event系统中。events是要对fd要监听的事件。
 * fde_ndx是fd对应的fdnode在ev->fdarray中的下标值的指针。
 * 如果fde_ndx==NULL，则表示在fd event系统中增加fd。如果不为NULL，则表示这个
 * fd已经在系统中存在，这个函数的功能就变为将对fd监听的事件变为events。
 */
int fdevent_event_add(fdevents * ev, int *fde_ndx, int fd, int events);
/*
 * 从fd event系统中删除fd。 fde_ndx的内容和上面的一致。
 */
int fdevent_event_del(fdevents * ev, int *fde_ndx, int fd);
/*
 * 返回ndx对应的fd所发生的事件。
 * 这里的ndx和上面的fde_ndx不一样，这个ndx是ev->epoll_events中epoll_event结构体的下标。
 * 第一次调用的时候，通常ndx为-1。
 * 这个ndx和其对应的fd没有关系。而fde_ndx等于其对应的fd。
 */
int fdevent_event_get_revent(fdevents * ev, size_t ndx);
/*
 * 返回ndx对应的fd。
 */
int fdevent_event_get_fd(fdevents * ev, size_t ndx);
/*
 * 返回下一个发生IO事件的fd。
 */
int fdevent_event_next_fdndx(fdevents * ev, int ndx);
/*
 * 开始等待IO事件。timeout_ms是超时限制。
 */
int fdevent_poll(fdevents * ev, int timeout_ms);
/**
 * 设置fd的状态，通常是设置为运行exec在子进程中关闭和非阻塞。
 */
int fdevent_fcntl_set(fdevents * ev, int fd);

 
在fdevent.c文件中，这些函数的实现基本上都是简单的调用fdevents结构体中对应的函数指针。对于lighttpd，通过调用上面的这些函数完成IO事件的处理，对于具体到底是谁处理了这些事件，lighttpd并不知道，也不关心。

其他的函数声明：

/*
 * 返回fd对应的事件处理函数地址。也就是fdnode中handler的值。
 */
fdevent_handler fdevent_get_handler(fdevents * ev, int fd);
/*
 * 返回fd对应的环境。也就是fdnode中ctx的值。
 */
void *fdevent_get_context(fdevents * ev, int fd);

/*
 * 注册和取消注册fd。
 * 就是生成一个fdnode，然后保存在ev->fdarray中。或者删除之。
 */
int fdevent_register(fdevents * ev, int fd, fdevent_handler handler, void *ctx);
int fdevent_unregister(fdevents * ev, int fd);
/**
 * 初始化各种多路IO。
 */
int fdevent_select_init(fdevents * ev);
int fdevent_poll_init(fdevents * ev);
int fdevent_linux_rtsig_init(fdevents * ev);
int fdevent_linux_sysepoll_init(fdevents * ev);
int fdevent_solaris_devpoll_init(fdevents * ev);
int fdevent_freebsd_kqueue_init(fdevents * ev);
