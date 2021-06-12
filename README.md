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


### In the following, we will explain the different parts of the code provided for the server.


The server creates a new socket using the __socket()__ system call. This returns a file descriptor that can be used to refer to the socket in future system calls .


 #include <sys/socket.h>
 
 int socket(int _domain_ , int _type_ , int _protocol_);
 
Therefore socket() creates an endpoint for communication and returns a file descriptor that refers to that endpoint. The file descriptor returned by a successful call will be the lowest-numbered file descriptor not currently open for the process.

The _domain_ argument specifies a communication domain; this selects the protocol family which will be used for communication. These families are defined in <sys/socket.h> ( In this case  _domain_ : __AF_UNIX__   with porpuse: Local communication)

The socket has the indicated _type_ , which specifies the communication semantics .(In this case  _type_ : __SOCK_STREAM__ Provides sequenced, reliable, two-way, connection-based byte streams.  An out of-band data transmission mechanism  may be supported.)

The _protocol_ specifies a particular protocol to be used with the socket.  Normally only a single protocol exists to support a particular socket type within a given protocol family, in which case protocol can be specified as 0.  However, it is possible that many protocols may exist, in which case a particular protocol must be specified in this manner.  The protocol number to use is specific to the “communication domain” in which  communication is to take place;

So we will have in the server code:

```
// Create a new server socket with domain: AF_UNIX, type: SOCK_STREAM, protocol: 0
  int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
  printf("Server socket fd = %d\n", sfd);
  
```
_____________________________________________________________________________________________________________________________________________________________________________

Using the following command, we make sure that socket's file descriptor is legit.

```
if (sfd == -1) {
    errExit("socket");
  }

```

________________________________________________________________________________________________________________________________________________________________________________

Using the following command, we make sure the address we're planning to use isn't too long.

```
if (strlen(SV_SOCK_PATH) > sizeof(addr.sun_path) - 1) {
    fatal("Server socket path too long: %s", SV_SOCK_PATH);
  }
  
```


_______________________________________________________________________________________________________________________________________________________________________________


 Delete any file that already exists at the address. Make sure the deletion succeeds. If the error is just that the file/directory doesn't exist, it's fine.
 
```
  if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT) {
    errExit("remove-%s", SV_SOCK_PATH);
  }
  
```
_______________________________________________________________________________________________________________________________________________________________________________

```
// Zero out the address, and set family and path.
  memset(&addr, 0, sizeof(struct sockaddr_un));
  addr.sun_family = AF_UNIX;
  strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);
  
```
_______________________________________________________________________________________________________________________________________________________________________________

The server uses the __bind()__ system call to bind the socket to a well-known address, so that the client can connect to it .


 #include <sys/socket.h>
 
 int bind (int _sockfd_ , const struct sockaddr _*addr_, socklen_t _addrlen_);

When a socket is created with socket(), it exists in a name space (address family) but has no address assigned to it.  bind() assigns the address specified by _addr_ to the socket referred to by the file descriptor _sockfd_ . _addrlen_ specifies the size, in bytes, of the address structure pointed to by _addr_. Traditionally, this operation is called “assigning a name to a socket”. It is normally necessary to assign a local address using bind() before a SOCK_STREAM socket may receive connections.

The actual structure passed for the _addr_ argument will depend on the address family.  The sockaddr structure is defined as something like:


           struct sockaddr {
               sa_family_t sa_family;
               char        sa_data[14];
           }

 The only purpose of this structure is to cast the structure pointer passed in addr in order to avoid compiler warnings.
 On success, zero is returned.  On error, -1 is returned, and errno is set to indicate the error.
 
 So we will have in the server code:

```
// Bind the socket to the address. Note that we're binding the server socket
// to a well-known address so that clients know where to connect.
  if (bind(sfd, (struct sockaddr *) &addr, sizeof(struct sockaddr_un)) == -1) {
    errExit("bind");
  }
 
```
_______________________________________________________________________________________________________________________________________________________________________________







































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
