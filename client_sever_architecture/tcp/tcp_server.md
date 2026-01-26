**Note: The lines of code in this documentation is not mine, it has been reproduced following the book "Hands On Network Programming in C" by Lewis Van Wrikle which you can fork/copy from [https://github.com/codeplea/Hands-On-Network-Programming-with-C](Hand_On_Network_Programming_with_C).Here would be a simple explanation, on what I learned following this book. Enjoy :)**

In the previous blog series, we answered to what tcp was, and how transmission controll protocol was used to transport data, from one point of the communication to another point over the internet. As to before we used a remote server to exchange data between nodes, while here we aim at using a local one.


## Getting started: The concept of a server. 

The concept of a server, like any other computer  said simply is nothing fancy, it is in the end, `just a pile of sand that was somehow made to give us physical detachment from reality`.

`A server is a computer that provides information to other computers called "clients" on a computer network`- Wikipedia. 
The server is an actor of what is known as the client-server architecture. In figure 1.1 is a breifly description of how conceptualy we can thinkof the client and server relationship. 


<img src='../images/drwaing1'>
*Note: This is my own drawing, however you are free to use it to your likings*

From one part of the schema, you have my computer (FLokshi's pooter) which in this scenario is the client (my computer is not actually the client,it is hardware part, what is the real client, if for example your web browser,say Chrome, what I use, I would use firefox, but cant because my memory crashes, once a tab is opened, but pro tip use it in the winter, the amount of ads it has is going to warm you continuesly)  and from the other side you have the server. 
The idea is quite simple, I open my web client (Chrome) and write the following url to the search bar:https://flokshi.github.io/flokshi and type enter. What just happend is that my client, send the server a request, for the following resource (flokshi) from the host (flokshi.github.io)

To demostrate it here we made a request to the site from a chrome browser, doesn't really matter, and intercept the traffic using burp suit.
The whole request header is display in figure 2. 

<img src='../images/burp_suit_request.jpg'> 
*Note: Screenshot taken from the owner of the blog*

The request made was a GET request, meaning that we are telling the server that we want to get the following resource form the host. What the server give to us is the respond, and the respond (res for shortly) can be part of a group of categories (called response status code). Have fun exploring them: [respons status code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)

If the respond is of the 200 code class, meaning that everything went well, with our request, and in response the server send the file we are looking for.  


## History of Servers 

Blank, due to not enough information gathered. 



## Getting our hands dirty 

I will start with the following image. 

<img src='../images/monotholic_to_microservices'>
*Note: The image is not mine. It was reproduced from the user @PR0GRAMMERHUM0R on X. ALL the rights reserved to the original author*

So yeah, as the picture suggest the nerds on the internet replaced a well fine designe architecture as monolithic (such as the Linux Kernel), with microservices. 
(Note: On our school paper, me and my partner, we did explain how software reeengineering was done through microservices. Check it out if you areinterested:[software-reengineering-paper](../../software-reengineering/software-reengineering.doc) )
But yeah, since 2014, and because of the massive adopting from companies such as Google, Amazone etc., microservice have become quite popular. The idea of microservices is that large programming problems can be split up into many small subsystems that communicate over a network. 
How does this help us. Well the example that is taken in the book, emphasis the problem of creating a server which we are going to use to transform our string to upper cases, something like ` aaa -> AAA` and this is a quite simple approach, but good to understand it.

Another  approach is to code this service inside our program, but coding and generally programming i hard.Alternatively, you could keep your program simple and instead connect to a service which isgoing to format the string for you. If the client connects to our server and send `hello`-> the server will respond to it back with `HELLO`  all capitalised. 
This will serve a very basic microservice. For our microservice to be useful, it does need to handle many simultaneous incomming connections. Forthis reason, as the author mentioned in the book, we are going to again be using selec() to handle those connections. 

The flow of how our program is going to work is shown in the picture below. 

<img src="../images/server_flow.jpg">

*Note: The image is being reproduced from the book Hands-on-network-programming-with-C by Lewis Van Wrikle. All right reserved to the author*

The image above is pretty much as standard as the one that we used to create the tcp client. We start by firstly getting a local address, then creating a socket, than binding the socket to the local address, then listening for incoming connections, and then using selectto handle the incomming connections, made by tcp clients, rather than using accept() immediatly, we first select the connection,and only when it is a new connection we accept it by using accept. 

The code  for the server is quite straightforward, starting by including "chap03.h" header file (or whatver is it if you have rename it), and also including the <ctype.h> header file. Based on google ctype.h (or <cctype> in C++) is a C standard library header providing functions to classify and transform single characters, such as checking if a character is a digit, letter, whitespace, or converting itbetween uppercase and lowercase, improving code readability and portability by abstracting ASCII value checks. Key functions include isalnum(), isalpha(), isdigit(), islower(), isupper(), isspace(), tolower(), and toupper(). The library that we are going to use to provide our microservice.

```
#include "chap03.h"
#include <ctype.h>

```
We handle the creation of the widndow socket (if we operate on window) and then we configure our remote address. (Pretty standard stuff, the same as the one that where used when we made the tcp client. Our specification for the connection,is that of IPv4 (note IPv6, we stated by setting the hints.ai_family = AF_INET, read man for more information), we want the type of our socket to be of tcp socket,andwe have also set, hints.ai_flags= AI_PASSIVE, which according to man: `If the AI_PASSIVE flag is specified in hints.ai_flags, and nodeis NULL, then returned socket address will be suitable for binding a socket  that will accept connections. In short words the socket that is going to be returned to us, will be able to use bind and accept.`

When we were working with tcp_client, we didn't really set anything regarding the flags (we only set the connection type is that of SOCK_STREAM), and according to man : `If the AI_PASSIVE flag is notset in hinst.ai_flags, then the returned socked addresses will be suitable for use with connect(), sendto(), or sendmsg().` And indeed, what we wanted when we worked with the tcp_client was a remote connection, which works with connect(), while the local connection, that we are doing with tcp_server works with bind() (*To be honest, while i was reading regarding the tcp_client, i couldn't really pitpoint why weren't were we using bind, but then the book and google explained it*

After configuring the local address, we create our socket, pretty standard. Then in the same way that we used connect, we also use bind, to bind the socket to a local address. Based on man: `WHen a socket is created with socket(), it exists in a name space(address family), but has no address assigned to it. bind() assigns the address specified by addr to the socket referred to by the file descriptor socfd (or in our case, socket_listen). Traditinally, this operation is called "assigning a name to a socket".` 

Bind() is part of the standard c library (libc) and the function signature is : **int bind(int sockfd, const struct sockaddr* addr, socklen_t addrlen)** On success, zero is returned. On error, -1 is return and errno is set to indicate the error. 

Only when we have successfully have bind our socket to a local address, we move one to listening for connections.All the connection's file descriptors are going to be set inside a fd_set named master, in which after we zero it, are going to put all of our active soket. .We also, keep a max_socket, which is firstly going to be set to our socket file descriptor.

We then print a status message `printf("Waiting for connections...\n"`, enter the main loop and then wait for connections.( The loop obviously is going to be looping forever, until either of the peers close the connection or Ctrl+C.
We copy the master content inside another fd_set called reads. Since we are using select, as also explained in the book, if we use it direclty into the master we are going to modify its values, that is why we first copy the content of the master into reads.   

We now loop through each possible socket and see wether it was flagged by select ad being ready. If the socket, x was flagged by selectthan `FD_ISSET(x, &read)` is ture (this isset reminds me the function in php, pretty nice).

*Something to remember as mentioned in the book is that, FD_ISSET is only true ofr sockets that are to be read* In the case of socket_listen, this means that a new connection is ready to be established with accept. For all other socket, it means that data is ready to be read with recv(). 

```
while(1) {
        fd_set reads;
        reads = master;
        if (select(max_socket+1, &reads, 0, 0, 0) < 0) {
            fprintf(stderr, "select() failed. (%d)\n", GETSOCKETERRNO());
            return 1;
        }

        SOCKET i;
        for(i = 1; i <= max_socket; ++i) {
            if (FD_ISSET(i, &reads)) {

                if (i == socket_listen) {
                    struct sockaddr_storage client_address;
                    socklen_t client_len = sizeof(client_address);
                    SOCKET socket_client = accept(socket_listen,
                            (struct sockaddr*) &client_address,
                            &client_len);
                    if (!ISVALIDSOCKET(socket_client)) {
                        fprintf(stderr, "accept() failed. (%d)\n",
                                GETSOCKETERRNO());
                        return 1;
                    }

                    FD_SET(socket_client, &master);
                    if (socket_client > max_socket)
                        max_socket = socket_client;

                    char address_buffer[100];
                    getnameinfo((struct sockaddr*)&client_address,
                            client_len,
                            address_buffer, sizeof(address_buffer), 0, 0,
                            NI_NUMERICHOST);
                    printf("New connection from %s\n", address_buffer);

                } else {
                    char read[1024];
                    int bytes_received = recv(i, read, 1024, 0);
                    if (bytes_received < 1) {
                        FD_CLR(i, &master);
                        CLOSESOCKET(i);
                        continue;
                    }

                    int j;
                    for (j = 0; j < bytes_received; ++j)
                        read[j] = toupper(read[j]);
                    send(i, read, bytes_received, 0);
                }

            } //if FD_ISSET
        } //for i to max_socket
    } //while(1)
```
If the socket is socket_listen, then we accpet() the connection. If not then it is instead a request for an established connection. In this case we need to read it with recv(), convert it into upper case using the built-in toupper from the ctype.h header file and send the data back. 

```
} else {
                    char read[1024];
                    int bytes_received = recv(i, read, 1024, 0);
                    if (bytes_received < 1) {
                        FD_CLR(i, &master);
                        CLOSESOCKET(i);
                        continue;
                    }

                    int j;
                    for (j = 0; j < bytes_received; ++j)
                        read[j] = toupper(read[j]);
                    send(i, read, bytes_received, 0);
                }

```
If the client has diconnected, then recv() returns a non-positive number. In this case, we remove that socket from the master socket set, and we also call CLOSESOCKET() on it to clean it up. 
