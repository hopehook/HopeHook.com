---
date: 2018-03-27
layout: post
title: epoll demo
thread: 2018-03-27-epoll_demo.md
categories: 计算机原理
tags: linux
---

1 客户端

<pre>
#include <stdio.h> 
#include <string.h> 
#include <sys/socket.h>
#include <netinet/in.h> 

#define MAXDATASIZE 1024
#define SERVERIP "127.0.0.1"
#define SERVERPORT 8000

int main( int argc, char * argv[] ) 
{
	char buf[MAXDATASIZE];
	int sockfd, numbytes;
	struct sockaddr_in server_addr;
	if ( ( sockfd = socket( AF_INET , SOCK_STREAM , 0) ) == -1) 
	{
		perror ( "socket error" );
		return 1;
	}
	memset ( &server_addr, 0, sizeof ( struct sockaddr ) );
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(SERVERPORT);
	server_addr.sin_addr.s_addr = inet_addr(SERVERIP);
	if ( connect ( sockfd, ( struct sockaddr * ) & server_addr, sizeof ( struct sockaddr ) ) == -1) 
	{
		perror ( "connect error" );
		return 1;
	}
	printf ( "send: Hello, world!\n" );
	if ( send ( sockfd, "Hello, world!" , 14, 0) == -1) 
	{
		perror ( "send error" );
		return 1;
	}
	if ( ( numbytes = recv ( sockfd, buf, MAXDATASIZE, 0) ) == -1) 
	{
		perror ( "recv error" );
		return 1;
	}
	if (numbytes) 
	{
		buf[numbytes] = '\0';
		printf ( "received: %s\n" , buf);
	}
	close(sockfd);
	return 0;
}
</pre>
<br></br>
2 服务端

<pre>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <errno.h>
#include <netinet/in.h>
#include <arpa/inet.h>


#define MAXDATASIZE 1024
#define MAXCONN_NUM 10
#define MAXEVENTS 64

struct sockaddr_in server_addr;
struct sockaddr_in client_addr;

static int make_socket_non_blocking (int sockfd)
{
    int flags, s;

    flags = fcntl (sockfd, F_GETFL, 0);
    if (flags == -1)
    {
        perror ("fcntl");
        return -1;
    }

    flags |= O_NONBLOCK;
    s = fcntl (sockfd, F_SETFL, flags);
    if (s == -1)
    {
        perror ("fcntl");
        return -1;
    }
    return 0;
}

static int create_and_bind(int port)
{
	int sockfd;
	if ( ( sockfd = socket ( AF_INET , SOCK_STREAM , 0) ) == -1) 
	{
        perror ( "socket error" );
        return -1;
    }

	memset(&client_addr, 0, sizeof(struct sockaddr));
	server_addr.sin_family = AF_INET;
	server_addr.sin_port = htons(port);
	server_addr.sin_addr.s_addr = INADDR_ANY;
	if ( bind( sockfd, ( struct sockaddr * ) &server_addr, sizeof(struct sockaddr) ) == -1) 
	{
        perror ( "bind error" );
        close (sockfd);
        return -1;
	}
	return sockfd;
}


int main (int argc, char *argv[])
{
    // 数据缓存区域
    char buf[MAXDATASIZE];

    // 检查是否指定端口
    if (argc != 2)
    {
        fprintf (stderr, "Usage: %s [port]\n", argv[0]);
        exit (EXIT_FAILURE);
    }
    char *port_argv = argv[1];
    int port = atoi(port_argv);
    
    // 创建并监听tcp socket
    int sockfd = create_and_bind (port);
    if (sockfd == -1)
        abort ();

    // 设置socket为非阻塞
    if (make_socket_non_blocking (sockfd) == -1)
        abort ();

    // 创建epoll句柄
    int epfd = epoll_create1 (0);
    if (epfd == -1)
    {
        perror ("epoll_create error");
        abort ();
    }

    // epoll_ctl
    struct epoll_event event;
    event.data.fd = sockfd;
    event.events = EPOLLIN | EPOLLET;
    if (epoll_ctl (epfd, EPOLL_CTL_ADD, sockfd, &event) == -1)
    {
        perror ("epoll_ctl error");
        abort ();
    }

    /* Buffer where events are returned */
    struct epoll_event *events;
    events = calloc (MAXEVENTS, sizeof event);

    // listen
	if ( listen(sockfd, MAXCONN_NUM ) == -1)
	{
        perror ( "listen error" );
        abort ();
	}

    /* The event loop */
    while(1)
    {
        int n, i, new_fd, numbytes;
        n = epoll_wait (epfd, events, MAXEVENTS, -1);
        for (i = 0; i < n; i++)
        {
            /* We have a notification on the listening socket, which
                 means one or more incoming connections. */
            if(events[i].data.fd == sockfd)
            {
                // accept
                int sin_size = sizeof( struct sockaddr_in );
                if ( ( new_fd = accept( sockfd, ( struct sockaddr * ) &client_addr, &sin_size) ) == -1) 
                {
                    perror ( "accept error" );
                    continue;
                }
                printf ("server: got connection from %s\n" , inet_ntoa( client_addr.sin_addr) ) ;

                // epoll_ctl
                event.data.fd = new_fd;
                event.events = EPOLLIN | EPOLLET;
                if (epoll_ctl (epfd, EPOLL_CTL_ADD, new_fd, &event) == -1)
                {
                    perror ("epoll_ctl error");
                    abort ();
                }
            } 
            else if(events[i].events & EPOLLIN)
            {
                if((new_fd = events[i].data.fd) < 0)
                    continue;
         
                if((numbytes = read(new_fd, buf, MAXDATASIZE)) < 0) {
                    if(errno == ECONNRESET) {
                        close(new_fd);
                        events[i].data.fd = -1;
                        epoll_ctl(epfd, EPOLL_CTL_DEL, new_fd, &event);
                    } 
                    else
                    {
                        printf("readline error");
                    }
                } 
                else if(numbytes == 0)
                {
                    close(new_fd);
                    events[i].data.fd = -1;
                    epoll_ctl(epfd, EPOLL_CTL_DEL, new_fd, &event);
                }
                // numbytes > 0
                else
                {
                    printf("received data: %s\n", buf);
                }
                event.data.fd = new_fd;
                event.events = EPOLLOUT | EPOLLET;
                epoll_ctl(epfd, EPOLL_CTL_MOD, new_fd, &event);
            }
            else if(events[i].events & EPOLLOUT)
            {
				new_fd = events[i].data.fd;
				write(new_fd, buf, numbytes);

                printf("written data: %s\n", buf);
                printf("written numbytes: %d\n", numbytes);

				event.data.fd = new_fd;
				event.events = EPOLLIN | EPOLLET;
				epoll_ctl(epfd, EPOLL_CTL_MOD, new_fd, &event);
			}
            else if ((events[i].events & EPOLLERR) || (events[i].events & EPOLLHUP))
            {
                /* An error has occured on this fd, or the socket is not
                ready for reading (why were we notified then?) */
                fprintf (stderr, "epoll error\n");
                new_fd = events[i].data.fd;
                close(new_fd);
                events[i].data.fd = -1;
                epoll_ctl(epfd, EPOLL_CTL_DEL, new_fd, &event);
                continue;
            }
        }
    }

}
</pre>
