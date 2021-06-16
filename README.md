# Getting Started With Unix Domain Sockets


Sockets provide a means of communication between processes, i.e. a way for them to exchange data. The way it usually works is that __process_a__ has __socket_x__ , __process_b__ has __socket_y__ , and the two sockets are connected. Each process can then use its socket to receive data from the other process and/or send data to the other process. One way to think about sockets is that they open up a communication channel where both sides can read and write.



![Image of Yaktocat](https://miro.medium.com/max/10240/1*3Ny5SBf14TwhpRR5vOqF2A.png)



Why do we need sockets? Well, because one process can’t normally talk to another process; this is true when the processes are on the same computer or on different computers. On a related note, there are two main domains of sockets: __Unix domain sockets__ , which allow processes on the same computer to communicate (IPC), and __Internet domain sockets__ , which allow processes to communicate over a network. If that’s confusing, just think of “Unix domain” and “Internet domain” as adjectives that describe the communication range of a socket.



![Image of Yaktocat](https://miro.medium.com/max/9040/1*ekw1o4xE_7ew9kYh6tVkCA.png)


Note: Each socket has two important attributes : a communication domain and a type. There are two main types, stream and datagram. In this post, I’m going to focus on the former. That is, I’m going to focus on streaming Unix domain sockets.
__________________________________________________________________________________________________________________________________________________________________

# Server Process (AKA the server)

This process binds its socket to a known location and accepts incoming connection requests from clients. For each connection request that is received, a new socket is created that is used to communicate with the peer socket (peer socket = the socket at the other end of the connection, in this case the socket created by some client process).


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
__________________________________________________________________________________________________________________________________________________________________

The server creates a new socket using the __socket()__ system call. This returns a file descriptor that can be used to refer to the socket in future system calls .


 #include <sys/socket.h>
 
 int socket(int _domain_ , int _type_ , int _protocol_);
 
Therefore socket() creates an endpoint for communication and returns a file descriptor that refers to that endpoint. The file descriptor returned by a successful call will be the lowest-numbered file descriptor not currently open for the process.

The _domain_ argument specifies a communication domain; this selects the protocol family which will be used for communication. These families are defined in <sys/socket.h> ( In this case  _domain_ : __AF_UNIX__   with porpuse: Local communication )

The socket has the indicated _type_ , which specifies the communication semantics . (In this case  _type_ : __SOCK_STREAM__ provides sequenced, reliable, two-way, connection-based byte streams.  An out of-band data transmission mechanism  may be supported.)

The _protocol_ specifies a particular protocol to be used with the socket.  Normally only a single protocol exists to support a particular socket type within a given protocol family, in which case protocol can be specified as 0.  However, it is possible that many protocols may exist, in which case a particular protocol must be specified in this manner.  The protocol number to use is specific to the “communication domain” in which  communication is to take place;

So we will have in the server code:

```
// Create a new server socket with domain: AF_UNIX, type: SOCK_STREAM, protocol: 0
  int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
  printf("Server socket fd = %d\n", sfd);
  
```
__________________________________________________________________________________________________________________________________________________________________

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


__________________________________________________________________________________________________________________________________________________________________


 Delete any file that already exists at the address. Make sure the deletion succeeds. If the error is just that the file/directory doesn't exist, it's fine.
 
```
  if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT) {
    errExit("remove-%s", SV_SOCK_PATH);
  }
  
```
________________________________________________________________________________________________________________________________________________________________________________

```
// Zero out the address, and set family and path.
  memset(&addr, 0, sizeof(struct sockaddr_un));
  addr.sun_family = AF_UNIX;
  strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);
  
```
________________________________________________________________________________________________________________________________________________________________________________

The server uses the __bind()__ system call to bind the socket to a well-known address, so that the client can connect to it .


 #include <sys/socket.h>
 
 int bind ( int _sockfd_ , const struct sockaddr _*addr_ , socklen_t _addrlen_ );

When a socket is created with socket(), it exists in a name space (address family) but has no address assigned to it.  bind() assigns the address specified by _addr_ to the socket referred to by the file descriptor _sockfd_ . _addrlen_ specifies the size, in bytes, of the address structure pointed to by _addr_. Traditionally, this operation is called “assigning a name to a socket”. It is normally necessary to assign a local address using bind() before a SOCK_STREAM socket may receive connections.

The actual structure passed for the _addr_ argument will depend on the address family.  The sockaddr structure is defined as something like:

struct sockaddr {

sa_family_t  sa_family;

char        sa_data[14];

}

The only purpose of this structure is to cast the structure pointer passed in addr in order to avoid compiler warnings.
 
__RETURN VALUE__ :  On success, zero is returned.  On error, -1 is returned, and errno is set to indicate the error.
 
 So we will have in the server code:

```

// Bind the socket to the address. Note that we're binding the server socket
// to a well-known address so that clients know where to connect.
  if (bind(sfd, (struct sockaddr *) &addr, sizeof(struct sockaddr_un)) == -1) {
    errExit("bind");
  }
 
 
```
________________________________________________________________________________________________________________________________________________________________________________

The server calls the __listen()__ system call to mark the socket as passive, i.e. as a socket that will accept incoming connection requests.

#include <sys/socket.h>

int listen ( int _sockfd_ , int _backlog_ );

listen() marks the socket referred to by _sockfd_ as a passive socket, that is, as a socket that will be used to accept incoming connection requests using __accept()__.

The _sockfd_ argument is a file descriptor that refers to a socket of _type_ SOCK_STREAM or SOCK_SEQPACKET.

The _backlog_ argument defines the maximum length to which the queue of pending connections for sockfd may grow.  If a connection request arrives when the queue is full, the client may receive an error with an indication of ECONNREFUSED or, if the underlying protocol supports retransmission, the request may be ignored so that a later reattempt at connection succeeds.


__RETURN VALUE__ : On success, zero is returned.  On error, -1 is returned, and errno is set to indicate the error.

So we will have in the server code:
 
```

// The listen call marks the socket as *passive*. The socket will subsequently
// be used to accept connections from *active* sockets.
// listen cannot be called on a connected socket (a socket on which a connect()
// has been succesfully performed or a socket returned by a call to accept()).
  if (listen(sfd, BACKLOG) == -1) {
    errExit("listen");
  }
  
  ```
________________________________________________________________________________________________________________________________________________________________________________

The server calls the __accept()__ system call to accept an incoming connection. This call blocks until a connection request arrives. Note that this function will error out if listen() is not called beforehand. Importantly, this call creates a new socket that is connected to the peer socket and returns a file descriptor associated with it. So, if you want to communicate with the peer socket, you should use the file descriptor returned from accept(), not the file descriptor returned from the call to socket() . The latter socket remains open and is used to accept further connection requests.

#include <sys/socket.h>

int accept (int _sockfd_ , struct sockaddr *restrict _addr_ , socklen_t *restrict _addrlen_ );

The accept() system call is used with connection-based socket types __(SOCK_STREAM, SOCK_SEQPACKET)__ .  It extracts the first connection request on the queue of pending connections for the listening socket, _sockfd_ , creates a new connected socket, and returns a new file descriptor referring to that socket.  The  newly created socket is not in the listening state.  The original  socket sockfd is unaffected by this call.

The argument _sockfd_ is a socket that has been created with __socket()__ , bound to a local address with __bind()__ , and is listening for connections after a __listen()__.

The argument _addr_ is a pointer to a sockaddr structure.  This structure is filled in with the address of the peer socket, as known to the communications layer.  The exact format of the address returned addr is determined by the socket's address family. When addr is NULL, nothing is filled in; in this case, addrlen is not used, and should also be NULL.
 
The _addrlen_ argument is a value-result argument: the caller must initialize it to contain the size (in bytes) of the structure pointed to by addr; on return it will contain the actual size of the peer address.

The returned address is truncated if the buffer provided is too small; in this case, addrlen will return a value greater than was supplied to the call.

__RETURN VALUE__ :  On success, these system calls return a file descriptor for the  accepted socket (a nonnegative integer).  On error, -1 is returned, errno is set to  indicate the error, and addrlen is left unchanged.

So we will have in the server code:

```

// Accept a connection. The connection is returned on a NEW
// socket, 'cfd'; the listening socket ('sfd') remains open
// and can be used to accept further connections. */

    printf("Waiting to accept a connection...\n");
    
// NOTE: blocks until a connection request arrives.
    int cfd = accept(sfd, NULL, NULL);
    printf("Accepted socket fd = %d\n", cfd);

    
```
________________________________________________________________________________________________________________________________________________________________________________

After this, the __read()__ and __write()__ system calls can be used to communicate with the peer socket (i.e. to communicate with the client).

## for read():

#include <unistd.h>

ssize_t read (int _fd_ , void _*buf_ , size_t _count_) ;

__read()__ attempts to read up to count bytes from file descriptor _fd_  into the buffer starting at _buf_.

On files that support seeking, the read operation commences at the file offset, and the file offset is incremented by the number of bytes read.  If the file offset is at or past the end of file, no bytes are read, and read() returns zero.

If count is zero, read() may detect the errors . In the absence of any errors, or if read() does not check for  errors, a read() with a count of 0 returns zero and has no other effects.

__RETURN VALUE__:  On success, the number of bytes read is returned (zero indicates end of file), and the file position is advanced by this number. It is not an error if this number is smaller than the number of bytes requested; this may happen for example because fewer bytes are actually available right now (maybe because we were close to end-of-file, or because we are reading from a pipe, or from a terminal), or because read() was interrupted by a signal.

On error, -1 is returned, and errno is set to indicate the error. In this case, it is left unspecified whether the file position (if any) changes.


## for write():


#include <unistd.h>

ssize_t write (int _fd_ , const void _*buf_ , size_t  _count_);

 __write()__ writes up to _count_ bytes from the buffer starting at _buf_ to the file referred to by the file descriptor _fd_.

The number of bytes written may be less than count if, for example, there is insufficient space on the underlying physical  medium, or the RLIMIT_FSIZE resource limit is encountered , or the call was interrupted by a signal handler after having written less than count bytes.

__RETURN VALUE__ : On success, the number of bytes written is returned.  On error, -1 is returned, and errno is set to indicate the error.

Note that a successful write() may transfer fewer than count bytes.  Such partial writes can occur for various reasons; for example, because there was insufficient space on the disk device to write all of the requested bytes, or because a blocked write() to a socket, pipe, or similar was interrupted by a signal handler after it had transferred some, but before it had transferred all of the requested bytes.  In the event of a partial write, the caller can make another write() call to transfer the remaining  bytes.  The subsequent call will either transfer further bytes or  may result in an error (e.g., if the disk is now full).

So we will have in the server code:

```
//
// Transfer data from connected socket to stdout until EOF */
//

// Read at most BUF_SIZE bytes from the socket into buf.
    while ((numRead = read(cfd, buf, BUF_SIZE)) > 0) {
 // Then, write those bytes from buf into STDOUT.
      if (write(STDOUT_FILENO, buf, numRead) != numRead) {
        fatal("partial/failed write");
      }
    }
    
     if (numRead == -1) {
      errExit("read");
    }

```

________________________________________________________________________________________________________________________________________________________________________________

Finally, when the server is done with the socket, it should call close().

```
if (close(cfd) == -1) {
      errMsg("close");
    }
    
```
________________________________________________________________________________________________________________________________________________________________________________

# Client Process (AKA the client)
This process connects its socket to a passive socket, after which it is free to communicate with the peer socket. Again, note that these two sockets — the passive socket and the peer socket — are different. The former is the one created by calling socket(), the latter is the one returned by calling accept() in the server code.



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
### In the following, we will explain the different parts of the code provided for the client.
__________________________________________________________________________________________________________________________________________________________________

This is the same as server code,  the process creates a new socket using the socket() system call, which returns a file descriptor that can be used to refer to the socket in future system calls. Note that by default, a socket created using socket() is marked as active, and can be used in a connect() call to connect to a passive socket.

So, similar to server code, we will have

```
// Create a new client socket with domain: AF_UNIX, type: SOCK_STREAM, protocol: 0
    int sfd = socket(AF_UNIX, SOCK_STREAM, 0);
    printf("Client socket fd = %d\n", sfd);

// Make sure socket's file descriptor is legit.
    if (sfd == -1) {
      errExit("socket");
    }
  
  ``` 
__________________________________________________________________________________________________________________________________________________________________

```
// Construct server address, and make the connection.
    
    memset(&addr, 0, sizeof(struct sockaddr_un));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);
```
__________________________________________________________________________________________________________________________________________________________________

The client calls the connect() system call, which connects to a passive socket. Remember that the server bound its socket to a well-known address — this is the address that should be used for connect(). Note that connect() should be called after listen() is called on the server socket, otherwise it will error. However, it can be called before accept().

#include <sys/socket.h>

int connect (int _sockfd_ , const struct sockaddr _*addr_ , socklen_t _addrlen_);

The connect() system call connects the socket referred to by the file descriptor _sockfd_ to the address specified by _addr_ .  The addrlen argument specifies the size of _addr_ . The format of the address in _addr_ is determined by the address space of the socket _sockfd_ ;

If the socket _sockfd_ is of type SOCK_DGRAM, then _addr_ is the address to which datagrams are sent by default, and the only address from which datagrams are received.  If the socket is of type SOCK_STREAM or SOCK_SEQPACKET, this call attempts to make a connection to the socket that is bound to the address specified by _addr_.

__RETURN VALUE__: If the connection or binding succeeds, zero is returned. On error, -1 is returned, and errno is set to indicate the error.

So we will have in the client code:


```

 // Connects the active socket referred to be sfd to the listening socket
 // whose address is specified by addr.
    if (connect(sfd, (struct sockaddr *) &addr,
                sizeof(struct sockaddr_un)) == -1) {
      errExit("connect");
    }

```

__________________________________________________________________________________________________________________________________________________________________

Similar to the descriptions for __read()__ and __write()__ in the server code, we have :


```
//
// Copy stdin to socket.
// 

// Read at most BUF_SIZE bytes from STDIN into buf.
    while ((numRead = read(STDIN_FILENO, buf, BUF_SIZE)) > 0) {
// Then, write those bytes from buf into the socket.
      if (write(sfd, buf, numRead) != numRead) {
        fatal("partial/failed write");
      }
    }

    if (numRead == -1) {
      errExit("read");
    }
```

_______________________________________________________________________________________________________________________________________________________________________________
# Running the Codes

First, run the server. The server will create a passive socket and start listening for incoming connection requests.

![Image of Yaktocat](/home/zeinab/Pictures/Screenshot from 2021-06-15 18-44-52.png)


Then, run the client — this will connect to the passive socket. After that, you can type input into the client’s terminal. When you press enter, it should be echoed on the server’s terminal window.






________________________________________________________________________________________________________________________________________________________________________________


## Conclusion

__•__ The first, unix_server_socket.c, creates a passive socket and accepts connections. For each connection, it will read from the socket and echo the results into STDOUT.

__•__ The second, unix_client_socket.c, creates an active socket and connects to the passive socket created by unix_server_socket.c. After connecting, it reads from STDIN and writes to the socket.

![Image of Yaktocat](https://miro.medium.com/max/700/1*wKeN12uTZiYT1LVwvr5xmg.png)

