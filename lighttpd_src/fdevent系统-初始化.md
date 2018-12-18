fdevent.h中对所有可能的多路IO系统都定义了初始化函数：

 int fdevent_select_init(fdevents * ev);
 int fdevent_poll_init(fdevents * ev);
 int fdevent_linux_rtsig_init(fdevents * ev);
 int fdevent_linux_sysepoll_init(fdevents * ev);
 int fdevent_solaris_devpoll_init(fdevents * ev);
 int fdevent_freebsd_kqueue_init(fdevents * ev);


因此，对于系统中没有的多路IO系统对应的初始化函数，预编译结束后，这些初始化函数被定义为报错函数。如epoll对应的为：

#ifdef USE_LINUX_EPOLL
/* 当定义了epoll时,epoll的函数实现代码 */
#else 
/* 当未定义epoll时,epoll只实现init函数（这是必须的，因为该函数在fdevent.h中定义了），并将它实现为报错函数 */
int fdevent_linux_sysepoll_init(fdevents *ev) {
    UNUSED(ev);

    fprintf(stderr, "%s.%d: linux-sysepoll not supported, try to set server.event-handler = \"poll\" or \"select\"\n",
        __FILE__, __LINE__);

    return -1;
}
#endif

们看一看配置中有关fdevent的设置。进入configfile.c文件中的config_set_defaults()函数
struct ev_map { fdevent_handler_t et; const char *name; } event_handlers[] = { /*
         * - poll is most reliable - select works everywhere -
* linux-* are experimental
         */
上面定义了一个struct ev_map类型的数组。数组的内容是当前系统中存在的多路IO系统的类型和名称。这里排序很有意思，从注释中可以看出，poll排在最前因为最可靠，select其次因为支持最广泛，epoll第三因为是最好的。


当子程序产生后，初始化fdevent系统：
    if (NULL == (srv->ev = fdevent_init(srv->max_fds + 1, srv->event_handler))) {
        log_error_write(srv, __FILE__, __LINE__,
                "s", "fdevent_init failed");
        return -1;
    }


//初始文件描述字符
 fdevents *fdevent_init(size_t maxfds, fdevent_handler_t type) {
    fdevents *ev;

    //内存初始0
    ev = calloc(1, sizeof(*ev));

    //分配数组
    ev->fdarray = calloc(maxfds, sizeof(*ev->fdarray));
    ev->maxfds = maxfds;

    //根据IO进行初始化
    switch(type) {
    case FDEVENT_HANDLER_POLL:
        if (0 != fdevent_poll_init(ev)) {
            fprintf(stderr, "%s.%d: event-handler poll failed\n",
                __FILE__, __LINE__);

            return NULL;
        }
        break;
    case FDEVENT_HANDLER_SELECT:
        if (0 != fdevent_select_init(ev)) {
            fprintf(stderr, "%s.%d: event-handler select failed\n",
                __FILE__, __LINE__);
            return NULL;
        }
        break;
    case FDEVENT_HANDLER_LINUX_RTSIG:
        if (0 != fdevent_linux_rtsig_init(ev)) {
            fprintf(stderr, "%s.%d: event-handler linux-rtsig failed, try to set server.event-handler = \"poll\" or \"select\"\n",
                __FILE__, __LINE__);
            return NULL;
        }
        break;
    case FDEVENT_HANDLER_LINUX_SYSEPOLL:
        if (0 != fdevent_linux_sysepoll_init(ev)) {
            fprintf(stderr, "%s.%d: event-handler linux-sysepoll failed, try to set server.event-handler = \"poll\" or \"select\"\n",
                __FILE__, __LINE__);
            return NULL;
        }
        break;
    case FDEVENT_HANDLER_SOLARIS_DEVPOLL:
        if (0 != fdevent_solaris_devpoll_init(ev)) {
            fprintf(stderr, "%s.%d: event-handler solaris-devpoll failed, try to set server.event-handler = \"poll\" or \"select\"\n",
                __FILE__, __LINE__);
            return NULL;
        }
        break;
    case FDEVENT_HANDLER_FREEBSD_KQUEUE:
        if (0 != fdevent_freebsd_kqueue_init(ev)) {
            fprintf(stderr, "%s.%d: event-handler freebsd-kqueue failed, try to set server.event-handler = \"poll\" or \"select\"\n",
                __FILE__, __LINE__);
            return NULL;
        }
        break;
    default:
        fprintf(stderr, "%s.%d: event-handler is unknown, try to set server.event-handler = \"poll\" or \"select\"\n",
            __FILE__, __LINE__);
        return NULL;
    }

    return ev;
}   

//fdevent_init()函数根据fdevent_handler_t的值调用相应的初始化函数
//fdevent_linux_sysepoll_init() 
int fdevent_linux_sysepoll_init(fdevents *ev) {
    ev->type = FDEVENT_HANDLER_LINUX_SYSEPOLL;
#define SET(x) \
    ev->x = fdevent_linux_sysepoll_##x; 
    //通过SET宏对fdevents结构体中的函数指针赋值。然后创建epoll，最后做一些设置。

    SET(free);
    SET(poll);

    SET(event_del);
    SET(event_add);

    SET(event_next_fdndx);
    SET(event_get_fd);
    SET(event_get_revent);

    //创建epoll
    if (-1 == (ev->epoll_fd = epoll_create(ev->maxfds))) {
        fprintf(stderr, "%s.%d: epoll_create failed (%s), try to set server.event-handler = \"poll\" or \"select\"\n",
            __FILE__, __LINE__, strerror(errno));

        return -1;
    }
    //epoll函数运行exec()是关闭
    if (-1 == fcntl(ev->epoll_fd, F_SETFD, FD_CLOEXEC)) {
        fprintf(stderr, "%s.%d: epoll_create failed (%s), try to set server.event-handler = \"poll\" or \"select\"\n",
            __FILE__, __LINE__, strerror(errno));

        close(ev->epoll_fd);

        return -1;
    }
    ////创建fd事件数组。在epoll_wait函数中使用。
    //存储发生了IO事件的fd和对应的IO事件。
    ev->epoll_events = malloc(ev->maxfds * sizeof(*ev->epoll_events));

    return 0;
}