# Getting Started With Unix Domain Sockets


Sockets provide a means of communication between processes, i.e. a way for them to exchange data. The way it usually works is that __process_a__ has __socket_x__ , __process_b__ has __socket_y__ , and the two sockets are connected. Each process can then use its socket to receive data from the other process and/or send data to the other process. One way to think about sockets is that they open up a communication channel where both sides can read and write.


![Image of Yaktocat](https://miro.medium.com/max/10240/1*3Ny5SBf14TwhpRR5vOqF2A.png)

Why do we need sockets? Well, because one process can’t normally talk to another process; this is true when the processes are on the same computer or on different computers. On a related note, there are two main domains of sockets: __Unix domain sockets__ , which allow processes on the same computer to communicate (IPC), and __Internet domain sockets__ , which allow processes to communicate over a network. If that’s confusing, just think of “Unix domain” and “Internet domain” as adjectives that describe the communication range of a socket.
