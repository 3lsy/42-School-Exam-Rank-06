# IRC Server Implementation README

This README provides an overview of the key sections of an IRC server implementation. The server is written in C and uses socket programming to handle multiple clients, allowing them to send and receive messages.

## Table of Contents

1. [Overview](#overview)
2. [Initialization](#initialization)
3. [Error Handling](#error-handling)
4. [Sending Messages to All Clients](#sending-messages-to-all-clients)
5. [Main Function](#main-function)
    - [Argument Checking](#argument-checking)
    - [Socket Setup](#socket-setup)
    - [Server Loop](#server-loop)
        - [Handling New Connections](#handling-new-connections)
        - [Handling Client Messages](#handling-client-messages)

## Overview

This IRC server allows multiple clients to connect, send messages, and receive messages from others. It uses the `select` system call to manage multiple connections concurrently. The implementation includes the following key sections:

- Initialization of server and client structures
- Error handling function
- Function to send messages to all connected clients
- Main function handling the server's core logic

## Initialization

```c
typedef struct s_client {
    int     id;
    char    msg[100000];
}   t_client;

t_client    clients[1024];
fd_set      read_set, write_set, current;
int         maxfd = 0, gid = 0;
char        send_buffer[120000], recv_buffer[120000];
```

- **t_client**: Structure to store client information (ID and message buffer).
- **clients[]**: Array to hold all connected clients.
- **fd_set variables**: File descriptor sets to track sockets ready for reading and writing.
- **maxfd**: Maximum file descriptor value.
- **gid**: Global client ID counter.
- **send_buffer and recv_buffer**: Buffers for sending and receiving messages.

## Error Handling

```c
void err(char *msg) {
    if (msg)
        write(2, msg, strlen(msg));
    else
        write(2, "Fatal error", 11);
    write(2, "\n", 1);
    exit(1);
}
```

- **err**: Function to print an error message and exit the program.

## Sending Messages to All Clients

```c
void send_to_all(int except) {
    for (int fd = 0; fd <= maxfd; fd++) {
        if (FD_ISSET(fd, &write_set) && fd != except)
            if (send(fd, send_buffer, strlen(send_buffer), 0) == -1)
                err(NULL);
    }
}
```

- **send_to_all**: Function to send the contents of `send_buffer` to all clients except the specified one.

## Main Function

### Argument Checking

```c
if (ac != 2)
    err("Wrong number of arguments");
```

- **Argument Check**: Ensures the server is started with the correct number of arguments (the port number).

### Socket Setup

```c
struct sockaddr_in serveraddr;
socklen_t len;
int serverfd = socket(AF_INET, SOCK_STREAM, 0);
if (serverfd == -1) err(NULL);
maxfd = serverfd;

FD_ZERO(&current);
FD_SET(serverfd, &current);
bzero(clients, sizeof(clients));
bzero(&serveraddr, sizeof(serveraddr));

serveraddr.sin_family = AF_INET;
serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
serveraddr.sin_port = htons(atoi(av[1]));

if (bind(serverfd, (const struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1 || listen(serverfd, 100) == -1)
    err(NULL);
```

- **Socket Creation**: Sets up the server socket.
- **File Descriptor Initialization**: Initializes the file descriptor sets and clears the client array.
- **Server Address Setup**: Configures the server address structure.
- **Bind and Listen**: Binds the server to the specified port and listens for incoming connections.

### Server Loop

#### Handling New Connections

```c
if (FD_ISSET(fd, &read_set)) {
    if (fd == serverfd) {
        int clientfd = accept(serverfd, (struct sockaddr *)&serveraddr, &len);
        if (clientfd == -1) continue;
        if (clientfd > maxfd) maxfd = clientfd;
        clients[clientfd].id = gid++;
        FD_SET(clientfd, &current);
        sprintf(send_buffer, "server: client %d just arrived\n", clients[clientfd].id);
        send_to_all(clientfd);
    }
```

- **New Connection Handling**: Accepts new client connections and initializes client data.

#### Handling Client Messages

```c
else {
    int ret = recv(fd, recv_buffer, sizeof(recv_buffer), 0);
    if (ret <= 0) {
        sprintf(send_buffer, "server: client %d just left\n", clients[fd].id);
        send_to_all(fd);
        FD_CLR(fd, &current);
        close(fd);
    } else {
        for (int i = 0, j = strlen(clients[fd].msg); i < ret; i++, j++) {
            clients[fd].msg[j] = recv_buffer[i];
            if (clients[fd].msg[j] == '\n') {
                clients[fd].msg[j] = '\0';
                sprintf(send_buffer, "client %d: %s\n", clients[fd].id, clients[fd].msg);
                send_to_all(fd);
                bzero(clients[fd].msg, strlen(clients[fd].msg));
                j = -1;
            }
        }
    }
}
```

- **Message Handling**: Receives messages from clients, processes them, and sends them to all other clients. Handles client disconnections and message formatting.

## Conclusion

This IRC server implementation handles multiple client connections, allowing them to communicate in real-time. The server initializes necessary structures, manages connections and messages, and ensures smooth operation using efficient error handling and the `select` system call for multiplexing.

---
# OLD README
---

## Expected File

mini_serv.c

## Allowed Functions

<unistd.h>
```
write, close and select
```

<sys/socket.h>
```
socket, accept, listen, send, recv and bind
```

<string.h>
```
strstr, strlen, strcpy, strcat, memset and bzero
```

<stdlib.h>
```
malloc, realloc, free, calloc, atoi and exit
```

<stdio.h>
```
- sprintf
```


## Subject Text

Write a program that will listen for client to connect on a certain port on 127.0.0.1 and will let clients to speak with each other. This program will take as first argument the port to bind to.

  - If no argument is given, it should write in stderr "Wrong number of arguments" followed by a \n and exit with status 1
  - If a System Calls returns an error before the program start accepting connection, it should write in stderr "Fatal error" followed by a \n and exit with status 1
  - If you cant allocate memory it should write in stderr "Fatal error" followed by a \n and exit with status 1

### Your Program

- Your program must be non-blocking but client can be lazy and if they don't read your message you must NOT disconnect them...
- Your program must not contains #define preproc
- Your program must only listen to 127.0.0.1

The fd that you will receive will already be set to make 'recv' or 'send' to block if select hasn't be called before calling them, but will not block otherwise. 

### When a client connect to the server:

- the client will be given an id. the first client will receive the id 0 and each new client will received the last client id + 1
- %d will be replace by this number
- a message is sent to all the client that was connected to the server: "server: client %d just arrived\n"

### Clients must be able to send messages to your program.

- message will only be printable characters, no need to check
- a single message can contains multiple \n
- when the server receive a message, it must resend it to all the other client with "client %d: " before every line!

### When a client disconnect from the server:

- a message is sent to all the client that was connected to the server: "server: client %d just left\n"

```
Memory or fd leaks are forbidden
```

## Exam Practice Tool
Practice the exam just like you would in the real exam using this tool - https://github.com/JCluzet/42_EXAM
    }
}
