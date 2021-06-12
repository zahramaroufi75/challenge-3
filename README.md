# Getting Started With Unix Domain Sockets


Sockets provide a means of communication between processes, i.e. a way for them to exchange data. The way it usually works is that __process_a__ has __socket_x__ , __process_b__ has __socket_y__ , and the two sockets are connected. Each process can then use its socket to receive data from the other process and/or send data to the other process. One way to think about sockets is that they open up a communication channel where both sides can read and write.



![Image of Yaktocat](https://miro.medium.com/max/10240/1*3Ny5SBf14TwhpRR5vOqF2A.png)



Why do we need sockets? Well, because one process can’t normally talk to another process; this is true when the processes are on the same computer or on different computers. On a related note, there are two main domains of sockets: __Unix domain sockets__ , which allow processes on the same computer to communicate (IPC), and __Internet domain sockets__ , which allow processes to communicate over a network. If that’s confusing, just think of “Unix domain” and “Internet domain” as adjectives that describe the communication range of a socket.



![Image of Yaktocat](https://miro.medium.com/max/9040/1*ekw1o4xE_7ew9kYh6tVkCA.png)


Note: Each socket has two important attributes : a communication domain and a type. There are two main types, stream and datagram. In this post, I’m going to focus on the former. That is, I’m going to focus on streaming Unix domain sockets.


## Server Code

```
#include "lib/error_functions.h"
#include "sockets/unix_socket.h"

#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#define BACKLOG 5

int main(int argc, char *argv[]) {
  struct sockaddr_un addr;

  int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
  printf("Server socket fd = %d\n", sfd);

  if (sfd == -1) {
    errExit("socket");
  }

  if (strlen(SV_SOCK_PATH) > sizeof(addr.sun_path) - 1) {
    fatal("Server socket path too long: %s", SV_SOCK_PATH);
  }

 
  if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT) {
    errExit("remove-%s", SV_SOCK_PATH);
  }


  memset(&addr, 0, sizeof(struct sockaddr_un));
  addr.sun_family = AF_UNIX;
  strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);

  if (bind(sfd, (struct sockaddr *) &addr, sizeof(struct sockaddr_un)) == -1) {
    errExit("bind");
  }

 
  if (listen(sfd, BACKLOG) == -1) {
    errExit("listen");
  }

  ssize_t numRead;
  char buf[BUF_SIZE];
  for (;;) {          /* Handle client connections iteratively */
   
    printf("Waiting to accept a connection...\n");
  
    int cfd = accept(sfd, NULL, NULL);
    printf("Accepted socket fd = %d\n", cfd);

    while ((numRead = read(cfd, buf, BUF_SIZE)) > 0) {
     
      if (write(STDOUT_FILENO, buf, numRead) != numRead) {
        fatal("partial/failed write");
      }
    }

    if (numRead == -1) {
      errExit("read");
    }

    if (close(cfd) == -1) {
      errMsg("close");
    }
  }
}

```





## Client Code

```
#include "lib/error_functions.h"
#include "sockets/unix_socket.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    struct sockaddr_un addr;
    ssize_t numRead;
    char buf[BUF_SIZE];

    int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    printf("Client socket fd = %d\n", sfd);

   
    if (sfd == -1) {
      errExit("socket");
    }

  
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);

    if (connect(sfd, (struct sockaddr *) &addr,
                sizeof(struct sockaddr_un)) == -1) {
      errExit("connect");
    }


    while ((numRead = read(STDIN_FILENO, buf, BUF_SIZE)) > 0) {
     
      if (write(sfd, buf, numRead) != numRead) {
        fatal("partial/failed write");
      }
    }

    if (numRead == -1) {
      errExit("read");
    }

    exit(EXIT_SUCCESS);
}

```
