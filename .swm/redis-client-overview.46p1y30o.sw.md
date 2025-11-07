---
title: Redis Client Overview
---
# Redis Client Overview

The Redis Client in this project is implemented using the hiredis library, a minimalistic C client designed specifically for communicating with Redis databases. It serves as the bridge between the application and the Redis server, managing all aspects of connection, command formatting, sending, and response handling.

# Purpose and Functionality

This client simplifies interaction with Redis by abstracting the complexities of the Redis protocol and network communication. It supports both synchronous and asynchronous communication modes, allowing the application to choose the most suitable interaction pattern. The client ensures commands are correctly formatted according to the Redis protocol before transmission and handles replies in a binary-safe manner.

# Connection Management

The client manages network communication details such as establishing connections, handling disconnections, and managing errors to maintain a stable link with the Redis server. It supports connections over TCP sockets as well as Unix domain sockets, providing flexibility depending on deployment scenarios.

<SwmSnippet path="/deps\hiredis\net.c" line="256">

---

The function <SwmToken path="deps\hiredis\net.c" pos="345:2:2" line-data="int redisContextConnectTcp(redisContext *c, const char *addr, int port,">`redisContextConnectTcp`</SwmToken> establishes a TCP connection to a Redis server using the server's IP address and port. Internally, it calls <SwmToken path="deps\hiredis\net.c" pos="256:4:4" line-data="static int _redisContextConnectTcp(redisContext *c, const char *addr, int port,">`_redisContextConnectTcp`</SwmToken>, which handles address resolution, socket creation, and connection setup. This function supports both <SwmToken path="deps\hiredis\net.c" pos="269:13:13" line-data="    /* Try with IPv6 if no IPv4 address was found. We do it in this order since">`IPv4`</SwmToken> and <SwmToken path="deps\hiredis\net.c" pos="269:7:7" line-data="    /* Try with IPv6 if no IPv4 address was found. We do it in this order since">`IPv6`</SwmToken> addresses and configures socket options like blocking mode and keep-alive to ensure connection stability.

```c
static int _redisContextConnectTcp(redisContext *c, const char *addr, int port,
                                   const struct timeval *timeout,
                                   const char *source_addr) {
    int s, rv;
    char _port[6];  /* strlen("65535"); */
    struct addrinfo hints, *servinfo, *bservinfo, *p, *b;
    int blocking = (c->flags & REDIS_BLOCK);

    snprintf(_port, 6, "%d", port);
    memset(&hints,0,sizeof(hints));
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;

    /* Try with IPv6 if no IPv4 address was found. We do it in this order since
     * in a Redis client you can't afford to test if you have IPv6 connectivity
     * as this would add latency to every connect. Otherwise a more sensible
     * route could be: Use IPv6 if both addresses are available and there is IPv6
     * connectivity. */
    if ((rv = getaddrinfo(addr,_port,&hints,&servinfo)) != 0) {
         hints.ai_family = AF_INET6;
         if ((rv = getaddrinfo(addr,_port,&hints,&servinfo)) != 0) {
            __redisSetError(c,REDIS_ERR_OTHER,gai_strerror(rv));
            return REDIS_ERR;
        }
    }
    for (p = servinfo; p != NULL; p = p->ai_next) {
        if ((s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1)
            continue;

        c->fd = s;
        if (redisSetBlocking(c,0) != REDIS_OK)
            goto error;
        if (source_addr) {
            int bound = 0;
            /* Using getaddrinfo saves us from self-determining IPv4 vs IPv6 */
            if ((rv = getaddrinfo(source_addr, NULL, &hints, &bservinfo)) != 0) {
                char buf[128];
                snprintf(buf,sizeof(buf),"Can't get addr: %s",gai_strerror(rv));
                __redisSetError(c,REDIS_ERR_OTHER,buf);
                goto error;
            }
            for (b = bservinfo; b != NULL; b = b->ai_next) {
                if (bind(s,b->ai_addr,b->ai_addrlen) != -1) {
                    bound = 1;
                    break;
                }
            }
            freeaddrinfo(bservinfo);
            if (!bound) {
                char buf[128];
                snprintf(buf,sizeof(buf),"Can't bind socket: %s",strerror(errno));
                __redisSetError(c,REDIS_ERR_OTHER,buf);
                goto error;
            }
        }
        if (connect(s,p->ai_addr,p->ai_addrlen) == -1) {
            if (errno == EHOSTUNREACH) {
                redisContextCloseFd(c);
                continue;
            } else if (errno == EINPROGRESS && !blocking) {
                /* This is ok. */
            } else {
                if (redisContextWaitReady(c,timeout) != REDIS_OK)
                    goto error;
            }
        }
        if (blocking && redisSetBlocking(c,1) != REDIS_OK)
            goto error;
        if (redisSetTcpNoDelay(c) != REDIS_OK)
            goto error;

        c->flags |= REDIS_CONNECTED;
        rv = REDIS_OK;
        goto end;
    }
    if (p == NULL) {
        char buf[128];
        snprintf(buf,sizeof(buf),"Can't create socket: %s",strerror(errno));
        __redisSetError(c,REDIS_ERR_OTHER,buf);
        goto error;
    }

error:
    rv = REDIS_ERR;
end:
    freeaddrinfo(servinfo);
    return rv;  // Need to return REDIS_OK if alright
}

int redisContextConnectTcp(redisContext *c, const char *addr, int port,
                           const struct timeval *timeout) {
    return _redisContextConnectTcp(c, addr, port, timeout, NULL);
}
```

---

</SwmSnippet>

<SwmSnippet path="/deps\hiredis\net.c" line="356">

---

For local Redis instances, the function <SwmToken path="deps\hiredis\net.c" pos="356:2:2" line-data="int redisContextConnectUnix(redisContext *c, const char *path, const struct timeval *timeout) {">`redisContextConnectUnix`</SwmToken> connects through a Unix domain socket specified by a filesystem path. It creates a local socket, sets it to non-blocking mode during connection, and restores blocking mode afterward if necessary. This method can offer faster and more secure communication compared to TCP connections.

```c
int redisContextConnectUnix(redisContext *c, const char *path, const struct timeval *timeout) {
    int blocking = (c->flags & REDIS_BLOCK);
    struct sockaddr_un sa;

    if (redisCreateSocket(c,AF_LOCAL) < 0)
        return REDIS_ERR;
    if (redisSetBlocking(c,0) != REDIS_OK)
        return REDIS_ERR;

    sa.sun_family = AF_LOCAL;
    strncpy(sa.sun_path,path,sizeof(sa.sun_path)-1);
    if (connect(c->fd, (struct sockaddr*)&sa, sizeof(sa)) == -1) {
        if (errno == EINPROGRESS && !blocking) {
            /* This is ok. */
        } else {
            if (redisContextWaitReady(c,timeout) != REDIS_OK)
                return REDIS_ERR;
        }
    }

    /* Reset socket to be blocking after connect(2). */
    if (blocking && redisSetBlocking(c,1) != REDIS_OK)
        return REDIS_ERR;

    c->flags |= REDIS_CONNECTED;
    return REDIS_OK;
}
```

---

</SwmSnippet>

# Using the Redis Client

To use the Redis Client, the application first establishes a connection to the Redis server, typically via `redisConnect`. After connection, commands are issued synchronously or asynchronously. The client handles command formatting and response parsing internally, allowing developers to focus on application logic without dealing with low-level protocol details. Error handling and connection stability are managed by the client to ensure reliable communication.

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBUmVkaXNDc2FtcGxlJTNBJTNBdW1hbGluZ2Fzd2FtaQ==" repo-name="RedisCsample"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
