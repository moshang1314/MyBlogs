# 1 高级I/O函数

## 1.1 pipe函数

> ​	pipe函数可用于创建一个管道，以实现进程间通信。pipe函数的定义如下：
>
> ```c
> #include <unistd.h>
> int pipe(int fd[2]);
> ```
>
> ​	pipe函数的参数是一个包含两个int型整数的数组指针。该函数成功时返回0，并将一对打开的文件描述符值填入其参数指向的数组。如果失败，则返回-1并设置errno。
>
> ​	通过pipe函数创建的这两个文件描述符fd[0]和fd[1]分别构成了管道的两端，往fd[1]写入的数据可以从fd[0]读出。并且，fd[0]只能用于从管道读出数据，fd[1]则只能用于往管道写入数据，而不能反过来使用。如果要实现双向的数据传输，就应该使用两个管道。**默认情况下，这一对文件描述符都是阻塞的。**此时如果我们用read系统调用来读取一个空的管道，则read将阻塞，直到管道内有数据可读；如果我们用write系统调用来往一个满的管道中写入数据，则write将被阻塞，直到管道有足够多的空闲空间可用。但如果应用程序将fd[0]和fd[1]都设置为非阻塞，则read和write会有不同的行为。如果管道的写端文件描述符fd[1]的引用计数减少至0，即没有任何进程需要往管道中写数据，**则针对该管道的读端文件描述符fd[0]的read操作将返回0，即读取到了文件结束标记（End Of File，EOF）；**反之，如果管道的读端文件描述符fd[0]的引用计数减少至0，即没有任何进程需要从管道读取数据，则针对该管道的写端文件描述符fd[1]的write操作将失败，**并引发SIGPIPE信号。**
>
> ​	管道内部传输的数据是字节流，这和TCP字节流的概念相同。但二者又有细微的区别。应用程序能往一个TCP连接中写入多少字节的数据，**取决于对方的接收通告窗口的大小和本端的拥塞窗口的大小**。而管道本身拥有一个容量限制，它规定如果应用程序不将数据从管道读走的话，该管道最多能被写入多少字节的数据。自Linux 2.6.11内核起，管道容量的大小默认是65536字节。我们可以使用fcntl函数来修改管道的容量（见后文）。
>
> ​	此外，socket的基础API中有一个socketpair函数。它能够方便地创建双向管道。其定义如下：
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> int socketpair(int domain, int type, int protocol, int fd[2]);
> ```
>
> ​	socketpair前三个参数的含义与socket系统调用的三个参数完全相同，但domain只能使用UNIX本地域协议族AF_UNIX，因为我们仅能在本地使用这个双向管道。最后一个参数则和pipe系统调用的参数一样，只不过socketpair创建的这对文件描述符都是既可读又可写的。socketpair成功时返回0，失败时返回-1并设置errno。

## 1.2  dup函数和dup2函数

> ​	有时我们希望把标准输入重定向到一个文件，或者把标准输出重定向到一个网络连接（比如CGI编程）。这可以通过下面的用于复制文件描述符的dup或dup2函数来实现：
>
> ```c
> #include <unistd.h>
> int dup(int file_descriptor);
> int dup2(int file_descriptor_one, int file_descriptor_two);	
> ```
>
> ​	dup函数创建一个新的文件描述符，该新文件描述符和原有文件描述符file_descriptor指向相同的文件、管道或者网络连接。并且dup返回的文件描述符总是取进程当前可用的最小整数值。dup2和dup类似，不过它将返回第一个不小于file_discriptor_two的整数值。dup和dup2系统调用失败时返回-1并设置errno。注意：通过dup和dup2等创建的文件描述符并不继承原文件描述符的close-on-exec属性。
>
> 以下代码利用dup函数实现了一个基本的CGI服务器。
>
> ```c
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <stdio.h>
> #include <stdlib.h>
> #include <errno.h>
> #include <string.h>
> 
> int main(int argc, char* argv[])
> {
> 	if(argc <= 2)
>     {
>         printf("usage: %s ip_adress port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
>     
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
>     
>     int sock = socket(PF_INET, SOCK_STREAM, 0);
>     assert(sock > 0);
>     
>     int ret = bind(sock, (struct sockaddr*)&address, sizof(address));
>     assert(ret != -1);
>     
>     struct sockaddr_in client;
>     socklen_t client_addrlength = sizeof(client);
>     int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlenght);
>     if(connfd < 0)
>     {
>         printf("errno is: %d\n", errno);
>     }
>     else
>     {
>         close(STDOUT_FILENO);
>         dup(connfd);
>         printf("abcd\n");
>         close(connfd);
>     }
>     
>     close(sock);
> 	return 0;
> }
> ```
>
> ​	在代码中，我们先关闭标准输出文件描述符STDOUST_FILENO（其值是1），然后复制socket文件描述符connfd。因为dup总是返回系统中最小的可用文件描述符，所以它的返回值实际上是1，即之前关闭的标准输出文件描述符的值。这样一来，服务器输出到标准输出的内容（这里是“abc”）就会直接发送到与客户端连接对应的socket上，**因此printf调用的输出将被客户端获得（而不是显示在服务器程序的终端上）**。这就是CGI服务器的基本工作原理。

## 1.3 readv函数和writev函数

> ​	readv函数将数据从文件描述符读到分散的内存块中，即分散读；writev函数则将多块分散的内存数据一并写入文件描述符中，即集中写。它们的定义如下：
>
> ```c
> #include <sys/uio.h>
> ssize_t readv(int fd, const struct iovec* vector, int count);
> ssize_t writev(int fd, const struct iovec* vector, int count);
> ```
>
> ​	fd参数是被操作的目标文件描述符。vector参数的类型是iovec结构数组。该结构类型（iovec）在网络编程数据读写章节讨论过，该结构体描述了一块内存区。count参数是vector数组的长度，即有多少块内存数据需要从fd读出或写到fd。readv和writev在成功时返回读出/写入fd的字节数，失败则返回-1并设置errno。**它们相当于简化版的recvmsg和sendmsg函数。**
>
> ​	考虑之前讨论过的Web服务器。当Web服务器解析完一个HTTP请求之后，如果目标文档存在且客户具有读取该文档的权限，那么它就需要发送一个HTTP应答来传输该文档。这个HTTP应答应包含1个状态行、多个头部字段、1个空行和文档的内容。其中，前3部分的内容可能被Web服务器放置在一块内存中，而文档的内容则通常被读入到另外一块单独的内存中（通过read函数或mmap函数）。我们并不需要把这两部分内容拼接到一起再发送，而是可以使用writev函数将它们同时写出，如以下代码所示：
>
> ```c
> #include <sys/socket.h>
> #include <netinet.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <stdlib.h>
> #include <errno.h>
> #include <string.h>
> #include <sys/stat.h>
> #include <sys/types.h>
> #include <fcntl.h>
> 
> #define BUFFer_SIZE 1024
> /* 定义两种HTTP状态和状态信息 */
> static const char* status_line[2] = {"200 OK", "500 Internal server error"};
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 3)
>     {
>         printf("usage: %s ip_address port_number filename\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
>     
>     /* 将目标文件作为程序的第三个参数传入 */
>     const char* file_name = argv[3];
>     
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
>     
>     int sock = socket(PF_INET, SOCK_STREAM, 0);
>     assert(sock >= 0);
>     
>     int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
>     
>     ret = listen(sock, 5);
>     assert(ret != -1);
>     
>     struct sockaddr_in client;
>     socklen_t client_addrlength = sizeof(client);
>     int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
>     if(confd < 0)
>     {
>         printf("errno is: %d\n", errno);
>     }
>     else
>     {
>         /* 用于保存HTTP应答的状态行、头部字段和一个空行的缓存区 */
>         char header_buf[BUFFER_SIZE];
>         memset(header_buf, '\0', BUFFER_SIZE);
>         /* 用于存放目标文件内容的应用程序缓存 */
>         char* file_buf;
>         /* 用于获取目标文件的属性，比如是否为目录，文件大小等 */
>         struct stat file_stat;
>         /* 记录目标文件是否是有效文件 */
>         bool valid = true;
>         /* 缓存区header_buf目前已经使用了多少字节的空间 */
>         int len = 0;
>         if(stat(file_name, &file_stat) < 0) /* 目标文件不存在 */
>         {
>             valid = false;
>         }
>         else
>         {
>             if(S_ISDIR(file_stat.st_mode))	/* 目标文件是一个目录 */
>             {
>                 valid = false;
>             }
>             else if(file_stat.st_mode & S_IROTH) /* 其它用户具有读取目标文件的权限 */
>             {
>                 /* 动态分配缓存区file_buf，并指定其大小为目标文件的大小，file_stat.st_size加1，然后将目标文件读入缓存区file_buf中 */
>                 int fd = open(file_name, O_RDONLY);
>                 file_buf = new char[file_stat.st_size + 1];
>                 memset(file_buf, '\0', file_stat.st_size + 1);
>                 if(read(fd, file_buf, file_stat.st_size) < 0)
>                 {
>                     valid = false;
>                 }
>             }
>             else
>             {
>                 valid = false;
>             }
>             /* 如果目标文件有效，则发送正常的HTTP应答 */
>             if(valid)
>             {
>                 /* 下面这部分内容将HTTP应答的状态行、"Content-Length"头部字段和一个空行依次加入header_buf中 */
>                 ret = snprintf(header_buf, BUFFER_SIZE-1, "%s %s\r\n", "HTTP/1.1", status_line[0]);
>                 assert(ret != -1);
>                 len += ret;
>                 ret = snprintf(header_buf + len, BUFFER_SIZE-1-len, "%s", "\r\n");
>                 /* 利用writev将header_buf和file_buf的内容一并写出 */
>                 struct iovec iv[2];
>                 iv[0].iov_base = header_buf;
>                 iv[0].iov_len = strlen(header_buf);
>                 iv[1].iov_base = file_buf;
>                 iv[1].iov_len = file_stat.st_size;
>                 ret = writev(connfd, iv, 2);
>             }
>             else	/* 如果目标文件无效，则通知客户端服务器发生了“内部错误” */
>             {
>                 ret = snprintf(header_buf, BUFFER_SIZE-1, "%s %s\r\n", "HTTP/1.1", status_line[1]);
>                 len += ret;
>                 ret = snprintf(header_buf+len, BUFFER_SIZE-1-len, "%s", "\r\n");
>                 send(connfd, header_buf, strlen(header_buf), 0);
>             }
>             close(connfd);
>             delete [] file_buf;
>         }
>         
>         close(sock);
>         return 0;
>     }
> }
> ```
>
> ​	以上代码中，我们省略了HTTP请求的接收及解析，因为现在重点是HTTP应答的发送。我们直接将目标文件作为第三个参数传递给服务器程序，客户telnet到该服务器上即可获得该文件。

## 1.4 sendfile函数

> ​	sendfile函数在两个文件描述符之间直接传递数据（完全在内核中操作），从而避免了内核缓冲区和用户缓冲区之间的数据拷贝，效率很高，这被称为零拷贝。sendfile函数定义如下：
>
> ```c
> ssize_t sendfile(int out_fd, int in_fd, off_t* offset, size_t count);
> ```
>
> ​	in_fd参数是待读出内容的文件描述符，out_fd参数是待写入内容的文件描述符。offset参数指定从读入文件流的哪个位置开始读，如果为空，则使用读入文件流默认的起始位置。count参数指定在文件描述符in_fd和out_fd之间传输的字节数。sendfile成功时返回传输的字节数，失败则返回-1并设置errno。该函数的man手册明确指出，**in_fd必须是一个支持类似mmap函数的文件描述符，即它必须指向真实的文件，不能是socket和管道；而out_fd则必须是一个socket。**由此可见，sendfile几乎是专门为在网络上传输文件而设计的。
>
> 下面的代码利用sendfile函数将服务器上的一个文件传送到客户端。
>
> ```c
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <stdlib.h>
> #include <errno.h>
> #include <string.h>
> #include <sys/types.h>
> #include <sys/stat.h>
> #include <sys/sendfile.h>
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 3 )
>     {
>         printf("usage: %s ip_address port_number filename\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
>     const char* file_name = argv[3];
>     
>     int filefd = open(file_name, O_RDONLY);
>     assert(filefd > 0);
>     struct stat stat_buf;
>     fstat(filefd, &stat_buf);
>     
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = PF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
>     
>     int sock = socket(AF_INET, SOCK_STREAM, 0);
>     assert(sock > 0);
>     
>     int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
>     
>     ret = listen(sock, 5);
>     assert(ret != -1);
>     
>     struct sockaddr_in client;
>     socklen_t client_addrlength = sizeof(client);
>     int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
>     if(connfd < 0)
>     {
>         printf("errno is: %d\n", errno);
>     }
>     else
>     {
>         //发送文件
>         sendfile(connfd, filefd, NULL, stat_buf.st_size);
>         close(connfd);
>     }
>     close(sock);
>     return 0;
> }
> ```
>
> ​	代码中，我们将目标文件作为第3个参数传递给服务器程序，客户端telnet到该服务器上即可获得该文件。该代码没有为目标文件分配任何用户空间的缓存，也没有执行读取文件的操作，但同样实现了文件的发送，其效率显然要高得多。

## 1.5 mmap函数和munmap函数

> ​	mmap函数用于申请一段内存空间。我们可以将这段内存作为进程间通信的共享内存，也可以将文件直接映射到其中。munmap函数则释放由mmap函数创建的这段内存空间。它们的定义如下：
>
> ```c
> #include <sys/mman.h>
> void* mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);
> int munmap(void *start, size_t length);
> ```
>
> ​	start参数允许用户使用某个特定的地址作为这段内存的起始地址。如果它被设置成NULL，则系统自动分配一个地址。length参数指定内存段的长度。prot参数用来设置内存段的访问权限。它可以取以下几个值的按位或：
>
> ```c
> /*
> PROT_READ，内存段可读
> PROT_WRITE，内存段可写
> PROT_EXEC，内存段可执行
> PROT_NONE，内存段不能被访问
> */
> ```
>
> ​	flags参数控制内存段内容被修改后程序的行为。它可以被设置为下表中的某些值（这里仅列出了常用的值）的按位或（其中MAP_SHARED和MAP_PRIVATE是互斥的，不能同时指定）。
>
> | 常用值        | 含义                                                         |
> | ------------- | ------------------------------------------------------------ |
> | MAP_SHARED    | 在进程间共享这段内存，对该内存段的修改将反映到被映射的文件中。它提供了进程间共享内存的POSIX方法。 |
> | MAP_PRIVATE   | 内存段为调用进程所私有。对该内存段的修改不会被反映到被映射的文件中。 |
> | MAP_ANONYMOUS | 这段内存不是从文件映射而来的。其内容被初始化为全0.这种情况下，mmap函数的最后两个参数将被忽略 |
> | MAP_HUGETLB   | 按照“大内存页面”来分配内存空间。“大内存页面”的大小可通过/proc/meminfo文件来查看 |
> | MAP_FIXED     | 内存段必须位于start参数指定的地址处。start必须是内存页面大小（4096字节）的整数倍 |
>
> ​	fd参数是被映射文件对应的文件描述符。它一般通过open系统调用获得。offset参数设置从文件的何处开始映射（对于不需要读入整个文件的情况）。
>
> ​	mmap函数成功时返回指向目标内存区域的指针，失败则返回MAP_FAILED((void*)-1)并设置errno。munmap函数成功时返回0，失败则返回-1并设置errno。

## 1.6 splice函数

> ​	splice函数用于在两个文件描述符之间移动数据，也是零拷贝操作。splice函数的定义如下：
>
> ```c
> #include <fcntl.h>
> ssize_t splice(int fd_in, loff_t* off_in, int fd_out, loff_t* off_out, size_t len, unsigned int flags);
> ```
>
> ​	fd_in参数是待输入数据的文件描述符。如果fd_in是管道文件描述符，那么off_in参数必须被设置为NULL。如果fd_in不是一个管道文件描述符（比如socket），那么off_in表示从输入数据流的何处开始读取数据。此时，若off_in被设置为NULL，则表示从输入数据流的当前偏移位置读入；若off_in不为NULL，则它将指出具体的偏移位置。fd_out/off_out参数的含义与fd_in、off_in相同，不过用于输出数据流。len参数指定移动数据的长度；flags参数则控制数据如何移动，它可以被设置为下表中的某些值的按位或。
>
> | 常用值            | 含义                                                         |
> | ----------------- | ------------------------------------------------------------ |
> | SPLICE_F_MOVE     | 如果合适的话，按整页内存移动数据。这只是给内核的一个提示。不过，因为它的实现存在BUG，所以自内核2.6.21后，它实际上没有任何效果 |
> | SPLICE_F_NONBLOCK | 非阻塞的splice操作，但实际效果还会受文件描述符本身的阻塞状态的影响 |
> | SPLICE_F_MORE     | 给内核一个提示：后续的splice调用将读取更多的数据             |
> | SPLICE_F_GIFT     | 对splice没有效果                                             |
>
> ​	使用splice函数时，f**d_in和fd_out必须至少有一个是管道文件描述符。**splice函数调用成功时返回移动字节的数量。它可能返回0，表示没有数据需要移动，这发生在从管道中读取数据（fd_in是管道文件描述符）而该管道没有被写入任何数据时。splice函数失败时返回-1并设置errno。常见的errno如下表所示：
>
> | 错误   | 含义                                                         |
> | ------ | ------------------------------------------------------------ |
> | EBADF  | 参数所指文件描述符有错                                       |
> | EINVAL | 目标文件系统不支持splice，或者目标文件以追加方式打开，或者两个文件描述符都不是管道文件描述符，或者某个offset参数被用于不支持随机访问的设备（比如字符设备） |
> | ENOMEM | 内存不够                                                     |
> | ESPIPE | 参数fd_in（或fd_out）是管道文件描述符，而off_in（off_in或off_out）不为NULL |
>
> ​	下面我们使用splice函数来实现一个零拷贝的回射服务器，它将客户端发送的数据原样返回给客户端，具体实现如下：
>
> ```c
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <stdlib.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
>     
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
>     
>     int sock = socket(PF_INET, SOCK_STREAM, 0);
>     assert(sock >= 0);
>     
>     int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
>     
>     ret = listen(sock, 5);
>     assert(ret != -1);
>     
>     struct sockaddr_in client;
>     socklen_t client_addrlength = sizeof(client);
>     int connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
>     if(connfd < 0)
>     {
>         printf("errno is: %d\n", errno);
>     }
>     else
>     {
>         int pipefd[2];
>         ret = pipe(pipefd);	/* 创建管道 */
>         assert(ret != -1);
>         /* 将connfd上流入的客户端数据定向到管道中 */
>         ret = splice(connfd, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>         assert(ret != -1);
>         /* 将管道的输出定向到connfd客户连接描述符 */
>         ret = splice(pipefd[0], NULL, connfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>         assert( ret != -1);
>         close(connfd);
>     }
>     close(sock);
>     return 0;
> }
> ```
>
> ​	我们通过splice函数将客户端的内容读入到pipefd[1]中，然后再使用splice函数从pipefd[0]中读出该内容到客户端，从而实现了简单高效的回射服务。整个过程未执行recv/send操作，因此也未涉及用户空间和内核空间之间的数据拷贝。

## 1.7 tee函数

> ​	tee函数在两个管道文件描述符之间复制数据，也是零拷贝操作。它不消耗数据，因此源文件描述符上的数据仍然可以用于后续的读操作。tee函数的原型如下：
>
> ```c
> #include <fcntl.h>
> ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
> ```
>
> ​	该函数的参数的含义与splice相同（**但fd_in和fd_out必须都是管道文件描述符**）。tee函数成功时返回在两个文件描述符之间复制的数据数量（字节数）。返回0表示没有复制任何数据。tee失败时返回-1并设置errno。
>
> ​	以下代码利用tee函数和splice函数，实现了Linux下tee程序（同时输出数据到终端和文件的程序，不要和tee函数混淆）的基本功能。
>
> ```c
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> 
> int main(int argc, char* argv[])
> {
>     if(argc != 2)
>     {
>         printf("usage: %s <file>\n", argv[0]);
>         return 1;
>     }
>     int filefd = open(argv[1], O_CREAT | O_WRONLY | O_TRUNC, 0666);
>     assert(filefd > 0);
>     
>     int pipefd_stdout[2];
>     int ret = pipe(pipefd_stdout);
>     assert(ret != -1);
>     
>     int pipefd_file[2];
>     ret = pipe(pipefd_file);
>     assert(ret != -1);
>     
>     /* 将标准输入内容输入管道pipefd_stdout */
>     ret = splice(STDIN_FILENO, NULL, pipefd_stdout[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>     assert(ret != -1);
>     
>     /* 将管道pipefd_stdout的输出复制到pipefd_file的输入端 */
>     ret = tee(pipefd_stdout[0], pipefd_file[1], 32768, SPLICE_F_NONBLOCK);
>     assert(ret != -1);
>     
>     /* 将管道pipefd_file的输出定向到文件描述符filefd上，从而将标准输入的内容写入文件 */
>     ret = splice(pipefd_file[0], NULL, filefd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>     assert(ret != -1);
>     
>     /* 将管道pipefd_stdout的输出定向到标准输出，其内容和写入文件的内容完全一致 */
>     ret = splice(pipfd_stdout[0], NULL, STDOUT_FILENO, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>     assert(ret != -1);
>     
>     close(filefd);
>     close(pipefd_stdout[0]);
>     close(pipefd_stdout[1]);
>     close(pipefd_file[0]);
>     close(pipefd_file[1]);
>     return 0;
> }
> ```

## 1.8 fcntl函数

> ​	fcntl函数，正如其名字（file control）描述的那样，提供了对文件描述符的各种控制操作。**另外一个常见的控制文件描述符属性和行为的系统调用是ioctl，而且ioctl比fcntl能够执行更多的控制**。但是，对于控制文件描述符常用的属性和行为，fcntl函数是由POSIX规范指定的首选方法。fcntl函数的定义如下：
>
> ```c
> #include <fcntl.h>
> int fcntl(int fd, int cmd, ……);
> ```
>
> ​	fd参数是被操作的文件描述符，cmd参数指定执行何种类型的操作。根据操作类型的不同，该函数还需要第三个可选参数arg。fcntl函数支持的常用操作及其参数如下表所示。
>
> | 操作分类                       | 操作            | 含义                                                         | 第三个参数的类型 | 成功时的返回值                    |
> | ------------------------------ | --------------- | ------------------------------------------------------------ | :--------------- | --------------------------------- |
> | 复制文件描述符                 | F_DUPFD         | 创建一个新的文件描述符，其值大于或等于arg                    | long             | 新创建的文件描述符的值            |
> | 复制文件描述符                 | F_DUPFD_CLOEXEC | 与F_DUPFD相似，不过在创建文件描述符的同时，设置其close-on-exec标志位 | long             | 新创建的文件描述符                |
> | 获取和设置文件描述符的标志     | F_GETFD         | 获取fd的标志，比如close-on-exec标志                          | 无               | fd的标志                          |
> | 获取和设置文件描述符的标志     | F_SETFD         | 设置fd的标志                                                 | long             | 0                                 |
> | 获取和设置文件描述符的状态标志 | F_GETFL         | 获取fd的状态标志，这些标志包括有open系统调用设置的标志（O_APPEND、O_CREAT等）和访问模式（O_RDONLY、和O_WRONLY和O_RDWR） | void             | fd的状态标志                      |
> | 获取和设置文件描述符的状态标志 | F_SETFL         | 设置fd的状态标志，但部分标志是不能被修改的（比如访问模式标志） | long             | 0                                 |
> | 管理信号                       | F_GETOWN        | 获得SIGIO和SIGURG信号的宿主进程的PID或进程组的组ID           | 无               | 信号的宿主进程的PID或进程组的组ID |
> | 管理信号                       | F_SETOWN        | 设置SIGIO和SIGURG信号的宿主进程的PID或者进程组的组ID         | long             | 0                                 |
> | 管理信号                       | F_GETSIG        | 获取当应用程序被通知fd可读或可写时，是哪个信号通知该事件的   | 无               | 信号值，0表示SIGIO                |
> | 管理信号                       | F_SETSIG        | 设置当fd可读或可写时，系统应该触发哪个信号来通知应用程序     | long             | 0                                 |
> | 操作管道容量                   | F_SETPIPE_SZ    | 设置有fd指定的管道的容量，/proc/sys/fs/pipe-size-max内核参数指定了fcntl能设置的管道容量的上限 | long             | 0                                 |
> | 操作管道容量                   | F_GETPIPE_SZ    | 获取由fd指定的管道的容量                                     | 无               | 管道容量                          |
>
> fcntl函数成功时的返回值如上表最后一列所示，失败返回-1并设置errno。
>
> 在网络编程中，fcntl函数通常用来将一个文件描述符设置为非阻塞的，如以下代码所示：
>
> ```c
> int setnonblocking(int fd)
> {
> 	int old_option = fcntl(fd, F_GETFL);	/* 获取文件描述符旧的状态标志 */
> 	int new_option = old_option | O_NONBLOCK;	/* 设置非阻塞标志 */
> 	fcntl(fd, F_SETFL, new_option);	
> 	
> 	return old_option;/* 返回文件描述符旧的状态标志，以便日后恢复该状态标志 */
> }
> ```
>
> ​	此外，**SIGIO和SIGURG这两个信号与其它Linux信号不同，它们必须与某个文件描述符相关联方可使用：**当被关联的文件描述符可读或可写时，系统将触发SIGIO信号；当被关联的文件描述符（而且必须是一个socket）上有带外数据可读时，系统将触发SIGURG信号。将信号和文件描述符关联的方法，就是使用fcntl函数为目标文件描述符指定宿主进程或进程组，那么被指定的宿主进程或进程组将捕获这两个信号。使用SIGIO时，还需要利用fcntl设置其O_ASYNC标志（异步I/O标志，不过SIGIO信号模型并非真正意义上的异步I/O模型）。

# 2 Linux服务器程序规范

> 除了网络通信外，服务器程序通常还必须考虑其它细节问题。这些细节问题涉及面广且零碎，而且基本上是模板式的，所以我们称之为服务器程序规范。比如：
>
> 1) Linux服务器程序一般以后台进程形式运行。后台进程又称为守护进程（daemon）。它没有控制终端，因而也不会意外接收到用户输入。守护进程的父进程通常是init进程（PID为1的进程）。
> 2) Linux服务器程序通常有一套日志系统，它至少能输出日志到文件，有的高级服务器还能输出日志到专门的UDP服务器。大部分后台进程都在/var/log目录下拥有自己的日志目录。
> 3) Linux服务器程序一般以某个专门的非root身份运行。比如mysqld、httpd、syslogd等后台进程，分别拥有自己的运行账户mysql、apache和syslog。
> 4) Linux服务器程序通常是可配置的。服务器程序通常能处理很多命令行选项，如果一次运行的选项太多，则可以用配置文件来管理。绝大多数服务器程序都有配置文件，并存放在/etc目录下。比如之前讨论的squid服务器的配置文件是/etc/squid3/squid.conf。
> 5) Linux服务器进程通常会在启动的时候生成一个PID文件并存入/var/run目录中，以记录该后台进程的PID。比如syslogd的PID文件是/var/run/syslogd.pid。
> 6) Linux服务器程序通常需要考虑系统资源和限制，以预测自身能承受多大负荷，比如进程可用文件描述符总数和内存总量等。

## 2.1 日志

### 2.1.1 系统日志

> ​	服务器的调试和维护都需要一个专业的日志系统。Linux提供了一个守护进程来处理系统日志——syslogd，不过现在的Linux系统上使用的都是它的升级版——rsyslogd。
>
> rsyslogd守护进程既能接收用户进程输出的日志，又能接收内核日志。**用户日志是通过调用syslog函数生成系统日志的。该函数将日志输出到一个UNIX本地域socket类型（AF_UNIX）的文件/dev/log中，rsyslogd则监听该文件以获取用户进程的输出。**内核日志在老的系统上是通过另外一个守护进程rklogd来管理的，rsyslogd利用额外的模块实现了相同的功能。内核日志由printk等函数打印至内核的环状缓存（ring buffer）中。环状缓存的内容直接映射到/proc/kmsg文件中。rsyslogd则通过读取该文件获得内核日志。
>
> ​	rsyslogd守护进程在接收到用户进程或内核输入的日志后，会把它们输出至某些特定的日志文件。默认情况下，调试信息会保存至/var/log/debug文件，普通信息保存至/var/log/messages文件，内核消息则保存至/var/log/kern.log文件。不过，日志信息具体如何分发，可以在rsyslogd的配置文件中设置。rsyslogd的主配置文件是/etc/rsyslog.conf，其中主要设置的项包括：内核日志输出路径，是否接收UDP日志及其监听端口（默认是514，见/etc/services文件），是否接收TCP日志及其监听端口，日志文件的权限，包含哪些子配置文件（比如/etc/rsyslog.d/*.conf）。rsyslogd的子配置文件则指定各类日志的目标存储文件。
>
> ![image-20221103203931844](../../images/Linux/2f34169d42b3ef51c6840ebb263c9a01f.png)

### 2.1.1 syslog函数

> ​	应用程序使用syslog函数与rsyslogd守护进程通信。syslog函数的定义如下：
>
> ```c
> #include <syslog.h>
> void syslog(int priority, const char* message, ...);
> ```
>
> ​	该函数采用可变参数（第二个参数message和第三个参数…）来结构化输出。priority参数是所谓的设施值与日志级别的按位或。设施值的默认值是LOG_USER，我们下面的讨论也只限于这一种设施值。日志级别有如下几个：
>
> ```c
> #include <syslog.h>
> #define LOG_EMERG	0	/* 系统不可用 */
> #define LOG_ALERT	1	/* 报警，需要立即采取动作 */
> #define LOG_CRIT	2	/* 非常严重的情况 */
> #define LOG_ERR		3	/* 错误 */
> #define LOG_WARNING	4	/* 警告 */
> #define LOG_NOTICE	5	/* 通知 */
> #define LOG_INFO	6	/* 信息 */
> #define LOG_DEBUG	7	/* 调试 */
> ```
>
> ​	下面这个函数可以改变syslog的默认输出方式，进一步结构化日志内容：
>
> ```c
> #include <syslog.h>
> void openlog(const char* ident, int logopt, int facility);
> ```
>
> ​	ident参数指定的字符串将被添加到日志信息的日期和时间之后，它通常被设置为程序的名字。logopt参数对后续syslog调用的行为进行配置，它可取下列值的按位或：
>
> ```c
> #define	LOG_PID		0x01	/* 在日志消息中包含程序PID */
> #define LOG_CONS	0x02	/* 如果消息不能记录到日志文件，则打印至终端 */
> #define LOG_ODELAY	0x04	/* 延迟打开日志功能直到第一次调用syslog */
> #define LOG_NDELAY	0x08	/* 不延迟打开日志功能 */
> ```
>
> ​	facility参数可用来修改syslog函数中的默认设施值。
>
> ​	此外，日志的过滤也很重要。程序在开发阶段可能需要输出很多调试信息，而发布之后我们又需要将这些调试信息关闭。解决这个问题的方法并不是在程序发布之后删除调试代码（因为日后可能还需要用到），而是简单地设置日志掩码，使日志级别大于日志掩码的日志信息被系统忽略。下面这个函数用于设置syslog的日志掩码：
>
> ```c
> #include <syslog.h>
> int setlogmask(int maskpri);
> ```
>
> ​	maskpri参数指定日志掩码值。该函数始终会成功，它返回调用进程先前的日志掩码值。
> 最后，不要忘了使用如下函数关闭日志功能：
>
> ```c
> #include <syslog.h>
> void closelog();
> ```

## 2.2 用户信息

### 2.2.1 UID、EUID、GID和EGID

> ​	用户信息对于服务器程序的安全性来说是很重要的，比如大部分服务器就必须以root身份启动，但不能以root身份运行。下面这一组函数可以获取和设置当前进程的真实用户ID（UID）、有效用户ID（EUID）、真实组ID（GID）和有效组ID（EGID）：
>
> ```shell
> #include <sys/types.h>
> #include <unistd.h>
> uid_t getuid();		/* 获取真实用户ID */
> uid_t geteuid();	/* 获取有效用户ID */
> gid_t getgid();		/* 获取真实组ID */
> gid_t getegid();	/* 获取有效组ID */
> int setuid(uid_t uid); /* 设置真实用户ID */
> int seteuid(uid_t uid);	/* 设置有效用户ID */
> int setgid(gid_t gid);	/* 设置真实组ID */
> int setegid(gid_t gid); /* 设置有效组ID */
> ```
>
> ​	需要指出的是，一个进程拥有两个用户ID：UID和EUID。EUID存在的目的是方便资源访问：<font color=orangered><b>它使得运行程序的用户拥有该程序的有效用户的权限。</b></font>比如su程序，任何用户都可以使用它来修改自己的账户信息，但修改账户时su程序不得不访问/etc/passwd文件，而访问该文件是需要root权限的。那么以普通用户身份启动的su程序如何能访问/etc/passwd文件呢？窍门就在EUID。用ls命令可以查看到，su程序的所有者是root，并且它被设置了set-user-id标志。这个标志表示，任何普通用户运行su程序时，其有效用户就是该程序的所有者root。那么根据有效用户的含义，任何运行su程序的普通用户都能访问/etc/passwd文件。**有效用户为root的进程称为特权进程（privileged processes）。**EGID的含义与EUID类似：给运行程序的组用户提供有效组的权限。
>
> ​	以下代码可以用来测试进程的UID和EUID的区别：
>
> ```c
> #include <unistd.h>
> #include <stdio.h>
> 
> int main()
> {
> 	uid_t uid = getuid();
> 	uid_t euid = geteuid();
> 	printf("userid is %d, effective userid is: %d\n", uid, euid);
> 	return 0;
> }
> ```
>
> ​	编译该文件，将生成的可执行文件（名为test_uid）的所有者设置为root，并设置该文件的set-user-id标志，然后运行该程序以查看UID和EUID。具体操作如下：
>
> ```shell
> sudo chown root:root test_uid	# 修改目标文件的所有者为root
> sudo chmod +s test_uid	#设置目标文件的set-user-id标志
> ./test_uid	# 运行程序
> userid is 1000, effective userid is: 0
> ```
>
> ​	从测试程序的输出来看，进程的UID是启动程序的用户的ID，而EUID则是root账户（文件所有者）的ID。

### 2.2.2 切换用户

> ​	下面的代码展示了如何将以root身份启动的进程切换为以一个普通用户身份运行：
>
> ```c
> static bool switch_to_user(uid_t user_id, gid_t gp_id)
> {
> 	/* 先确保目标用户不是root */
> 	if((user_id == 0) && (gp_id == 0))
> 	{
> 		return false;
> 	}
> 	
> 	/* 确保当前用户是合法用户：root或者目标用户，其它用户间转换需要密码，所以只可以是root用户转其它用户，或用户转自身*/
> 	gid_t gid = getgid();
> 	uid_t uid = getuid();
> 	if(((gid != 0) || (uid != 0)) && ((gid != gp_id) || (uid != user_id)))
> 	{
> 		return false;
> 	}
>     /* 如果当前用户非root用户，则其已经为目标用户 */
>     if(uid != 0)
>     {
>         return true;
>     }
>     
>     /* 切换到目标用户 */
>     if((setgid(gp_id) < 0) || (setuid(user_id) < 0))
>     {
>         return false;
>     }
>     
>     return true;
> }
> ```

## 2.3 进程间关系

### 2.3.1 进程组

> ​	Linux下每个进程都隶属于一个进程组，因此它们除了PID信息外，还有进程组ID（PGID）。我们可以用如下函数来获取指定进程的PGID：
>
> ```c
> #include <unistd.h>
> pid_t getpgid(pid_t pid);
> ```
>
> ​	该函数成功时返回进程pid所属进程组的PGID，失败则返回-1并设置errno。
>
> ​	每个进程组都有一个首领进程，其PGID和和PID相同。进程组将一直存在，直到其中所有进程都退出，或者加入到其它进程组。
>
> 下面的函数用于设置PGID：
>
> ```c
> #include <unistd.h>
> int setpgid(pid_t pid, pid_t pgid);
> ```
>
> ​	该函数将PID为pid的进程的PGID设置为pgid。如果pid和pgid相同，则由pid指定的进程将被设置为进程组首领；如果pid为0，则表示设置当前进程的PGID为pgid；如果pgid为0，则使用pid作为目标PGID。setpgid函数成功时返回0，失败则返回-1并设置errno。
>
> ​	**一个进程只能设置自己或者其子进程的PGID。并且当子进程调用exec系列函数后，我们也不能再在父进程中对它设置PGID。**

### 2.3.2  会话

> ​	一些关联的进程组将形成了一个会话（session）。下面的函数用于创建一个会话：
>
> ```c
> #include <unistd.h>
> pid_t setsid(void);
> ```
>
> ​	该函数不能由进程组的首领进程调用，否则将产生一个错误。对于非组首领的进程，调用该函数不仅创建新会话，而且有如下额外效果：
>
> * 调用进程成为了会话的首领，此时该进程是新会话的唯一成员。
>
> * 新建一个进程组，其PGID就是调用进程的PID，调用进程成为该组的首领。
>
> * 调用进程将甩开终端（如果有的话）。
>
>   该函数成功时返回新的进程组的PGID，失败则返回-1并设置errno。
>
>   Linux进程并未提供所谓会话ID（SID）的概念，但Linux系统认为它等于会话首领所在的进程组的PGID，并提供了如下函数来读取SID：
>
>   ```c
>   #include <unistd.h>
>   pid_t getsid(pid_t pid);
>   ```

### 2.3.3 用ps命令查看进程关系

> ​	执行ps命令可查看进程、进程组和会话之间的关系：
>
> ```shell
> $ ps -o pid,ppid,pgid,sid,comm | less
> PID		PPID	PGID	SID	COMMAND
> 1943	1942	1943	1943	bash
> 2298	1943	2298	1943	ps
> 2299	1943	2298	1943	less
> $ pstree
> ```
>
> ​	我们是在bash shell下执行ps和less命令的，所以ps和less命令的父进程是bash命令，这可以从PPID（父进程PID）一列看出。这3条命令创建了1个会话（SID是1943）和2个进程组（PGID分别是1943和2298）。bash命令的PID、PGID和SID都相同，很明显它即是会话的首领，也是组1943的首领。ps命令则是组2298的首领，因为其PID也是2298。下图描述了此三者的关系：
>
> ![image-20221106155128589](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221106155128589.png)

## 2.4 系统资源限制

> ​	Linux上运行的程序都会受到资源限制的影响，比如物理设备限制（CPU数量、内存数量等）、系统策略限制（CPU时间等），以及具体实现的限制（比如文件名的最大长度）。Linux系统资源限制可以通过如下一对函数来读取和设置：
>
> ```c
> #include <sys/resource.h>
> int getrlimit(int resource, struct rlimit *rlim);
> int setrlimit(int resource, const struct rlimit *rlim);
> ```
>
> ​	rlim参数是rlimit结构体类型的指针，rlimit结构体的定义如下：
>
> ```c
> struct rlimit
> {
> 	rlim_t rlim_cur;
> 	rlim_t rlim_max;
> }
> ```
>
> ​	rlim_t是一个整数类型，它描述了资源级别。rlim_cur成员指定资源的软限制，rlim_max成员指定资源的硬限制。软限制是一个建议性的、最好不要超越的限制，如果超越的话，系统可能向进程发送信号以终止其运行。例如，当进程CPU时间超过其软限制时，系统将向进程发送SIGXCPU信号；当文件尺寸超过其软限制时，系统将向进程发送SIGXFSZ信号。**硬限制一般是软限制的上限。普通程序可以减少硬限制，而只有root身份运行的程序才能增加硬限制**。此外，我们可以使用ulimit命令修改当前shell环境下的资源限制（软限制或/和硬限制），这种修改将对该shell启动的所有后续程序有效。我们也可以通过修改配置文件来改变系统软限制和硬限制。下表列举了部分比较重要的资源限制类型。
>
> 软限制: 对进程的资源数的限制的当前值, 可用getrlimit读取, setrlimit设置, 参数struct rlimitr.lim_cur. 软限制是限制的当前值, 小于等于 硬限制, 实际进程可以调用setrlimit增长到硬限制值. 也就是说, 软限制对进程并不是真正的限制.
> 硬限制: 对进程的资源数的限制的最大值, 也可以用getrlimit读取/setrlimit设置, 参数struct rlimitr.rlim_max. 硬限制是绝对上限值, 进程增长资源数不会超过硬限制.
>
> | 资源限制类型      | 含义                                                         |
> | ----------------- | ------------------------------------------------------------ |
> | RLIMIT_AS         | 进程虚拟内存总量限制（单位是字节）。超过该限制将使得某些函数（比如mmap）产生ENOMEM错误。 |
> | RLIMIT_CORE       | 进程核心转储文件（core dump）的大小限制（单位是字节）。其值为0表示不产生核心转储文件 |
> | RLIMIT_CUP        | 进程CPU时间限制（单位是秒）                                  |
> | RLIMIT_DATA       | 进程数据段（初始化数据data段、未初始化数据段bss段和堆）限制（单位是字节） |
> | RLIMIT_FSIZE      | 文件大小限制（单位是字节），超过该限制将使得某些函数（比如write）产生EFBIG错误 |
> | RLIMIT_NOFILE     | 文件描述符数量限制，超过该限制将使得某些函数（比如pipe）产生EMFILE错误 |
> | RLIMIT_NPROC      | 用户能创建的进程数量限制，超过该限制将使得某些函数（比如fork）产生EAGAIN错误 |
> | RLIMIT_SIGPENDING | 用户能够挂起的信号数量限制                                   |
> | RLIMIT_STACK      | 进程栈内存限制（单位是字节），超过该限制将引起SIGSEGV信号    |
>
> setrlimit和getrlimit成功时返回0，失败时则返回-1并设置errno。

## 2.5 改变工作目录和根目录

> ​	有些服务器程序还需要改变工作目录和根目录，比如Web服务器。一般来说，Web服务器的逻辑根目录并非文件系统的根目录“/"，而是站点的根目录（对于Linux的Web服务器来说，该目录一般是/var/www/）。
>
> ​	获取进程当前工作目录和改变进程工作目录的函数分别是：
>
> ```c
> #include <unistd.h>
> char* getcwd(char* buf, size_t size);
> int chdir(const char* path);
> ```
>
> ​	buf参数指向的内存用于存储进程当前工作目录的绝对路径名，其大小由size参数指定，如果当前工作目录的绝对路径的长度（再加上一个空结束字符"\0"）超过了size，则getcwd将返回NULL，并设置errno为ERANGE。如果buf为NULL并且size非0，则getcwd可能在内部使用malloc动态分配内存，并将进程的当前目录存储在其中。如果是这种情况，则我们必须自己来释放getcwd在内部创建的这块内存。getcwd函数成功时返回一个指向目标存储区（buf指向的缓存区或是getcwd在内部动态创建的缓存区）的指针，失败则返回NULL并设置errno。
>
> ​	chdir函数的path参数指定要切换到的目标目录。它成功时返回0，失败时返回-1并设置errno。
>
> ​	改变进程根目录的函数是chroot，其定义如下：
>
> ```c
> #include <unistd.h>
> int chroot(const char* path);
> ```
>
> ​	path参数指定要切换到的目标根目录。它成功时返回0，失败时返回-1并设置errno。chroot并不改变进程的当前工作目录，所以调用chroot之后，我们仍然需要使用chdir("/")将工作目录切换至新的根目录。改变进程的根目录之后，程序可能无法访问类似/dev的文件（和目录），因为这些文件（和目录）并非处于新的根目录之下。不过好在调用chroot之后，进程原先打开的文件描述符仍然生效，所以我们可以利用这些早打开的文件描述符来访问调用chroot之后不能直接访问的文件（和目录），尤其是一些日志文件，**此外，只有特权进程才能改变根目录。**

## 2.6 服务器程序后台化

> 最后，我们讨论如何在代码中让一个进程以守护进程的方式运行。守护进程的编写遵循一定的步骤，下面我们通过一个具体实现来探讨，如以下代码所示：
>
> ```c
> bool daemonize()
> {
> 	/* 创建子进程，关闭父进程，这样可以使程序在后台运行 */
> 	pid_t pid = fork();
> 	if(pid < 0)
> 	{
> 		return false;
> 	}
> 	else if(id > 0)
> 	{
> 		exit(0);
> 	}
> 	
> 	/* 设置文件权限掩码。当进程创建新文件（使用open(const char* pathname, int flags, mode_t mode)系统调用时，文件的权限是mode & 0777 */
> 	umask(0);
> 	
> 	/* 创建新的会话，设置本进程为进程组的首领 */
> 	pid_t sid = setsid();
> 	if(sid < 0)
> 	{
> 		return false;
> 	}
> 	
> 	/* 切换工作目录 */
> 	if((chdir("/")) < 0)
> 	{
> 		return false;
> 	}
> 	
> 	/* 关闭标准输入设备、标准输出设备和标准输出设备 */
> 	close(STDIN_FILENO);
> 	close(STDOUT_FILENO);
> 	close(STDERR_FILENO);
> 	
> 	/* 关闭其它已经打开的文件描述符， 代码省略 */
> 	/* 将标准输入、标准输出和标准错误输出都定向到/dev/null文件 */
> 	open("/dev/null", O_RDONLY);
> 	open("/dev/null", O_RDWR);
> 	open("/dev/null", O_RDWR);
> 	return true;
> }
> ```
>
> ​	实际上，Linux提供了完成同样功能的库函数：
>
> ```c
> #include <unistd.h>
> int daemon(int nochdir, int noclose);
> ```
>
> ​	其中，nochdir参数用于指定是否改变工作目录，如果给它传递0，则工作目录被设置为"/"（根目录），否则继续使用当前工作目录。noclose参数为0时，标准输入、标准输出和标准错误输出都被重定向到/dev/null文件，否则依然使用原来的设备。该函数成功时返回0，失败则返回-1并设置errno。

# 3 高性能服务器程序框架

## 3.1 服务器模型

### 3.1.1 C/S模型

> ​	TCP/IP协议在设计和实现上并没有客户端和服务器的概念，在通信过程中所有机器都是对等的。但由于资源（视频、新闻、软件等）都被数据提供者所垄断，所以几乎所有的网络应用程序都很自然地采用了如下图所示的C/S（客户端/服务器）模型：所有客户端都通过访问服务器来获取所需的资源。
>
> ![image-20221107194354685](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107194354685.png)
>
> ​	采用C/S模型的TCP服务器和TCP客户端的工作流程如下图8-2所示。
>
> ​	C/S模型的逻辑很简单。服务器启动后，首先创建一个（或多个）监听socket，并调用bind函数将其绑定到服务器感兴趣的端口上，然后调用listen函数等待客户连接。服务器稳定运行之后，客户端就可以调用connect函数服务器发起连接了。**由于客户连接请求是随机到达的异步事件，服务器需要使用某种I/O模型来监听这一事件。**I/O模型有多种，图8-2中，服务器使用的是I/O复用技术之一的select系统调用。当监听到连接请求后，**服务器就调用accept函数接受它，并分配一个逻辑单元为新的连接服务。**逻辑单元可以是新创建的子进程、子线程或其它。图8-2中，服务器给客户端分配的逻辑单元是由fork系统调用创建的子进程。逻辑单元读取客户端请求，处理该请求，然后将处理结果返回给客户端。客户端接收到服务器的反馈结果之后，可以继续向服务器发送请求，也可以立即主动关闭连接。如果客户端主动关闭连接，则服务器执行被动关闭连接。至此，双方的通信结束。需要注意的是，服务器在处理一个客户请求的同时还会继续监听其它客户请求，否则就变成了效率低下的串行服务器了（必须先处理完前一个客户的请求，才能继续处理下一个客户请求）。图8-2中，服务器同时监听多个客户请求是通过select系统调用实现的。
>
> ![image-20221107195914814](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107195914814.png)
>
> ​	C/S模型非常适合资源相对集中的场合，并且它的实现也很简单，但其缺点也很明显：服务器是通信的中心，当访问量过大时，可能所有客户都将得到很慢的响应。下面讨论的P2P模型解决了这个问题。

### 3.1.2 P2P模型

> ​	P2P（Peer to Peer，点对点）模型比C/S模型更符合网络通信的实际情况。它摒弃了以服务器为中心的格局，让网络上所有主机重新回归对等的地位。P2P模型如图8-3a所示。
>
> ​	P2P模型使得每台机器在消耗服务的同时也给别人提供服务，这样资源就能够充分、自由地共享。云计算机群可以看作P2P模型的一个典范。但P2P模型的缺点也很明显：当用户之间传输请求过多时，网络的负载将加重。
>
> ​	图8-3a所示的P2P模型存在一个显著的问题，即主机之间很难相互发现。所以实际使用的P2P模型通常带有一个专门的发现服务器，如图8-3b所示。这个发现服务器通常还提供查找服务（甚至还可以提供内容服务），使每个客户都能尽快地找到自己需要的资源。
>
> ![image-20221107201943967](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107201943967.png)
>
> ​	从编程角度来讲，P2P模型可以看作是C/S模型的扩展：每台主机既是客户端，又是服务器。

## 3.2 服务器编程框架

> ​	虽然服务器程序种类繁多，但其基本框架都一样，不同之处在于逻辑处理。如8-4所示：
>
> ![image-20221107202326885](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107202326885.png)
>
> ​	该图既能描述一台服务器，也能用来描述一个服务器机群。两种情况下各个部件的含义和功能如表8-1所示：
>
> | 模块         | 单个服务器程序             | 服务器机群                   |
> | ------------ | -------------------------- | ---------------------------- |
> | I/O处理单元  | 处理客户连接，读写网络数据 | 作为接入服务器，实现负载均衡 |
> | 逻辑单元     | 业务进程或线程             | 逻辑服务器                   |
> | 网络存储单元 | 本地数据库、文件或缓存     | 数据库服务器                 |
> | 请求队列     | 各单元之间的通信方式       | 各服务器之间的永久TCP连接    |
>
> ​	I/O处理单元是服务器管理客户连接的模块。它通常要完成以下工作：等待并接受新的客户连接，接收客户连接，将服务器响应数据返回给客户端。但是，数据的收发不一定在I/O处理单元中执行，也可能在逻辑单元中执行，具体在何处执行取决于事件处理模式。对于一个服务器机群来说，I/O处理单元是一个专门的接入服务器。它实现负载均衡，从所有逻辑服务器中选取负荷最小的一台来为新客户服务。
>
> ​	一个逻辑单元通常是一个进程或线程。它分析并处理客户数据，然后将结果传递给I/O处理单元或者直接发送给客户端（具体使用哪种方式取决于事件处理模式）。对服务器机群而言，一个逻辑单元本身就是一台逻辑服务器。服务器通常拥有多个逻辑单元，以实现对多个客户任务的并行处理。
>
> ​	网络存储单元可以是数据库、缓存和文件，甚至是一台独立的服务器。但它不是必须的，比如ssh、telnet等服务就不需要这个单元。
>
> ​	请求队列是各单元之间的通信方式的抽象。I/O处理单元接收到客户请求时，需要以某种方式通知一个逻辑单元来处理该请求。同样，多个逻辑单元同时访问一个存储单元时，也需要采用某种机制来协调处理竞态条件。请求队列通常被实现为池的一部分，我们将在后面讨论池的概念。对于服务器机群而言，请求队列是各台服务器之间预先建立的，静态的，永久的TCP连接。这种TCP连接能提高服务器之间交换数据的效率，因为它避免了动态建立TCP连接导致的额外的系统开销。

## 3.3 I/O模型

> ​	socket在创建的时候默认是阻塞的。我们可以给socket系统调用的第2个参数传递SOCK_NONBLOCK标志，或者通过fcntl系统调用的F_SETFL命令，将其设置为非阻塞的。阻塞和非阻塞的概念能应用于所有文件描述符，而不仅仅是socket。我们称阻塞的文件描述符为阻塞I/O，称非阻塞的文件描述符为非阻塞I/O。
>
> ​	**针对阻塞I/O执行的系统调用可能因为无法立即完成而被操作系统挂起，直到等待的事件发生为止。**比如，客户端通过connect向服务器发起连接时，connect将首先发送同步报文给服务器，然后等待服务器返回确认报文。如果服务器的确认报文段没有立即到达客户端，则connect调用将被挂起，直到客户端收到确认报文段并唤醒connect调用。**socket的基础API中，可能被阻塞的系统调用包括accept、send、recv和connect。**
>
> ​	针对非阻塞I/O执行的系统调用则总是立即返回，而不管事件是否已经发生。**如果事件没有立即发生，这些系统调用就返回-1，和出错的情况一样。**此时我们必须根据errno来区分这两种情况。**对accep、send和recv而言，事件未发生时errno通常被设置成EAGAIN（意为“再来一次”）或者EWOULDBLOCK（意为“期望阻塞”）；对connect而言，errno则被设置成EINPROGRESS（意为“在处理中”）。**
>
> ​	很显然，**我们只有在事件已经发生的情况下操作非阻塞I/O（读、写等），才能提高程序的效率。**因此，非阻塞I/O通常要和其它I/O通知机制一起使用，比如I/O复用和SIGIO信号。
>
> ​	I/O复用是最常使用的I/O通知机制。它指的是，**应用程序通过I/O复用函数向内核注册一组事件，内核通过I/O复用函数把其中就绪的事件通知给应用程序。**Linux上常用的I/O复用函数是**select、poll和epoll_wait**，我们将在后续的章节讨论它们。需要指出的是，**I/O复用函数本身是阻塞的，它们能提高程序效率的原因在于它们具有同时监听多个I/O事件的能力。**
>
> ​	SIGIO信号也可以用来报告I/O事件。前面提到过，fcntl函数可以为一个目标文件描述符指定宿主进程，那么被指定的宿主进程将捕获到SIGIO信号。这样，当目标文件描述符上有事件发生时，SIGIO信号的信号处理函数将被触发，我们也就可以在该信号处理函数中对目标文件描述符执行非阻塞I/O操作了。
>
> ​	从理论上来说，<font color=green><b>阻塞I/O、I/O复用和信号驱动I/O都是同步I/O模型。因为在这三种I/O模型中，I/O的读写操作，都是在I/O事件发生之后，由应用程序来完成的。</b></font>而POSIX规范所定义的异步I/O模型则不同。对异步I/O而言，用户可以直接对I/O执行读写操作，这些操作告诉内核用户读写缓冲区的位置，以及I/O操作完成之后内核通知应用程序的方式。异步I/O的读写操作总是立即返回，而不论I/O是否是阻塞的，因为真正的读写操作已经由内核接管。<font color=orangered><b>也就是说，同步I/O模型要求用户代码自行执行I/O操作（将数据从内核缓冲区读入用户缓冲区，或将数据从用户缓冲区写入内核缓冲区），而异步I/O机制则由内核来执行I/O操作（数据在内核缓冲区和用户缓冲区之间的移动时由内核在“后台”完成的）。你可以这样认为，同步I/O向应用程序通知的是I/O就绪事件，而异步I/O向应用程序通知的是I/O完成事件。Linux环境下，aio.h头文件定义的函数提供了对异步I/O的支持。</b></font>
>
> ​	作为总结，我们将上面讨论的几种I/O模型的差异列于下表中。
>
> | I/O模型   | 读写操作和阻塞阶段                                           |
> | --------- | ------------------------------------------------------------ |
> | 阻塞I/O   | 程序阻塞于读写函数                                           |
> | I/O复用   | 程序阻塞于I/O复用系统调用，但可以同时监听多个I/O事件。对I/O本身的读写操作是非阻塞的 |
> | SIGIO信号 | 信号触发读写就绪事件，用户程序执行读写操作。程序没有阻塞阶段 |
> | 异步I/O   | 内核执行读写操作并触发读写完成事件。程序没有阻塞阶段         |

## 3.4 两种高效的事件处理模式

> ​	服务器程序通常需要处理三类事件：I/O事件、信号及定时事件。我们将在后续章节依次讨论这三种类型的事件，这一节先从整体上介绍一下两种高效的事件处理模式：Reactor和Proactor。
>
> ​	随着网络设计模式的兴起，Reactor和Proactor事件处理模式应运而生。<font color=orange><b>同步I/O模型通常用于实现Reactor模式，</b></font>异步I/O模型则用于实现Proactor模式。不过后面我们将看到如何使用同步I/O方式模拟出Proactor模式。

### 3.4.1 Reactor模式

> ​	Reactor是这样一种模式，<font color=green><b>它要求主线程（I/O处理单元，下同）只负责监听文件描述符上是否有事件发生，有的话就立即将该事件通知工作线程（逻辑单元，下同）。除此之外，主线程不做任何其他实质性的工作。读写数据，接受新的连接，以及处理客户请求均在工作线程中完成。</b></font>
>
> ​	使用同步I/O模型（以epoll_wait为例）实现的Reactor模式的工作流程是：
>
> > 1) 主线程往epoll内核事件表中注册socket上的读就绪事件
> > 2) 主线程调用epoll_wait等待socket上有数据可读。
> > 3) 当socket上有数据可读时，epoll_wait通知主线程。主线程则将socket可读事件放入请求队列。
> > 4) 睡眠在请求队列上的某个工作线程被唤醒，它从socket读取数据，并处理客户请求，然后完epoll内核事件表中注册socket上的写就绪事件。
> > 5) 主线程调用epoll_wait等待socket可写。
> > 6) 当socket可写时，epoll_wait通知主线程。主线程将socket可写事件放入请求队列。
> > 7) 睡眠在请求队列上的某个工作线程被唤醒，它往socket上写入服务器处理客户请求的结果。
>
> 图8-5总结了Reactor模式的工作流程。
>
> ![image-20221109125102782](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221109125102782.png)
>
> ​	图8-5中，工作线程从请求队列中取出事件后，将根据事件的类型来决定如何处理它：
> 对于可读事件，执行读数据和处理请求的操作：对于可写事件，执行写数据的操作。因此，图8-5所示的Reactor模式中，没必要区分所谓的“读工作线程”和“写工作线程”。

### 3.4.2 Proactor模式

> 与Reactor模式不同，<font color=green><b>Proactor模式将所有I/O操作都交给主线程和内核来处理，工作线程仅仅负责业务逻辑。</b></font>因此Proactor模式更符合图8-4所描述的服务器编程框架。
>
> ​	使用异步I/O模型（以aio_read和aio_write为例）实现的Proactor模式的工作流程是：
>
> 1) 主线程调用aio_read函数向内核注册socket上的读完成事件，并告诉内核用户读缓冲区的位置，以及读操作完成时如何通知应用程序（这里以信号为例，详情请参考sigevent的man手册）。
> 2) 主线程继续处理其它逻辑。
> 3) 当socket上的数据被读入用户缓冲区后，内核将向应用程序发送一个信号，以通知应用程序数据已经可用。
> 4) 应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求。工作线程处理完客户请求之后，调用aio_write函数向内核注册socket上的写完成事件，并告诉内核用户写缓冲区的位置，以及写操作完成时如何通知应用程序（仍然以信号为例）。
> 5) 主线程继续处理其他逻辑。
> 6) 当用户缓冲区的数据被写入socket之后，内核将向应用程序发送一个信号，以通知应用程序数据已经发送完毕。
> 7) 应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，比如决定是否关闭socket。

> 图8-6总结了Proactor模式的工作流程。
>
> ![image-20221108215110003](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221108215110003.png)
>
> ​	在图8-6中，连接socket上的读写事件是通过aio_read/aio_write向内核注册的，因此内核将通过信号来向应用程序报告连接socket上的读写事件。所以，主线程中的epoll_wait调用仅能用来检测监听socket上的连接请求事件，而不能用来检测连接socket上的读写事件。

### 3.4.3 模拟Proactor模式

> ​	其原理是：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一“完成事件”。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。
>
> ​	使用同步I/O模型（仍然以epoll_wait为例）模拟出的Proactor模式的工作流程如下：
>
> 1) 主线程往epoll内核事件表中注册socket上的读就绪事件。
> 2) 主线程调用epoll_wait等待socket上有数据可读。
> 3) 当socket上有数据可读时，epoll_wait通知主线程。主线程从socket循环读取数据，直到没有更多数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。
> 4) 睡眠在请求队列上的某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往epoll内核事件表中注册socket上的写就绪事件。
> 5) 主线程调用epoll_wait等待socket可写。
> 6) 当socket可写时，epoll_wait通知主线程。主线程往socket上写入服务器处理客户请求的结果。
>
> 图8-7总结了用同步I/O模型模拟出的Proactor模式的工作流程。
>
> ![image-20230112205120754](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230112205120754.png)

## 3.5 两种高效的并发模式

> 并发编程的目的是让程序“同时”执行多个任务。如果程序是计算密集型的，并发编程并没有优势，反而由于任务的切换使效率降低。但如果程序是I/O密集型的，比如经常读写文件，访问数据库等，则情况就不同了。由于I/O操作的速度远没有CPU的计算速度快，所以让程序阻塞于I/O操作将浪费大量的CPU时间。如果程序有多个执行线程，则当前被I/O操作所阻塞的执行线程可主动放弃CPU（或由操作系统来调度），并将执行权转移到其它线程。这样一来，CPU就可以用来做更加有意义的事情（除非所有线程都同时被I/O操作所阻塞），而不是等待I/O操作的完成，因此CPU的利用率显著提升。
>
> ​	从实现上来说，并发编程主要有**多进程**和**多线程**两种方式。我们将在后续章节详细讨论它们，这一节讨论并发模式。对应于图8-4，<font color=green>**并发模式是指I/O处理单元和多个逻辑单元之间协调完成任务的方法**</font>。服务器主要有两种并发编程模式：半同步/半异步（half-sync/half-async）模式和领导者/追随者（Leader/Follower）模式。

### 3.5.1 半同步/半异步模式

> ​	首先，半同步/半异步模式中的“同步”和“异步”与前面讨论的I/O模型中的“同步”和“异步”是完全不同的概念。在I/O模型中，“同步”和“异步”区分的是内核向应用程序通知的是何种I/O事件（ 是就绪事件还是完成事件），以及该由谁来完成I/O读写（是应用程序还是内核）。<font color=orangered><b>在并发模式中，“同步”指的是程序完全按照代码序列的顺序执行；“异步”指的是程序的执行需要系统事件来驱动。常见的系统事件包括中断、信号等。</b></font>比如，图8-8a描述了同步的读操作，而8-8b描述了异步的读操作。
>
> ![image-20230114175907734](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog/image/image-20230114175907734.png)
>
> ​	按照同步方式运行的线程称为同步线程，按照异步方式运行的线程称为异步线程。显然，异步线程的执行效率高，实时性强，这是很多嵌入式程序采用的模型。但编写以异步方式执行的程序相对复杂，难以调试和扩展，而且不适合于大量的并发。而同步线程则相反，它虽然效率相对较低，实时性较差，但逻辑简单。因此，对于像服务器这种既要求较好实时性，又要求能同时处理多个客户请求的应用程序，我们就应该使用同步线程和异步线程来实现，即采用半同步/半异步模式来实现。
>
> ​	半同步/半异步模式中，**同步线程用于处理客户逻辑**，相当于图8-4中的逻辑单元；**异步线程用于处理I/O事件**，相当于图8-4中的I/O处理单元。异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中。请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象。具体选择哪个工作线程来为新的客户请求服务，则取决于请求队列的设计。比如最简单的轮流选取工作线程的Round Robin算法，也可以通过条件变量或信号量来随机地选择一个工作线程。图8-9总结了半同步/半异步模式的工作流程。
>
> ![image-20230217164411022](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230217164411022.png)
>
> ​	在服务器程序中，如果结合考虑两种事件处理模式和几种I/O模型，则半同步/半异步模式就存在多种变体。其中有一种变体称为半同步/半反应堆（half-sync/half-reactive)模式，如图8-10所示。
>
> ![image-20230220202800549](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230220202800549.png)
>
> ​	图8-10中，**异步线程只有一个，由主线程来充当。它负责监听所有所有socket上的事件**。如果监听socket上有可读事件发生，即有新的连接请求到来，主线程就接受之以得到新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件。如果连接socket上有读写事件发生，即有新的客户到来或有数据要发送到客户端，主线程插入请求队列中。所有工作线程都睡眠在请求队列上，当有任务到来时，它们将通过竞争（比如申请互斥锁）获得任务的接管权。这种竞争机制使得只有空闲的工作线程才有机会来处理新任务，这是很合理的。
>
> ​	图8-10中，主线程插入请求队列中的任务是就绪的连接socket。这说明该图所示的半同步/半异步反应堆模式采用的事件处理模式是Reactor模式：它要求工作线程自己从socket上读取客户请求和往socket写入服务器应答。这就是该模式的名称“half-reactive”的含义。实际上，半同步/半反应堆模式也可以使用模拟的Proactor事件处理模式，即由主线程来完成数据的读写。在这种情况下，主线程一般会将应用程序数据、任务类型等信息封装为一个任务对象，然后将其（或者指向该任务对象的一个指针）插入请求队列。工作线程从请求队列中取得任务对象之后，即可处理之，而无须执行读写操作了。
>
> ​	半同步/半反应堆模式模式存在如下缺点：
>
> * 主线程和工作线程共享请求队列。主线程往请求队列中添加任务，或者工作线程从请求队列中取任务，都需要对请求队列加锁保护，从而白白耗费CPU时间。
> * 每个工作线程在同一时间只能处理一个客户请求。如果客户数量较多，而工作线程较少，则请求队列中将堆积很多任务对象

### 3.5.2 领导者/追随者模式

## 3.6 有限状态机

> ​	前面两节探讨的是服务器的I/O处理单元、请求队列和逻辑单元之间协调完成任务的各种模式，这一节我们介绍逻辑单元内部的一种高效编程方法：有限状态机（finite state machine）。
>
> ​	有的应用层协议头部包含数据包类型字段，每种类型可以映射为逻辑单元的一种执行状态，服务器可以根据它来编写相应的处理逻辑，如以下代码所示：
>
> ```c
> STATE_MACHINE(Package_pack)
> {
> 	PackageType _type = _pack.GetType();
> 	switch(_type)
> 	{
> 		case type_A:
> 			process_package_A(_pack);
> 		case type_B:
> 			process_package_B(_pack);
> 			break;
> 	}
> }
> ```
>
> ​	这是一个简单的有限状态机，只不过该状态机的每个状态都是相互独立的，即状态之间没有相互转移。状态之间的转移是需要状态机内部驱动的，如代码所示：
>
> ```c
> STATE_MACHINE()
> {
> 	State cur_State = type_A;
> 	while(cur_State != type_C)
> 	{
> 		Package _pack = getNewPackage();
> 		switch(cur_State)
> 		{
> 			case type_A:
> 				process_package_state_A(_pack);
> 				cur_State = type_B;
> 				break;
> 			case type_B:
> 				process_package_state_B(_pack);
> 				cur_State = type_C;
> 				break;
> 		}
> 	}
> }
> ```
>
> ​	该状态机包含三种状态：type_A、type_B和type_C，其中type_A是状态机的开始状态，type是状态机的结束状态。状态机的当前状态记录在cur_State变量中。在一趟循环过程中，状态机先通过getNewPackage方法获得一个新的数据包，然后根据cur_State变量的值判断如何处理该数据包。数据包处理完之后，状态机通过给cur_State变量传递目标状态值来实现状态转移。那么当状态机进入下一趟循环时，它将执行新的状态对应的逻辑。
>
> ​	下面我们考虑有限状态机应用的一个实例：HTTP请求的读取和分析。很多网络协议，包括TCP协议和IP协议，都在其头部中提供头部长度字段。程序根据该字段的值就可以知道是否接收到一个一个完整的协议头部。但HTTP协议并未提供这样的头部长度字段，并且其头部变化很大，可以只有十几字节，也可以有上百字节。根据协议规定，我们判断HTTP头部结束的依据是遇到一个空行，该空行仅包含一对回车换行符（`<CR><LF>`)。如果一次读操作没有读入HTTP请求的整个头部，即没有遇到空行，那么我们必须等待客户继续写数据并再次读入。因此，我们每完成因此读操作，就要分析新读入的数据中是否有空行。不过在寻找空行的过程中，我们可以同时完成对整个HTTP请求头部的分析（记住，空行前面还有请求行和头部域），以提高解析HTTP请求的效率。以下代码使用主、从两个有限状态机实现了最简单的HTTP请求的读取和分析。为了使表述简洁，我们约定，直接称HTTP请求的一行（包括请求行和头部字段）为行。
>
> ```c++
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <stdlib.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <strings.h>
> #include <fcntl.h>
> #define BUFFER_SIZE 4096 /* 读缓冲区大小 */
> 
> /* 主状态机的两种可能状态，分别表示：当前正在分析请求行，当前正在分析头部字段 */
> enum CHECK_STATE { CHECK_STATE_REQUESTLINE = 0, CHECK_STATE_HEADER };
> /* 从状态机的三种可能的状态，即行的读取状态，分别表示：读取到一个完整的行、行出错和行数据尚且不完整 */
> enum LINE_STATUS { LINE_OK = 0, LINE_BAD, LINE_OPEN };
> 
> /* 服务器处理HTTP请求的结果：NO_REQUEST表示请求不完整，需要继续读取客户端数据；GET_REQUEST表示获得了一个完整的客户请求；BAD_REQUEST
> 表示客户请求有语法错误；FORBIDDEN_REQUEST表示客户对资源没有足够的访问权限；INTERNAL_ERROR表示服务器内部错误；CLOSED_CONNECTION表示客户端
> 已经关闭连接了 */
> enum HTTP_CODE { NO_REQUEST, GET_REQUEST, BAD_REQUEST,
>                 FORBIDDEN_REQUEST, INTERNAL_ERROR, CLOSED_CONNECTION };
> /* 为了简化问题，我们没有给客户端发送一个完整的HTTP应答报文，而只是根据服务器的处理结果发送如下成功或失败信息 */
> static const char* szret[] = { "I get a correct result\n", "Something wrong\n"};
> 
> /* 从状态机，用于解析出一行内容 */
> LINE_STATUS parse_line(char* buffer, int& checked_index, int& read_index)
> {
>     char temp;
>     /* checked_index指向buffer（应用程序的读缓冲区）中当前正在分析的字节，read_index指向buffer中客户数据的尾部的下一个字节。
>     buffer中第0~checked_index字节都已分析完毕，第checked_index~（read_index-1）字节由下面的循环挨个分析 */
>     for( ; checked_index < read_index; ++checked_index)
>     {
>         /* 获得当前要分析的字节 */
>         temp = buffer[checked_index];
>         /* 如果当前的字节是“\r”，即回车符，则说明可能读到一个完整的行 */
>         if(temp == '\r')
>         {
>             /* 如果“\r”字符碰巧是目前buffer中最后一个已经被读入的客户数据，那么这次
>             分析没有读到一个完整的行，返回LINE_OPEN以表示还需要继续读取客户数据才能进一步分析 */
>             if((checked_index + 1) == read_index)
>             {
>                 return LINE_OPEN;
>             }
>             /* 如果下一个字符是“\n”，则说明我们成功读取到一个完整的行 */
>             else if(buffer[checked_index + 1] == '\n')
>             {
>                 buffer[checked_index++] = '\0';
>                 buffer[checked_index++] = '\0';
>                 return LINE_OK;
>             }
>             /* 否则的话，说明客户发送的HTTP请求存在语法问题 */
>             return LINE_BAD;
>         }
>         /* 如果当前的字节是“\n”，即换行符，则也说明可能读取到一个完整的行 */
>         else if(temp == '\n')
>         {
>             if((checked_index > 1) && buffer[checked_index - 1] == '\r')
>             {
>                 buffer[checked_index - 1] = '\0';
>                 buffer[checked_index++] = '\0';
>                 return LINE_OK;
>             }
>             return LINE_BAD;
>         }
>     }
>     /* 如果所有内容都分析完毕也没有遇到“\r”字符，则返回LINE_OPEN，表示还需要继续读取客户数据才能进一步分析 */
>         return LINE_OPEN;
> }
> /* 分析请求行 */
> HTTP_CODE parse_requestline(char* temp, CHECK_STATE& checkstate)
> {
>     char* url = strpbrk(temp, " \t");   /* 如果请求行中没有空白字符或“\t”字符，则HTTP请求必有问题 */
>     if(!url)
>     {
>         return BAD_REQUEST;
>     }
>     *url++ = '\0';
> 
>     char* method = temp;
>     if(strcasecmp(method, "GET") == 0)  /* 仅支持GET方法 */
>     {
>         printf("The request method is GET\n");
>     }
>     else
>     {
>         return BAD_REQUEST;
>     }
> 
>     url += strspn(url, " \t"); /* 跳过多余空格或“\t”字符 */
>     char* version = strpbrk(url, " \t");
>     if(!version)
>     {
>         return BAD_REQUEST;
>     }
>     *version++ = '\0';
>     version += strspn(version, " \t");
>     /* 仅支持HTTP/1.1 */
>     if(strcasecmp(version, "HTTP/1.1") != 0)
>     {
>         return BAD_REQUEST;
>     }
>     /* 检查URL是否合法 */
>     if(strncasecmp(url, "http://", 7) == 0)
>     {
>         url += 7;
>         url = strchr(url, '/');
>     }
>     if(!url || url[0] != '/')
>     {
>         return BAD_REQUEST;
>     }
>     printf("The request URL is: %s:\n", url);
>     /* HTTP 请求行处理完毕，状态转移到头部字段的分析 */
>     checkstate = CHECK_STATE_HEADER;
>     return NO_REQUEST;
> }
> 
> /* 分析请求行 */
> HTTP_CODE parse_headers(char* temp)
> {
>     /* 遇到一个空行，说明我们得到了一个正确的HTTP请求 */
>     if(temp[0] == '\0')
>     {
>         return GET_REQUEST;
>     }
>     else if(strncasecmp(temp, "Host:", 5) == 0) /* 处理“Host"头部字段 */
>     {
>         temp += 5;
>         temp += strspn(temp, " \t");
>         printf("the request host is: %s\n", temp);
>     }
>     else    /* 其他头部字段都不处理 */
>     {
>         printf("I can not handle this header\n");
>     }
>     return NO_REQUEST;
> }
> 
> 
> /* 分析HTTP请求的入口函数 */
> HTTP_CODE parse_content(char* buffer, int& checked_index, CHECK_STATE& checkstate, int& read_index, int& start_line)
> {
>     LINE_STATUS linestatus = LINE_OK;   /*记录当前行的读取状态 */
>     HTTP_CODE retcode = NO_REQUEST;     /* 记录HTTP请求的处理结果 */
>     /* 主状态机，用于从buffer中取出所有完整的行 */
>     while((linestatus = parse_line(buffer, checked_index, read_index)) == LINE_OK)
>     {
>         char* temp = buffer + start_line;   /* start_line是行在buffer中的起始位置 */
>         start_line = checked_index;         /* 记录下一行的起始位置 */
>         /* checkstate 记录主状态机当前的状态 */
>         switch(checkstate)
>         {
>            case CHECK_STATE_REQUESTLINE:    /* 第一个状态，分析请求行 */ 
>            {
>                 retcode = parse_requestline(temp, checkstate);
>                 if(retcode == BAD_REQUEST)
>                 {
>                     return BAD_REQUEST;
>                 }
>                 break;
>            }
>             case CHECK_STATE_HEADER:    /* 第二个状态，分析头部字段 */
>             {
>                 retcode = parse_headers(temp);
>                 if(retcode == BAD_REQUEST)
>                 {
>                     return BAD_REQUEST;
>                 }
>                 else if(retcode == GET_REQUEST)
>                 {
>                     return GET_REQUEST;
>                 }
>                 break;
>             }
>             default:
>             {
>                 return INTERNAL_ERROR;
>             }
>         }
>     }
>     /* 若没有读取到一个完整的行，则表示还需要继续读取客户数据才能进一步分析 */
>     if(linestatus == LINE_OPEN)
>     {
>         return NO_REQUEST;
>     }
>     else
>     {
>         return BAD_REQUEST;
>     }
> }
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
>     int ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
>     struct sockaddr_in client_address;
>     socklen_t client_addrlength = sizeof(client_address);
>     int fd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>     if(fd < 0)
>     {
>         printf("errno is: %d\n", errno);
>     }
>     else
>     {
>         char buffer[BUFFER_SIZE]; /* 读缓冲区 */
>         memset(buffer, '\0', BUFFER_SIZE);
>         int data_read = 0;
>         int read_index = 0; /* 当前已经读取了多少字节的客户数据 */
>         int checked_index = 0; /* 当前已经分析完了多少字节的客户数据 */
>         int start_line = 0; /* 行在buffer中的起始位置 */
>         /* 设置主状态机的初始状态 */
>         CHECK_STATE checkstate = CHECK_STATE_REQUESTLINE;
>         while(1)    /* 循环读取数据并分析之 */
>         {
>             data_read = recv(fd, buffer + read_index, BUFFER_SIZE - read_index, 0);
>             if(data_read == -1)
>             {
>                 printf("reading failed\n");
>                 break;
>             }
>             else if(data_read == 0)
>             {
>                 printf("remote client has closed thee connection\n");
>                 break;
>             }
>             read_index += data_read;
>             /* 分析目前已经获得的所有客户端数据 */
>             HTTP_CODE result = parse_content(buffer, checked_index, checkstate, read_index, start_line);
>             if(result == NO_REQUEST)    /* 尚未得到一个完整的HTTP请求 */
>             {
>                 continue;
>             }
>             else if(result == GET_REQUEST) /* 得到一个完整的，正确的HTTP请求 */
>             {
>                 send(fd, szret[0], strlen(szret[0]), 0);
>                 break;
>             }
>             else    /* 其他情况表示发生错误 */
>             {
>                 send(fd, szret[1], strlen(szret[1]), 0);
>                 break;
>             }
>         }
>         close(fd);
>     }
>     close(listenfd);
>     return 0;
> }
> ```
>
> ​	我们将代码中的两个有限状态机分别称为主状态机和从状态机，这体现了它们之间的关系：主状态机在内部调用从状态机。下面分析从状态机，即parse_line函数，它从buffer中解析出一个行。图8-15描述了其可能的状态及状态转移过程。
>
> ![image-20230527105502472](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230527105502472.png)
>
> ​	这个状态机的初始状态时LINE_OK，其原始驱动动力来自于buffer中新到达的客户数据。在main函数中，我们循环调用recv函数往buffer中读入客户数据。每次成功读取数据后，我们就调用parse_content函数来分析新读入的数据。parse_content函数首先要做的就是调用parse_line函数来获取一个行。现在假设服务器经过一次recv调用之后，buffer的内容以及部分变量的值如8-16a所示。
>
> ​	parse_line函数处理后的结果如图8-16b所示，它挨个检查图8-16a所示的buffer中checked_index到（read_index - 1）之间的字节，判断是否存在行结束符，并更新checked_index的值。当前buffer中不存在行结束符，所以parse_line返回LINE_OPEN。接下来，程序继续调用recv以读取更多的客户数据，这次读操作后buffer中的内容以及部分变量的值如图8-16c所示。然后parse_line函数就又开始处理这部分新到来的数据，如图8-16d所示。这次它读取到了一个完整的行，即“HOST:localhost\r\n”。此时parse_line函数就可以将这行内容递交给parse_content函数中的主状态机来处理了。
>
> ![image-20230527110333720](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230527110333720.png)
>
> ​	主状态机使用checkstate变量来记录当前状态。如果当前的状态时CHECK_STATE_REQUESTLINE，则表示parse_line函数解析出的行是请求行，于是主状态机调用parse_requestline函数来分析请求行；如果当前状态是CHECK_STATE_HEADER，则表示parse_line函数解析出的是头部字段，于是主状态机调用parse_headers函数来分析头部字段。checkstate变量的初始值是CHEC_STATE_REQUESTLINE，parse_requestline函数在成功地分析完请求行之后将其设置为CHECK_STATE_HEADER，从而实现状态转移。
>
> ```c
> #include <string.h>
> size_t strspn(const char* str1, const char* str2);
> /* 
> 它的作用是返回在一个字符串中连续包含另一个字符串任意字符的最长起始子串的长度。
> 通常用来跳过连续多余的空格或“\t”符号。
> */
> //eg:
> strspn(url, " \t");
> 
> #include <string.h>
> char *strpbrk(const char *s1, const char *s2);
> /*
> 在源字符串（s1）转给你找出最先含有搜索字符串（s2）中任一字符的位置并返回，若找不到则返回空指针。
> 通常用来找到HTTP报文请求行的分隔符。
> */
> //eg:
> strpbrk(temp, " \t");
> 
> #include <strings.h> /* 非C/C++标准库中的头文件，只在Linux中提供，相当于windows平台的stricmp */
> int strcasecmp(const char *s1, const char *s2);
> /*
> 忽略大小写地比较字符串，相等返回0，s1大于s2只返回大于0的值，s1小于s2只返回小于0的值（即最后比较字符的差值）。
> */
> 
> int strncasecmp(const char *s1, const char *s2, size_t n);
> /*
> 字符串s1和s2自左向右比较n个字符，且忽略英文字母的大小，直至比较字符不同或比较完前n个字符或遍历完某一字符串。
> */
> //eg:
> strcasecmp(url, "http://", 7);
> 
> #include <string.h>
> char* strchr(const char *str, int c);
> /*
> 在str字符串中搜索第一次出现字符c的位置。
> 返回一个指向该字符串中第一次出现的字符的指针，如果不包含该字符则返回NULL空指针。
> */
> ```



## 3.7 提高服务器性能的其他建议

> ​	性能对服务器来说是至关重要的，毕竟每个客户都期望其请求能很快地得到响应。影响服务器性能的首要因素就是系统的硬件资源，比如CPU的个数、速度，内存的大小等。不过由于硬件技术的飞快发展，现代服务器都不缺乏硬件资源。因此，我们需要考虑的主要问题是如何从“软环境”来提升服务器的性能。服务器的“软环境”，一方面是指系统的软件资源，比如操作系统允许用户打开的最大文件描述符数量；另一方面指的是服务器程序本身，即如何从编程的角度来确保服务器的性能，这是本节要讨论的问题。
>
> ​	前面介绍的几种高效的事件处理模式和并发模式，以及高效的逻辑处理方式——有限状态机，它们都有助于提高服务器的整体性能。下面我们进一步分析高性能服务器需要注意的其他几个方面：池、数据复制、上下文切换和锁。

### 3.7.1 池

> ​	既然服务器的硬件资源“充裕”，那么提高服务器性能的一个很直接的方法就是以空间换时间，即“浪费”服务器的硬件资源，以换取其运行效率。这就是池（pool）的概念。池是一组资源的集合，这组资源在服务器启动之初就被完全创建号并初始化，这称为静态资源分配。当服务器进入正式运行阶段，即开始处理客户请求的时候，如果它需要相关的资源，就可以直接从池中获取，无须动态分配。很显然，直接从池中取得所需资源比动态分配资源的速度要快得多，因为分配系统资源的系统调用都是很耗时的。当服务器处理完一个客户连接后，可以把相关的资源返回池中，无须执行系统调用来释放资源。从最终的效果来看，池相当于服务器管理系统资源的应用层设施，它避免了服务器对内核的频繁访问。
>
> ​	不过，既然池中的资源是预先静态分配的，我们就无法预期应该分配多少资源。这个问题又该如何解决呢？最简单的解决方案就是分配“足够多”的资源，即针对每个可能的客户连接都分配必要的资源。这通常会导致资源的浪费，因为任一时刻的客户数量都可能远远没有达到服务器支持的最大客户数量。好在这种资源的浪费对服务器来说一般不会构成问题。还有一种解决方案是预先分配一定的资源，此后如果发现资源不够用，就再动态分配一些并加入池中。
>
> ​	根据不同的资源类型，池可以分为多种，常见的有内存池、进程池、线程池和连接池。它们的含义都很明确。
>
> ​	内存池通常用于socket的接收缓存和发送缓存。对于某些长度有限的客户请求，比如HTTP请求，预先分配一个大小足够（比如5000字节）的接收缓存区是很合理的。当客户请求的长度超过接收缓存区的大小时，我们可以选择丢弃请求或动态扩大接收缓存区。
>
> ​	进程池和线程池都是并发编程常用的“伎俩”。当我们需要一个工作进程或工作线程来处理新到来的客户请求时，我们可以直接从进程池或线程池中取得一个执行实体，而无须动态地调用fork或pthread_create等函数来创建进程和线程。
>
> ​	连接池通常用于服务器或服务器机群的内部永久连接。图8-4中，每个逻辑单元可能都需要频繁地访问本地的某个数据库。简单的做法是：逻辑单元每次需要访问数据库的时候，就向数据库程序发起连接，而访问完毕后释放连接。很显然，这种做法的效率太低。一种解决方案是使用连接池。连接池是服务器预先和数据库程序建立的一组连接的集合。当某个逻辑单元需要访问数据库时，它可以直接从连接池中取得一个连接的实体并使用之。待完成数据库的访问之后，逻辑单元再将该连接返还给连接池。

### 3.7.2 数据复制

> ​	高性能服务器应该避免不必要的数据复制，尤其是当数据复制发生在用户代码和内核之间的时候。如果内核可以直接处理从socket或者文件读入的数据，则应用程序就没必要将这些数据从内核缓冲区复制到应用程序缓冲区。这里说的“直接处理”指的是应用程序不关心这些数据的内容，不需要对它们做任何分析。比如ftp服务器，当客户请求一个文件时，服务器只需要检测目标文件是否存在，以及客户是否有读取它的权限，而绝对不会关心文件的具体内容。这样的话，ftp服务器就无须把目标文件的内容完整地读入应用程序缓冲区中并不调用send函数来发送，而是可以使用“零拷贝”函数sendfile来直接将其发送给客户端。
>
> ​	此外，用户代码内部（不访问内核）的数据复制也是应该避免的。举例来说，当两个工作进程之间要传递大量数据时，我们就应该考虑使用共享内存来在它们之间直接共享这些数据，而不是使用管道或者消息队列来传递。又比如代码清单8-3所示的解析HTTP请求的实例中，我们用指针（start_line）来指出每个行在buffer中的起始位置，以便随后对行内容进行访问，而不是把行的内容复制到另外一个缓冲区中来使用，因为这样既浪费空间，又效率低下。

### 3.7.3 上下文切换和锁

> ​	并发程序必须考虑上下文切换（context switch）的问题，即进程切换或线程切换导致的系统开销。即使I/O密集型的服务器，也不应该使用过多的工作线程（或工作进程，下同），否则线程间的切换将占用大量的CPU时间，服务器真正用于处理业务逻辑的CPU时间的比重就显得不足了。因此，为每个客户连接都创建一个工作线程的服务器模型是不可取的。图8-11所描述的半同步/半异步模式是一种比较合理的解决方案，它允许一个线程同时处理多个客户连接。此外，多线程服务器的一个优点是不同的线程可以同时运行在不同的CPU上。当线程的数量不大于CPU数目时，上下文切换就不是问题了。
>
> ​	并发程序需要考虑的另外一个问题是共享资源的加锁保护。锁通常被认为是导致服务器效率低下的一个因素，因为由它引入的代码不仅不处理任何业务逻辑，而且需要访问内核资源。因此，服务器如果有更好的解决方案，就应该避免使用锁。显然，图8-11所描述的半同步/半异步模式就比图8-10所描述的半同步/半反应堆模式的效率高。如果服务器必须使用“锁”，则可以考虑减少锁的粒度，比如使用读写锁。当所有工作线程都只读取一块共享内存的内容时，读写锁并不会增加系统的额外开销。只有当其中某一个工作线程需要写这块内存时，系统才必须去锁住这块区域。

# 4 I/O复用

> I/O复用使得程序能同时监听多个文件描述符，这对程序的性能至关重要。通常，网络程序在下列情况下需要使用I/O复用技术：
>
> * 客户端程序要同时处理多个socket。比如本章将要讨论的非阻塞connect技术。
>
> * 客户端程序要同时处理用户输入和网络连接。比如本章将要讨论的聊天程序。
>
> * TCP服务器要同时处理监听socket和连接socket。这是I/O复用使用最多的场合。
>
> * 服务器要同时处理TCP请求和UDP请求。比如本章要讨论的回射服务器。
>
> * 服务器要同时监听多个端口，或者处理多种服务。比如本章将要讨论的xinetd服务器。
>
>   需要指出的是，<font color=orange>**I/O复用虽然能同时监听多个文件描述符，但它本身是阻塞的。并且当多个文件描述符同时就绪时，如果不采取额外的措施，程序就只能按顺序依次处理其中的每一个文件描述符**</font>，这使得服务器程序看起来像是串行工作的。如果要实现并发，只能使用多进程或多线程等编程手段。
>
>   Linux下实现I/O复用的系统调用主要有<font color=green>**select、poll和epoll**</font>，本章将依次讨论之。

## 4.1 select系统调用

> ​	select系统调用的用途是：在一段指定时间内，监听感兴趣的文件描述符上的可读、可写和异常等事件。本节先介绍select系统调用的API，然后讨论select判断select判断文件描述符就绪的条件，最后给出他在处理带外数据中的实际应用。

### 4.1.1 select API

> select系统调用的原型如下：
>
> ```c
> #include <sys/select.h>
> int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
> ```
>
> ​	1)nfds参数指定被监听的文件描述符的总数。它通常被设置为select监听的所有文件描述符中的最大值加1，因为文件描述符是从0开始计数的。
>
> ​	2）readfds、writefds和exceptfds参数分别指向可读、可写和异常等事件对应的文件描述符集合。应用程序调用select函数时，通过这3个参数传入自己感兴趣的文件描述符。**select调用返回时，内核将修改它们来通知应用程序哪些文件描述符已经就绪**。这3个参数是fd_set结构指针类型。fd_set结构体的定义如下：
>
> ```c
> #include <typesizes.h>
> #define __FD_SETSIZE 1024
> 
> #include <sys/seleclt.h>
> #define FD_SETSIZE __FD_SETSIZE
> typedef long int __fd_mask;
> #undef __NFDBITS
> #define __NFDBITS (8 * (int) sizeof(__fd_mask))
> typedef struct
> {
>  #ifdef __USE_XOPEN
>  	__fd_mask fds_bits[__FD_SETSIZE/__NFDBITS];
>  #else
>  	__fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
>  #define __FDS_BITS(set) ((set)->__fds_bits)
>  #endif
> } fd_set;
> ```
>
> ​	由以上定义可见，fd_set结构体仅包含一个整型数组，该数组的每个元素的每一位（bit）标记一个文件描述符。fd_set能容纳的文件描述符有FD_SETSIZE指定，这就限制了select能同时处理的文件描述符的总量。
>
> ​	由于位操作过于繁琐，我们应该使用下面的一系列宏来访问fd_set结构体中的位：
>
> ```c
> #include <sys/select.h>
> FD_ZERO(fd_set *fdset);		/* 清除fdset的所有位 */
> FD_SET(int fd, fd_set *fdset);	/* 设置fdset的位fd */
> FD_CLR(int fd, fd_set *fdset);	/* 清除fdset的位fd */
> int FD_ISSET(int fd, fd_set *fdset);	/* 测试fdset的位fd是否被设置 */
> ```
>
> ​	3）timeout参数用来设置select函数的超时时间。它是一个timeval结构类型的指针，**采用指针参数是因为内核将修改它以告诉应用程序select等待了多久**。不过我们不能完全信任select调用返回后的timeout值，比如调用失败时timeout值是不确定的。timeval结构体的定义如下：
>
> ```c
> struct timeval
> {
> 	long tv_sec;	/* 秒数 */
> 	long tv_usec;	/* 微秒数 */
> };
> ```
>
> ​	由以上定义可见，select给我们提供了一个微妙级的定时方式。如果给timeout变量的tv_sec成员和tv_usec成员都传递0，则select将立即返回。如果给timeout传递NULL，则select将一直阻塞，直到某个文件描述符就绪。
>
> ​	select成功时返回就绪（可读、可写和异常）文件描述符总数。如果在超时时间内没有任何文件描述符就绪，select将返回0，select失败时返回-1并设置errno。如果在select等待期间，程序接收到信号，则select立即返回-1，并设置errno为EINTR。

### 4.1.2 文件描述符就绪条件

> ​	哪些情况下文件描述符可以被认为是可读、可写或者出现异常，对于select的使用非常关键。在网络编程中，
>
> 下列情况下socket可读：
>
> * socket内核接收缓存区中的字节数大于或等于其低水位标记SO_RCVLOWAT。此时我们可以无阻塞地读该socket,并且读操作返回的字节书大于0。
> * socket通信的对方关闭连接。此时对该socket的读操作将返回0.
> * 监听socket上有新的连接请求。
> * **socket上有未处理的错误。此时我们可以使用getsocketopt来读取和清除该错误。**
>
> 下列情况下socket可写：
>
> * socket内核发送缓冲区中的可用字节数大于或等于其低水位标记SO_SNDLOWAT.此时我们可以无阻塞地写该socket，并且写操作返回的字节数大于0.
>
> * socket的写操作被关闭。对写操作被关闭的socket执行写操作将触发一个SIGPIPE信号。
>
> * socket使用非阻塞connect连接成功或者失败（超时）之后。
>
> * socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。
>
>   网络程序中，select能处理的异常情况只有一种：socket上接收到带外数据。下面我们详细讨论之。

### 4.1.3 处理带外数据

> socket上接收到普通数据和带外数据都将使select返回，但socket处于不同的就绪状态：前者处于可读状态，后者处于异常状态。以下代码描述了select是如何同时处理二者的。
>
> ```c
> // test_select_server.c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <libgen.h>
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd > 0);
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     struct sockaddr_in client_address;
>     socklen_t client_addrlength = sizeof(client_address);
>     int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
> 
>     if(connfd < 0)
>     {
>         printf("errno is: %d\n", errno);
>         close(listenfd);
>         return 0;
>     }
> 
>     char buf[1024];
>     fd_set read_fds;
>     fd_set exception_fds;
>     FD_ZERO(&read_fds);
>     FD_ZERO(&exception_fds);
> 
>     while(1)
>     {
>         memset(buf, '\0', sizeof(buf));
>         /* 每次调用select前都要重新在read_fds和exception_fds中设置文件描述符connfd，因为事件发生之后，文件描述符集合将被内核修改 */
>         FD_SET(connfd, &read_fds);
>         FD_SET(connfd, &exception_fds);
>         ret = select(connfd + 1, &read_fds, NULL, &exception_fds, NULL);
>         if(ret < 0)
>         {
>             printf("selection failure\n");
>             break;
>         }
> 
>         /* 对于可读事件，采用普通的recv函数读取数据 */
>         if(FD_ISSET(connfd, &read_fds))
>         {
>             ret = recv(connfd, buf, sizeof(buf)-1, 0);
>             if(ret <= 0)
>             {
>                 break;
>             }
>             printf("get %d bytes of normal data: %s\n", ret, buf);
>         }
>         /* 对于异常事件，采用带MSG_OOB标志的recv函数读取带外数据 */
>         else if(FD_ISSET(connfd, &exception_fds))
>         {
>             ret = recv(connfd, buf, sizeof(buf) - 1, MSG_OOB);
>             if(ret <= 0)
>             {
>                 break;
>             }
>             printf("get %d bytes of oob data: %s\n", ret, buf);
>         }
>     }
>     close(connfd);
>     close(listenfd);
>     return 0;
> }
> ```
>
> ```c
> // test_select_client.c
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <string.h>
> #include <stdlib.h>
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
> 
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     struct sockaddr_in server_address;
>     bzero(&server_address, sizeof(server_address));
>     server_address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &server_address.sin_addr);
>     server_address.sin_port = htons(port);
> 
>     int sockfd = socket(AF_INET, SOCK_STREAM, 0);
>     assert(sockfd > 0);
>     if(connect(sockfd, (struct sockaddr*)&server_address, sizeof(server_address)) < 0)
>     {
>         printf("connect failed\n");
>     }
>     else
>     {
>         const char* oob_data = "b";
>         const char* normal_data = "123";
>         // send(sockfd, normal_data, strlen(normal_data), 0);
>         send(sockfd, oob_data, strlen(oob_data), MSG_OOB);
>         //send(sockfd, normal_data, strlen(normal_data), 0);
>     }
>     close(sockfd);
>     return 0;
> }
> ```

## 4.2 poll 系统调用

> ​	poll系统调用和select类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有就绪者。poll的原型如下：
>
> ```c
> #include <poll.h>
> int poll(struct pollfd* fds, nfds_t, int timeout);
> ```
>
> ​	1）fds参数是一个pollfd结构类型的数组，它指定所有我们感兴趣的文件描述符上发生的可读、可写和异常等事件。pollfd结构体如下：
>
> ```c
> struct pollfd
> {
> 	int fd;	/* 文件描述符 */
> 	short events;	/* 注册的事件 */
> 	short revents;	/* 实际发生的事件，由内核填充 */
> }
> ```
>
> ​	其中fd成员指定文件描述符；events成员告诉poll监听fd上的哪些事件，它是一系列事件的按位或；revents成员则由内核修改，以通知应用程序fd上实际发生了哪些事件。poll支持的事件类型如下表所示：
>
> | 事件       | 描述                                                        | 是否可作为输入 | 是否可作为输出 |
> | ---------- | ----------------------------------------------------------- | -------------- | -------------- |
> | POLLIN     | 数据（包括普通数据和优先数据）可读                          | 是             | 是             |
> | POLLRDNORM | 普通数据可读                                                | 是             | 是             |
> | POLLRDBAND | 优先级带数据可读（Linux不支持）                             | 是             | 是             |
> | POLLPRI    | 高优先级数据可读，比如TCP带外数据                           | 是             | 是             |
> | POLLOUT    | 数据（包括普通数据和优先数据）可写                          | 是             | 是             |
> | POLLWRNORM | 普通数据可写                                                | 是             | 是             |
> | POLLWRBAND | 优先级数据可写                                              | 是             | 是             |
> | POLLRDHUP  | TCP连接被对方关闭，或者对方关闭了写操作。它由GNU引入        | 是             | 是             |
> | POLLERR    | 错误                                                        | 否             | 是             |
> | POLLHUP    | 挂起。比如管道的写端被关闭后，读端描述符上将收到POLLHUP事件 | 否             | 是             |
> | POLLNVAL   | 文件描述符没有打开                                          | 否             | 是             |
>
> ​	表中，POLLRDNORM、POLLRDBAND、POLLWRNORM、POLLWRBAND由XOPEN规范定义。它们实际上是POLLIN事件和POLLOUT事件分得更细致，以区别对待普通数据和优先数据。但Linux并不完全支持它们。
>
> ​	通常，应用程序需要根据recv调用的返回值来区分socket上接收的是有效数据还是对方关闭连接的请求，并做相应的的处理。不过，自Linux内核2.6.17开始，GNU为poll系统调用增加了一个POLLRDHUP事件，它在socket上接收到对方关闭连接的请求之后触发。这为我们区分上述两种情况提供了一种更简单的方式。但使用POLLRDHUP事件时，我们需要在代码最开始处定义_GNU_SOURCE。
>
> ​	2)nfds参数指定被监听事件集合fds的大小。其类型nfds_t的定义如下：
>
> ```c
> typedef unsigned long int nfds_t;	
> ```
>
> ​	3）timeout参数指定poll的超时值，单位是毫秒。**当timeout为-1时，poll调用将永远阻塞，直到某个事件发生；当timeout为0时，poll调用将立即返回。**
>
> ​	poll系统调用的返回值的含义与select相同。

## 4.3 epoll序列系统调用

### 4.3.1 内核事件表

> ​	epoll是Linux特有的I/O复用函数。它在实现和使用上与select、poll有很大差异。首先，epoll使用一组函数来完成任务，而不是单个函数。其次，epoll把用户关心的文件描述符上的事件放在内核的一个事件表中，**从而无须像select和poll那样每次调用都要重复传入文件描述符集或事件集**。**但epoll需要使用一个额外的文件描述符，来唯一标识内核中的这个事件表。**这个文件描述符使用如下epoll_create函数来创建：
>
> ```c
> #include <sys/epoll.h>
> int epoll_create(int size);
> ```
>
> ​	size参数现在并不起作用，只是给内核一个提示，告诉它事件表需要多大。**该函数返回的文件描述符将用作其他所有epoll系统调用的第一个参数，以指定要访问的内核事件表。**
>
> ​	下面的函数用来操作epoll的内核事件表：
>
> ```c
> #include <sys/epoll.h>
> int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
> ```
>
> fd参数是要操作的文件描述符，op参数则指定操作类型。操作类型有如下3种：
>
> * EPOLL_CTL_ADD，往事件表中注册fd上的事件
> * EPOLL_CTL_MOD，修改fd上的注册事件
> * EPOLL_CTL_DEL，删除fd上的注册事件。
>
> event参数指定事件，它是epoll_event结构指针类型。epoll_event的定义如下：
>
> ```c
> struct epoll_event
> {
> 	__uint32_t events;	/* epoll事件 */
> 	epoll_data_t data;	/* 用户数据 */
> };
> ```
>
> ​	其中events成员描述事件类型。epoll支持的事件类型和poll基本相同。表示epoll事件类型的宏是在poll对应的宏上加上“E”，比如epoll的数据可读事件时EPOLLIN。但epoll有两个额外的事件类型——**EPOLLLET和EPOLLONESHOT**。它们对于epoll的高效运作非常关键，我们将在后面讨论它们。data成员用于存储用户数据，其类型epoll_data_t的定义如下：
>
> ```c
> typedef union epoll_data
> {
> 	void* ptr;
> 	int fd;
> 	uint32_t u32;
> 	uint64_t u64;
> } epoll_data_t;
> ```
>
> ​	epoll_data_t是一个联合体，其4个成员中使用最多的是fd，它指定事件从属的目标文件描述符。ptr成员可用来指定与fd相关的用户数据。但由于epoll_data_t是一个联合体，我们不能同时使用其ptr成员和fd成员，因此，如果要将文件描述符和用户数据关联起来，以实现快速的数据访问，只能使用其他手段，比如放弃使用epoll_data_t的fd成员，而在ptr指向的用户数据中包含fd。
>
> ​	epoll_ctl成功时返回0，失败返回-1并设置errno。

### 4.3.2 epoll_wait函数

> ​	epoll系列系统调用的主要接口是epoll_wait函数。它在一段超时时间内等待一组文件描述符上的事件，其原型如下：
>
> ```c
> #include <sys/epoll.h>
> int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
> ```
>
> ​	该函数成功时返回就绪的文件描述符的个数，失败时返回-1并设置errno。
>
> ​	timeout参数的含义与poll接口的timeout参数相同。maxevents参数指定最多监听多少个事件，它必须大于0。
>
> ​	epoll_wait函数如果检测到事件，就将所有就绪的事件从内核事件表（由epfd参数指定）复制到它的第二个参数events指向的数组中。**这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数那样既用于传入用户注册的事件，又用于输出内核检测到的就绪事件。**这就极大地提高了应用程序索引就绪文件描述符的效率。以下代码体现了这个差别：
>
> ```c
> /* 如何索引poll返回的就绪文件描述符 */
> int ret = poll(fds, MAX_EVENT_NUMBER, -1);
> /* 必须遍历所有已注册文件描述符并找到其中就绪者（当然，可以利用ret来稍做优化）*/
> for(int i = 0; i < MAX_EVENT_NUMBER; ++i)
> {
>     if(fds[i].revents & POOLIN)	/* 判断第i个文件描述符是否就绪 */
>     {
>         int sockfd = fds[i].fd;
>         /* 处理sockfd */
>     }
> }
> 
> /* 如何索引epoll返回的就绪文件描述符 */
> int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
> /* 仅遍历就绪的ret个文件描述符 */
> for(int i = 0; i < ret; i++)
> {
>     int sockfd = events[i].data.fd;
>     /* sockfd 肯定就绪，直接处理 */
> }
> ```

### 4.3.3 LT和ET模式

> ​	epoll对文件描述符的操作有两种模式：LT（Level Trigger，电平触发）模式和ET（Edge Trigger，边沿触发）模式。LT模式是默认的工作模式，这种模式下epoll相当于一个效率较高的poll。当往epoll内核事件表中注册一个文件描述符上的EPOLLET事件时，epoll将以ET模式来操作该文件描述符。ET模式是epoll的高效工作模式。
>
> ​	对于采用LT工作模式的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序可以不立即处理该事件。这样，当应用程序下一次调用epoll_wait时，epoll_wait还会再次向应用程序通告此事件，直到该事件被处理。而对于采用ET工作模式的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理该事件，因为后续的epoll_wait调用将不再向应用程序通知这一事件。可见，ET模式在很大程度上降低了同一个epoll事件被重复触发的次数，因此效率要比LT高。以下代码体现了LT和ET在工作方式上的差异。
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <pthread.h>
> #include <libgen.h>
> #include <stdbool.h>
> 
> #define MAX_EVENT_NUMBER 1024
> #define BUFFER_SIZE 10
> 
> /* 将文件描述符设置成非阻塞的 */
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, new_option);
>     return old_option;
> }
> 
> /* 将文件描述符fd上的EPOLLIN祖册到epollfd指示的epoll内核事件表中，参数enable_et指定是否对fd启用ET模式 */
> void addfd(int epollfd, int fd, bool enable_et)
> {
>     struct epoll_event event;
>     event.data.fd = fd;
>     if(enable_et)
>     {
>         event.events |= EPOLLET;
>     }
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> 
> /* LT模式的工作流程 */
> void lt(struct epoll_event* events, int number, int epollfd, int listenfd)
> {
>     char buf[BUFFER_SIZE];
>     for(int i = 0; i < number; i++)
>     {
>         int sockfd = events[i].data.fd;
>         if(sockfd == listenfd)
>         {
>             struct sockaddr_in client_address;
>             socklen_t client_addrlength = sizeof(client_address);
>             int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>             addfd(epollfd, connfd, false); /* 对connfd 禁用ET模式 */
>         }
>         else if(events[i].events & EPOLLIN)
>         {
>             /* 只要socket读缓存中还有未读出的数据，这段代码就被触发 */
>             printf("event trigger once\n");
>             memset(buf, '\0', BUFFER_SIZE);
>             int ret = recv(sockfd, buf, BUFFER_SIZE-1, 0);
>             if(ret <= 0)
>             {
>                 close(sockfd);
>                 continue;
>             }
>             printf("get %d bytes of content: %s", ret, buf);
>         }
>         else
>         {
>             printf("something else happened \n");
>         }
>     }
> }
> 
> /* ET 模式的工作流程 */
> void et(struct epoll_event* events, int number, int epollfd, int listenfd)
> {
>     char buf[BUFFER_SIZE];
>     for(int i = 0; i < number; i++)
>     {
>         int sockfd = events[i].data.fd;
>         if(sockfd == listenfd)
>         {
>             struct sockaddr_in client_address;
>             socklen_t client_addrlength = sizeof(client_address);
>             int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>             addfd(epollfd, connfd, true); /* 对connfd开启ET模式 */
>         }
>         else if(events[i].events & EPOLLIN)
>         {
>             /* 这段代码不会被重复触发，所以我们循环读取数据，以确保把socket读缓存中的所有数据读出 */
>             printf("event trigger once\n");
>             while(1)
>             {
>                 memset(buf, '\0', BUFFER_SIZE);
>                 int ret = recv(sockfd, buf, BUFFER_SIZE-1, 0);
>                 if(ret < 0)
>                 {
>                     /* 对于非阻塞IO, 下面的条件成立表示数据已经全部读取完毕。此后，epoll
>                     就能再次触发sockfd上的EPOLLIN事件，以驱动下一次读操作*/
>                     if((errno == EAGAIN) || (errno == EWOULDBLOCK))
>                     {
>                         printf("read later\n");
>                         break;
>                     }
>                     close(sockfd);
>                     break;
>                 }
>                 else
>                 {
>                     printf("get %d bytes of content: %s\n", ret, buf);
>                 }
>             }
>         }
>         else
>         {
>             printf("something else happened \n");
>         }
>     }
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
> 
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     struct epoll_event events[MAX_EVENT_NUMBER];
>     int epollfd = epoll_create(5);
>     assert(epollfd != -1);
>     addfd(epollfd, listenfd, true);
> 
>     while(1)
>     {
>         int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
>         if(ret < 0)
>         {
>             printf("epoll failure\n");
>             break;
>         }
> 
>         lt(events, ret, epollfd, listenfd); /* 使用LT模式 */
>         // et(events, ret, epollfd, listenfd); /* 使用ET模式 */
>     }
> 
>     close(listenfd);
>     return 0;
> }
> ```
>
> ​	运行这段代码，然后telnet到这个服务器程序上并一次传输超过10字节（BUFFER_SIZE的大小）的数据，然后比较LT模式和ET模式的异同。你将会发现，正如我们预期的，ET模式下事件被触发的次数要比LT模式下少很多。
>
> ​	每个使用ET模式的文件描述符都应该是非阻塞的。如果文件描述符是阻塞的，那么读或写操作会因为没有后续的事件而一直处于阻塞状态。

### 9.3.4 EPOLLONESHOT事件

> ​	即使我们使用ET模式，一个socket上的某个事件还是可能被多次触发。这在并发程序中就会引起一个问题。比如一个线程（或进程，下同）在读取完某个socket上的数据后开始处理这些数据，而在数据的处理过程中该socket上又有新的数据可读（EPOLLIN再次被触发），此时另外一个线程被唤醒来读取这些新的数据。于是就出现了两个线程同时操作一个socket的局面。这当然不是我们期望的。我们期望的是一个socket连接在任一时刻都只被一个线程处理。这一点使用epoll的EPOLLONESHOT事件实现。
>
> ​	对于注册了EPOLLONESHOT事件的文件描述符，操作系统最多触发其上注册的一个可读、可写或者异常事件，且只触发一次，除非我们使用epoll_ctl重置该文件描述符上注册的EPOLLONESHOT事件。这样，当一个线程在处理某个socket时，其它线程是不可能有机会操作该socket的。但反过来思考，注册了EPOLLONESHOT事件的socket一旦被某个线程处理完毕，该线程就应该立即重置这个socket上的EPOLLONESHOT事件，以确保这个socket下一次可读时，其EPOLLIN事件能被触发，进而让其他的工作线程有机会继续处理这个socket。
>
> ​	以下代码展示了EPOLLONESHOT事件的使用：
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <pthread.h>
> #include <stdbool.h>
> #include <libgen.h>
> #include <fcntl.h>
> 
> #define MAX_EVENT_NUMBER 1024
> #define BUFFER_SIZE 1024
> 
> struct fds
> {
>     int epollfd;
>     int sockfd;
> };
> 
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> /* 将fd上的EPOLLIN和EPOLLET事件注册到epollfd指示的epoll内核事件表中，参数oneshot指定是否注册fd
> 上的EPOLLONESHOT事件 */
> void addfd(int epollfd, int fd, bool oneshot)
> {
>     struct epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET;
>     if(oneshot)
>     {
>         event.events |= EPOLLONESHOT;
>     }
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> 
> /* 重置fd上的事件。这样操作之后，尽管fd上的EPOLLONESHOT事件被注册，但是操作系统仍然会触发fd上的EPOLLIN事件，且只触发一次 */
> void reset_oneshot(int epollfd, int fd)
> {
>     struct epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
>     epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
> }
> 
> /* 工作线程 */
> void* worker(void* arg)
> {
>     int sockfd = ((struct fds*)arg)->sockfd;
>     int epollfd = ((struct fds*)arg)->epollfd;
>     printf("start new thread to receive data on fd: %d\n", sockfd);
>     char buf[BUFFER_SIZE];
>     memset(buf, '\0', BUFFER_SIZE);
>     /* 循环读取sockfd上的数据，直到遇到EAGAIN错误 */
>     while(1)
>     {
>         int ret = recv(sockfd, buf, BUFFER_SIZE-1, 0);
>         if(ret == 0)
>         {
>             close(sockfd);
>             printf("foreiner closed the connection\n");
>             break;
>         }
>         else if(ret < 0)
>         {
>             if(errno == EAGAIN)
>             {
>                 reset_oneshot(epollfd, sockfd);
>                 printf("read later\n");
>                 break;
>             }
>         }
>         else
>         {
>             printf("get content: %s\n", buf);
>             /* 休眠5s, 模拟数据处理工程 */
>             sleep(5);
>         }
>     }
>     printf("end thread receiving data on fd: %d\n", sockfd);
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     struct epoll_event events[MAX_EVENT_NUMBER];
>     int epollfd = epoll_create(5);
>     assert(epollfd != -1);
>     /* 注意，监听socket listenfd上是不能注册EPOLLONESHOT事件的，否则应用程序只能处理一个客户连接！因为后续的客户连接请求将不再触发
>     listenfd上的EPOLLIN事件 */
>     addfd(epollfd, listenfd, false);
> 
>     while(1)
>     {
>         int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
>         if(ret < 0)
>         {
>             printf("epoll failure\n");
>             break;
>         }
> 
>         for(int i = 0; i < ret; i++)
>         {
>             int sockfd = events[i].data.fd;
>             if(sockfd == listenfd)
>             {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
>                 int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                 /* 对每个监听文件描述符都注册EPOLLONESHOT事件 */
>                 addfd(epollfd, connfd, true);
>             }
>             else if(events[i].events & EPOLLIN)
>             {
>                 pthread_t thread;
>                 struct fds fds_for_new_worker;
>                 fds_for_new_worker.epollfd = epollfd;
>                 fds_for_new_worker.sockfd = sockfd;
>                 /* 新启动一个工作线程sockfd服务 */
>                 pthread_create(&thread, NULL, worker, (void*)&fds_for_new_worker);
>             }
>             else
>             {
>                 printf("something else happened \n");
>             }
>         }
>         close(listenfd);
>         return 0;
>     }
> }
> ```
>
> ​	从工作线程函数worker来看，如果一个工作线程处理完某个socket上的一次请求（我们用休眠5秒来模拟这个过程）之后，又接收该socket上新的客户请求，则线程将继续为这个socket服务。并且因为该socket上注册了EPOLLONESHOT事件，其他线程没有机会接触这个socket，如果工作线程等待5s秒后仍然没有收到该socket上的下一批客户数据，则它将放弃为该socket服务。同时，它调用reset_oneshot函数来重置该socket上的注册事件，这将使epoll有机会再次检测到该socket上的EPOLLIN事件，进而使得其他线程有机会为该socket服务。
>
> ​	由此看来，尽管一个socket在不同时间可能被不同的线程处理，但同一时刻肯定只有一个线程在为它服务。这就保证了连接的完整性，从而避免了很多可能的竞态条件。

## 4.4 三组I/O复用函数的比较

> ​	前面我们讨论了select、poll、和epoll三组I/O复用系统调用，这3组系统调用都能同时监听多个文件描述符。它们将等待由timeout参数指定的超时时间，直到一个或者多个文件描述符上有事件发生时返回，返回值是就绪的文件描述符的数量。返回0表示没有事件发生。现在我们从事件集、最大支持文件描述符数、工作模式和具体实现等四个方面进一步比较它们的异同，以明确在实际应用中应该选择使用哪个（或哪些）。
>
> ​	这3组函数都通过某种结构体变量来告诉内核监听哪些文件描述符上的哪些事件，并使用该结构体类型的参数来获取内核处理的结果。select的参数类型fd_set没有将文件描述符和时间绑定，它仅仅是一个文件描述符集合，因此select需要提供3个这种类型的参数来分别传入和输出可读、可写及异常等事件。这一方面使得select不能处理更多类型的事件，另一方面由于内核对fd_set集合的在线修改，应用程序下次调用select前不得不重置这3个fd_set集合。poll的参数类型pollfd则多少“聪明”一些。它把文件描述符和时间都定义其中，任何事件都被统一处理，从而使得编程接口简洁得多。并且内核每次修改的是pollfd结构体的revents成员，而events成员保持不变，因此下次调用poll时应用程序无须重置pollfd类型的事件参数。由于每次select和poll调用都返回整个用户注册的事件集合（其中包括就绪和未就绪的），所以应用程序索引就绪文件描述符的时间复杂度为O(n)。epoll则采用与select和poll完全不同的方式来管理用户注册的事件。它在内核中维护一个事件表，并提供了一个独立的系统调用epoll_ctl来控制往其中添加、删除、修改事件。这样，每次epoll_wait调用都直接从该内核事件表中取得用户注册的事件，而无须反复从用户空间读入这些事件。epoll_wait系统调用的events参数仅用来返回就绪的事件，这使得应用程序索引就绪文件描述符的时间复杂度为O(1)。
>
> ​	poll和epoll_wait分别用nfds和maxevens参数指定最多监听多少个文件描述符和事件。这两个数值都能达到系统允许打开的最大文件描述符数目，即65535（cat /proc/sys/fs/filemax）。而select允许监听的最大文件描述符数量通常有限制。虽然用户可以修改这个限制，但这可能导致不可预期的后果。
>
> ​	select和poll都只能工作在相对低效的LT模式，而epoll则可以工作在ET高效模式。并且epoll还支持EPOLLONESHOT事件。该事件能进一步减少可读、可写和异常等事件被触发的次数。
>
> ​	从实现原理上来说，**select和poll采用的都是轮询的方式，**即每次调用都要扫描整个注册文件描述符集合，并将其中就绪的文件描述符返回给用户程序，因此它们检测就绪事件的算法时间复杂度是O(n)。epoll_wait则不同，它采用的是回调方式。**内核检测到就绪的文件描述符时，将触发回调函数，回调函数就将该文件描述符上对应的事件插入内核就绪队列**。内核最后在适当的时机将该就绪事件队列中的内容拷贝到用户空间。因此epoll_wait无须轮询整个文件描述符集合来检测哪些事件已经就绪，其算法时间复杂度是O(1)。但是，当活动连接比较多的时候，epoll_wait的效率未必比select和poll高，因为此时回调函数被触发得过于频繁。**所以epoll_wait使用于连接数量多，但活动连接少的情况。**
>
> ![image-20230227171441683](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230227171441683.png)

## 4.5 I/O复用的高级应用一：非阻塞connect

> ​	connect系统调用的man手册有如下一段内容：
>
> ![image-20230227210820205](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230227210820205.png)
>
> ​	这段话描述了connect出错时的一种errno值：EINPROGRESS。这中错误发生在对非阻塞的socket调用connect，而连接又没有建立时。根据man文档的解释，在这种情况下，我们可以调用select、poll等函数来监听这个连接失败的socket上的可写事件。当select、poll等函数返回后，再利用getsockopt来读取错误码并清除该socket上的错误。如果错误码是0，表示连接建立成功，否则失败。
>
> 通过上面描述的非阻塞connect方式，我们就能同时发起多个连接并一起等待。下面看看非阻塞connect的一种实现，代码如下：
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <stdlib.h>
> #include <assert.h>
> #include <stdio.h>
> #include <time.h>
> #include <errno.h>
> #include <fcntl.h>
> #include <sys/ioctl.h>
> #include <unistd.h>
> #include <string.h>
> 
> #define BUFFER_SIZE 1023
> 
> int setnonblocking(int fd)
> {
>     int old_option = fnctl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> /* 超时连接函数，参数分别是服务器IP地址、端口号和超时时间（毫秒）。函数成功时返回已经处于连接状态的socket，失败则返回-1 */
> int unblock_connect(const char* ip, int port, int time)
> {
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int sockfd = socket(PF_INET, SOCK_STREAM, 0);
>     int fdopt = setnonblocking(sockfd);
>     ret = connect(sockfd, (struct sockaddr*)&address, sizeof(address));
>     if(ret == 0)
>     {
>         /* 如果连接成功，则恢复socket的属性，并立即返回之 */
>         printf("connect with server immediately\n");
>         fcntl(sockfd, F_SETFL, fdopt);
>         return sockfd;
>     }
>     else if(errno != EINPROGRESS)
>     {
>         /* 如果连接没有立即建立，那么只有当errno是EINPROGRESS时才表示连接还在进行，否则出错返回 */
>         printf("unblock connect not support\n");
>         return -1;
>     }
>     fd_set readfds;
>     fd_set writefds;
>     struct timeval timeout;
> 
>     FD_ZERO(&readfds);
>     FD_SET(sockfd, &writefds);
> 
>     timeout.tv_sec = time;
>     timeout.tv_usec = 0;
> 
>     ret = select(sockfd + 1, NULL, &writefds, NULL, &timeout);
>     if(ret <= 0)
>     {
>         /* select超时或者出错，立即返回 */
>         printf("connection time out\n");
>         close(sockfd);
>         return -1;
>     }
>     if(! FD_ISSET(sockfd, &writefds))
>     {
>         printf("no events on sockfd found\n");
>         close(sockfd);
>         return -1;
>     }
> 
>     int error = 0;
>     socklen_t length = sizeof(error);
>     /* 调用getsockopt来获取并清除sockfd上的错误 */
>     if(getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &length) < 0)
>     {
>         printf("get socket option failed\n");
>         close(sockfd);
>         return -1;
>     }
>     /* 错误号不为0表示连接出错 */
>     if(error != 0)
>     {
>         printf("connection failed after select with error: %d\n", error);
>         return -1;
>     }
>     /* 连接成功 */
>     printf("connection ready after select with the socket: %d\n", sockfd);
>     fcntl(sockfd, F_SETFL, fdopt);
>     return sockfd;
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc < 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int sockfd = unblock_connect(ip, port, 10);
>     if(sockfd < 0)
>     {
>         return 1;
>     }
>     close(sockfd);
>     return 0;
> }
> ```

## 4.6 I/O复用的高级应用二：聊天室程序

> ​	像ssh这样的登录服务通常要同时处理网络连接和用户输入，这也可以使用I/O复用来实现。本节我们以poll为例实现一个简单的聊天程序，以阐述如何使用I/O复用技术来同时处理网络连接和用户输入。该聊天室程序能让所有用户同时在线群聊，它分别为客户端和服务器两个部分。其中客户端程序有两个功能：**一是从标准输入终端读入用户数据，并将用户数据发送至服务器；二是往标准输出终端打印服务器发送给它的数据。**服务器的功能是**接收客户数据，并把客户数据发送给每一个登录到该服务器上的客户端（数据发送者除外）**。下面我们依次给出客户端程序和服务器程序的代码：

### 4.6.1 客户端

> ​	客户端程序使用poll同时监听用户输入和网络连接，并利用splice函数将用户输入内容直接定向到网络连接上以发送之，从而实现数据零拷贝，提高了程序执行效率。客户端程序代码如下：
>
> ```c
> #define _GNU_SOURCE 1
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <string.h>
> #include <stdlib.h>
> #include <poll.h>
> #include <fcntl.h>
> 
> #define BUFFER_SIZE 64
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     struct sockaddr_in server_address;
>     bzero(&server_address, sizeof(server_address));
>     server_address.sin_amily = AF_INET;
>     inet_pton(AF_INET, ip, &server_address.sin_addr);
>     server_address.sin_port = htons(port);
> 
>     int sockfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(sockfd >= 0);
>     if(connect(sockfd, (struct sockaddr*)&server_address, sizeof(server_address)) < 0)
>     {
>         printf("connection failed\n");
>         close(sockfd);
>         return 1;
>     }
> 
>     struct pollfd fds[2];
>     /* 注册文件描述符0（标准输入）和文件描述符sockfd上的可读事件 */
>     fds[0].fd = 0;
>     fds[0].events = POLLIN;
>     fds[0].revents = 0;
>     fds[1].fd = sockfd;
>     fds[1].events = POLLIN | POLLRDHUP;
>     fds[1].revents = 0;
> 
>     char read_buf[BUFFER_SIZE];
>     int pipefd[2];
>     int ret = pipe(pipefd);
>     assert(ret != -1);
> 
>     while(1)
>     {
>         ret = poll(fds, 2, -1);
>         if(ret < 0)
>         {
>             printf("poll failure\n");
>             break;
>         }
> 
>         if(fds[1].revents & POLLRDHUP)
>         {
>             printf("server close the connection\n");
>             break;
>         }
>         else if(fds[1].revents & POLLIN)
>         {
>             memset(read_buf, '\0', BUFFER_SIZE);
>             recv(fds[1].fd, read_buf, BUFFER_SIZE-1, 0);
>             printf("%s\n", read_buf);
>         }
> 
>         if(fds[0].revents & POLLIN)
>         {
>             /* 使用splice将用户输入的数据直接写到sockfd上（零拷贝） */
>             ret = splice(0, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>             ret = splice(pipefd[0], NULL, sockfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
>         }
>     }
> 
>     close(sockfd);
>     return 0;
> }
> ```

### 4.6.2 服务器

> ​	服务器程序使用poll同时管理监听socket和连接socket，并且使用牺牲空间换取时间的策略来提高服务器性能，代码如下：
>
> ```c
> #define _GNU_SOURCE 1
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <poll.h>
> 
> #define USER_LIMIT 5 /* 最大用户数量 */
> #define BUFFER_SIZE 64 /* 读缓冲区的的大小 */
> #define FD_LIMIT 65535 /* 文件描述符数量限制 */
> /* 客户数据：客户端socket地址、待写到客户端的数据的位置、从客户端读入的数据 */
> struct client_data
> {
>     struct sockaddr_in address;
>     char* write_buf;
>     char buf[BUFFER_SIZE];
> };
> 
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
>     
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     /* 创建users数组，分配FD_LIMIT个client_data对象。可以预期：每个可能的socket连接都可以获得一个这样的对象，并且socket
>     的值可以直接用来索引（作为数组的小标）socket连接对应的client_data对象，这是将socket和客户数据关联的简单而高效的方式 */
>     struct client_data* users = new struct client_data[FD_LIMIT];
>     /* 尽管我们分配了足够多的client_data对象，但为了提高poll的性能，仍然有必要限制用户的数量 */
>     struct pollfd fds[USER_LIMIT+1];
>     int user_counter = 0;
>  
>     for(int i = 1; i <= USER_LIMIT; ++i)
>     {
>         fds[i].fd = -1;
>         fds[i].events = 0;
>     }
>     fds[0].fd = listenfd;
>     fds[0].events = POLLIN | POLLERR;
>     fds[0].revents = 0;
> 
>     while(1)
>     {
>         ret = poll(fds, user_counter+1, -1);
>         if(ret < 0)
>         {
>             printf("poll failure\n");
>             break;
>         }
> 
>         for(int i = 0; i < user_counter+1; ++i)
>         {
>             if((fds[i].fd == listenfd) && (fds[i].revents & POLLIN))
>             {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
>                 int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                 if(connfd < 0)
>                 {
>                     printf("errno is: %d\n", errno);
>                     continue;
>                 }
>                 /* 如果请求太多，则关闭新到的连接 */
>                 if(user_counter >= USER_LIMIT)
>                 {
>                     const char* info = "too many users\n";
>                     printf("%s", info);
>                     send(connfd, info, strlen(info), 0);
>                     close(connfd);
>                     continue;
>                 }
>                 /* 对于新的连接， 同时修改fds和users数组。前文已经提到，users[connfd]对应于新连接文件描述符
>                 connfd的客户数据 */
>                 user_counter++;
>                 users[connfd].address = client_address;
>                 setnonblocking(connfd);
>                 fds[user_counter].fd = connfd;
>                 fds[user_counter].events = POLLIN | POLLRDHUP | POLLERR;
>                 fds[user_counter].revents = 0;
>                 printf("comes a new user, now have %d users\n", user_counter);
>             }
>             else if(fds[i].revents & POLLERR)
>             {
>                 printf("get an error from %d\n", fds[i].fd);
>                 char errors[100];
>                 memset(errors, '\0', 100);
>                 socklen_t length = sizeof(errors);
>                 if(getsockopt(fds[i].fd, SOL_SOCKET, SO_ERROR, &errors, &length) < 0)
>                 {
>                     printf("get socket option failed\n");
>                 }
>                 continue;
>             }
>             else if(fds[i].revents & POLLRDHUP)
>             {
>                 /* 如果客户端关闭连接，则服务器也关闭对应的连接，并将用户数减1 */
>                 users[fds[i].fd] = users[fds[user_counter].fd];
>                 close(fds[i].fd);
>                 fds[i] = fds[user_counter];
>                 i--;
>                 user_counter--;
>                 printf("a client left\n");
>             }
>             else if(fds[i].revents & POLLIN)
>             {
>                 int connfd = fds[i].fd;
>                 memset(users[connfd].buf, '\0', BUFFER_SIZE);
>                 ret = recv(connfd, users[connfd].buf, BUFFER_SIZE-1, 0);
>                 printf("get %d bytes of client data %s from %d\n", ret, users[connfd].buf, connfd);
>                 if(ret < 0)
>                 {
>                     /* 如果读操作出错，则关闭连接 */
>                     if(errno != EAGAIN)
>                     {
>                         close(connfd);
>                         users[fds[i].fd] = users[fds[user_counter].fd];
>                         fds[i] = fds[user_counter];
>                         i--;
>                         user_counter--;
>                     }
>                 }
>                 else if(ret == 0)
>                 {
> 
>                 }
>                 else
>                 {
>                     /* 如果接收到客户数据，则通知其它socket连接准备写数据 */
>                     for(int j = 1; j <= user_counter; ++j)
>                     {
>                         if(fds[j].fd == connfd)
>                         {
>                             continue;
>                         }
>                         fds[j].events |= ~POLLIN;
>                         fds[j].events |= POLLOUT;
>                         users[fds[j].fd].write_buf = users[connfd].buf;
>                     }
>                 }
>             }
>             else if(fds[i].revents & POLLOUT)
>             {
>                 int connfd = fds[i].fd;
>                 if(! users[connfd].write_buf)
>                 {
>                     continue;
>                 }
>                 ret = send(connfd, users[connfd].write_buf, strlen(users[connfd].write_buf), 0);
>                 users[connfd].write_buf = NULL;
>                 /* 写完数据后需要重新注册fds[i]上的可读事件 */
>                 fds[i].events |= ~POLLOUT;
>                 fds[i].events |= POLLIN;
>             }
>         }
>     }
>     delete[] users;
>     close(listenfd);
>     return 0;
> }
> ```

## 4.7 I/O复用的高级应用三：同时处理TCP和UDP服务

> ​	至此，我们讨论过的服务器程序都只监听一个端口。在实际应用中，有不少服务器程序能同时监听多个端口，比如超级服务inetd和android的调试服务adbd。
>
> ​	从bind系统调用的参数来看，一个socket只能与一个socket地址绑定，即一个socket只能用来监听一个端口。因此，服务器如果同时监听多个端口，就必须创建多个socket，并将它们分别绑定到各个端口上。这样一来，服务器程序就需要同时管理多个监听socket，I/O复用技术就有了用武之地。另外，即使同一个端口，如果服务器要同时处理该端口上的TCP和UDP请求，则也需要创建两个不同的socket：一个是流socket，另一个是数据报socket，并将它们都绑定到该端口上。比如如下代码所示的回射服务器就能同时处理一个端口上的TCP和UDP请求。
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <pthread.h>
> 
> #define MAX_EVENT_NUMBER 1024
> #define TCP_BUFFER_SIZE 512
> #define UDP_BUFFER_SIZE 1024
> 
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> void addfd(int epollfd, int fd)
> {
>     struct epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET;
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     /* 创建TCP socket，并将其绑定到端口port上 */
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
> 
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     /* 创建UDP socket，并将其绑定到端口port上 */
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
>     int udpfd = socket(PF_INET, SOCK_DGRAM, 0);
>     assert(udpfd >= 0);
> 
>     ret = bind(udpfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     epoll_event events[MAX_EVENT_NUMBER];
>     int epollfd = epoll_create(5);
>     assert(epollfd != -1);
>     /* 注册TCP socket 和 UDP socket上的可读事件 */
>     addfd(epollfd, listenfd);
>     addfd(epollfd, udpfd);
> 
>     while(1)
>     {
>         int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
>         if(number < 0)
>         {
>             printf("epoll failure\n");
>             break;
>         }
> 
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             if(sockfd == listenfd)
>             {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
>                 int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                 addfd(epollfd, connfd);
>             }
>             else if(sockfd == udpfd)
>             {
>                 char buf[UDP_BUFFER_SIZE];
>                 memset(buf, '\0', UDP_BUFFER_SIZE);
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
> 
>                 ret = recvfrom(udpfd, buf, UDP_BUFFER_SIZE-1, 0, (struct sockaddr*)&client_address, &client_addrlength);
>                 if(ret > 0)
>                 {
>                     sendto(udpfd, buf, UDP_BUFFER_SIZE-1, 0, (struct sockaddr*)&client_address, client_addrlength);
>                 }
>             }
>             else if(events[i].events & EPOLLIN)
>             {
>                 char buf[TCP_BUFFER_SIZE];
>                 while(1)
>                 {
>                     memset(buf, '\0', TCP_BUFFER_SIZE);
>                     ret = recv(sockfd, buf, TCP_BUFFER_SIZE-1, 0);
>                     if(ret < 0)
>                     {
>                         if((errno == EAGAIN) || (errno == EWOULDBLOCK))
>                         {
>                             break;
>                         }
>                         close(sockfd);
>                         break;
>                     }
>                     else if(ret == 0)
>                     {
>                         close(sockfd);
>                     }
>                     else
>                     {
>                         send(sockfd, buf, ret, 0);
>                     }
>                 }
>             }
>             else
>             {
>                 printf("something else happened \n");
>             }
>         }
>     }
>     close(listenfd);
>     return 0;
> }
> ```

## 4.8 超级服务xinetd

> ​	Linux因特网inetd是超级服务。它同时管理着多个子服务，即监听多个端口。现在Linux系统上使用的inetd服务程序通常是其升级版本xinetd。xinetd程序的原理与inetd相同，但增加了一些控制选项，并提高了安全性。下面我们从配置文件和工作流程两个方面对xinetd进行介绍。

### 4.8.1 xinetd配置文件

> ​	xinetd采用/etc/xinetd.conf主配置文件和/etc/xinetd.d目录下的子配置文件来管理所有服务。主配置文件包含的是通用选项，这些选项将被所有子配置文件继承。不过子配置文件可以覆盖这些选项。每个子配置文件用于设置一个子服务的参数。比如，telnet子服务的配置文件/etc/xinetd.d/telnet的典型内容如下：
>
> ```shell
> # default: on
> # description: The telnet server serves telnet sessions; it uses \
> # unencrypted username/password pairs for authentication.
> service telnet
> {
> 	flags			= REUSE
> 	socket_type		= stream
> 	wait			= no
> 	user			= root
> 	server			= /user/sbin/in.telnetd
> 	log_on_faailure	+= USERID
> 	disabel			= no
> }
> ```
>
> /etc/xinetd.d/telnet文件的项目及其含义如下表：
>
> | 项目           | 含义                                                         |
> | -------------- | ------------------------------------------------------------ |
> | service        | 服务名                                                       |
> | flags          | 设置连接的标志。REUSE表示复用telnet连接的socket。该标志已经过时，每个连接都默认启用REUSE标志。 |
> | socket_type    | 服务类型                                                     |
> | wait           | 服务采用单线程方式（wait=yes)还是多线程方式（wait=no）。单线程方式表示xinetd只accept第一次连接，此后将由子服务进程来accept新连接。多线程表示xinetd一直负责accept连接，而子服务进程仅处理连接socket上的数据读写。 |
> | user           | 子服务进程将以user指定的用户身份运行                         |
> | server         | 子服务程序的完整路径                                         |
> | log_on_failure | 定义当前服务不能启动时输出日志的参数                         |
> | disable        | 是否启动该子服务                                             |
>
> ​	xinetd配置文件的内容相当丰富，远不止上面这些。读者可参考其man手册来获得更多信息。

### 4.8.2 xinetd工作流程

> ​	xinetd管理的子服务中有的是标准服务，比如时间日期服务daytime、回射服务echo和丢弃服务discard。xinetd服务器在内部直接处理这些服务。还有的子服务则需要调用外部的服务器程序来处理。xinetd通过调用fork和exec函数来加载运行这些服务器程序。比如telnet、ftp服务都是这种类型的子服务。我们仍以telnet服务为例来探讨xinetd的工作流程。
>
> ​	首先，查看xinetd守护进程的PID：
>
> ```shell
> $ cat /var/run/xinetd.pid
> 9543
> ```
>
> ​	然后开启两个终端并分别使用如下命令telnet到本机：
>
> ```c
> $ telnet 192.168.1.109
> ```
>
> 接下来使用ps命令查看与进程9543相关的进程：
>
> ```shell
> $ ps -eo pid,ppid,pgid,sid,comm | grep 9543
> PID		PPID		PGID		SESS		COMMAND
> 9543	1			9543		9543		xinetd
> 9810	9543		9810		9810		in.telnetd
> 10355	9543		10355		10355		in.telnetd
> ```
>
> ​	由此可见，我们每次使用telnet登录到xinetd服务，它都创建一个子进程来为该telnet客户服务。子进程运行in.telnetd程序，这是在/etc/xinetd.d/telnet配置文件中定义的。每个子进程都处于自己独立的进程组和会话中。我们可以使用lsof进一步查看子进程都打开哪些文件描述符：
>
> ```shell
> $ sudo lsof -p 9810 # 以子进程9810为例
> 	in.telnet 9810 root 0u Ipv4 48189 0t0 TCP 
> 	in.telnet 9810 root 1u IPv4 48189 0t0 TCP
> 	in.telnet 9810 root 2u IPv4 48189 0t0 TCP
> ```
>
> ​	这里省略了一些无关的输出。通过lsof的输出我们知道，子进程9810关闭了其标准输入、标准输出和标准错误，而将socket文件描述符dup到它们上面。因此，**telnet服务器程序将网络连接上的输入当做标准输入，并把标准输出定向到一个网络连接上。**
>
> ​	在进一步，对xinetd进程使用lsof命令：
>
> ```shell
> $ sudo lsof -p 9543
> xinetd 9543 root 5u IPv6 47265 0t0 TCP *:telnet(LISTEN)
> ```
>
> ​	这一条输出说明xinetd将一直监听telnet连接请求，因此in.telnetd子进程只处理连接socket，而不处理监听socket。这是子配置文件中wait参数所定义的行为。
>
> ​	对于内部标准服务，xinetd的处理流程也可以用上述方法来分析，这里不再赘述。
>
> ​	综合上面的讨论，我们将xinetd的工作流程（wait选项的值是no的情况）绘制为下图：
>
> ![image-20230228193514089](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230228193514089.png)

# 5 信号

> ​	信号是由用户、系统或者进程发送给目标进程的信息，以通知目标进程某个状态的改变或者系统异常。Linux信号可由如下条件产生：
>
> * 对于前台进程，用户可以通过输入特殊的终端字符来给它发送信号。比如输入Ctl+C通常会给进程发送一个中断信号。
>
> * 系统异常。比如浮点异常和非法内存段访问。
>
> * 系统状态变化。比如alarm定时器到期将引起SIGALRM信号。
>
> * 运行kill命令或调用kill函数。
>
>   服务程序必须处理（或至少忽略）一些常见的信号，以免异常终止。
>
>   本章先讨论如何在程序中发送信号和处理信号，然后讨论Linux支持的信号种类，并详细探讨其中和网络编程密切相关的几个。

## 5.1 Linux信号概述

### 5.1.1 发送信号

> ​	Linux下，一个进程给其他进程发送信号的API是kill函数。其定义如下：
>
> ```c
> #include <sys/types.h>
> #include <signal.h>
> int kill(pid_t pid, int sig);
> ```
>
> ​	该函数把信号sig发送给目标进程；目标进程由pid参数指定，其可能的取值及含义如下表所示：
>
> | pid参数  | 含义                                                         |
> | -------- | ------------------------------------------------------------ |
> | pid>0    | 信号发送给PID为pid的进程                                     |
> | pid = 0  | 信号发送给进程组内的其他进程                                 |
> | pid = -1 | 信号发给除init进程外的所有进程，但发送者需要拥有对目标进程发送信号的权限 |
> | pid < -1 | 信号发送给组ID为-pid的进程组中的所有成员                     |
>
> ​	Linux定义的信号值都大于0，如果sig取值为0，则kill函数不发送任何信号。但将sig设置为0可以用来检测目标进程或进程组是否存在，因为检查工作总是在信号发送之前就执行。不过这种检测方式是不可靠的。一方面进程PID的回绕，可能导致被检测的PID不是我们期望的进程的PID；另一方面，这中检测方法不是原子操作。
>
> ​	该函数成功时返回0，失败则返回-1并设置errno。几种可能的errno如下表所示。
>
> | errno  | 含义                                     |
> | ------ | ---------------------------------------- |
> | EINVAL | 无效的信号                               |
> | EPERM  | 该进程没有权限发送信号给任何一个目标进程 |
> | ESRCH  | 目标进程或进程组不存在                   |

### 5.1.2 信号处理方式

> ​	目标进程在收到信号时需要定义一个接收函数来处理之。信号处理函数的原型如下：
>
> ```c
> #include <signam.h>
> typedef void (*__sighander_t)(int);
> ```
>
> ​	信号处理函数只带有一个整型参数，该参数用来指示信号类型。信号处理函数应该是可重入的，否则很容易引发一些竞态条件。所以在信号处理函数中严禁调用一些不安全的函数。
>
> ​	除了用户自定义信号处理函数外，bits/signum.h头文件中还定义了信号的两种其他处理方式————SIG_IGN和SIG_DEL：
>
> ```c
> #include <bits/signum.h>
> #define SIG_DFL ((__sighandler_t) 0)
> #define SIG_IGN ((__sighandler_t))
> ```
>
> ​	SIG_IGN表示忽略目标信号，SIG_DFL表示使用信号的默认处理方式。信号的默认处理方式有如下几种：结束进程（Term）、忽略信号（Ign）、结束进程并生成核心转储文件（Core)、暂停进程（Stop)，以及继续进程（Cont）

### 5.1.3 Linux信号

> ​	Linux的可用信号都定义在bits/signum.h头文件中，其中包括标准信号和POSIX实时信号。我们仅讨论标准信号，如下表所示：
>
> | 信号        | 起源      | 默认行为 | 含义                                                         |
> | ----------- | --------- | -------- | ------------------------------------------------------------ |
> | SIGHUP      | POSIX     | Term     | 控制终端挂起                                                 |
> | SIGINT      | ANSI      | Term     | 键盘输入以中断进程（Ctrl+C)                                  |
> | SIGQUIT     | POSIX     | Core     | 键盘输入使进程退出（Ctrl+\\)                                 |
> | SIGILL      | ANSI      | Core     | 非法指令                                                     |
> | SIGTRAP     | POSIX     | Core     | 断点陷阱，用于调试                                           |
> | SIGABRT     | ANSI      | Core     | 进程调用abort函数时生成该信号                                |
> | SIGIOT      | 4.2BSD    | Core     | 和SIGABRT相同                                                |
> | SIGBUS      | 4.2BSD    | Core     | 总线错误，错误内存访问                                       |
> | SIGFPE      | ANSI      | Core     | 浮点异常                                                     |
> | **SIGKILL** | **POSIX** | **Term** | **终止该进程，该信号不可被捕获或者忽略**                     |
> | SIGUSR1     | POSIX     | Term     | 用户自定义信号之一                                           |
> | SIGSEGV     | ANSI      | Core     | 非法内存段引用                                               |
> | SIGUSER2    | POSIX     | Term     | 用户自定义信号之二                                           |
> | SIGPIPE     | POSIX     | Term     | 往读端被关闭的管道或者socket连接中写数据                     |
> | SIGALRM     | POSIX     | Term     | 由alarm或者setitimer设置的实时闹钟超时引起                   |
> | SIGTERM     | ANSI      | Term     | 终止进程。kill命令默认发送的信号就是SIGTERM                  |
> | SIGSTKFLT   | Linux     | Term     | 早期Linux使用该信号来报告数学协处理器栈错误                  |
> | SIGCLD      | System V  | Ign      | 和SIGCHLD相同                                                |
> | SIGCHLD     | POSIX     | Ign      | 子进程状态发生变化（退出或暂停）                             |
> | SIGCONT     | POSIX     | Cont     | **启动被暂停的进程（Ctrl+Q)。如果目标进程未处于暂停状态，则信号被忽略** |
> | SIGSTOP     | POSIX     | Stop     | **暂停进程（Ctrl+S)。该信号不可被捕获或者忽略**              |
> | SIGTSTP     | POSIX     | Stop     | **挂起进程（Ctrl+Z)**                                        |
> | SIGTTIN     | POSIX     | Stop     | **后台进程试图从终端读取输入**                               |
> | SIGTTOU     | POSIX     | Stop     | **后台进程试图往终端输出内容**                               |
> | SIGURG      | 4.2BSD    | Ign      | socket连接上接收到紧急数据                                   |
> | SIGXCPU     | 4.2BSD    | Ign      | 进程的CPU使用时间超过其软限制                                |
> | SIGXFSZ     | 4.2BSD    | Core     | 文件尺寸超过其软限制                                         |
> | SIGVTALRM   | 4.2BSD    | Term     | 与SIGALRM类似，不过它只统计本进程用户空间代码的运行时间      |
> | SIGPROF     | 4.2BSD    | Term     | 与SIGALRM类似，不过它同时统计用户代码和内核的运行时间        |
> | SIGWINCH    | 4.3BSD    | Ign      | 终端窗口大小发生变化                                         |
> | SIGPOLL     | System V  | Term     | 与SIGIO类似                                                  |
> | SIGIO       | 4.2BSD    | Ign      | IO就绪，比如socket上发生可读、可写事件。因为TCP服务器可触发SIGIO的条件很多，故SIGIO无法在TCP服务器中使用。SIGIO信号可用在UDP服务器中，不过也非常少见 |
> | SIGPWR      | System V  | Term     | 对于使用UPS(Uninterruptable Power Supply)的系统，当电池电量过低时，SIGPWR信号被触发 |
> | SIGSYS      | POSIX     | Core     | 非法系统调用                                                 |
> | SIGUNUSERD  |           | Core     | 保留，通常和SIGSYS效果相同                                   |
>
> ​	我们并不需要在代码中处理所有这些信号。本章后面将重点介绍与网络编程关系紧密的几个信号：SIGHUP、SIGPIPE和SIGURG。后续章节还将介绍SIGALRM、SIGCHLD等信号的使用。

### 5.1.4 中断系统调用

> ​	如果程序在执行处理阻塞状态的系统调用时接收到信号，并且我们为该信号设置了信号处理函数，**默认情况下系统调用将被中断，并且errno被设置为EINTR。**我们可以使用sigaction函数（见后文）为信号设置SA_RESTART标志以自动重启被该信号中断的系统调用。
>
> ​	对于默认行为时暂停进程的信号（比如SIGSTOP、SIGTTIN)，如果我们没有为它们设置信号处理函数，则它们也可以中断某些系统调用（比如connect、epoll_wait）。POSIX没有这种行为，这是Linux独有的。

## 5.2 信号函数

### 5.2.1 signal系统调用

> ​	要为一个信号设置处理函数，可以使用下面的signam系统调用：
>
> ```c
> #include <signal.h>
> _sighandler_t signal(int sig, _sighandler_t _handler);
> ```
>
> ​	sig参数指出要捕获的信号类型。\_handler参数时\_sighandler_t类型的函数指针，用于指定信号sig的处理函数。
>
> ​	signal函数成功时返回一个函数指针，该函数指针的类型也是\_sighandler_t。这个返回值是前一次调用signal函数时传入的函数指针，或者时信号sig对应的默认处理函数指针SIG_DEF(如果是第一次调用signal的话)。
>
> ​	**signal系统调用出错时返回SIG_ERR，并设置errno。**

### 10.2.2 sigaction系统调用

> 设置信号处理函数的更健壮的接口是如下的系统调用：
>
> ```c
> #include <signal.h>
> int sigaction(int sig, const struct sigaction* act, struct sigaction* oact);
> ```
>
> ​	sig参数指出要捕获的信号类型，act参数指定新的信号处理方式，oact参数则输出信号先前的处理方式（如果不为NULL的话）。act和oact都是sigaction结构体类型的指针，sigaction结构体描述了信号处理的细节，其定义如下：
>
> ```c
> struct sigaction
> {
> #ifdef __USER_POSIX199309
> 	union
> 	{
> 		_sighandler_t sa_handler;
> 		void (*sa_sigaction)(int, siginfo_t*, void*);
> 	}
> 	_sigaction_handler;
> #define sa_handler	__sigaction_handler.sa_handler
> #define sa_sigaction 	__sigaction_handler.sa_sigaction
> #else
> 	_sighandler_t sa_handler;
> #endif
> 
> 	_sigset_t sa_mask;
> 	int sa_flags;
> 	void (*sa_restorer) (void);
> };
> ```
>
> ​	该结构体中的sa_hander成员指定信号处理函数。sa_mask成员设置进程的信号掩码（确切的说是在进程原有信号掩码的基础上增加信号掩码），以指定哪些信号不能发送给本进程。sa_mask是信号集sigset_t（\_sigset_t的同义词）类型指定一组信号。关于信号集，我们将在后面介绍。
>
> **sa_mask成员：**
> 功能：sa_mask是一个信号集，当接收到某个信号，并且调用sa_handler函数对信号处理之前，把该信号集里面的信号加入到进程的信号屏蔽字当中，当sa_handler函数执行完之后，这个信号集中的信号又会从进程的信号屏蔽字中移除
> 为什么这样设计？？这样保证了当正在处理一个信号时，如果此种信号再次发生，信号就会阻塞。如果阻塞期间产生了多个同种类型的信号，那么当sa_handler处理完之后。进程又只接受一个这种信号
> 即使没有信号需要屏蔽，也要初始化这个成员（sigemptyset()），不能保证sa_mask=0会做同样的事情
> sigset_t数据类型见文章：![img](file:///C:\Users\pc\AppData\Roaming\Tencent\QQTempSys\%W@GJ$ACOF(TYDYECOKVDYB.png)https://blog.csdn.net/qq_41453285/article/details/89228297
>
> sa_flags成员用于设置程序收到信号时的行为，其可选值如下表所示：
>
> | 选项          | 含义                                                         |
> | ------------- | ------------------------------------------------------------ |
> | SA_NOCLDSSTOP | **如果sigaction的sig参数是SIGCHLD，则设置该标志表示子进程暂停时不生成SIGCHLD信号。** |
> | SA_NOCLDWAIT  | 如果sigaction的sig参数时SIGCHLD，则设置该标志表示子进程结束时不产生僵尸进程 |
> | SA_SIGINFO    | 使用sa_sigaction作为信号处理函数（而不是默认的sa_handler），它给进程提供更多相关的信息。 |
> | SA_ONSTACK    | 调用由sigaltstack函数设置的可选信号栈上的信号处理函数        |
> | SA_RESTART    | **重新调用被该信号终止的系统调用**                           |
> | SA_NODEFER    | 当接收到信号并进入信号处理函数时，不屏蔽该信号，默认情况下，我们期望进程在处理一个信号时不再接收到同种信号，否则将引起竞态条件。 |
> | SA_RESETHAND  | 信号处理函数执行完以后，回复信号的默认处理方式               |
> | SA_INTERRUPT  | 中断系统调用                                                 |
> | SA_NOMASK     | 同SA_NODEFFER                                                |
> | SA_ONESHOT    | 同SA_RESETHAND                                               |
> | SA_STACK      | 同SA_ONSTACK                                                 |
>
> ​	sa_restorer成员已经过时，最好不要使用。sigaction成功时返回0，失败则返回-1并设置errno。

## 5.3 信号集

### 5.3.1 信号集函数

> ​	Linux使用数据结构sigset_t来表示一组信号。其定义如下：
>
> ```c
> #include <bits/sigset.h>
> # define _SIGSET_NWORDS (1024 / (8 * sizeof(unsigned long int)))
> typedef struct
> {
> 	unsigned long int int __val[_SIGSET_NWORDS];
> }__sigset_t;
> ```
>
> ​	由该定义可见，sigset_t实际上是一个长整型数组，数组的每个元素的每个位表示一个信号。这种定义方式和文件描述符集fd_set类似。Linux提供了如下一组函数来设置、修改、删除和查询信号集：
>
> ```c
> #include <signal.h>
> int sigemptyset(sigset_t* _set)	/* 清空信号集 */
> int sigfillset(sigset_t* _set);	/* 在信号集中设置所有信号 */
> int sigaddset(sigset_t* _set, int _signo)	/* 将信号_signo添加至信号集中 */
> int sigdelset(sigset_t* _set, int _signo)	/* 将信号_signo从信号集中删除 */
> int sigismember(_const sigset_t* _set, int _signo) /* 测试_signo是否在信号集中 */
> ```

### 5.3.2 进程信号掩码

> ​	我们可以利用sigaction结构体的sa_mask成员来设置进程的信号掩码。此外，如下函数也可以用于设置或查看进程的信号掩码：
>
> ```c
> #include <signal.h>
> int sigprocmask(int _how, _const sigset_t* _set, sigset_t* _oset);
> ```
>
> ​	\_set参数指定新的信号掩码，\_oset参数则输出原来的信号掩码（如果不为NULL的话）。如果\_set参数不为NULL，则_how参数指定设置进程信号掩码的方式，其可选值如下表所示：
>
> | \_how参数   | 含义                                                         |
> | ----------- | ------------------------------------------------------------ |
> | SIG_BLOCK   | 新的进程信号掩码是其当前值和\_set指定的信号集的并集          |
> | SIG_UNBLOCK | 新的进程信号掩码是其当前值和~\_set信号集的交集。因此\_set指定的信号集将不被屏蔽 |
> | SIG_SETMASK | 直接将进程信号掩码设置为\_set                                |
>
> ​	如果\_set为NULL，则进程信号掩码不变，此时我们仍然可以利用\_oldset参数来获得进程的当前的信号掩码。
>
> ​	sigprocmask成功时返回0，失败则返回-1并设置errno。

### 5.3.3 被挂起的信号

> ​	设置进程信号掩码后，被屏蔽的信号将不能被进程接收。如果给进程发送一个被屏蔽的信号，则操作系统将该信号设置为进程的一个被挂起的信号。如果我们取消对被挂起信号的屏蔽，则它立即被进程接收到。如下函数可以获得进程当前被挂起的信号集：
>
> ```c
> #include <signal.h>
> int sigpending(sigset_t* set);
> ```
>
> ​	set参数用于保存被挂起的信号集。显然，进程即使多次接收到同一个被挂起的信号，sigpendign函数也只能反映一次。并且，当我们再次使用sigprocmask使能该挂起的信号时，该信号的处理函数也只被触发一次。
>
> ​	sigpending成功时返回0,失败时返回-1并设置errno。
>
> 需要注意的是，要始终清楚地知道进程每个运行时刻的信号掩码，以及如何适当地处理捕获到的信号。在多进程、多线程环境中，我们要以进程、线程为单位来处理信号和信号掩码。我们不能设想新创建的进程、线程具有和父进程、主线程完全相同的信号特征。**比如，fork调用产生的子进程将继承父进程的信号掩码，但具有一个空的挂起的信号集。**

## 5.4 统一事件源

> ​	信号是一种异步事件：信号处理函数和程序的主循环是两条不同的执行路线。很显然，信号处理函数需要尽可能快地执行，以确保该信号不被屏蔽（为了避免一些竞态条件，信号在处理期间，系统不会再次触发它）太久。一种典型的解决方案是：**把信号的主要处理逻辑放到程序的主循环中，当信号处理函数被触发时，它只是简单地通知主循环程序接收到信号，并把信号值传递给主循环，主循环再根据接收到的信号值执行目标信号对应的逻辑代码**。信号处理函数通常使用管道来将信号“传递”给主循环：信号处理函数往管道的写端写入信号值，主循环则从管道的读端读出该信号值。那么主循环怎么知道管道上何时有数据可读？这很简单，我们只需要使用I/O复用系统调用来监听管道的读端文件描述符上的可读事件。如此一来，信号事件就能和其他I/O事件一样被处理，即统一事件源。
>
> ​	很多优秀的I/O框架库和后台服务器程序都统一处理信号和I/O事件，比如Libevent I/O框架库和xinetd超级服务。以下代码给出了统一事件源的一个简单实现。
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <signal.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <pthread.h>
> 
> #define MAX_EVENT_NUMBER 1024
> static int pipefd[2];
> 
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> void addfd(int epollfd, int fd)
> {
>     epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET;
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> /* 信号处理函数 */
> void sig_handler(int sig)
> {
>     /* 保留原来的errno，在函数最后恢复，以保证函数的可重入性 */
>     int save_errno = errno;
>     int msg = sig;
>     send(pipefd[1], (char*)&msg, 1, 0); /* 将信号值写入管道，以通知主循环 */
>     errno = save_errno;
> }
> 
> /* 设置信号的处理函数 */
> void addsig(int sig)
> {
>     struct sigaction sa;
>     memset(&sa, '\0', sizeof(sa));
>     sa.sa_handler = sig_handler;
>     sa.sa_flags |= SA_RESTART;
>     sigfillset(&sa.sa_mask);
>     assert(sigaction(sig, &sa, NULL) != -1);
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
> 
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
> 
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     if(ret == -1)
>     {
>         printf("errno is %d\n", errno);
>         return 1;
>     }
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     epoll_event events[MAX_EVENT_NUMBER];
>     int epollfd = epoll_create(5);
>     assert(epollfd != -1);
>     addfd(epollfd, listenfd);
> 
>     /* 设置一些信号的处理函数 */
>     addsig(SIGHUP);
>     addsig(SIGCHLD);
>     addsig(SIGTERM);
>     addsig(SIGINT);
>     bool stop_server = false;
> 
>     while(!stop_server)
>     {
>         int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
>         if((number < 0) && (errno != EINTR))
>         {
>             printf("epoll failure\n");
>             break;
>         }
> 
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             /* 如果有就绪的文件描述符是listenfd，则处理新的连接 */
>             if(sockfd == listenfd)
>             {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
>                 int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                 addfd(epollfd, connfd);
>             }
>             /* 如果就绪文件描述符是pipefd[0]，则处理信号 */
>             else if((sockfd == pipefd[0]) && (events[i].events & EPOLLIN))
>             {
>                 int sig;
>                 char signals[1024];
>                 ret = recv(pipefd[0], signals, sizeof(signals), 0);
>                 if(ret == -1)
>                 {
>                     continue;
>                 }
>                 else if(ret == 0)
>                 {
>                     continue;
>                 }
>                 else
>                 {
>                     /* 因为每个信号占1字节，所以按字节来逐个接收信号，我们以SIGTERM为例，来说明如何安全地终止服务器主循环 */
>                     for(int i = 0; i < ret; ++i)
>                     {
>                         switch(signals[i])
>                         {
>                             case SIGCHLD:
>                             case SIGHUP:
>                             {
>                                 continue;
>                             }
>                             case SIGTERM:
>                             case SIGINT:
>                             {
>                                 stop_server = true;
>                             }
>                         }
>                     }
>                 }
>             }
>             else
>             {
> 
>             }
>         }
>     }
> 
>     printf("close fds\n");
>     close(listenfd);
>     close(pipefd[1]);
>     close(pipefd[0]);
>     return 0;
> }
> ```

## 5.5 网络编程相关信号

### 5.5.1 SIGHUP

> ​	当挂起进程的控制终端时，SIGHUP信号将被触发。对于没有控制终端的网络后台程序而言，它们通常利用SIGHUP来强制服务器重读配置文件。一个典型的例子是xinetd超级服务程序。
>
> ​	xinetd程序在接收到SIGHUP信号之后将调用hard_reconfig函数（见xinetd源码），它循环读取/etc/xinetd.d/目录下的每个子配置文件，并检测其变化。如果某个正在运行的子服务的配置文件被修改以停止服务，则xinetd主进程将给该子服务进程发送SIGTERM信号以结束它。如果某个子服务的配置文件被修改以开启服务，则xinetd将创建新的socket并将其绑定到该服务对应的端口上。下面我们简单地分析xinetd处理SIGHUP信号的流程。
>
> 测试机Kongming20上具有如下环境：
>
> ![image-20230302212631106](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230302212631106.png)
>
> ​	从ps的输出来看，xinetd创建了子进程7442，它运行echo-stream内部服务。从lsof的输出来看，xinetd打开了一个管道。该管道的读端文件描述符的值是3，写端文件描述符的值是4。后面我们将看到，它们的作用就是统一事件源。现在我们修改/etc/xinetd.d/目录下的部分配置文件，并给xinetd发送一个SIGHUP信号。具体操作如下：
>
> ![image-20230302214019975](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230302214019975.png)

<font color=green>**trace命令能跟踪程序执行时调用的系统调用和接收到的信号。**</font>这里我们利用trace命令跟踪进程7438，即xinetd服务器程序，以观察xinetd是如何处理SIGHUP信号的。此次trace命令的部分输出如下所示：

![image-20230302214839190](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230302214839190.png)

![image-20230302214916606](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230302214916606.png)

> ​	该输出分为4个部分，我们用空行将每个部分分隔开。
>
> ​	第一部分描述程序接收到SIGHUP信号时，信号处理函数使用管道通知主程序该信号的到来。信号处理函数往文件描述符4（管道的写端）写入信号值1（SIGHUP信号），而主程序使用poll检测到文件描述符3（管道的读端）上有可读事件，就将管道上的数据读入。
>
> ​	第二部分描述了xinetd重新读取一个子配置文件的过程。
>
> ​	第三部分描述了xinetd给子进程echo-stream（PID为7442）发送一个SIGTERM信号来终止该子进程，并调用waitpid来等待该子进程结束。
>
> ​	第四部分描述了xinetd启动telnet服务的过程：创建一个流服务socket并将其绑定到端口23上，然后监听该端口。

### 10.5.2 SIGPIPE

> ​	默认情况下，往一个读端关闭的管道或socket连接中写数据将引发SIGPIPE信号。我们需要在代码中捕获并处理该信号，或者至少忽略它，因为程序接收到SIGPIPE信号的默认行为是结束进程，而我们绝对不希望因为错误的写操作而导致程序退出。**引起SIGPIPE信号的写操作将设置errno为EPIPE**。
>
> ​	之前提到过，我们可以使用send函数的MSG_NOSIGNAL标志来禁止写操作触发SIGPIPE信号。**在这种情况下，我们应该使用send函数反馈的errno值来判断管道或者socket连接的读端是否已经关闭。**
>
> ​	此外，我们也可以利用I/O复用系统调用来检测管道和socket连接的读端是否已经关闭。以poll为例，当管道的读端关闭时，**写端文件描述符上的POLLHUP事件将被触发；**当socket连接被对方关闭时，socket上的POLLRDHUP事件将被触发。

### 10.5.3 SIGURG

> ​	在Linux环境下，内核通知应用程序带外数据到达主要有两种方式：一种是之前介绍的I/O复用技术，select等系统调用在接收到带外数据时将返回，并向应用程序报告socket上的异常事件，在select系统调用讲解中给出了一个这方面的例子；另外一种方法就是使用SIGURG信号，如以下代码所示：
>
> ```c
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <stdlib.h>
> #include <errno.h>
> #include <string.h>
> #include <signal.h>
> #include <fcntl.h>
> 
> #define BUF_SIZE 1024
> 
> static int connfd;
> /* SIGURG 信号处理函数 */
> void sig_urg(int sig)
> {
>     int save_errno = errno;
>     char buffer[BUF_SIZE];
>     memset(buffer, '\0', BUF_SIZE);
>     int ret = recv(connfd, buffer, BUF_SIZE-1, MSG_OOB);    /* 接收带外数据 */
>     printf("got %d bytes of oob data '%s'\n", ret, buffer);
>     errno = save_errno;
> }
> 
> void addsig(int sig, void(*sig_handler)(int))
> {
>     struct sigaction sa;
>     memset(&sa, '\0', sizeof(sa));
>     sa.sa_handler = sig_handler;
>     sa.sa_flags |= SA_RESTART;
>     sigfillset(&sa.sa_mask);
>     assert(sigaction(sig, &sa, NULL) != -1);
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename((argv[0])));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int sock = socket(PF_INET, SOCK_STREAM, 0);
>     assert(sock >= 0);
> 
>     int ret = bind(sock, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(sock, 5);
>     assert(ret != -1);
> 
>     struct sockaddr_in client;
>     socklen_t client_addrlength = sizeof(client);
>     connfd = accept(sock, (struct sockaddr*)&client, &client_addrlength);
>     if(connfd < 0)
>     {
>         printf("errno is: %d\n", errno);
>     }
>     else
>     {
>         addsig(SIGURG, sig_urg);
>         /* 使用SIGURG信号之前，我们必须设置socket的宿主进程或进程组 */
>         fcntl(connfd, F_SETOWN, getpid());
> 
>         char buffer[BUF_SIZE];
>         while(1)
>         {
>             memset(buffer, '\0', BUF_SIZE);
>             ret = recv(connfd, buffer, BUF_SIZE-1, 0);
>             if(ret <= 0)
>             {
>                 break;
>             }
>             printf("got %d bytes of normal data '%s'\n", ret, buffer);
>         }
>         close(connfd);
>     }
>     close(sock);
>     return 0;
> }
> ```
>
> ​	至此，我们讨论完了TCP带外数据相关的所有知识。3.8节中我们介绍了TCP带外数据的基本知识，其中探讨了TCP模块时如何发送和接收带外数据的。5.8.1小节描述了如何在应用程序中使用MSG_OOB标志的send/recv系统调用来发送/接收带外数据，并给出了相关代码。9.1.3小节和10.5.3小节分别介绍了检测带外数据是否到达的两种方法：I/O复用系统调用报告的异常事件和SIGURG信号。但应用程序检测到带外数据到达后，我们还需要进一步判断带外数据在数据流中的具体位置，才能够准确无误地读取带外数据。5.9节介绍的**sockatmark系统调用就是专门用于解决这个问题的。它判断一个socket是否处于带外标记，即socket上下一个将被读取到的数据是否是带外数据。**

# 6 定时器

> ​	网络程序需要处理的第三类事件是定时事件，比如定期检测一个客户连接的活动状态。服务器程序通常管理着众多定时事件，因此有效地组织这些定时事件，使之能在预期的时间点被触发且不影响服务器的主要逻辑，对于服务器的性能有着至关重要的影响。为此，我们要将每个定时事件分别封装成定时器，并使用某种容器类数据结构，比如链表、排序链表和时间轮，将所有定时器串联起来，以实现对定时器的统一管理。本章主要讨论的就是两种高效的管理定时器的容器：时间轮和时间堆。
>
> ​	不过，在讨论如何组织定时器之前，我们先要介绍定时的方法。**定时是指在一段时间之后触发某段代码的机制**，我们可以在这段代码中依次处理所有到期的定时器。换言之，定时机制是定时器得以被处理的原动力。Linux提供了三种定时方法，它们是：
>
> * socket选项SO_RCVTIMEO和SO_SNDTIMEO。
> * SIGALRM信号
> * I/O复用系统调用的超时参数

## 6.1 socket选项 SO_RCVTIMEO和SO_SNDTIMEO

> ​	我们介绍过socket选项SO_RCVTIMEO和SO_SNDTIMEO，它们分别用来设置socket接收数据超时时间和发送数据超时时间。因此，这两个选项仅对与数据接收和发送相关的socket专用系统调用（socket专用系统调用指的是5.2~5.11节介绍的哪些socketAPI)有效，这些系统调用包括send、sendmsg、recv、recvmsg、accept和connect。我们将选项SO_RCVTIMEO和SO_SNDTIMEO对这些系统调用的影响总结于下表中。
>
> | 系统调用 | 有效选项    | 系统调用超时后的行为                   |
> | -------- | ----------- | -------------------------------------- |
> | send     | SO_SNDTIMEO | 返回-1，设置errno为EAGAIN或EWOULDBLOCK |
> | sendmsg  | SO_SNDTIMEO | 返回-1，设置errno为EAGAIN或EWOULDBLOCK |
> | recv     | SO_RCVTIMEO | 返回-1，设置errno为EAGAIN或EWOULDBLOCK |
> | recvmsg  | SO_RCVTIMEO | 返回-1，设置errno为EAGAIN或EWOULDBLOCK |
> | accept   | SO_RCVTIMEO | 返回-1，设置errno为EAGAIN或EWOULDBLOCK |
> | connect  | SO_SNDTIMEO | 返回-1，设置errno为EINPROGRESS         |
>
> ​	由表可知，在程序中，我们可以根据系统调用（send、sendmsg、recv、recvmsg、accept和connect）的返回值以及errno来判断超时时间是否已到，进而决定是否开始处理定时任务。以下代码以connect为例，说明程序中如何使用SO_SNDTIMEO选项来定时。
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <stdlib.h>
> #include <assert.h>
> #include <stdio.h>
> #include <errno.h>
> #include <fcntl.h>
> #include <unistd.h>
> #include <string.h>
> 
> /* 超时连接函数 */
> int timeout_connect(const char* ip, int port, int time)
> {
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int sockfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(sockfd >= 0);
>     /* 通过选项SO_RCVTIMEO和SO_SNDTIMEO所设置的超时时间类型是timeval，这和select系统调用的超时参数类型相同 */
>     struct timeval timeout;
>     timeout.tv_sec = time;
>     timeout.tv_usec = 0;
>     socklen_t len = sizeof(timeout);
>     ret = setsockopt(sockfd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len);
>     assert(ret != -1);
> 
>     ret = connect(sockfd, (struct sockaddr*)&address, sizeof(address));
>     if(ret == -1)
>     {
>         /* 超时对应的错误号是EINPROGRESS.下面这个条件如果成立。我们就可以处理定时任务了 */
>         if(errno == EINPROGRESS)
>         {
>             printf("connecting timeout, process timeout logic \n");
>             return -1;
>         }
>         printf("error occur when connecting to server\n");
>         return -1;
>     }
>     return sockfd;
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int sockfd = timeout_connect(ip, port, 10);
>     if(sockfd < 0)
>     {
>         return 1;
>     }
>     return 0;
> }
> ```

## 6.2 SIGALRM信号

> ​	由alarm和seitimer函数设置的实时闹钟一旦超时，将触发SIGALRM信号。因此，我们可以利用该信号的信号处理函数来处理定时任务。但是，如果要处理多个定时任务，我们就需要不断地触发SIGALRM信号，并在其信号处理函数中执行到期的任务。一般而言，**SIGALRM信号按照固定的频率生成，即由alarm或setitimer函数设置的定时周期T保持不变。如果某个定时任务的超时时间不是T的整数倍，那么它实际被执行的时间和预期的时间略有偏差。因此定时周期T反映了定时的精度。**
>
> ​	本节中我们通过一个实例————处理非活动连接来介绍如何使用SIGALRM信号定时。不过，我们需要先给出一种简单的定时器实现————基于升序链表的定时器，并把它应用到处理非活动连接这个实例中。这样，我们才能观察到SIGALRM信号处理函数是如何定时器并执行定时任务的。此外，我们介绍这种定时器也是为了和后面要讨论的高效定时器————时间轮和时间堆做对比。

### 6.2.1 基于升序链表的定时器

> ​	定时器通常至少要包含两个成员：**一个超时时间（相对时间或者绝对时间）和一个任务回调函数**。有的时候还可能包含回调函数被执行时需要传入的参数，以及是否重启定时器等信息。如果使用链表作为容器来串联所有的定时器，则每个定时器还要包含指向下一个定时器的指针成员。进一步，如果链表是双向的，则每个定时器还需要包含前一个定时器的指针成员。
>
> 以下代码实现了一个简单的升序定时器链表。升序定时器链表将其中的定时器按照超时时间做升序排序。
>
> ```c
> #ifndef LST_TIMER
> #define LST_TIMER
> 
> #include <time.h>
> #define BUFFER_SIZE 64
> class util_timer; /* 前向声明 */
> 
> /* 用户数据结构：客户端socket地址、socket文件描述符、读缓存和定时器 */
> struct client_data
> {
>     sockaddr_in address;
>     int sockfd;
>     char buf[BUFFER_SIZE];
>     util_timer* timer;
> };
> 
> /* 定时器类 */
> class util_timer
> {
> public:
>     util_timer(): prev(NULL), next(NULL){}
> 
> public:
>     time_t expire;  /* 任务的超时时间，这里使用绝对时间 */
>     void (*cb_func)(client_data*); /* 任务回调函数 */
>     /* 回调函数处理的客户数据，由定时器的执行者传递给回调函数 */
>     client_data* user_data;
>     util_timer* prev;   /* 指向前一个定时器 */
>     util_timer* next;   /* 指向下一个定时器 */
> };
> 
> /* 定时器链表，它是一个升序、双向链表，且带有头节点和尾结点 */
> class sort_timer_lst
> {
> public:
>     sort_timer_lst(): head(NULL), tail(NULL) {}
>     /* 链表被销毁时，删除其中的定时器 */
>     ~sort_timer_lst()
>     {
>         util_timer* tmp = head;
>         while(tmp)
>         {
>             head = tmp->next;
>             delete tmp;
>             tmp = head;
>         }
>     }
> 
>     /* 将目标定时器timer添加到链表中 */
>     void add_timer(util_timer* timer)
>     {
>         if(!timer)
>         {
>             return;
>         }
>         if(!head)
>         {
>             head = tail = timer;
>             return;
>         }
>         /* 如果目标定时器的超时时间小于当前链表中所有定时器的超时时间，则把该定时器插入链表头部，作为
>         链表，作为链表新的头节点。否则就需要调用重载函数add_timer(util_timer* timer, util_timer* lst_head)，把它插入链表中
>         合适的位置，以保证链表的升序特性 */
>         if(timer->expire < head->expire)
>         {
>             timer->next = head;
>             head->prev = timer;
>             head = timer;
>             return;
>         }
>         add_timer(timer, head);
>     }
>     /* 当某个定时任务发生变化时，调整对应的定时器在链表中的位置，这个函数只考虑被调整的定时器
>     的超时时间延长的情况，即该定时器需要往链表的尾部移动 */
>     void adjust_timer(util_timer* timer)
>     {
>         if(!timer)
>         {
>             return;
>         }
>         util_timer* tmp = timer->next;
>         /* 如果被调整的目标定时器处在链表尾部，或者该定时器新的超时值仍然小于其下一个定时器的超时值，则不用调整 */
>         if(!tmp || (timer->expire < tmp->expire))
>         {
>             return;
>         }
>         /* 如果目标定时器是链表的头节点，则将该定时器从链表中取出并重新插入链表 */
>         if(timer == head)
>         {
>             head = head->next;
>             head->prev = NULL;
>             timer->next = NULL;
>             add_timer(timer, head);
>         }
>         /* 如果目标定时器不是链表的头节点，则将该定时器从链表中取出，然后插入其原来所在位置之后的部分链表中 */
>         else 
>         {
>             timer->prev->next = timer->next;
>             timer->next->prev = timer->prev;
>             add_timer(timer, timer->next);
>         }
>     }
>     /* 将目标定时器timer从链表中删除 */
>     void del_timer(util_timer* timer)
>     {
>         if(!timer)
>         {
>             return;
>         }
>         /* 下面这个条件成立表示链表中只有一个定时器，即目标定时器 */
>         if((timer == head) && (timer == tail))
>         {
>             delete timer;
>             head = NULL;
>             tail = NULL;
>             return ;
>         }
>         /* 如果链表中至少有两个定时器，且目标定时器是链表的头节点，则将链表的头节点重置为原节点的
>         下一个节点，然后删除目标定时器 */
>         if(timer == head)
>         {
>             head = head->next;
>             head->prev = NULL;
>             delete timer;
>             return;
>         }
>         /* 如果链表中至少有两个定时器，且目标定时器是链表的尾结点，则将链表的尾结点
>         重置为原尾节点的前一个节点，然后删除目标定时器 */
>         if(timer == tail)
>         {
>             tail = tail->prev;
>             tail->next = NULL;
>             delete timer;
>             return;
>         }
>         /* 如果目标定时器位于链表的中间， 则把它的前后的定时器串联起来，然后删除目标定时器 */
>         timer->prev->next = timer->next;
>         timer->next->prev = timer->prev;
>         delete timer;
>     }
>     /* SIGALRM信号每次被触发就在其信号处理函数（如果统一事件源，则是主函数）中执行一次tick函数，以处理链表上到期的任务 */
>     void tick()
>     {
>         if(!head)
>         {
>             return;
>         }
>         printf("timer tick\n");
>         time_t cur = time(NULL); /* 获得系统当前的时间 */
>         util_t timer* tmp = head;
>         /* 从头节点开始依次处理每个定时器，直到遇到一个尚未到期的定时器，这就是定时器的核心逻辑 */
>         while(tmp)
>         {
>             /* 因为每个定时器都使用绝对时间作为超时值，所以我们可以把定时器的超时值和系统当前时间，比较以判断定时器是否到期 */
>             if(cur < tmp->expire)
>             {
>                 break;
>             }
>             /* 调用定时器的回到函数，以执行定时任务 */
>             tmp->cb_func(tmp->user_data);
>             /* 执行完定时器中的定时器任务之后， 就将它从链表中删除，并重置链表头节点 */
>             head = tmp->next;
>             if(head)
>             {
>                 head->prev = NULL;
>             }
>             delete tmp;
>             tmp = head;
>         }
>     }
> private:
>     /* 一个重载的辅助函数，它被公有的add_timer函数和adjust_timer函数调用。该函数表示将目标定时器timer添加
>     到节点lst_head之后的部分链表中 */
>     void add_timer(util_timer* timer, util_timer* lst_head)
>     {
>         util_timer* prev = lst_head;
>         util_timer* tmp = prev->next;
>         /* 遍历lst_head节点之后的部分链表，直到找到一个超时时间大于目标定时器的超时时间的节点
>         ，并将目标定时器插入该节点之前 */
>         while(tmp)
>         {
>             if(timer->expire < tmp->expire)
>             {
>                 prev->next = timer;
>                 timer->next = tmp;
>                 tmp->prev = timer;
>                 timer->prev = prev;
>                 break;
>             }
>             prev = tmp;
>             tmp = tmp->next;
>         }
>         /* 如果遍历完lst_head节点之后的部分链表，仍未找到超时时间大于目标定时器的节点，
>         则将目标定时器插入链表尾部，并把它设置为链表新的尾结点 */
>         if(!tmp)
>         {
>             prev->next = timer;
>             timer->prev = prev;
>             timer->next = NULL;
>             tail = timer;
>         }
>     }
> private:
>     util_timer* head;
>     util_timer* tail;
> };
> 
> #endif
> ```
>
> ​	为了便于阅读，我们将实现包含在头文件中。sort_timer_lst是一个升序链表。其核心函数tick相当于一个心博函数，它每隔一段固定的时间就执行一次，以检测并处理到期的任务。判断定时任务到期的依据是定时器的expire值小于当前的系统时间。从执行效率来看，添加定时器的时间复杂度是O(n)，删除定时器的时间复杂度是O(1)，执行定时任务的时间复杂度是O(1)。

### 6.2.2 处理非活动连接

> ​	现在我们考虑上述定时器链表的实际应用————处理非活动连接。服务器程序通常要定期处理非活动连接：给客户端发一个重连请求，或者关闭连接，或者其它。Linux在内核中提供了**对连接是否处于活动状态的定期检查机制，我们可以通过socket选项KEEPALIVE来激活它**。不过使用这种方式使得应用程序对连接的管理变得复杂。因此，我们可以考虑在应用层实现类似于KEEPALIVE的机制，以管理所有长时间处于非活动状态的连接。比如，以下代码利用alarm函数周期性地触发SIGALRM信号，该信号的信号处理函数利用管道通知主循环执行定时器链表上上的定时任务————关闭非活动的连接。
>
> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <signal.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <pthread.h>
> #include "lst_timer.h"
> 
> #define FD_LIMIT 65535
> #define MAX_EVENT_NUMBER 1024
> #define TIMESLOT 5
> static int pipefd[2];
> /* 利用升序链表来管理定时器 */
> static sort_timer_lst timer_lst;
> static int epollfd = 0;
> 
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> void addfd(int epollfd, int fd)
> {
>     struct epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET;
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> 
> void sig_handler(int sig)
> {
>     int save_errno = errno;
>     int msg = sig;
>     send(pipefd[1], (char*)&msg, 1, 0);
>     errno = save_errno;
> }
> 
> void addsig(int sig)
> {
>     struct sigaction sa;
>     memset(&sa, '\0', sizeof(sa));
>     sa.sa_handler = sig_handler;
>     sa.sa_flags |= SA_RESTART;
>     sigfillset(&sa.sa_mask);
>     assert(sigaction(sig, &sa, NULL) != -1);
> }
> 
> void timer_handler()
> {
>     /* 定时处理任务， 实际上就是调用tick函数 */
>     timer_lst.tick();
>     /* 因为一次alarm调用只会引起一次SIGALRM信号，所以我们需要重新定时，以不断触发SIGALRM信号 */
>     alarm(TIMESLOT);
> }
> 
> /* 定时器回调函数， 它删除非活动连接socket上的注册事件，并关闭之 */
> void cb_func(client_data* user_data)
> {
>     epoll_ctl(epollfd, EPOLL_CTL_DEL, user_data->sockfd, 0);
>     assert(user_data);
>     close(user_data->sockfd);
>     printf("close fd %d\n", user_data->sockfd);
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1; 
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
> 
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     struct epoll_event events[MAX_EVENT_NUMBER];
>     int epollfd = epoll_create(5);
>     assert(epollfd != -1);
>     addfd(epollfd, listenfd);
> 
>     ret = socketpair(PF_UNIX, SOCK_STREAM, 0, pipefd);
>     assert(ret != -1);
>     setnonblocking(pipefd[1]);
>     addfd(epollfd, pipefd[0]);
> 
>     /* 设置信号处理函数 */
>     addsig(SIGALRM);
>     addsig(SIGTERM);
>     bool stop_server = false;
> 
>     client_data* users = new client_data[FD_LIMIT];
>     bool timeout = false;
>     alarm(TIMESLOT); /* 定时 */
> 
>     while(!stop_server)
>     {
>         int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
>         if((number < 0) && (errno != EINTR))
>         {
>             printf("epoll failure\n");
>             break;
>         }
>         
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             /* 处理新到的客户连接 */
>             if(sockfd == listenfd)
>             {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
>                 int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                 addfd(epollfd, connfd);
>                 users[connfd].address = client_address;
>                 users[connfd].sockfd = connfd;
>                 /* 创建定时器，设置其回调函数与超时时间，然后绑定定时器与用户数据，最后将定时器添加到链表timer_lst中 */
>                 util_timer* timer = new util_timer;
>                 timer->user_data = &users[connfd];
>                 timer->cb_func = cb_func;
>                 time_t cur = time(NULL);
>                 timer->expire = cur + 3 * TIMESLOT;
>                 users[connfd].timer = timer;
>                 timer_lst.add_timer(timer);
>             }
>             /* 处理信号 */
>             else if((sockfd == pipefd[0]) && (events[i].events & EPOLLIN))
>             {
>                 int sig;
>                 char signals[1024];
>                 ret = recv(pipefd[0], signals, sizeof(signals), 0);
>                 if(ret == -1)
>                 {
>                     // handle the error
>                     continue;
>                 }
>                 else if(ret == 0)
>                 {
>                     continue;
>                 }
>                 else
>                 {
>                     for(int i = 0; i < ret; ++i)
>                     {
>                         switch (signals[i])
>                         {
>                             case SIGALRM:
>                             {
>                                 /* 用timeout变量标记有定时任务需要处理，但不立即处理定时任务。
>                                 这是因为定时任务的优先级不是很高，我们优先处理其它更重要的任务 */
>                                 timeout = true;
>                                 break;
>                             }
>                             case SIGTERM:
>                             {
>                                 stop_server = true;
>                             }
>                         }
>                     }
>                 }
>             }
>             /* 处理客户连接上接收到的数据 */
>             else if(events[i].events & EPOLLIN)
>             {
>                 memset(users[sockfd].buf, '\0', BUFFER_SIZE);
>                 ret = recv(sockfd, users[sockfd].buf, BUFFER_SIZE-1, 0);
>                 printf("get %d bytes of client data %s from %d\n", ret, users[sockfd].buf, sockfd);
>                 util_timer* timer = users[sockfd].timer;
>                 if(ret < 0)
>                 {
>                     /* 如果发生读错误，则关闭连接，并移除其对应的定时器 */
>                     if(errno != EAGAIN)
>                     {
>                         cb_func(&users[sockfd]);
>                         if(timer)
>                         {
>                             timer_lst.del_timer(timer);
>                         }
>                     }
>                 }
>                 else if(ret == 0)
>                 {
>                     /* 如果对方已经关闭连接，则我们也关闭连接，并移除对应的定时器 */
>                     cb_func(&users[sockfd]);
>                     if(timer)
>                     {
>                         timer_lst.del_timer(timer);
>                     }
>                 }
>                 else
>                 {
>                     /* 如果某个客户连接上有数据可读，则我们要调整连接对应的定时器，以延迟该连接被关闭的时间 */
>                     if(timer)
>                     {
>                         time_t cur = time(NULL);
>                         timer->expire = cur + 3 * TIMESLOT;
>                         printf("adjust timer once\n");
>                         timer_lst.adjust_timer(timer);
>                     }
>                 }
>             }
>             else
>             {
>                 //other
>             }
>         }
>         /* 最后处理定时事件， 因为I/O事件有更高的优先级。当然，这样做将导致定时任务不能精确地按照预期的时间执行 */
>         if(timeout)
>         {
>             timer_handler();
>             timeout = false;
>         }
>     }
> 
>     close(listenfd);
>     close(pipefd[1]);
>     close(pipefd[0]);
>     delete [] users;
>     return 0;
> }
> ```

## 6.3 I/O复用系统调用的超时参数

> Linux下的3组I/O复用系统调用都带有超时参数，因此它们不仅能统一处理信号和I/O事件，也能统一处理定时事件。但是由于I/O复用系统调用可能在超时时间到期之前就返回（有I/O事件发生），所以我们要利用它们来定时，就需要不断更新定时参数以反映剩余的时间，如以下代码所示：
>
> ```c
> #define TIMEOUT 5000
> 
> int timeout = TIMEOUT;
> time_t start = time(NULL);
> time_t ene = time(NULL);
> while(1)
> {
>     printf("the timeout is now %d mil-seconds\n", timeout);
>     start = time(NULL);
>     int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, timeout);
>     if((number < 0) && (errno != EINTR))
>     {
>         printf("epolll failure\n");
>         break;
>     }
>     /* 如果epoll_wait成功返回0，则说明超时时间到，此时便可以处理定时任务，并重置定时时间 */
>     (if number == 0)
>     {
>         timeout = TIMEOUT;
>         continue;
>     }
>     end = time(NULL);
>     /* 如果epoll_wait的返回值大于0，则本次epoll_wait调用持续的时间是（end - start) * 1000ms，我们需要将定时时间timeout减去这段
>     时间，以获得下次epoll_wait调用的超时参数 */
>     timeout -= (end - start) * 1000;
>     /* 重新计算之后的timeout值有可能等于0，说明本次epoll_wait调用返回时，不仅有文件描述符就绪，
>     而且其超时时间也刚刚好到达，此时我们也要处理定时任务，并重置定时时间 */
>     if(timeout <= 0)
>     {
>         timeout = TIMEOUT;
>     }
>     // handle connections
> }
> ```

## 6.4 高性能定时器

### 6.4.1 时间轮

> ​	前文提到，基于排序链表的定时器处在一个问题：添加定时器的效率偏低。下面我们要讨论的时间轮解决了这个问题。一种简单的时间轮如图所示：
>
> ![image-20230307151441606](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230307151441606.png)
>
> ​	如图所示的时间轮内，（实线）指针指向轮子上的一个槽（slot）。它以恒定的速度顺时针转动，每转动一步就指向下一个槽（虚线指针指向的槽），每次转动称为一个滴答（tick）。一个滴答的时间称为时间轮的槽间隔si（slot interval），它实际上就是心博时间。该时间轮共有N个槽，因此它运转一周的时间是N*si。每个槽指向一条定时器链表，每条链表上的定时器具有相同的特征：**它们的定时器时间相差N\*si的整数倍**。时间轮正是利用这个关系将定时器散列到不同的链表中。假如现在指针指向槽cs，我们要添加一个定时时间为ti的定时器，则该定时器将被插入槽ts（timer slot）对应的链表中：
>
> ​                                              $ ts = (cs + (ti/si)) \% N $
>
> ​	基于排序链表的定时器使用唯一的一条链表来管理所有定时器，所以插入操作的效率随着定时器数目的增多而降低。而时间轮使用哈希表的思想，将定时器插入不同的链表上。这样每条链表上的定时器数目都将明显少于原来的排序链表上的定时器数目，插入操作的效率不受定时器数目的影响。
>
> ​	很显然，对时间轮而言，要提高定时精度，就要使si值足够小；要提高执行效率，则要N值足够大。
>
> ​	上图描述的是一种简单的时间轮，因为它只有一个轮子。而复杂的时间轮可能有多个轮子，不同的轮子拥有不同的粒度。相邻的两个轮子，精度高的转一圈，精度低的仅往前移动一槽，就像水表一样。下面将按照上图来编写一个较为简单的时间轮的实现代码：
>
> ```c
> #ifndef TIME_WHEEL_TIMER
> #define TIMER_WHEEL_TIMER
> 
> #include <time.h>
> #include <netinet/in.h>
> #include <stdio.h>
> 
> #define BUFFER_SIZE 64
> class tw_timer;
> /* 绑定socket和定时器 */
> struct client_data
> {
>     sockaddr_in address;
>     int sockfd;
>     char buf[BUFFER_SIZE];
>     tw_timer* timer;
> };
> 
> /* 定时器类 */
> class tw_timer
> {
> public: 
>     tw_timer(int rot, int ts) : next(NULL), prev(NULL), rotation(rot), time_slot(ts){}
> 
> public:
>     int rotation;   /* 记录定时器在时间轮转多少圈后生效 */
>     int time_slot;  /* 记录定时器属于时间轮上哪个槽（对应的链表，下同） */
>     void (*cb_func)(client_data*);  /* 定时器回调函数 */
>     client_data* user_data; /* 客户数据 */
>     tw_timer* next; /* 指向下一个定时器 */
>     tw_timer* prev; /* 指向前一个定时器 */
> };
> 
> class time_wheel
> {
> public:
>     time_wheel() :cur_slot(0)
>     {
>         for(int i = 0; i < N; ++i)
>         {
>             slots[i] = NULL;    /* 初始化每个槽的头节点 */
>         }
>     }
> 
>     ~time_wheel()
>     {
>         /* 遍历每个槽，并销毁其中的定时器 */
>         for(int i = 0; i < N; ++i)
>         {
>             tw_timer* tmp = slots[i];
>             while(tmp)
>             {
>                 slots[i] = tmp->next;
>                 delete tmp;
>                 tmp = slots[i];
>             }
>         }
>     }
> 
>     /* 根据定时值timeout创建一个定时器，并把它插入合适的槽中 */
>     tw_timer* add_timer(int timeout)
>     {
>         if(timeout < 0)
>         {
>             return NULL;
>         }
>         int ticks = 0;
>         /* 下面根据待插入定时器的超时值计算它将在时间轮转动多少滴答后被触发，并将该滴答数存
>         储于变量ticks中。如果待插入定时器的超时值小于时间轮的槽间隔SI，则将ticks向上折合为1，否则就
>         将ticks向下折合为timeout/SI */
>         if(timeout < SI)
>         {
>             ticks = 1;
>         }
>         else
>         {
>             ticks = timeout / SI;
>         }
>         /* 计算待插入的定时器在时间轮转动多少圈后被触发 */
>         int rotation = ticks / N;
>         /* 计算待插入的定时器应该被插入哪个槽中 */
>         int ts = (cur_slot + (ticks % N)) % N;
>         /* 创建新的定时器，它在时间轮转动rotation圈之后被触发，且位于第ts个槽上 */
>         tw_timer* timer = new tw_timer(rotation, ts);
>         /* 如果第ts个槽中尚无任何定时器，则把新建的定时器插入其中，并将该定时器设置为
>         该槽的头节点 */
>         if(!slots[ts])
>         {
>             printf("add timer, rotation is %d, ts is %d, cur_slot is %d\n", rotation, ts, cur_slot);
>             slots[ts] = timer;
>         }
>         /* 否则，将定时器插入第ts个槽中 */
>         else
>         {
>             timer->next = slots[ts];
>             slots[ts]->prev = timer;
>             slots[ts] = timer;
>         }
>         return timer;
>     }
>     /* 删除目标定时器timer */
>     void del_timer(tw_timer* timer)
>     {
>         if(!timer)
>         {
>             return;
>         }
>         int ts = timer->time_slot;
>         /* slots[ts]是目标定时器所在头节点，如果目标定时器就是该头节点，则需要重置第ts个槽的头节点 */
>         if(timer == slots[ts])
>         {
>             slots[ts] = slots[ts]->next;
>             if(slots[ts])
>             {
>                 slots[ts]->prev = NULL;
>             }
>             delete timer;
>         }
>         else
>         {
>             timer->prev->next = timer->next;
>             if(timer->next)
>             {
>                 timer->next->prev = timer->prev;
>             }
>             delete timer;
>         }
>     }
>     /* SI时间到后，调用该函数，时间轮向前滚动一个槽的间隔 */
>     void tick()
>     {
>         tw_timer* tmp = slots[cur_slot]; /* 取得时间轮上当前槽的头节点 */
>         printf("current slot is %d\n", cur_slot);
>         while(tmp)
>         {
>             printf("tick the timer once\n");
>             /* 如果定时器的rotation值大于0，则它在这一轮不起作用 */
>             if(tmp->rotation > 0)
>             {
>                 tmp->rotation--;
>                 tmp = tmp->next;
>             }
>             /* 否则， 说明定时器已经到期，于是执行定时任务， 然后删除该定时器 */
>             else
>             {
>                 tmp->cp_func(tmp->user_data);
>                 if(tmp == slots[cur_slot])
>                 {
>                     printf("delete header in cur_slot\n");
>                     slots[cur_slot] = tmp->next;
>                     delete tmp;
>                     if(slots[cur_slot])
>                     {
>                         slots[cur_slot]->prev = NULL;
>                     }
>                     tmp = slots[cur_slot];
>                 }
>                 else
>                 {
>                     tmp->prev->next = tmp->next;
>                     if(tmp->next)
>                     {
>                         tmp->next->prev = tmp->prev;
>                     }
>                     tw_timer* tmp2 = tmp->next;
>                     delete tmp;
>                     tmp = tmp2;
>                 }
>             }
>         }
>         cur_slot = ++cur_slot % N;  /* 更新时间轮的当前槽，以反映时间轮的转动 */
>     }
> private:
>     /* 时间轮上槽的数目 */
>     static const int N = 60;
>     /* 每1s时间轮转动一次，即槽间隔为1s */
>     static const int SI = 1;
>     /* 时间轮的槽，其中每个元素指向一个定时器链表，链表无序 */
>     tw_timer* slots[N];
>     int cur_slot;   /* 时间轮的当前槽 */
> };
> 
> #endif
> ```
>
> ​	可见，对时间轮而言，添加一个定时器的时间复杂度是O(1),删除一个定时器的时间复杂度也是O(1)，执行一个定时器的时间复杂度是O(n)。但实际上执行一个定时器任务的效率要比O(n)好得多，因为时间轮将所有的定时器散列到了不同的链表上。时间轮的槽越多，等价于散列表的入口越多，从而每条链表上的定时器数量越少。此外，我们的代码仅使用了一个时间轮。当使用多个轮子来实现时间轮时，执行一个定时器任务的时间复杂度接近O(1)。

### 6.4.2 时间堆

> ​	前面讨论的定时方案都是以固定的频率调用心博函数tick，并在其中依次检测到期的定时器，然后执行到期定时器上的回调函数。设计定时器的另一种思路是：将所有定时器中超时时间最小的一个定时器的超时值作为心博间隔。这样，一旦心博函数tick被调用，超时时间最小的定时器必然到期，我们就可以在tick函数中处理该定时器。然后，再从剩余的定时器中找出超时时间最小的一个，并将这段最小时间设置为下一次心博间隔。如此反复，就实现了较为精确的定时。
>
> ​	最小堆很适合处理这种定时方案。最小堆是指每个节点的值都小于或等于其子节点的值的完全二叉树。下图为一个具有6个元素的最小堆。
>
> ![image-20230307165735334](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230307165735334.png)
>
> ​	树的基本操作是插入节点和删除节点。对最小堆而言，它们都很简单。为了将一个元素插入最小堆，我们可以在树的下一个空闲位置创建一个空穴。如果X可以放在空穴中而不破坏堆序，则插入完成。否则执行上虑操作，即交换空穴和它的父节点上的元素。不断执行上虑操作，直到X可以被放入空穴，则插入操作完成。比如，我们要往上图的最小堆中插入值为14的元素，则可以按照下图所示的步骤来操作。
>
> ![image-20230307170132647](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230307170132647.png)
>
> ​	最小堆的删除操作指的是删除其根节点上的元素，并且不破坏堆序性质。执行删除操作时，我们需要先在根节点处创建一个空穴。由于堆先在少了一个元素，因此我们可以把堆的最后一个元素X移动到该堆的某个地方。如果X可以被放入空穴，则删除完成。否则就执行下虑操作，即交换空穴和它的两个儿子节点中的较小者。不断进行上述过程，直到X可以被放入空穴，则删除操作完成。比如，我们要对上图所示的最小堆执行删除操作，则可以按照下图的步骤来执行。
>
> ![image-20230307170540259](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230307170540259.png)
>
> ​	由于最小堆是一种完全二叉树，所以我们可以用数组来组织其中的元素。比如，上图所示的最小堆可以用下图所示的数组表示。对于数组中的任意一个位置i上的元素，其左儿子节点在位置2i+1上，其右儿子在位置2i+2上，其父节点则在位置[(i-1)/2]\(i>0)上。与用链表来表示堆相比，用数组表示堆不仅节省空间，而且更容易实现堆的插入、删除等操作。
>
> ![image-20230307170936632](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230307170936632.png)
>
> ​	假设我们已经有一个包含N个元素的数组，现在要把它初始化为一个最小堆。那么最简单的方法是：初始化一个空堆，然后将数组中的每个元素插入到该堆中。不过这样做的效率偏低。实际上，我们只需要对数组中的第[(N-1)/2]~2个元素执行下虑操作，即可确保该数组构成一个最小堆。这是因为对包含N个元素的完全二叉树而言，它具有[(N-1)/2]个非叶子节点，这些非叶子节点正是该完全二叉树的第0~[(N-1)/2]个节点。我们只要确保这些非叶子节点构成的子树都具有堆序性质，整个树就具有堆序性质。
>
> ​	我们称用最小堆实现的定时器为时间堆。以下代码给出了一种时间堆的实现，其中，最小堆使用数组来表示。
>
> ```c
> #ifndef MIN_HEAP
> #define MIN_HEAP
> 
> #include <iostream>
> #include <netinet/in.h>
> #include <time.h>
> usint std::exception;
> 
> #define BUFFER_SIZE 64
> 
> class heap_timer; /* 前向声明 */
> /* 绑定socket和定时器 */
> struct client_data
> {
>     sockaddr_in address;
>     int sockfd;
>     char bu[BUFFER_SIZE];
>     heap_timer* timer;
> };
> 
> /* 定时器类 */
> class heap_timer
> {
> public:
>     heap_timer(int delay)
>     {
>         expire = time(NULL) + delay;
>     }
> 
> public:
>     time_t expire;  /* 定时器生效的绝对时间 */
>     void (*cb_func)(client_data*);  /* 定时器的回调函数 */
>     client_data* user_data; /* 用户数据 */
> };
> 
> /* 时间堆类 */
> class time_heap
> {
> public:
>     /* 构造函数之一，初始化一个大小为cap的空堆 */
>     time_heap(int cap) throw(std::exception) : capacity(cap), cur_size(0)
>     {
>         array = new heap_timer* [capacity]; /* 创建堆数组 */
>         if(!array)
>         {
>             throw std::exception();
>         }
>         for(int i = 0; i < capacity; ++i)
>         {
>             array[i] = NULL;
>         }
>     }
>     /* 构造函数之二，用已有数组来初始化堆 */
>     time_heap(heap_timer** init_array, int size, int capacity) throw(std::exception) : cur_size(size), capacity(capacity)
>     {
>         if(capacity < size)
>         {
>             throw std::exception();
>         }
>         array = new heap_timer* [capacity]; /* 创建堆数组 */
>         if(!array)
>         {
>             throw std::exception();
>         }
>         for(int i = 0; i < capacity; ++i)
>         {
>             array[i] = NULL;
>         }
>         if(size != 0)
>         {
>             /* 初始化堆数组 */
>             for(int i = 0; i < size; ++i)
>             {
>                 array[i] = init_array[i];
>             }
>             for(int i = (cur_size-1)/2; i>=0; --i)
>             {
>                 /* 对数组中第[(cur_size-1)/2]~0个元素执行下虑操作 */
>                 percolate_down(i);
>             }
>         }
>     }
>     /* 销毁时间堆 */
>     ~time_heap()
>     {
>         for(int i = 0; i < cur_size; ++i)
>         {
>             delete array[i];
>         }
>         delete [] array;
>     }
> 
> public:
>     /* 添加目标定时器timer */
>     void add_timer(heap_timer* timer) throw(std::exception)
>     {
>         if(!timer)
>         {
>             return ;
>         }
>         if(cur_size >= capacity)    /* 如果当前堆数组容量不够，则将其扩大1倍 */
>         {
>             resize();
>         }
>         /* 新插入了一个元素，当前堆大小加1，hole是新建空穴的位置 */
>         int hole = cur_size++;
>         int parent = 0;
>         /* 对从空穴到根节点的路径上的所有节点进行上虑操作 */
>         for(; hole > 0; hole=parent)
>         {
>             parent = (hole-1)/2;
>             if(array[parent]->expire <= timer->expire)
>             {
>                 break;
>             }
>             array[hole] = array[parent];
>         }
>         array[hole] = timer;
>     }
>     /* 删除目标定时器timer */
>     void del_timer(heap_timer* timer)
>     {
>         if(!timer)
>         {
>             return;
>         }
>         /* 仅仅将目标定时器的回调函数设置为空，即所谓的延迟销毁。这将节省真正删除该定时器造成的开销，但这样做容易
>         使堆数组膨胀 */
>         timer->cb_func = NULL;
>     }
>     /* 获得堆顶部的定时器 */
>     heap_timer* top() const
>     {
>         if(empty())
>         {
>             return NULL;
>         }
>         return array[0];
>     }
>     /*删除堆顶部的定时器 */
>     void pop_timer()
>     {
>         if(empty())
>         {
>             return;
>         }
>         if(array[0])
>         {
>             delete array[0];
>             /* 将原来的堆顶元素替换为堆数组中最后一个元素 */
>             array[0] == arr[--cur_size];
>             percolate_down(0); /* 对新的堆顶元素执行下虑操作 */
>         }
>     }
>     /* 心博函数 */
>     void tick()
>     {
>         heap_timer* tmp = array[0];
>         time_t cur = time(NULL) /* 循环处理堆中到期的定时器 */
>         while(!empty())
>         {
>             if(!tmp)
>             {
>                 break;
>             }
>             /* 如果堆顶定时器没有到期，则退出循环 */
>             if(tmp->expire > cur)
>             {
>                 break;
>             }
>             /* 否则就执行堆顶定时器中的任务 */
>             if(array[0]->cb_func)
>             {
>                 array[0]->cb_func(array[0]->user_data);
>             }
>             /* 将堆顶元素删除，同时生成新的堆顶定时器（array[0]） */
>             pop_timer();
>             tmp = array[0];
>         }
>     }
>     bool empty() const {return cur_size == 0;}
> private:
>     /* 最小堆的下虑操作，它确保堆数组中以第hole个节点作为根的子树拥有最小堆性质 */
>     void percolate_down(int hole)
>     {
>         heap_timer* temp = array[hole];
>         int child = 0;
>         for(; ((hole*2+1) <= (cur_size-1)); hole=chile)
>         {
>             child = hole*2+1;
>             if((child < (cur_size-1)) && (array[child+1]->expire < array[child]->expire))
>             {
>                 ++child;
>             }
>             if(array[child]->expire < temp->expire)
>             {
>                 array[hole] = array[chile];
>             }
>             else
>             {
>                 break;
>             }
>         }
>         array[hole] = temp;
>     }
>     /* 将堆数组容量扩大1倍 */
>     void resize() throw(std::exception)
>     {
>         heap_timer** temp = new heap_timer* [2*capacity];
>         for(int i = 0; i < 2*capacity; ++i)
>         {
>             temp[i] = NULL;
>         }
>         if(!temp)
>         {
>             throw std::exception;
>         }
>         capacity = 2*capacity;
>         for(int i = 0; i < cur_size; ++i)
>         {
>             temp[i] = array[i];
>         }
>         delete [] array;
>         array = temp;
>     }
> 
> private:
>     heap_timer** array; /* 堆数组 */
>     int capacity;   /* 堆数组的容量 */
>     int cur_size; /* 堆数组当前包含元素的个数 */
> };
> 
> #endif
> ```
>
> ​	由代码可见，对时间堆而言，添加一个定时器的时间复杂度是O(logn)，删除一个定时器的时间复杂度是O(1)，执行一个定时器的时间复杂度是O(1)，因此时间堆的效率是很高的。

# 7 高性能I/O框架库Libevent

> 前面我们利用三章篇幅较为细致地讨论了Linux服务器程序必须处理的三类事件：I/O事件、信号和定时事件。在处理这三类事件时我们通常需要考虑如下三个问题：
>
> * 统一事件源。很明显，统一处理这三类事件既能使代码简单易懂，又能避免一些潜在的逻辑错误。前面我们已经讨论了实现统一事件源的一般方法————利用I/O复用系统调用来管理所有事件。
>
> * 可移植性。不同的操作系统具有不同的I/O复用方式，比如Solaris的dev/poll文件，FreeBSD的kqueue机制，Linux的epoll系列系统调用。
>
> * 对并发编程的支持。在多进程和多线程环境下，我们需要考虑各执行实体如何协同处理客户连接、信号和定时器，以避免竞态条件。
>
>   所幸的是，开源社区提供了诸多优秀的I/O框架库。它们不仅解决了上述问题，让开发者可以将精力放在程序逻辑上，而且稳定性、性能等方面都相当出色。比如ACE、ASIO和Libevent。本章将介绍其中相对轻量级的Libeven框架库。

## 7.1 I/O框架库概述

> ​	I/O框架库以库函数的形式，封装了较为底层的系统调用，给应用程序提供了一组更为便于使用的接口。这些库函数往往比程序员自己实现的同样功能的函数更合理、更高效，且更健壮。因为它们经受住了真实网络环境下的高压测试，以及时间的考验。
>
> ​	各种I/O框架库的实现原理基本相似，要么以Reacto模式实现，要么以Proactor模式实现，要么同时以这两种模式实现。举例来说，基于Reactor模式的I/O框架库包含如下几个组件：句柄（Handle）、事件多路分发器（EventDemultiplexer）、事件处理器（EventHandler）和具体的事件处理器（ConcreteEventHandler）、Reactor。这些组件的关系如下图所示：
>
> ![image-20230308163750140](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230308163750140.png)
>
> **1.句柄**
>
> ​	I/O框架库要处理的对象，即I/O事件、信号和定时事件，统一称为事件源。一个事件源通常和一个句柄绑定在一起。句柄的作用是，当内核检测到就绪事件时，它将通过句柄来通知应用程序这一事件。在Linux环境下，I/O事件对应的句柄是文件描述符，信号事件对应的句柄就是信号值。
>
> **2.事件多路分发器**
>
> ​	事件的到来是随机的、异步的。我们无法预知何时收到一个客户连接请求，又亦或收到一个暂停信号。所以程序需要循环地等待并处理事件，这就是事件循环。在事件循环中，等待事件一般使用I/O复用技术来实现。I/O框架库一般将系统支持的各种I/O复用系统调用封装成统一的接口，称为事件多路分发器。事件多路分发器的demultiplex方法是等待事件的核心函数，其内部调用的是select、poll、epoll_wait等函数。
>
> ​	此外，事件多路分发器还需要实现register_event和remove_event方法，以供调用者往事件分发器中添加事件和从事件多路分发器中删除事件。
>
> **3.事件处理器和具体事件处理器**
>
> ​	**事件处理器执行事件对应的业务逻辑。**它通常包含一个或多个handle_event回调函数，这些回调函数在事件循环中被执行。I/O框架库提供的事件处理器通常是一个接口，用户需要继承它来实现自己的事件处理器，即具体事件处理器。因此，事件处理器中的回调函数一般被声明为虚函数，以支持用户的扩展。
>
> ​	此外，事件处理器一般还提供一个get_handle方法，它返回与该事件处理器关联的句柄。那么，事件处理器和句柄有什么关系？当事件多路分发器检测到有事件发生时，它是通过句柄来通知应用程序的。因此，**我们必须将事件处理器和句柄绑定，才能在事件发生时获取到正确的事件处理器。**
>
> **4.Reactor**
>
> ​	Reactor是I/O框架库的核心。它提供的几个主要方法是：
>
> * handle_events。该方法执行事件循环。它重复如下过程：等待事件，然后依次处理所有就绪事件对应的事件处理器。
> * register_handler。该方法调用事件多路分发器的register_event方法来往多路分发器中注册一个事件。
> * remove_handler。该方法调用事件多路器的remove_event方法来删除事件多路分发器中的一个事件。
>
> ![image-20230308170927028](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230308170927028.png)

## 7.2 Libevent源码分析

> ​	Libevent是开源社区的一款高性能的I/O框架库，其学习者和使用者众多。使用Libevent的著名案例有：高性能的分布式内存对象缓存软件memcached，Google浏览器、Chromium的Linux版本。作为一个I/O框架库，Libevent具有如下特点：
>
> * 跨平台支持。Libevent支持Linux、UNIX和Windows。
> * 统一事件源。Linbevent对I/O事件、信号和定时器事件提供统一的处理。
> * 线程安全。Libevent使用libevent_pthreads库来提供线程安全支持。
> * 基于Reactor模式的实现。
>
> 这一节中我们将简单地研究一下Libevent源代码的主要部分。分析它除了更好地学习网络编程外，还有如下好处：
>
> * 学习编写一个产品级的函数要考虑哪些细节
>
> * 提高C语言功底。Libevent源码中使用了大量的函数指针，用C语言实现了多态机制，并提供了一些基础数据结构的高效实现，比如双向链表、最小堆等。
>
>   Libevent的官方网站是http://libevent.org/，其中提供Libevent源代码的下载，以及学习Libevent框架的第一手文档，并且源码和文档的更新也较为频繁。

### 7.2.1 一个实例

> 分析一筐软件的源代码，最简单有效的方式是从使用入手，这样才能从整体上把握该软件的逻辑结构。以下代码是使用Libevent实现的一个“Hello World”程序。
>
> ```c
> #include <sys/signal.h>
> #include <event.h>
> 
> void signal_cb(int fd, short event, void* argc)
> {
>     struct event_base* base = (event_base*)argc;
>     struct timeval delay = {2, 0};
>     printf("Caught an interrupt signal; exiting cleanly in two seconds...\n");
>     event_base_loopexit(base, &delay);
> }
> void timeout_cb(int fd, short event, void* argc)
> {
>     printf("timeout\n");
> }
> 
> int main()
> {
>     struct event_base* base = event_init();
> 
>     struct event* signal_event = evsignal_new(base, SIGINT, signal_cb, base);
> 
>     timeval tv = {1, 0};
>     struct event* timeout_event = evtimer_new(base, timeout_cb, NULL);
>     event_add(timeout_event, &tv);
> 
>     event_base_dispatch(base);
> 
>     event_free(timeout_event);
>     event_free(signal_event);
>     event_free(base);
> }
> ```
>
> ​	以上代码虽然简单，但却基本上描述了Libevent库的主要逻辑：
>
> 1) 调用event_init函数创建event_base对象。一个event_base相当于一个Reactor实例。
> 2) 创建具体的事件处理器，并设置它们所从属的Reactor实例。evsignal_new和evtimer_new分别用于创建信号事件处理器和定时事件处理器，它们是定义在include/event2/event.h文件中的宏：
>
> ```c
> #define evsignal_new(b, x, cb, arg) \
> 	event_new((b), (x), EV_SIGNAL|EV_PERSIST, (cb), (arg))
> #define evtimer_new(b, cb, arg) 	event_new((b), -1, 0, (cb), (arg))
> ```
>
> ​	可见，它们的统一入口是event_new函数，即用于创建通用事件处理器（图12-1中的EventHandler）的函数。其定义是：
>
> ```c
> struct event* event_new(struct event_base* base, evutil_socket_t fd, short events, void(*cb)(evutil_socket_t, short, void*), void* arg)
> ```
>
> ​	其中，base参数指定新创建的事件处理器从属的Reactor。fd参数指定与该事件处理器关联的句柄。创建I/O事件处理器时，应该给fd参数传递文件描述符值；创建信号事件处理器时，应该给fd参数传递信号值，比如代码中的SIGINT；创建定时事件处理器时，则应该给fd参数传递-1。events参数指定事件类型，其可选值都定义在include/event2/event.h文件中，如下所示：
>
> ```c
> #define EV_TIMEOUT	0x01	/* 定时事件 */
> #define EV_READ		0x02	/* 可读事件 */
> #define EV_WRITE	0x04	/* 可写事件 */
> #define EV_SIGNAL	0x08	/* 信号事件 */
> #define EV_PERSIST	0x10	/* 永久事件 */
> /* 边沿触发事件，需要I/O复用系统调用支持，比如epoll */
> #define EV_ET		0x20
> ```
>
> EV_PERSIST的作用是：事件被触发后，自动重新对这个event调用event_add函数。
>
> ​	cb参数指定目标事件对应的回调函数，相当于图12-1中的事件处理器的handle_event方法。arg参数则是Reactor传递给回调函数的参数。
>
> ​	event_new函数成功时返回应该event类型的对象，也就是Libevent的事件处理器。Libevent用单词“event”来描述事件处理器，而不是事件，会使读者觉得有些混乱，故而我们约定如下：
>
> * 事件指的是应该句柄上绑定的事件，比如文件描述符0上的可读事件。
> * 事件处理器，也就是event结构体类型的对象，除了包含事件具备的两个要素（句柄和事件类型）外，还有很多其他成员，比如回调函数。
> * 事件由事件多路分发器管理，事件处理器则由事件队列管理。事件队列包括多种，比如event_base中的注册事件队列、活动事件队列和通用定时器队列，以及evmap中的I/O事件队列、信号事件队列。关于这些队列，我们将在后文依次讨论。
> * 事件循环对一个被激活事件（就绪事件）的处理，指的是执行该事件对应的事件处理器中的回调函数。
>
> 3) 调用event_add函数，将事件处理器添加到注册事件队列中，并将该事件处理器对应的事件添加到事件多路分发器中。event_add函数相当于Reactor中的register_handler方法。
> 4) 调用event_base_dispatch函数来执行事件循环。
> 5) 事件循环结束后，使用\*_free系列函数来释放系统资源。

### 7.2.2 源代码组织结构

> Libevent源代码中的目录和文件按照功能可划分为如下部分：
>
> * 头文件目录include/event2。该目录是自Libevent主版本升级到2.0之后引入的，在1.4及更老的版本中并无此目录。该目录中的头文件是Libevent提供给应用程序使用的，比如，event.h文件提供核心函数，http.h头文件提供HTTP协议相关服务，rpc.h头文件提供远程过程调用支持。
>
> * 源代码目录下的头文件。这些头文件分为两类：一类是对include/event2目录下的部分头文件的包装，另外一类是提供Libevent内部使用的辅助性头文件，它们的文件名都具有\*-internal.h的形式。
>
> * 通用数据结构目录compat/sys。该目录下仅有一个文件————queue.h。它封装了跨平台的基础数据结构，包括单向链表、双向链表、队列、尾队列和循环队列。
>
> * sample目录。他提供一些示例程序。
>
> * test目录。它提供一些测试代码。
>
> * WIN32-Code目录。它提供Windows平台上的一些专用代码。
>
> * event.c文件。该文件实现Libevent的整体框架，主要是event和event_base两个结构体的相关操作
>
> * devpoll.c、kqueue.c、evport.c、select.c、win32select.c、poll.c和epoll.c文件。它们分别封装了如下I/O复用机制：/dev/poll、kqueue、event ports、POSIX select、Windows select、poll和epoll。这些文件的主要内容相似，都是针对结构体eventtop所定义的接口函数的具体实现。
>
> * minheap-internal.h文件。该文件实现了一个时间堆，以提供对定时事件的支持。
>
> * signal.c文件。它提供对信号的支持。其内容也是针对结构体eventop所定义的接口函数的具体实现。
>
> * evmap.c文件。它维护句柄（文件描述符或信号）与事件处理器的映射关系。
>
> * event_tagging.c文件。它提供往缓冲区中添加标记数据（比如一个整数），以及从缓冲区中读取标记数据的函数。
>
> * event_iocp.c文件。它提供对Windows IOCP（Input/Output Completion Port，输入输出完成端口）的支持。
>
> * buffer*.c文件，它提供对网络I/O缓冲的控制，包括：输入输出数据过滤，传输速率限制，使用SSL(Secure Sockets Layer)协议对应用数据进行保护，以及零拷贝文件传输等。
>
> * evthread*.c文件。它提供对多线程的支持。
>
> * listener.c文件。它封装了对监听socket的操作，包括监听连接和接受连接。
>
> * logs.c文件。它是Libevent的日志系统。
>
> * evutil.c、evutil_rand.c、strlcpy.c和arcrrandom.c文件。它们提供一些基本操作，比如生成随机数、获取socket地址信息、读取文件、设置socket属性等。
>
> * evdns.c、http.c和evrpc.c文件。它们分别提供了对DNS协议、HTTP协议和RPC(Remote Procedure Call，远程过程调用)协议的支持。
>
> * epoll_sub.c文件。该文件未见使用。
>
>   整个源码中，event-internal.h、include/event2/event_struct.h、event.c和evmap.c等4个文件最为重要。它们定义了event和event_base结构体，并实现了这两个结构体的相关操作。

# 8 多进程编程

> ​	进程是Linux操作系统环境的基础，它控制着系统上几乎所有的活动。本章从系统程序员的角度来讨论Linux多进程编程，包括如下内容：
>
> * 复制进程映像的fork系统调用和替换进程映像的exec系列系统调用。
> * 僵尸进程以及如何避免僵尸进程
> * 进程间通信（Inter-Process Communication, IPC）最简单的方式：管道。
> * 3种System V进程间通信方式：信号量、消息队列和共享内存。它们都是由AT&T System V2版本的UNIX引入的，所以统称为System V IPC
> * 在进程间传递文件描述符的通用方法：通过UNIX本地域socket传递特殊的辅助数据

## 8.1 fork系统调用

> Linux下创建新进程的系统调用是fork。其定义如下：
>
> ```c
> #include <sys/types.h>
> #include <unistd.h>
> pid_t fork(void)
> ```
>
> ​	该函数的每次调用都返回两次，在父进程中返回的是子进程的PID，在进程中则返回0.该返回值是后续代码判断当前进程是父进程还是子进程的依据。fork调用失败时返回-1，并设置errno。
>
> ​	fork函数复制当前进程，在内核进程表中创建一个新的进程表项。新的进程表项有很多属性和原进程相同，比如堆指针、栈指针和标志寄存器的值。但也有许多属性被赋予了新的值，比如该进程的PPID被设置成原进程的PID，信号位图被清除（原进程设置的信号处理函数不再对新进程起作用）。
>
> ​	子进程的代码域父进程完全相同，同时它还会复制父进程的数据（堆数据、栈数据和静态数据）。数据的复制采用的是所谓的写时复制（copy on write），即只有任一进程（父进程或子进程）对数据执行了写操作时，复制才会发生（先时缺页中断，然后操作系统给子进程分配内存并复制父进程的数据）。即便如此，如果我们在程序中分配了大量内存，那么使用fork也应当十分谨慎，尽量避免没必要的内存分配和数据复制。
>
> ​	此外，创建子进程后，父进程中打开的文件描述符默认在子进程中也是打开的，且文件描述符的引用计数加1.不仅如此，父进程的用户根目录，当前工作目录等变量的引用计数均会加1.

## 8.2 exec系列系统调用

> ​	有时我们需要在子进程中执行其他程序，即替换当前进程映像，这就需要使用如下exec系列函数之一：
>
> ```c
> #include <unistd.h>
> extern char** environ;
> 
> int execl(const char* path, const char* arg, ...);
> int execlp(const char* file, const char* arg, ...);
> int execle(const char* path, const char* arg, ..., char* const envp[]);
> int execv(const char* file, char* const argv[]);
> int execvp(const char* file, char* const argv[]);
> int execve(const char* path, char* const argv[], char* const envp[]);
> ```
>
> ​	path参数指定可执行文件的完整路径，file参数可以接受文件名，该文件的具体位置则在环境变量PATH中搜寻。arg接受可变参数，argv则接受参数数组，它们都会被传递给新程序（path或file指定的程序）的main函数。envp参数用于设置新程序的环境变量。如果未设置它，则新程序将使用由全局变量environ指定的环境变量。
>
> ​	一般情况下，exec函数时不返回的，除非出错。它出错时返回-1，并设置errno。如果没出错，则原程序中exec调用之后的代码都不会执行，因为此时原程序已经被exec的参数指定的程序完全替换（包括代码和数据）。
>
> ​	exec函数不会关闭原程序打开的文件描述符，除非该文件描述符被设置了类似SOCK_CLOEXEC的属性。

## 8.3 处理僵尸进程

> ​	对于多进程程序而言，父进程一般需要跟踪子进程的退出状态。因此，当子进程结束运行时，内核不会立即释放该进程的进程表表项，以满足父进程后续对该子进程退出信息的查询（如果父进程还在运行）。在子进程结束运行之后，父进程读取其退出状态之前，我们称该子进程处于僵尸态。另外一种使子进程进入僵尸态的情况是：父进程结束或者异常终止，而子进程继续运行。此时子进程的PPID将被操作系统设置为1，即init进程。init进程接管了该子进程，并等待它结束。在父进程退出之后，子进程退出之前，该进程处于僵尸态。
>
> ​	由此可见，无论哪种情况，如果父进程没有正确地处理子进程的返回信息，子进程都将停留在僵尸态，并占据着内核资源。这是绝对不能容许的，毕竟内核资源是有限的。下面这对函数在父进程中调用，以等待子进程的结束，并获取子进程的返回信息，从而避免了僵尸进程的产生，或者使子进程的僵尸态立即结束：
>
> ```c
> #include <sys/types.h>
> #include <sys/wait.h>
> pid_t wait(int* stat_loc);
> pid_t waitpid(pid_t pid, int* stat_loc, int options);
> ```
>
> ​	wait函数将阻塞进程，直到该进程的某个子进程结束运行为止。它返回结束运行的子进程的PID,并将该子进程的退出状态信息存储于stat_loc参数指向的内存中。sys/wait.h头文件中定义了几个宏来帮助解释子进程的退出状态信息，如下表所示：
>
> | 宏                    | 含义                                                      |
> | --------------------- | --------------------------------------------------------- |
> | WIFEXITED(stat_val)   | 如果子进程正常结束，它就返回一个非0值                     |
> | WEXITSTATUS(stat_val) | 如果WIFEXITED非0，它返回子进程的退出码                    |
> | WIFSIGNALED(atat_val) | 如果子进程使因为一个未捕获的信号而终止，它就返回一个非0值 |
> | WTERMSIG(stat_val)    | 如果WIFSIGNALED非0，它返回一个信号值                      |
> | WIFSTOPPED(stat_val)  | 如果子进程意外终止，它就返回一个非0值                     |
> | WSTOPSIG(stat_val)    | 如果WIFSTOPPED非0，它就返回一个信号值                     |
>
> ​	wait函数的阻塞特性显然不是服务器期望的，而waitpid函数解决了这个问题。waitpid只等待由pid参数指定的子进程。如果pid取值-1，那么它就和wait函数相同，即等待任意一个子进程结束。stat_loc参数的含义和wait的函数的stat_loc参数相同。options参数可以控制waitpid函数的行为。该参数最常用的取值是WNOHANG。当options的取值是WNOHANG时，waitpid调用将是非阻塞的：如果pid指定的目标子进程还没有结束或意外终止，则waitpid立即返回；如果目标子进程确实正常退出了，则waitpid返回该子进程的PID。waitpid调用失败时返回-1并设置errno。
>
> ​	要在事件已经发生的情况下执行非阻塞调用才能提高程序的效率。对waitpid函数而言，我们最好在某个子进程退出之后再调用它。那么父进程从何得知某个子进程已经退出了呢？这正是SIGCHLD信号的用途。当一个进程结束时，它将给其父进程发送一个SIGCHLD信号。因此我们可以再父进程中捕获SIGCHLD信号，并再信号处理函数中调用waitpid函数以“彻底结束”一个子进程，如下代码所示：
>
> ```c
> static void handle_child(int sig)
> {
> 	pid_t pid;
> 	int stat;
> 	while((pid = waitpid(-1, &stat, WNOHANG)) > 0)
> 	{
> 		/* 对结束的子进程进行善后处理 */
> 	}
> }
> ```

## 8.4 管道

> ​	管道是父进程和子进程之间通信的常用手段。
>
> ​	管道能在父、子进程间传递数据，利用的是fork调用之后两个管道文件描述符（fd[0]和fd[1]）都保持打开。一对这样的文件描述符只能保证父、子进程间一个方向的数据传输，父进程和子进程必须有一个关闭fd[0]，另一个关闭fd[1]。比如，我们要使用管道实现从父进程向子进程写数据，就应该按照下图所示来操作。
>
> ![image-20230312155905464](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230312155905464.png)
>
> ​	显然，如果要实现父、子进程之间的双向数据传输，就必须使用两个管道。socket编程接口提供了一个创建全双工管道的系统调用：socketpair。squid服务器程序就是利用socketpair创建管道，以实现父进程和日志服务子进程之间传递日志信息，下面我们简单分析之。在测试机器Kongming20上有如下环境：
>
> ![image-20230312160358996](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230312160358996.png)
>
> ​	这些输出说明Kongming20上开启了squid服务。该服务创建了几个子进程，其中子进程12492专门用于输出日志到/var/log/squid/access.log文件。父进程12491使用socketpair创建了一对UNIX域socket，然后关闭了其中一个，剩下的那个socket的值是9。子进程12492则从父进程12491继承了这一对UNIX域socket，并关闭了其中的另外一个，剩下的那个socket则被dup到标准输入和标准输出上。下面我们telnet到squid服务上，并向它发送部分数据。同时开启另外两个终端，分别运行strace命令以查看进程12491和12492在这个过程中交换的数据。具体操作如以下代码所示：
>
> ```shell
> $ telnet 192.168.1.109 squid
> Trying 192.168.1.109...
> Connected to 192.168.1.109.
> Escape character is '^]'
> a（回车）
> $ sudo strace -p 12491
> write(9, "L1338385956.213		40 192.168.1"...,104) = 104
> $ sudo strace -p 12492
> read(0, "L1338385956.213		40 192.168.1"..., 4096) = 104
> write(3, "1338385956.213		40 192.168.1."..., 101) = 101
> ```
>
> ​	由此可见，进程12491接收到客户数据后将日志信息输出至管道（写文件描述符9）。日志服务子进程使用阻塞读操作等待管道上有数据可读（读文件描述符0），然后将读取到的日志信息写入/var/log/squid/access.og文件（写文件描述符3）。
>
> ​	不过，管道只能用于关联两个进程（比如父、子进程）间的通信。而下面要讨论的3种System V IPC能用于无关联的多进程之间的通信，因为它们都使用一个全局唯一的键值来标识一条信道。不过，有一种特殊的管道称为FIFO(First In First Out，先进先出)，也叫命名管道。它也能用于无关联进程之间的通信。

## 8.5 信号量

### 8.5.1 信号量原语

> ​	当多个进程同时访问系统上的某个资源的时候，比如同时写一个数据库的某条记录，或者同时修改某个文件，就需要考虑进程的同步问题，以确保任一时刻只有一个进程可以拥有对资源的独占式访问。通常，程序对共享资源的访问的代码只是很短的一段，但就是这一段代码引发了进程之间的竞态条件。**我们称这段代码为关键代码段，或者临界区。对进程同步，也就是确保任一时刻只有一个进程能进入关键代码段。**
>
> ​	要编写具有通用目的的代码，以确保关键代码段的独占式访问是非常困难的。有两个名为Dekker算法和Peterson算法的解决方案，它们试图从语言本身（不需要内核支持）解决并发问题。但它们依赖于忙等待，即进程要持续不断地等待某个内存位置状态的改变。这种方式下CPU利用率太低，显然是不可取的。
>
> ​	Dijkstra提出的信号量（Semaphore）概念是并发编程领域迈出的重要一步。信号量是一种特殊的变量，它只能去自然数值并且只支持两种操作：等待（wait）和信号（signal）。不过在Linux/UNIX中，”等待“和”信号“都已经具有特殊的含义，所以对信号量的这两种操作更常用的称呼是P、V操作。这两个字母来自于荷兰单词passeren（传递，就好像进入临界区）和vrijgeven（释放，就好像退出临界区）。假设有信号量SV，则对它的P、V操作含义如下：
>
> * P(SV)，如果SV的值大于0，就将它减1；如果SV的值为0，则挂起进程的执行。
>
> * V(SV)，如果有其他进程因为等待SV而挂起，则唤醒之；如果没有，则将SV加1。
>
>   信号量的取值可以是任何自然数。但最常用的、最简单的信号量是二进制信号量，它只能取0和1这两个值。使用二进制信号量同步两个进程，以确保关键代码段的独占式访问的一个经典例子如下图所示。
>
>   ![image-20230313165922165](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230313165922165.png)
>
>   在图中，当关键代码段可用时，二进制信号量SV的值为1，进程A和B都有机会进入关键代码段。如果此时进程A执行了P(SV)操作就将SV减1，则进程B若再执行P(SV)操作就会被挂起。直到进程A离开关键代码段，并执行V(SV)操作将SV加1，关键代码段才重新变得可用。如果此时进程B因为等待SV而处于挂起状态，则将它唤醒，并进入关键代码段。同样，这时进程A如果再执行P(SV)操作，则也只能被操作系统挂起以等待进程B退出关键代码段。
>
>   > <font color=green>**注意：使用一个普通变量来模拟二进制信号量时行不通的，因为所有高级语言都没有一个原子操作可以同时完成如下两步操作：检测变量是否为true/false，如果是则再将它设置为false/true。**</font>
>
>   ​	Linux信号量的API都定义在sys/sem.h头文件中，主要包含3个系统调用：**semget、semop和semctl**。它们都被设计为操作一组信号量，即信号量集，而不是单个信号量，因此这些接口看上去多少比我们期望的要复杂一点。

### 8.5.2 Semget系统调用

> ​	semget系统调用创建一个新的信号量集，或者获取一个已经存在的信号量集。其定义如下：
>
> ```c
> #include <sys/sem.h>
> int semget(key_t key, int num_sems, int sem_flags);
> ```
>
> ​	key参数是一个键值，**用来标识一个全局唯一的信号量集**，就像文件名全局唯一地标识一个文件一样。要通过信号量通信的进程需要使用相同的键值来创建/获取该信号量。
>
> ​	num_sems参数指定要创建/获取的信号量集中**信号量的数目**。如果是创建信号量，则该值必须被指定；<font color=green>**如果是已经存在的信号量，则可以把它设置为0.**</font>
>
> ​	sem_flags参数指定一组标志。**它低端的9个比特是该信号量的权限，其格式和含义都与相同调用open的mode参数相同。**此外，它还可以和IPC_CREAT标志做按位“或”运算以创建新的信号量级。此时即使信号量集已经存在，semget也不会产生错误。我们还可以联合使用<font color=red>**IPC_CREAT和IPC_EXCL标志来确保创建一组新的，唯一的信号量集。在这种情况下，如果信号量集已经存在，则semget返回错误并设置errno为EEXIST，可用于创建信号量集。**</font>这种创建信号量的行为与用O_CREATE和O_EXCL标志调用open来排他式地打开一个文件相似。
>
> ​	semget成功时返回一个整数数值，它是信号量集的标识符；semget失败时返回-1，并设置errno。
>
> ​	**如果semget用于创建信号量集，则与之关联的内核数据结构体semid_ds将被创建并初始化。semid_ds结构体的定义如下：**
>
> ```c
> #include <sys/sem.h>
> /* 该结构体用于描述IPC对象（信号量、共享内存和消息队列）的权限 */
> struct ipc_perm
> {
>     key_t key;	/* 键值 */
>     uid_t uid;	/* 所有者的有效用户ID */
>     gid_t gid;	/* 所有者的有效组ID */
>     uid_t cuid;	/* 创建者的有效用户ID */
>     gid_t cgid;	/* 创建者的有效组ID */
>     mode_t mode;	/* 访问权限 */
>     				/* 省略其他填充字段 */
> }
> 
> struct semid_ds
> {
>     struct ipc_perm sem_perm;	/* 信号量的操作权限 */
>     unsigned long int sem_nsems;	/* 该信号量集中的信号量数目 */
>     time_t sem_otime;	/* 最后一次调用semop的时间 */
>     time_t sem_ctime;	/* 最后一次调用semctl的时间 */
>     					/* 省略其他填充字段 */
> }
> ```
>
> semget对semid_ds结构体的初始化包括：
>
> * 将sem_perm.cuid和sem_perm.uid设置为调用进程的有效用户ID。
> * 将sem_perm.cgid和sem_perm.gid设置为调用进程的有效组ID。
> * 将sem_perm.mode的最低9位设置为sem_flags参数的最低9位。
> * 将sem_nsems设置为num_sems。
> * 将sem_otime设置为0.
> * 将sem_ctime设置为当前的系统时间。

### 8.5.3 semop系统调用

> ​	**semop系统调用改变信号量的值，即执行P、V操作。**在讨论semop之前，我们需要先介绍与每个信号量关联的一些重要的内核变量：
>
> ```c
> unsigned short semval;	/* 信号量的值 */
> unsigned short semzcnt;	/* 等待信号量变为0的进程数量 */
> unsigned short semncnt;	/* 等待信号量值增加的进程数量 */
> pid_t sempid;	/* 最后一次执行semop操作的进程ID */
> ```
>
> ​	semop对信号量的操作实际上就是对这些内核变量的操作。semop的定义如下：
>
> ```c
> #include <sys/sem.h>
> int semop(int sem_id, struct sembuf* sem_ops, size_t num_sem_ops);
> ```
>
> ​	sem_id参数是由semget调用返回的信号量集标识符，用以指定被操作的目标信号量集。
>
> sem_ops参数指向一个sembuf结构体类型的数组，sembuf结构体的定义如下：
>
> ```c
> struct sembuf
> {
> 	unsigned short int sem_num;
> 	short int sem_op;
> 	short int sem_flg;
> }
> ```
>
> ​	其中，sem_num成员是信号量集中信号量的编号，0表示信号量集中的第一个信号量。sem_op成员指定操作类型，其可选值为正整数、0和负整数。每种类型的操作的行为又受到sem_flag成员的影响。**sem_flag的可选值是IPC_NOWAIT和SEM_UNDO。**IPC_NOWAIT的含义是，无论信号量操作是否成功，semop调用都将立即返回，这类似于非阻塞I/O操作。SEM_UNDO的含义是，当进程退出时取消正在进行的semop操作。具体来说，sem_op和sem_flg将按照如下方式来影响semop的行为：
>
> * 如果sem_op大于0<font color=green><b>（V操作）</b></font>，则semop将被操作的信号量的值semval增加sem_op。该操作要求调用进程对操作信号量集拥有<font color=green><b>写权限</b></font>。此时若设置了SEM_UNDO标志，则系统将更新进程的semadj变量（用以跟踪进程对信号量的修改情况）。
>
> * 如果sem_op等于0，则表示这是一个“等待0”（wait-for-zero）操作。**该操作要求调用进程对被操作信号拥有读权限**。如果此时信号量的值是0，则调用立即成功返回。如果信号量的值不是0，则semop失败返回或者阻塞进程以等待信号量变为0.在这种情况下，当IPC_NOWAIT标志被指定时，semop立即返回一个错误，并设置errno为EAGAIN。 如果为未指定IPC_NOWAIT标志，则信号量值的semzcnt值加1，进程被投入睡眠直到下列3个条件之一发生：信号量的值semval变为0，此时系统将该信号量的semzcnt值减1；被操作信号量所在的信号量集被进程移除，此时semop调用失败，errno被设置为EIDRM；调用被信号中断，此时semop调用失败返回，errno被设置为EINTR，同时系统将该信号量的semzcnt值减1.
>
> * 如果sem_op小于0，则表示对信号量值进行减操作<font color=green><b>（P操作）</b></font>，即期望获得信号量。该操作要求调用进程对操作信号量集拥有<font color=green><b>写权限</b></font>。如果信号量的值semval大于或等于sem_op的绝对值，则semop操作成功，调用进程立即获得信号量，并且系统将该信号量的semval值减去sem_op的绝对值。此时如果设置了SEM_UNDO标志，则系统将更新进程的semadj变量。如果信号量的值semval小于sem_op的绝对值，则semop失败返回或者阻塞进程以等待信号量可用。在这种情况下，当IPC_NOWAIT标志被指定时，semop立即返回一个错误，并设置errno为EAGAIN。如果未指定IPC_NOWAIT标志，则信号量的semncnt值加1，进程被投入睡眠直到下列3个条件之一发生：信号量的值semval变得大于或等于sem_op的绝对值，此时系统将该信号量的semncnt值减1，并将semval减去sem_op的绝对值，同时，如果SEM_UNDO标志被设置，则系统更新semadj变量；被操作信号量所在的信号量集被进程移除，此时semop调用失败返回，errno被设置未EINTR，同时系统将该信号量的semncnt值减1.
>
>   ​	semop系统调用的第3个参数num_sem_ops指定要执行的操作个数，即sem_ops数组中元素的个数。semop对数组sem_ops中的每个成员按照数组顺序依次执行操作，**并且该过程是原子操作**，以避免别的进程在同一时刻按照不同的顺序对该信号集中的信号量执行semop操作导致的竞态条件。
>
>   ​	semop成功返回0，失败则返回-1并设置errno。失败的时候，sem_ops数组中指定的所有操作都不被执行。

### 8.5.4 semctl系统调用

> semctl系统调用允许调用者对信号量进行直接控制。其定义如下：
>
> ```c
> #include <sys/sem.h>
> int semctl(int sem_id, int sem_num, int command, ...);
> ```
>
> ​	sem_id参数是由semget调用返回的信号量集标识符，用以指定被操作的信号量集。sem_num参数指定被操作的信号量在信号量集中的编号。command参数指定要执行的命令。有的命令需要调用者二传递第四个参数。第4个参数的类型由用户自己定义，但sys/sem.h头文件给出了它的推荐格式，具体如下：
>
> ```c
> union semun
> {
> 	int val;	/* 用于SETVAL命令 */
> 	struct semid_ds* buf;	/* 用于IPC_STAT和IPC_SET命令 */
> 	unsigned short* array;	/* 用于GETALL和SETALL命令 */
> 	struct seminfo* __buf;	/* 用于IPC_INFO命令 */
> };
> 
> struct seminfo
> {
> 	int semmap;		/* Linux内核没有使用 */
> 	int semmni;		/* 系统最多可以拥有的信号量集数目 */
> 	int semmns;		/* 系统最多可以拥有的信号量数目 */
> 	int semmnu;		/* Linux 内核没有使用 */
> 	int semmsl;		/* 一个信号量集最多允许包含的信号量数目 */
> 	int semopm;		/* semop一次最多能执行的sem_op操作数目 */
> 	int semume;		/* Linux内核没有使用 */
> 	int semusz;		/* sem_undo结构体大小 */
> 	int semvmx;		/* 最大允许的信号量值 */
> 	int semaem;		/* 最多允许的UNDO次数（带SEM_UNDO标志的semop操作的次数） */
> };
> ```
>
> ​	semctl支持的所有命令如下表所示：
>
> | 命令     | 含义                                                         | semctl成功时的返回值                       |
> | -------- | ------------------------------------------------------------ | ------------------------------------------ |
> | IPC_STAT | 将信号量集关联的内核数据结构复制到semun.buf中                | 0                                          |
> | IPC_SET  | 将semun.buf中的部分成员复制到信号量集关联的内核数据结构中，同时内核数据中的semid_ds.sm_ctime被更新 | 0                                          |
> | IPC_RMID | 立即移除信号量集，唤醒所有等待该信号量集的进程（semop返回错误，并设置errno为EIDRM） | 0                                          |
> | IPC_INFO | 获取系统信号量资源配置信息，将结果存储在semun.__buf中。这些信息的含义见结构体seminfo的注释部分 | 内核信号量集数组中已经被使用项的最大索引值 |
> | SEM_INFO | 与IPC_INFO类似，不过此时sem_id参数不是用来表示信号量集标识符，而是内核中信号量集数组的索引（系统的所有信号量集都是该数组中的一项） | 同IPC_INFO                                 |
> | SEM_STAT | 与IPC_STAT类似，不过semun.\_\_buf.semusz被设置为系统目前拥有的信号量集数目，而semnu.\_\_buf.semaem被设置为系统目前拥有的信号量数目 | 同IPC_INFO                                 |
> | GETALL   | 将由sem_id标识的信号量集中的所有信号量的semval值导出到semun.array中 | 0                                          |
> | GETNCNT  | 获取信号量的semncnt值                                        | 信号量的semncnt值                          |
> | GETPID   | 获取信号量的sempid值                                         | 信号量的sempid值                           |
> | GETVAL   | 获得信号量的semval值                                         | 信号量的semval值                           |
> | GETZCNT  | 获得信号量的semzcnt值                                        | 信号量的semzcnt值                          |
> | SETALL   | 用semun.array中的数据填充由sem_id标识的信号量集中的所有信号量的semval值，同时内核数据中的semid_ds.sem_ctime被更新 | 0                                          |
> | SETVAL   | 将信号量的semval值设置为semun.val，同时内核数据中的semid_ds.sem_ctime被更新 | 0                                          |
>
> 注意：这些操作中，GETNCNT、GETPID、GETVAL、GETZCNT和SETVAL操作的是单个信号量，它是由标识符sem_id指定的信号量集中的第sem_num个信号量；而其他操作针对的是整个信号量集，此时semctl的参数sem_num被忽略。
>
> ​	semctl成功时的返回值取决于command参数，如上表所示。semctl失败时返回-1，并设置errno。

### 8.5.5 特殊键值IPC_PRIVATE

> ​	semget的调用者可以给其key参数传递应该特殊的键值IPC_PRIVATE(其值为0)，这样无论该信号量是否已经存在，semget都将创建一个新的信号量。使用该键值创建的信号量并非像它的名字声称的那样是进程私有的。其他进程，尤其是子进程，也有方法来访问这个信号量。所以semget的man手册的BUTGS部分上说，使用名字IPC_PRIVATE有些误导（历史原因），应该称为IPC_NEW。比如下面的代码就在父、子进程间使用应该IPC_PRIVATE信号量来同步。
>
> ```c
> #include <sys/sem.h>
> #include <stdio.h>
> #include <stdlib.h>
> #include <unistd.h>
> #include <sys/wait.h>
> 
> union semun
> {
>     int val;
>     struct semid_ds* buf;
>     unsigned short int* array;
>     struct seminfo* __buf;
> };
> 
> /* op为-1时执行P操作，op为1时执行V操作 */
> void pv(int sem_id, int op)
> {
>     struct sembuf sem_b;
>     sem_b.sem_num = 0;
>     sem_b.sem_op = op;
>     sem_b.sem_flg = SEM_UNDO;
>     semop(sem_id, &sem_b, 1);
> }
> 
> int main(int argc, char* argv[])
> {
>     int sem_id = semget(IPC_PRIVATE, 1, 0666);
> 
>     union semun sem_un;
>     sem_un.val = 1;
>     semctl(sem_id, 0, SETVAL, sem_un);
> 
>     pid_t id = fork();
>     if(id < 0)
>     {
>         return 1;
>     }
>     else if(id == 0)
>     {
>         printf("child try to get binary sem\n");
>         /* 在父、子进程间共享IPC_PRIVATE信号量的关键就在于二者都可以操作该信号量的标识符sem_id */
>         pv(sem_id, -1); /* P操作 */
>         printf("child get the sem and would release it after 5 seconds\n");
>         sleep(5);
>         pv(sem_id, 1); /* V操作 */
>         exit(0);
>     }
>     else
>     {
>         printf("parent try to get binary sem\n");
>         pv(sem_id, -1);
>         printf("parent get the sem and would release it after 5 seconds\n");
>         sleep(5);
>         pv(sem_id, 1);
>     }
> 
>     waitpid(id, NULL, 0);
>     semctl(sem_id, 0, IPC_RMID, sem_un);    /* 删除信号量集 */
>     return 0;
> }
> ```
>
> ​	另外一个例子是：工作在prefork模式下的httpd网页服务器程序使用一个IPC_PRIVATE信号量来同步各子进程对epoll_wait的调用权。下面我们简单分析一下这个例子。在测试机器Kongming20上，使用strace命令依次查看httpd的各子进程是如何协调工作的：
>
> ![image-20230427114808163](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230427114808163.png)
>
> ​	由此可见，httpd的子进程1703~1708和710都在等待信号量393222（这是一个信号量集标识符）可用：只有进程1709暂时拥有该信号量，因为进程1709调用epoll_wait以等待新的客户端连接。当有新连接到来时，进程1709将接受之，并对信号量393222执行V操作，此时将有另外一个子进程获得信号量并调用epoll_wait来等待新的客户连接。那么我们如何知道信号量393222是使用键值IPC_PRIVATE创建的呢？答案将在8.8节揭晓。
>
> ​	下面要讨论另外两种IPC————共享内存和消息队列。这两种IPC在创建资源的时候也支持IPC_PRIVATE键值，其含义与信号量的IPC_PREVATE键值完全相同。

## 8.6 共享内存

> ​	共享内存是最高效的IPC机制，因为它不涉及进程之间的任何数据传输。这种高效率带来的问题是，我们必须用其他辅助手段来同步进程对共享内存的访问，否则会产生竞态条件。因此，共享内存通常和其他进程间通信方式一起使用。
>
> ​	Linux共享内存的API都定义在sys/shm.h头文件中，包括4个系统调用：shmget、shmat、和shmctl。

### 8.6.1 shmget相同调用

> ​	shmget系统调用创建一段新的共享内存，或者获取一段已经存在的共享内存。其定义如下：
>
> ```c
> #include <sys/shm.h>
> int shmget(key_t key, size_t size, int shmflg);
> ```
>
> ​	和semget系统调用一样，key参数是一个键值，用来标识一段全局唯一的共享内存。
>
> size参数指定共享内存的大小，单位是字节。如果是创建新的共享内存，则size值必须被指定。如果是获取已经存在的共享内存，则可以把size设置为0.
>
> ​	shmflg参数的使用和含义与sem_flags参数相同。不过shmget支持两个额外的标志————SHM_HUGETLB和SHM_NORESERVE。它们的含义如下：
>
> * SHM_HUGETLB，类似于mmap的MAP_HUGETLB标志，系统将使用“大页面”来来为共享内存分配空间。
>
> * SHM_NORESERVE，类似于mmap的MAP_NORESERVEE标志，不为共享内存保留交换区（swap空间）。这样，当物理内存不足的时候，对该共享内存执行写操作将触发SIGSEGV信号。
>
>   shmget成功时返回一个正整数值，它是共享内存的标识符。shmget失败时返回-1，并设置errno。
>
>   如果shmget用于创建共享内存，则这段共享内存的所有字节都被初始化为0，与之关联的内核数据结构shmid_ds将被创建并初始化。shmid_ds结构体的定义如下：
>
>   ```c
>   struct shmid_ds
>   {
>   	struct ipc_perm shm_perm;	/* 共享内存的操作权限 */
>   	size_t shm_segsz;	/* 共享内存大小，单位是字节 */
>   	__time_t shm_atime;	/* 对这段内存最后一次调用shmat的时间 */
>   	__time_t shm_dtime;	/* 对这段内存最后一次调用shmdt的时间 */
>   	__time_t shm_ctime;	/* 对这段内存最后一次调用shmctl的时间 */
>   	__pid_t shm_cpid;	/* 创建者的PID */
>   	__pid_t shm_lpid;	/* 最后一次执行shmat或shmdt操作的进程的PID */
>   	shmatt_t shm_nattach;	/* 目前关联到此共享内存的进程数量 */
>   	/* 省略一些填充字段 */
>   };
>               
>   /* 该结构体用于描述IPC对象（信号量、共享内存和消息队列）的权限 */
>   struct ipc_perm
>   {
>    key_t key;	/* 键值 */
>    uid_t uid;	/* 所有者的有效用户ID */
>    gid_t gid;	/* 所有者的有效组ID */
>    uid_t cuid;	/* 创建者的有效用户ID */
>    gid_t cgid;	/* 创建者的有效组ID */
>    mode_t mode;	/* 访问权限 */
>    				/* 省略其他填充字段 */
>   }
>   ```
>
>   shmget对shmid_ds结构体的初始化包括：
>
>   * 将shm_perm.cuid和shm_perm.uid设置为调用进程的有效用户ID。
>   * 将shm_perm.cgid和shm_perm.gid设置为调用进程的有效组ID。
>   * 将shm_perm.mode的最低9位设置为shmflg参数的最低9位。
>   * 将shm_segsz设置为size。
>   * 将shm_lpid、shm_nattach、shm_atime、shm_dtime设置为0。
>   * 将shm_ctime设置为当前的时间。

### 8.6.2 shmat和shmdt系统调用

> ​	共享内存被创建/获取之后，我们不能立即访问它，而是需要先将它关联到进程的地址空间中。使用完共享内存之后，我们也需要将它从进程地址空间中分离。这两项任务分别由如下两个系统调用实现：
>
> ```c
> #include <sys/shm.h>
> void* shmat(int shm_id, const void* shm_addr, int shmflg);
> int shmdt(const void* shm_addr);
> ```
>
> ​	其中，shm_id参数是由shmget调用返回的共享内存标识符。shm_addr参数指定将共享内存关联到进程的哪块地址空间，最终的效果还受到shmflg参数的可选标志SHM_RND的影响：
>
> * 如果shm_addr为NULL，则被关联的地址由操作系统选择。这是推荐的做法，以确保代码的可移植性。
> * 如果shm_addr非空，并且SHM_RND标志未被设置，则共享内存被关联到addr指定的地址处。
> * 如果shm_addr非空，，并且设置了SHM_RND标志，则被关联的地址是[shm_addr-(shm_addr%SHMLBA)]。SHMLBA的含义是“段低端边界地址倍数”（Segment Low Boundary Address Multiple），它必须是内存页大小（PAGE_SIZE）的整数倍。现在的Linux内核中，它等于一个内存页的大小。SHM_RND的含义是圆整（round），即将共享内存被关联的地址向下圆整到离shm_addr最近的SHMLBA的整数倍地址处。
>
> 除了SHM_RND标志外，shmflg参数还支持如下标志：
>
> * SHM_RDONLY。进程仅能读取共享内存中的内容。若没有指定该标志，则进程可同时对共享内存进行读写操作（当然，这需要创建共享内存的时候指定其读写权限）。
> * SHM_REMAP。如果地址shmaddr已经倍关联到一段共享内存上，则重新关联。
> * SHM_EXEC。它指定对共享内存段的执行权限。对共享内存而言，执行权限实际上和读权限是一样的。
>
> shmat成功是返回共享内存被关联到的地址，失败则返回（void*)-1并设置errno。shmat成功时，将修改内核数据结构shmid_ds的部分字段，如下：
>
> * 将shm_nattach加1
> * 将shm_lpid设置为调用进程的PID。
> * 将shm_atime设置为当前的时间。
>
> shmdt函数将关联到shm_addr处的共享内存从进程中分离。它成功时返回0，失败则返回-1并设置errno。shmdt在成功调用时将修改内核数据结构shmid_ds的部分字段，如下：
>
> * 将shm_nattach减1。
> * 将shm_lpid设置为调用进程的PID。
> * 将shm_dtime设置为当前的时间。

### 8.6.3 shmctl系统调用

> ​	shmctl系统调用控制共享内存的某些属性。其定义如下：
>
> ```c
> #include <sys/shm.h>
> int shmctl(int shm_id, int command, struct shmid_ds* buf);
> ```
>
> ​	其中，shm_id参数是由shmget调用返回的共享内存标识符。command参数指定要执行的命令。shmctl支持的所有命令如表所示：
>
> | 命令       | 含义                                                         | shmctl成功时的返回值                                   |
> | ---------- | ------------------------------------------------------------ | ------------------------------------------------------ |
> | IPC_ATAT   | 将共享内存相关的内核数据结构复制到buf（第3个参数，下同）中   | 0                                                      |
> | IPC_SET    | 将buf中部分成员复制到共享内存相关的内核数据结构中，同时内核数据中的shmid_ds.shm_ctime被更新 | 0                                                      |
> | IPC_RMID   | 将共享内存打上删除的标记，这样当最后一个使用它的进程调用shmdt将它从进程中分离时，该共享内存就被删除了 | 0                                                      |
> | IPC_INFO   | 获取系统共享内存资源配置信息，将结果存储在buf中。应用程序需要将buf转换成shminfo结构体类型来读取这些系统信息。shminfo结构体与seminfo类似。 | 内核共享内存信息数组中已经被使用的项的最大索引值       |
> | SHM_STAT   | 与IPC_STAT类似，不过此时shm_id参数不是用来表示共享内存标识符，而是内核中共享内存信息数组的索引（每个共享内存的信息都是该数组中的一项） | 内核共享内存信息数组中索引值为shm_id的共享内存的标识符 |
> | SHM_LOCK   | 禁止共享内存被移动到交换分区                                 | 0                                                      |
> | SHM_UNLOCK | 允许共享内存被移动至交换分区                                 | 0                                                      |
>
> ​	shmctl成功时返回值取决于command参数，如表所示。shmctl失败时返回-1，并设置errno。

### 8.6.4 共享内存的POSIX方法

> ​	前面的1.5章节中介绍过mmap函数。利用它的MAP_ANONYMOUS标志我们可以实现父、子进程之间的匿名共享内存。通过打开同一个文件，mmap也可以实现无关进程之间的内存共享。Linux提供了另外一种利用mmap在无关进程之间共享内存的方式。这种方式无须任何文件的支持，但它需要先使用如下函数来创建或打开一个POSIX共享内存对象：
>
> ```c
> #include <sys/mman.h>
> #include <sys/stat.h>
> #include <fcntl.h>
> int shm_open(const char* name, int oflag, modea_t mode);
> ```
>
> ​	shm_open的使用方法与open系统调用完全相同。
>
> ​	name参数指定要创建/打开的共享内存对象。从可移植性的角度考虑，该参数应该使用"/somename"的格式：以"/"开始，后接多个字符，且这些字符都不是"/"：以"\0"结尾，长度不超过NAME_MAX（通常是255）。
>
> ​	oflag参数指定创建方式。它可以是下列标志中的一个或者多个的按位或：
>
> * O_RDONLY。以只读方式打开共享内存对象。
>
> * O_RDWR。以可读、可写方式打开共享内存对象。
>
> * O_CREAT。如果共享内存对象不存在，则创建之。此时mode参数的最低9位将指定该共享内存对象的访问权限。共享内存对象被创建的时候，其初始长度为0。
>
> * O_EXCL。和O_CREAT一起使用，如果由name指定的共享内存对象已经存在，则shm_open调用返回错误，否则就创建一个新的共享内存对象。
>
> * O_TRUNC。如果共享内存对象已经存在，则把它截断，使其长度为0。
>
>   shm_open调用成功时返回一个文件描述符。该文件描述符可用于后续的mmap调用，从而将共享内存关联到调用进程。shm_open失败时返回-1，并设置errno。
>
>   和打开的文件最后需要关闭一样，由shm_open创建的共享内存对象使用完之后也需要被删除。这个过程是通过如下函数实现的：
>
>   ```c
>   #include <sys/mman.h>
>   #include <sys/stat.h>
>   #include <fcntl.h>
>   int shm_unlink(const char *name);
>   ```
>
>   该函数将name参数指定的共享内存对象标记为等待删除。当所有该共享内存对象的进程都使用ummap将它从进程中分离之后，系统将销毁这共享对象所占据的资源。如果代码使用了上述POSIX共享内存函数，则编译的时候需要指定链接选项-lrt。
>
> 在Linux系统中使用共享内存，一般用到以下几个函数：
>
> ```c
> int shm_open(const char* name, int oflag, mode_t mode);
> int ftruncate(int fd, off_t length);
> void *mmap(void *addr, size_t length, int prot, int flages, int fd, off_t offset);
> int munmap(void *addr, size_t length);
> int shm_unlink(const char *name);
> ```

### 8.6.5 共享内存实例

> ​	在4.6.2小节中，介绍过一个聊天服务器程序。下面将它修改为一个多进程服务器：一个子进程处理一个客户连接。同时，我们将所有客户socket连接的读缓冲区设计为一块共享内存，如代码所示：
>
> ```c
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <signal.h>
> #include <sys/wait.h>
> #include <sys/mman.h>
> #include <sys/stat.h>
> #include <fcntl.h>
> #include <stdbool.h>
> 
> #define USER_LIMIT 5
> #define BUFFER_SIZE 1024
> #define FD_LIMIT 65535
> #define MAX_EVENT_NUMBER 1024
> #define PROCESS_LIMIT 65536
> 
> /* 处理一个客户连接必要的数据 */
> struct client_data
> {
>     struct sockaddr_in address;    /* 客户端的socket地址 */
>     int connfd;     /* socket文件描述符 */
>     pid_t pid;      /* 处理这个连接的子进程的PID */
>     int pipefd[2];  /* 和父进程通信用的管道 */
> };
> 
> static const char* shm_name = "/my_shm";
> int sig_pipefd[2];
> int epollfd;
> int listenfd;
> int shmfd;
> char* share_mem = 0;
> /* 客户连接数组，进程用客户连接的编号来索引这个数组，即可取得相关的开花连接数据 */
> struct client_data* users = 0;
> /* 子进程和客户连接的映射关系表，用进程的PID来索引这个数组，即可取得该进程所处理的客户连接的编号 */
> int* sub_process = 0;
> /* 当前客户数量 */
> int user_count = 0;
> bool stop_child = false;
> 
> int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> void addfd(int epollfd, int fd)
> {
>     struct epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET;
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> 
> void sig_handler(int sig)
> {
>     int save_errno = errno;
>     int msg = sig;
>     send(sig_pipefd[1], (char*)&msg, 1, 0);
>     errno = save_errno;
> }
> 
> void addsig(int sig, void(*handler)(int), bool restart = true)
> {
>     struct sigaction sa;
>     memset(&sa, '\0', sizeof(sa));
>     sa.sa_handler = handler;
>     if(restart)
>     {
>         sa.sa_flags |= SA_RESTART;  /* 重新启动被中断的系统调用 */
>     }
>     sigfillset(&sa.sa_mask); /* 信号处理过程中，屏蔽信号 */
>     assert(sigaction(sig, &sa, NULL) != -1); 
> }
> 
> void del_resource()
> {
>     close(sig_pipefd[0]);
>     close(sig_pipefd[1]);
>     close(listenfd);
>     shm_unlink(shm_name);
>     delete [] users;
>     delete [] sub_process;
> }
> 
> /* 停止一个子进程 */
> void child_term_handler(int sig)
> {
>     stop_child = true;
> }
> 
> /* 子进程允许的函数， 参数idx指出该子进程处理的客户连接的编号，users是保存所有客户连接数据的数组
> ，参数share_mem指出共享内存的起始地址 */
> int run_child(int idx, client_data* users, char* share_mem)
> {
>     epoll_event events[MAX_EVENT_NUMBER];
>     /* 子进程使用I/O复用技术来同时监听两个文件描述符：客户连接socket、与父进程通信的管道文件描述符 */
>     int child_epollfd = epoll_create(5);
>     assert(child_epollfd != -1);
>     int connfd = users[idx].connfd;
>     addfd(child_epollfd, connfd);
>     int pipefd = users[idx].pipefd[1];
>     addfd(child_epollfd, pipefd);
>     int ret;
> 
>     /* 子进程需要设置自己的信号处理函数 */
>     addsig(SIGTERM, child_term_handler, false);
> 
>     while(!stop_child)
>     {
>         int number = epoll_wait(child_epollfd, events, MAX_EVENT_NUMBER, -1);
>         if((number < 0) && (errno != EINTR))
>         {
>             printf("epoll failure\n");
>             break;
>         }
> 
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             /* 本子进程负责的客户连接的数据到达 */
>             if((sockfd == connfd) && (events[i].events & EPOLLIN))
>             {
>                 memset(share_mem + idx*BUFFER_SIZE, '\0', BUFFER_SIZE);
>                 /* 将客户数据读取到对应的读缓存中。该读缓存时共享内存的一段，它开始于idx*BUFFER_SIZE处，
>                 长度为BUFFER_SIZE字节。因此，各个客户连接的读缓存是共享的 */
>                 ret = recv(connfd, share_mem + idx*BUFFER_SIZE, BUFFER_SIZE-1, 0);
>                 if(ret < 0)
>                 {
>                     if(errno != EAGAIN)
>                     {
>                         stop_child = true;
>                     }
>                 }
>                 else if(ret == 0) /* 客户端关闭了连接 */
>                 {
>                     stop_child = true;
>                 }
>                 else
>                 {
>                     /* 成功读取客户数据后通知主进程（通过管道）来处理 */
>                     send(pipefd, (child*)&idx, sizeof(idx), 0);
>                 }
>             }
> 
>             /* 主进程通知本进程（通过管道）将第client个客户的数据发送到本进程负责的客户端 */
>             else if((sockfd == pipefd) && (events[i].events & EPOLLIN))
>             {
>                 int client = 0;
>                 /* 接收主进程发送来的数据，即有客户数据到达的连接的编号 */
>                 ret = recv(sockfd, (char*)&client, sizeof(client), 0);
>                 if(ret < 0)
>                 {
>                     if(errno != EAGAIN)
>                     {
>                         stop_child = true;
>                     }
>                 }
>                 else if(ret == 0)
>                 {
>                     stop_child = true;
>                 }
>                 else
>                 {
>                     send(connfd, share_mem + client * BUFFER_SIZE, BUFFER_SIZE, 0);
>                 }
>             }
>             else
>             {
>                 continue;
>             }
>             close(connfd);
>             close(pipefd);
>             close(child_epollfd);
>             return 0;
>         }
>     }
> }
> 
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizoef(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
> 
>     user_count = 0;
>     users = new client_data[USER_LIMIT+1];
>     sub_process = new int[PROCESS_LIMIT];
>     for(int i = 0; i < PROCESS_LIMIT; ++i)
>     {
>         sub_process[i] = -1;
>     }
> 
>     epoll_event events[MAX_EVENT_NUMBER];
>     epollfd = epoll_create(5);
>     assert(epollfd != -1);
>     addfd(epollfd, listenfd);
> 
>     ret = socketpair(PF_UNIX, SOCK_STREAM, 0, sig_pipefd);
>     assert(ret != -1);
>     setnonblocking(sig_pipefd[1]);
>     setnonblocking(sig_pipefd[0]);
>     addfd(epollfd, sig_pipefd[0]);
> 
>     addsig(SIGCHLD, sig_handler);
>     addsig(SIGTERM, sig_handler);
>     addsig(SIGINT, sig_handler);
>     addsig(SIGPIPE, SIG_IGN); /* 忽略该信号 */
>     bool stop_server = false;
>     bool terminate = false;
> 
>     /* 创建共享内存，作为所有开花socket连接的读缓存 */
>     shmfd = shm_open(shm_name, O_CREAT | O_RDWR, 0666);
>     assert(shmfd != -1);
>     /* 设置共享内存大小 */
>     ret = ftruncate(shmfd, USER_LIMIT * BUFFER_SIZE);
>     assert(ret != -1);
> 
>     /* 将共享内存映射到进程的地址空间 */
>     share_mem = (char*)mmap(NULL, USER_LIMIT* BUFFER_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, shmfd, 0);
>     assert(share_mem != MAP_FAILED);
>     close(shmfd);
> 
>     while(!stop_server)
>     {
>         int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
>         if((number < 0) && (errno != EINTR))
>         {
>             printf("epoll failure\n");
>             break;
>         }
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             /* 新的客户连接到来 */
>             if(sockfd == listenfd)
>             {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof(client_address);
>                 int connfd = accept(listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                 if(connfd < 0)
>                 {
>                     printf("errno is: %d\n", errno);
>                     continue;
>                 }
>                 if(user_count >= USER_LIMIT)
>                 {
>                     const char* info = "too many users\n";
>                     printf("%s", info);
>                     send(connfd, info, strlen(info), 0);
>                     close(connfd);
>                     continue;
>                 }
>                 /* 保存第user_count个客户连接的相关数据 */
>                 users[user_count].address = client_address;
>                 users[user_count].connfd = connfd;
>                 /* 在主进程和子进程间建立管道， 以传递必要的数据 */
>                 ret = socketpair(PF_UNIX, SOCK_STREAM, 0, users[user_count].pipefd);
>                 assert(ret != -1);
> 
>                 pid_t pid = fork();
>                 if(pid < 0)
>                 {
>                     close(connfd);
>                     continue;
>                 }
>                 else if(pid == 0)
>                 {
>                     close(epollfd);
>                     close(listenfd);
>                     close(users[user_count].pipefd[0]);
>                     close(sig_pipefd[0]);
>                     close(sig_pipefd[1]);
>                     run_child(user_count, users, share_mem);
>                     munmap((void*)share_mem, USER_LIMIT * BUFFER_SIZE);
>                     exit(0);
>                 }
>                 else
>                 {
>                     close(connfd);
>                     close(users[user_count].pipefd[1]);
>                     addfd(epollfd, users[user_count].pipefd[0]);
>                     users[user_count].pid = pid;
>                     /* 记录新的客户连接的在数组users中的索引值，建立进程pid和该索引值之间的映射关系 */
>                     sub_process[pid] = user_count;
>                     user_count++;
>                 }
>             }
>             /* 处理信号事件 */
>             else if((sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN))
>             {
>                 int sig;
>                 char signals[1024];
>                 ret = recv(sig_pipefd[0], signals, sizeof(signal), 0);
>                 if(ret == -1)
>                 {
>                     continue;
>                 }
>                 else if(ret == 0)
>                 {
>                     continue;
>                 }
>                 else
>                 {
>                     for(int i = 0; i < ret; ++i)
>                     {
>                         switch(signals[i])
>                         {
>                             /* 子进程退出，表示有某个客户端关闭了连接 */
>                             case SIGCHLD:
>                             {
>                                 pid_t pid;
>                                 int stat;
>                                 while((pid = waitpid(-1, &stat, WNOHANG)) > 0)
>                                 {
>                                     /* 用子进程的pid取得被关闭的客户连接的编号 */
>                                     int del_user = sub_process[pid];
>                                     sub_process[pid] = -1;
>                                     if((del_user < 0) || (del_user > USER_LIMIT))
>                                     {
>                                         continue;
>                                     }
>                                     /* 清除第del_user个客户连接使用的相关数据 */
>                                     epoll_ctl(epollfd, EPOLL_CTL_DEL, users[del_user].pipefd[0], 0);
>                                     close(users[del_user].pipefd[0]);
>                                     users[del_user] = users[--user_count];
>                                     sub_process[users[del_user].pid] = del_user;
>                                 }
>                                 if(terminate && user_count == 0)
>                                 {
>                                     stop_server = true;
>                                 }
>                                 break;
>                             }
>                             case SIGTERM:
>                             case SIGINT:
>                             {
>                                 /* 结束服务器程序 */
>                                 printf("kill all the clild now\n");
>                                 if(user_count == 0)
>                                 {
>                                     stop_server = true;
>                                     break;
>                                 }
>                                 for(int i = 0; i < user_count; ++i)
>                                 {
>                                     int pid = users[i].pid;
>                                     kill(pid, SIGTERM);
>                                     terminate = true;
>                                     break;
>                                 }
>                                 default:
>                                 {
>                                     break;
>                                 }
>                             }
>                         }
>                     }
>                 }
>                 /* 某个子进程向父进程写入了数据 */
>                 else if(events[i].events & EPOLLIN)
>                 {
>                     int child = 0;
>                     /* 读取管道数据， child变量记录了是哪个客户连接有数据到达 */
>                     ret = recv(sockfd, (char*)&child, sizeof(child), 0);
>                     printf("read data from child accross pipe\n");
>                     if(ret == -1)
>                     {
>                         continue;
>                     }
>                     else if(ret == 0)
>                     {
>                         continue;
>                     }
>                     else
>                     {
>                         /* 向除负责处理第child个客户连接的子进程之外的其他子进程发送消息，通知它们有客户数据要写 */
>                         for(int j = 0; j < user_count; ++j)
>                         {
>                             if(users[j].pipefd[0] != sockfd)
>                             {
>                                 printf("send data to child accross pipe\n");
>                                 send(users[j].pipefd[0], (char*)&child, sizeof(child), 0);
>                             }
>                         }
>                     }
>                 }
>             }
>         }
>     }
>     del_resource();
>     return 0;
> }
> ```
>
> 上面的代码有两点需要注意：
>
> * 虽然我们使用了共享内存，但每个子进程都只会往自己所处理的客户连接所对应的那一部分缓存中写入数据，所以我们使用共享内存的目的只是为了“共享读”。因此，每个子进程在使用共享内存的时候都无须加锁。
> * 我们的服务器程序在启动的时候给数组users分配了足够多的空间，使得它可以存储所有可能的客户连接的相关数据。同样，我们一次性给数组sub_process分配的空间也足以存储所有可能的子进程的相关数据。

## 8.7 消息队列

> ​	消息队列是在两个进程之间传递二进制块数据的一种简单有效的方式。每个数据块都有一个特定的类型，接收方可以根据类型来选择地接收数据，而不一定像管道和命名管道那样必须以先进先出的方式接收数据。
>
> ​	Linux消息队列的API都定义在sys/msg.h头文件中，包括4个系统调用：msgget、msgsnd、msgrcv和msgctl。我们将依次讨论之。

### 8.7.1 msgget系统调用

> msgget系统调用创建一个消息队列，或者获取一个已有的消息队列。其定义如下：
>
> ```c
> #include <sys/msg.h>
> int msgget(key_t key, int msgflg);
> ```
>
> ​	和semget系统调用一样，key参数是一个键值，用来标识一个全局唯一的消息队列。
> ​	msgflg参数的使用和含义与semget系统调用的sem_flags参数相同。
> ​	msgget成功时返回一个正整数值，它是消息队列的标识符。msgget失败时返回-1，并设置errno。
> ​	如果msgget用于创建消息队列，则与之关联的内核数据结构msqid_ds将被创建并初始化。msqid_ds结构体的定义如下：
>
> ```c
> struct msqid_ds
> {
> 	struct ipc_perm msg_perm;	/* 消息队列的操作权限 */
> 	time_t msg_stime;	/* 最后一次调用msgsnd的时间 */
> 	time_t msg_rtime;	/* 最后一次调用msgrcv的时间 */
> 	time_t msg_ctime;	/* 最后一次被修改的时间 */
> 	unsigned long __msg_cbytes;	/* 消息队列中已有的字节数 */
> 	msgqnum_t msg_qnum;	/* 消息队列中已有的消息数 */
> 	msglen_t msg_qbytes;	/* 消息队列允许的最大字节数 */
> 	pid_t msg_lspid;	/* 最后执行msgsnd的进程的PID */
> 	pid_t msg_lrpid;	/* 最后执行msgrcv的进程的PID */
> }
> ```

### 8.7.2 msgsnd系统调用

> ​	msgsnd系统调用把一条消息添加到消息队列中，其定义如下：
>
> ```c
> #include <sys/msg.h>
> int msgsnd(int msqid, const void* msg_ptr, size_t msg_sz, int msgflg);
> ```
>
> ​	msqid参数是由msgget调用返回的消息队列标识符。
>
> msgptr参数指向一个准备发送的消息，消息必须被定义为如下类型：
>
> ```c
> struct msgbuf
> {
> 	long mtype;	/* 消息类型 */
> 	char mtext[512]; /* 消息数据 */
> };
> ```
>
> ​	其中，mtype成员指定消息的类型，它必须是一个正整数。mtext是消息数据。msg_sz参数是消息的数据部分（mtext)的长度。这个长度可以为0，表示没有消息数据。
>
> ​	msgflg参数控制msgsnd的行为。它通常仅支持IPC_NOWAIT标志，即以非阻塞的方式发送消息。默认情况下，发送消息时如果消息队列满了，则msgsnd将阻塞。若IPC_NOWAIT标志被指定，则msgsnd将立即返回并设置errno为EAGAIN。
>
> ​	处于阻塞状态的msgsnd调用可能被如下两种异常情况所中断：
>
> * 消息队列被移除。此时msgsnd调用将立即返回并设置errno为EIDRM。
> * 程序接收到信号。此时msgsnd调用将立即返回并设置errno为EINTR。
>
> msgsnd成功时返回0，失败则返回-1并设置errno。msgsnd成功时将修改内核数据结构msqid_ds的部分字段，如下所示：
>
> * 将msg_qnum加1
> * 将msg_lspid设置为调用进程的PID。
> * 将msg_stime设置为当前的时间。

### 8.7.3 msgrcv系统调用

> ​	msgrcv系统调用从消息队列获取消息。其定义如下：
>
> ```c
> #include <sys/msg.h>
> int msgrcv(int msqid, void* msg_ptr, size_t msg_sz, long int msgtype, int msgflg);
> ```
>
> ​	msqid参数是由msgget调用返回的消息队列标识符。
>
> msg_ptr参数用于存储接收的消息，msg_sz参数指的是消息数据部分的长度。
>
> msgtype参数指定接收何种类型的消息。我们可以使用如下几种方式来指定消息类型：
>
> * msgtype等于0.读取消息队列中的第一个消息。
> * msgtype大于0.读取消息队列中第一个类型为msgtype的消息（除非指定了标志MSG_EXCEPT）。
> * msgtype小于0.读取消息队列中第一个类型值比msgtype的绝对值小的消息。
>
> 参数msgflg控制msgrcv函数的行为。它可以是如下一些标志的按位或：
>
> * IPC_NOWAIT。如果消息队列中没有消息，则msgrcv调用立即返回并设置errno为ENOMSG。
> * MSG_EXCEPT。如果msgtype大于0，则接收消息队列中第一个非msgtype类型的消息。
> * MSG_NOERROR。如果消息数据部分的长度超过了msg_sz，就将它截断。
>
> 处于阻塞状态的msgrcv调用还可能被如下两种异常情况所中断：
>
> * 消息队列被移除。此时msgrcv调用将立即返回并设置errno为EIDRM。
> * 程序接收到信号。此时msgrcv调用将立即返回并设置errno为EINTR。
>
> msgrcv成功时返回0，失败则返回-1并设置errno. msgrcv成功时将修改内核数据结构msqid_ds的部分字段，如下所示：
>
> * 将msg_qnum减1.
> * 将msg_lrpid设置为调用进程进程的PID。
> * 将msg_rtime设置为当前的时间。

### 8.7.4 msgctl系统调用

> msgctl系统调用控制消息队列的某些属性。其定义如下：
>
> ```c
> #include <sys/msg.h>
> int msgctl(int msqid, int command, struct msqid_ds* buf);
> ```
>
> ​	msqid参数是由msgget调用返回的共享内存标识符。command参数指定要执行的命令。msgctl支持的所有命令如下表所示：
>
> | 命令     | 含义                                                         | msgctl成功时的返回值                                  |
> | -------- | ------------------------------------------------------------ | ----------------------------------------------------- |
> | IPC_STAT | 将消息队列关联的内核数据结构复制到buf中                      | 0                                                     |
> | IPC_SET  | 将buf中的部分成员复制到消息队列关联的内核数据结构中，同时内核数据中的msqid_ds.msg_ctime被更新 | 0                                                     |
> | IPC_RMID | 立即移除消息队列，唤醒所有等待读消息和写消息的进程（这些调用立即返回并设置errno为EIDRM） | 0                                                     |
> | IPC_INFO | 获取系统消息队列资源配置信息，将结果存储在buf中。应用程序需要将buf转换成msginfo结构体类型来读取这些系统信息。msginfo结构体与seminfo类似。 | 内核消息队列信息数组中已经被使用的项的最大索引值      |
> | MSG_INFO | 与IPC_INFO类似，不过返回的是已经分配的消息队列占用的资源     | 同IPC_INFO                                            |
> | MSG_STAT | 与IPC_STAT类似，不过此时msqid参数不是用来表示消息队列标识符，而是内核消息队列信息数组的索引（每个消息队列的信息都是该数组中的一项） | 内核消息队列信息数组中索引值为msqid的消息队列的标识符 |
>
> msgctl成功时的返回值取决于command参数。msgctl函数失败时返回-1并设置errno。

## 8.8 IPC命令

> ​	上述3种System V IPC进程间通信方式都使用一个全局唯一的键值（key）来描述一个共享资源。当程序调用semget、shmget或者msgget时，就创建了这些共享资源的一个实例。Linux提供了ipcs命令，以观察当前系统上拥有哪些共享资源实例。比如：
>
> ![image-20230510171606822](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230510171606822.png)
>
> ​	输出结果分段显示了系统拥有的共享内存、信号量和消息队列资源。可见，该系统目前尚未使用任何共享内存和消息队列，却分配了一组键值为0（IPC_PRIVATE）的信号量。这些信号量的所有者是apache，因此它们是由httpd服务器程序构建的。其中标识符为393222的信号量正是8.5.5小节讨论的那个用于在httpd各个子进程之间同步epoll_wait使用权的信号量。
>
> ​	此外，我们可以使用ipcrm命令来删除遗留在系统中的共享资源。

## 8.9 在进程间传递文件描述符

> ​	由于fork调用之后，父进程中打开的文件描述符在子进程中仍然保持打开，所以文件描述符可以很方便地从父进程传递到子进程。需要注意的是，传递一个文件描述符并不是传递一个文件描述符的值，而是要在接收过程中创建一个新的文件描述符，并且该文件描述符和发送进程中被传递的文件描述符**指向内核中相同的文件表项**。
>
> ​	那么如何把子进程中打开的文件描述符传递给父进程呢？或者更通俗地说，如何在两个不相干的进程之间传递文件描述符呢？在Linux下，我们可以利用UNIX域socket在进程间传递特殊的辅助数据，以实现文件描述符的传递。以下代码给出了一个实例，它在子进程打开一个文件描述符，然后将它传递给父进程，父进程则通过读取该文件描述符来获得文件的内容。
>
> ```c
> #include <sys/socket.h>
> #include <fcntl.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <stdlib.h>
> #include <assert.h>
> #include <string.h>
> 
> static const int CONTROL_LEN = CMSG_LEN(sizeof(int));
> /* 发送文件描述符，fd参数是用来传递信息的UNIX域socket，fd_to_send参数是待发送的文件描述符 */
> void send_fd(int fd, int fd_to_send)
> {
>     struct iovec iov[1];
>     struct msghdr msg;
>     char buf[0];
> 
>     iov[0].iov_base = buf;
>     iov[0].iov_len = 1;
>     msg.msg_name = NULL;
>     msg.msg_namelen = 0;
>     msg.msg_iov = iov;
>     msg.msg_iovlen = 1;
> 
>     cmsghdr cm;
>     cm.cmsg_len = CONTROL_LEN;
>     cm.cmsg_level = SOL_SOCKET;
>     cm.cmst_type = SCM_RIGHTS;
>     *(int *)CMSG_DATA(&cm) = fd_to_send;
> 
>     msg.msg_control = &cm; /* 设置辅助数据 */
>     msg.msg_controllen = CONTROL_LEN;
> 
>     sendmsg(fd, &msg, 0);
> }
> 
> /* 接收目标文件描述符 */
> int recv_fd(int fd)
> {
>     struct iovec iov[1];
>     struct msghdr msg;
>     char buf[0];
> 
>     iov[0].iov_base = buf;
>     iov[0].iov_len = 1;
>     msg.msg_name = NULL;
>     msg.msg_namelen = 0;
>     msg.msg_iov = iov;
>     msg.msg_iovlen = 1;
> 
>     cmsghdr cm;
>     msg.msg_control = &cm;
>     msg.msg_controllen = CONTROL_LEN;
> 
>     recvmsg(fd, &msg, 0);
> 
>     int fd_to_read = *(int *)CMSG_DATA(&cm);
>     return fd_to_read;
> }
> 
> int main()
> {
>     int pipefd[2];
>     int fd_to_pass = 0;
>     /* 创建父、子进程间的管道，文件描述符pipefd[0]和pipefd[1]都是UNIX域socket */
>     int ret = socketpair(PF_UNIX, SOCK_DGRAM, 0, pipefd);
>     assert(ret != -1);
> 
>     pid_t pid = fork();
>     assert(pid >= 0);
> 
>     if(pid == 0)
>     {
>         close(pipefd[0]);
>         fd_to_pass = open("test.txt", O_RDWR, 0666);
>         /* 子进程通过管道文件描述符发送到父进程，如果文件test.txt打开失败，则子进程将标准
>         输入文件描述符发送到父进程 */
>         send_fd(pipefd[1], (fd_to_pass > 0) ? fd_to_pass : 0);
>         close(fd_to_pass);
>         exit(0);
>     }
> 
>     close(pipefd[1]);
>     fd_to_pass = recv_fd(pipefd[0]); /* 父进程从管道接收目标文件描述符 */
>     char buf[1024];
>     memset(buf, '\0', 1024); 
>     read(fd_to_pass, buf, 1024); /* 读目标文件描述符，以验证其有效性 */
>     printf("I got fd %d and data %s\n", fd_to_pass, buf);
>     close(fd_to_pass);
> }
> ```

# 9 多线程编程

> ​	早期Linux不支持线程，直到1996年，Xavier Leroy等人才开发出第一个基本符合POSIX标准的线程库LinuxThreads。但LinuxThreads效率低而且问题很多。自内核2.6开始，Linux才真正提供内核级的线程支持，并有两个组织致力于编写新的线程库：NGPT(Next Generation POSIX Threads)和NPTL(Native POSIX Thread Library)。不过前者在2003年就放弃了，因此新的线程库就称为NPTL。NPTL比LinuxThreads效率高，且更符合POSIX规范，所以它已经成为glibc的一部分。本书所有线程相关的例程使用的线程库都是NPTL。
>
> ​	本章要讨论的线程相关的内容都属于POSIX线程（简称pthread）标准，而不局限于NPTL实现，具体包括：
>
> * 创建线程和结束线程。
>
> * 读取和设置线程属性
>
> * POSIX线程同步方式：POSIX信号量，互斥锁和条件变量。
>
>   在本章的最后，我们还将介绍在Linux环境下，库函数、进程、信号与多线程程序之间的相互影响。

## 9.1 Linux线程概述

### 9.1.1 线程模型

> ​	线程是程序中完成一个独立任务的完整执行序列，即一个可调度的实体。根据运行环境和调度者的身份，线程可分为内核线程和用户线程。内核线程，在有的系统上也称为LWP(Light Weight Process, 轻量级进程)，运行在内核空间，由内核来调度；用户线程运行在用户空间，由线程库来调度。当进程的一个内核线程获得CPU的使用权时，它就加载并运行一个用户线程。可见，内核线程相当于用户线程运行的“容器”。一个进程可以拥有M个内核线程和N个用户线程，其中$M \leq N$。并且在一个系统的所有进程中，$M$和$N$的比值都是固定的。按照$M:N$的取值，线程的实现方式可分为三种模式：完全在用户空间实现、完全由内核调度和双层调度(two level scheduler)。
>
> ​	完全在用户空间实现的线程无须内核的支持，内核甚至不知道这些线程的存在。线程库负责管理所有执行的线程，比如线程的优先级、时间片等。线程库利用`longjmp`来切换线程的执行，使它们看起来像是“并发”执行的。但实际上内核仍然是把整个进程作为最小单位来调度的。换句话说，一个进程的所有执行线程共享该进程的时间片，它们对外表现出相同的优先级。因此，就实现方式而言，$N=1$，即$M$个用户空间线程对应一个内核线程，而该内核线程实际上就是进程本身。完全在用户空间实现的线程的优点是：创建和调度线程都无须内核的干预，因此速度相当快。并且由于它不占用额外的内核资源，所以即使一个进程创建了很多线程，也不会对系统性能造成明显的影响。其缺点是：对于多处理器系统，一个进程的多个线程无法运行在不同的CPU上，因为内核是按照其最小调度单位来分配CPU的。此外，线程的优先级只对同一个进程中的线程有效，比较不同进程中的线程的优先级没有意义。早期的伯克利UNIX线程就是采用这种方式实现的。
>
> ​	完全由内核调度的模式将创建、调度线程的任务都交给了内核，运行在用户空间的线程库无须执行管理任务，这与完全在用户空间实现的线程恰恰相反。二者的优缺点也正好互换。较早的Linux内核对内核线程的控制能力有限，线程库通常还要提供额外的控制能力，尤其是线程同步机制，不过现代Linux内核已经大大增强了对线程的支持。完全由内核调度的这种线程实现方式满足$M:N=1:1$，即1个用户空间线程被映射为1个内核线程。
>
> ​	双层调度模式是前两种实现模式的混合体：内核调度$M$个内核线程，线程库调度$N$个用户线程。这种线程实现方式结合了前两种方式的优点：不但不会消耗过多的内核资源，而且线程切换速度也比较快，同时它可以充分利用多处理器的优势。

### 9.1.2 Linux线程库

> ​	Linux上两个最有名的线程库是LinuxThreads和NPTL，它们都是采用$1:1$的方式实现的。由于LinuxThreads在开发的时候，Linux内核对线程的支持还非常有限，所以其可用性、稳定性以及POSIX兼容性都远远不及NPTL。现代Linux上默认使用的线程库是NPTL。用户可以使用如下命令来查看当前系统上所使用的线程库：
>
> ```shell
> $ getconf GNU_LIBPTHREAD_VERSION
> NPTL 2.14.90
> ```
>
> ​	LinuxThreads线程库的内核线程是用clone系统调用创建的进程模拟的。clone系统调用和fork系统调用作用类似：**创建调用进程的子进程**。不过我们可以为clone系统调用指定CLONE_THREAD标志，这种情况下它创建的子进程与调用进程共享相同的**虚拟地址空间、文件描述符和信号处理函数**，这些都是线程的特点。不过，用进程来模拟内核线程会导致很多语义问题，比如：
>
> * 每个线程拥有不同的PID，因此不符合POSIX规范。
> * Linux信号处理本来是基于进程的，但现在一个进程内的所有线程都能而且必须处理信号。
> * 用户ID、组ID对一个进程中的不同线程来说可能是不一样的。
> * 程序产生的核心转储文件不会包含所有线程的信息，而只包含产生该核心转储文件的线程的信息。
> * 由于每个线程都是一个进程，因此系统允许最大进程数也就是最大线程数。
>
> LinuxThreads线程库一个有名的特性是所谓的管理线程。它是进程中专门用于管理其他工作线程的线程。其作用包括：
>
> * 系统发送给进程的终止信号先由管理线程接收，管理线程再给其他工作线程发送同样的信号以终止它们。
> * 当终止工作线程或者工作线程主动退出时，管理线程必须等待它们结束，以避免僵尸进程。
> * 如果主线程先于其他工作线程退出，则管理线程将阻塞它，直到所有其他工作线程都结束之后才唤醒它。
> * 回收每个线程堆栈使用的内存。
>
> 管理线程的引入，增加了额外的系统开销。并且由于它只能运行在一个CPU上，所以LinuxThreads线程库也不能充分利用多处理器系统的优势。
>
> ​	要解决LinuxThreads线程库的一系列问题，不仅需要改进线程库，最主要的是需要内核提高更完善的线程主持。因此，Linux内核2.6版本开始，提供了真正的内核线程。新的NPTL线程库也应运而生。相比LinuxThread，NPTL的主要优势在于：
>
> * 内核线程不再是一个进程，因此**避免了很多用进程模拟内核线程导致的语义问题**。
> * **摒弃了管理线程，终止线程、回收线程堆栈等工作都可以由内核来完成**。
> * **由于不存在管理线程，所以一个进程的线程可以运行在不同的CPU上，从而充分利用了多处理器系统的优势**。
> * 线程的同步由内核来完成。**隶属于不同进程的线程之间也能共享互斥锁**，因此可实现跨进程的线程同步。

## 9.2 创建线程和结束线程

> ​	下面我们讨论创建和结束线程的基础API。Linux系统上，它们都定义在pthread.h头文件中。
>
> 1. **pthread_create**
>
> 创建一个线程的函数是pthread_create。其定义如下：
>
> ```c
> #include <pthread.h>
> int pthread_create(pthread_t* thread, const pthread_attr_t attr, void* (*start_routine)(void*), void* arg);
> ```
>
> ​	thread参数是新线程的标识符，后续pthread_*函数通过它来引用新线程。其类型pthread_t的定义如下：
>
> ```c
> #include <bits/pthreadtypes.h>
> typedef unsigned long int pthread_t
> ```
>
> ​	可见，pthread_t是一个整型类型。实际上，Linux上几乎所有的资源标识符都是一个整型数，比如socket、各种System V IPC标识符等。
>
> ​	attr参数用于设置新线程的属性。给它传递NULL表示使用默认线程属性。线程拥有众多属性，我们将在后面详细讨论之。start_routine和arg参数分别指定新线程将运行的函数及其参数。
>
> ​	pthread_create成功时返回0，**失败时返回错误码**。一个用户可以打开的线程数量不能超过RLIMIT_NPROC软资源限制（见表4-1）。此外，系统上所有用户能创建的线程总数也不得超过/proc/sys/kernel/threads-max内核参数所定义的值。
>
> 2. **pthread_exit**
>
>    线程一旦被创建好，内核就可以调度内核线程来执行start_routine函数所指向的函数了。**线程函数在结束时最好调用如下函数**，以确保安全、干净地退出：
>
> ```c
> #include <pthread.h>
> void pthread_exit(void* retval);
> ```
>
> ​	pthread_exit函数通过retval参数向线程的回收者传递其退出信息。它执行完之后不会返回到调用者，而且永远不会失败。
>
> 3. **pthread_join**
>
> **一个进程中的所有线程都可以调用pthread_join函数来回收其他线程（前提是目标线程是可以回收的，见后文**），即等待其他线程结束，这类似于回收进程的wait和waitpid系统调用。prhread_join的定义如下：
>
> ```c
> #include <pthread.h>
> int pthread_join(pthread_t thread, void** retval);
> ```
>
> ​	thread参数是目标线程的标识符，**retval参数则是目标线程返回的退出信息**。该函数会一直阻塞，直到被回收的线程结束为止。该函数成功时返回0，失败则返回错误码。可能的错误如下表所示：
>
> | 错误码  | 描述                                                         |
> | ------- | ------------------------------------------------------------ |
> | EDEADLK | 可能引起死锁。比如两个线程相互针对对方调用pthread_join，或者线程对自身调用pthread_join |
> | EINVAL  | 目标线程是不可回收的，或者已经有其他线程在回收该目标线程     |
> | ESRCH   | 目标线程不存在                                               |
>
> 4. **pthread_cancel**
>
>    有时候我们希望异常终止一个线程，即取消线程，它是通过如下函数实现的：
>
> ```c
> #include <pthread.h>
> int pthread_cancel(pthread_t thread);
> ```
>
> ​	thread参数是目标线程的标识符。该函数成功时返回0，失败则返回错误码。不过，接收到取消请求的目标线程可以决定是否允许被取消以及如何取消，这分别由如下两个函数完成：
>
> ```c
> #include <pthread.h>
> int pthread_setcancelstate(int state, int *oldstate);
> int pthread_setcanceltype(int type, int *oldtype);
> ```
>
> ​	这两个函数的第一个参数分别用于设置线程的取消状态（是否允许取消）和取消类型（如何取消），第二个参数则分别记录线程原来的取消状态和取消类型。state参数有两个可选值：
>
> * PTHREAD_CANCEL_ENABLE，允许线程被取消。它是线程被创建时的默认取消状态。
> * PTHREAD_CANCEL_DISBALE，禁止线程被取消。这种情况下，如果一个线程收到取消请求，则它会将请求挂起，直到该线程允许被取消。
>
> type参数也有两个可选值：
>
> * PTHREAD_CANCEL_ASYNCHRONOUS，线程随时都可以被取消。它将使得接收到取消请求的目标线程立即采取行动。
> * PTHREAD_CANCEL_DEFERRED，允许目标线程推迟行动，直到它调用了下面几个所谓的取消点函数中的一个：pthread_join、pthread_testcancel、pthread_cond_wait、pthread_cond_timedwait、sem_wait和sigwait。根据POSIX标准，其他可能阻塞的系统调用，比如read、wait，也可以成为取消点。不过为了安全起见，我**们最好在可能会被取消的代码中调用pthread_testcancel函数以设置取消点**。
>
> pthread_setcancelstate和pthread_setcancletype成功时返回0，失败则返回错误码。

## 9.3 线程属性

> ​	pthread_attr_t结构体定义了一套完整的线程属性，如下所示：
>
> ```c
> #include <bits/pthreadtypes.h>
> #define __SIZEOF_PTHREAD_ATTR_T 36
> typedef union
> {
> 	char __size[__SIZEOF_PTHREAD_ATTR_T];
> 	long int __align;
> }pthread_attr_t;
> ```
>
> ​	可见，各种线程属性全部包含在一个字符数组中。线程库定义了一系列函数来操作pthread_attr_t类型的变量，以方便我们获取和设置线程属性。这些函数包括：
>
> ```c
> #include <pthread.h>
> /* 初始化线程属性对象 */
> int pthread_attr_init(pthread_attr_t* attr);
> /* 销毁线程属性对象，被销毁的线程属性对象只有再次初始化之后才能继续使用 */
> int pthread_attr_destroy(pthread_attr_t* attr);
> /* 下面这些函数用于获取和设置线程属性对象的某个属性 */
> int pthread_attr_getdetachstate(const pthread_attr_t* attr, int* detachstate);
> int pthread_attr_setdetachstate(pthread_attr_t* attr, int detachstate);
> int pthread_attr_getstackaddr(const pthread_attr_t* attr, void ** stackaddr);
> int pthread_attr_setstackaddr(pthread_attr_t* attr, void* stackaddr);
> int pthread_attr_getstacksize(const pthread_attr_t* attr, size_t* stacksize);
> int pthread_attr_setstacksize(pthread_attr_t* attr, size_t stacksize);
> int pthread_attr_getstack(const pthread_attr_t* attr, void** stackaddr, size_t* stacksize);
> int pthread_attr_setstack(pthread_attr_t* attr, void* stackaddr, size_t stacksize);
> int pthread_attr_getguardsize(const pthread_attr_t* attr, size_t* guardsize);
> int pthread_attr_setgrardsize(pthread_attr_t* attr, size_t guardsize);
> int pthread_attr_getschedparam(const pthread_attr_t* attr, struct sched_param* param);
> int pthread_attr_setschedparam(pthread_attr_t* attr, const struct sched_param* param);
> int pthread_attr_getschedpolicy(const pthread_attr_t* attr, int* policy);
> int pthread_attr_setschedpolicy(pthread_attr_t* attr, int policy);
> int pthread_attr_getinheritsched(const pthread_attr_t* attr, int* inherit);
> int pthread_attr_setinheritsched(pthread_attr_t* attr, int inherit);
> int pthread_attr_getscope(const pthread_attr_t* attr, int* scope);
> int pthread_attr_setscope(pthread_attr_t* attr, int scope);
> ```
>
> 下面我们详细讨论每个线程属性的含义：
>
> * detachstate，线程的脱离状态。它有PTHREAD_CREATE_JOINABLE和PTHREAD_CREATE_DETACH两个可选值。前者指定线程是可以被回收的，后者使调用线程脱离与进程中其他线程的同步。脱离了与其他线程的同步的线程称为“脱离线程”。脱离线程在退出时将自行释放其占用的系统资源。线程创建时该属性默认值是PTHREAD_CREATE_JOINBLE。此外，我们也可以使用pthread_detach函数直接将线程设置为脱离线程。
> * stackaddr和stacksize，线程堆栈的起始地址和大小。一般来说，我们不需要自己来管理线程堆栈，因为Linux默认为每个线程分配了足够的堆栈空间（一般是8MB）。我们可以使用`ulimt -s`命令来查看或修改这个默认值。
> * guardsize，保护区大小。如果guardsize大于0，则系统创建线程的时候会在其堆栈的尾部额外分配guardsize字节的空间，作为保护堆栈不被错误地覆盖的区域。如果guardsize等于0，则系统不为新创建的线程设置堆栈保护区。如果使用者通过pthread_attr_setstackaddr或pthread_attr_setstack函数手动设置线程的堆栈，则guardsize属性将被忽略。
> * schedparam，线程调度参数。其类型是sched_param结构体。该结构体目前还只有一个整型的成员————sched_priority，该成员表示线程的运行优先级。
> * schedpolicy，线程调度策略。该属性有SCHED_FIFO、SCHED_RR和SCHED_OTHER三个可选值，其中SCHED_OTHER是默认值。SCHED_RR表示采用轮转算法（round-robin）调度，SCHED_FIFO表示使用先进先出的方法调度，这两种调度方法都具备实时调度功能，但只能用于超级用户身份运行的进程。
> * inheritsched，是否继承调用线程的调度属性。该属性有PTHREAD_INHERIT_SCHED和PTHREAD_EXPLICIT_SCHED两个可选值。前者表示新线程沿用其创建者的线程的调度参数，这种情况下再设置线程的调度参数属性将没有任何效果。后者表示调用者要明确地指定新线程的调度参数。
> * scope，线程间竞争CPU的范围，即线程优先级的有效范围。POSIX标准定义了该属性的PTHREAD_SCOPE_SYSTEM和PTHREAD_SCOPE_PROCESS两个可选值，前者表示目标线程与系统中所有线程一起竞争CPU的使用，后者表示目标线程仅与其它隶属于同一进程的线程竞争CPU的使用。目前Linux只支持PTHREAD_SCOPE_SYSTEM这一种取值。

## 9.4 POSIX信号量

> ​	和多进程程序一样，多线程程序也必须考虑同步问题。pthread_join可以看作一种简单的线程同步方式，不过很显然，它无法高效地实现复杂的同步需求，比如控制对共享资源的独占式访问，又抑或是在某个条件满足之后唤醒一个线程。接下来我们讨论3种专门用于线程同步的机制：POSIX信号量、互斥量和条件变量。
>
> ​	在Linux上，信号量API有两组。一组是第8章讨论过的System V IPC信号量，另外一组是我们现在要讨论的POSIX信号量。这两组接口很相似，但不保证能互换。由于这两组信号量的语义完全相同，因此我们不再赘述信号量的原理。
>
> ​	POSIX信号量函数的名字都以sem_开头，并不像大多数线程函数那样以pthread\_开头。常用的POSIX信号量函数是下面5个：
>
> ```c
> #include <semaphore.h>
> int sem_init(sem_t* sem, int pshared, unsigned int value);
> int sem_destroy(sem_t* sem);
> int sem_wait(sem_t* sem);
> int sem_trywait(sem_t* sem);
> int sem_post(sem_t* sem);
> ```
>
> ​	这些函数的第一个参数sem指向被操作的信号量。
>
> ​	sem_init函数用于初始化一个未命名的信号量（POSIX信号量API支持命名信号量，不过这里不讨论它）。pshared参数指定信号量的类型。如果其值为0，就表示这个信号量是当前进程的局部信号量，否则该信号量就可以在多个进程之间共享。**value参数指定信号量的初始值**。此外，初始化一个已经被初始化的信号量将导致不可预测的结果。
>
> ​	sem_destroy函数用于销毁信号量，以释放其占用的内核资源。**如果销毁一个正被其他线程等待的信号量，则将导致不可预期的结果。**
>
> ​	sem_wait函数以原子操作的方式将信号量的值减1.如果信号量的值为0，则sem_wait将被阻塞，直到这个信号量具有非0值。
>
> ​	sem_trywait与sem_wait函数相似，不过它始终立即返回，而不论被操作的信号量是否具有非0值，相当于sem_wait的非阻塞版本。当信号量的值非0时，sem_trywait对信号量执行减1操作。当信号量的值为0时，它将返回-1并设置errno为EAGAIN.
>
> ​	sem_post函数以原子操作的方式将信号量的值加1.当信号量的值大于0时，其他正在调用sem_wait等待的线程将被唤醒。
>
> ​	上面这些函数成功时返回0，失败则返回-1并设置errno。

## 9.5 互斥锁

> ​	互斥锁（也称互斥量）可以用于保护关键代码段，以确保其独占式的访问，这有点像一个二进制信号量。当进入关键代码段时，我们需要获得互斥锁并将其加锁，这等价于二进制信号量的P操作；当离开关键代码段时，我们需要对互斥锁解锁，以唤醒其他等待该互斥锁的线程，这等价于二进制信号量的V操作。

### 9.5.1 互斥锁基础API

> POSIX互斥锁的相关函数主要有如下5个：
>
> ```c
> #include <pthread.h>
> int pthread_mutex_init(pthread_mutex_t* mutex, const pthread_mutexattr_t* mutexattr);
> int pthread_mutex_destroy(pthread_mutex_t* mutex);
> int pthread_mutex_lock(pthread_mutex_t* mutex);
> int pthread_mutex_trylock(pthread_mutex_t* mutex);
> int pthread_mutex_unlock(pthread_mutex_t* mutex);
> ```
>
> ​	这些函数的第一个参数mutex指向要操作的目标互斥锁，互斥锁的类型是pthread_mutex_t结构体。
>
> ​	pthread_mutex_init函数用于初始化互斥锁。mutexattr参数指定互斥锁的属性。如果将它设置为NULL，则表示使用默认属性。我们将在下一小节讨论互斥锁的属性。除了这个函数外，我们还可以使用如下方式来初始化一个互斥锁：
>
> ```c
> pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
> ```
>
> ​	宏PTHREAD_MUTEX_INITIALIZER实际上只是把互斥锁的各个字段都初始化为0.
>
> ​	pthread_mutex_destroy函数用于销毁互斥锁，以释放其占用的内核资源。**销毁一个已经加锁的互斥锁将导致不可预期的后果**。
>
> ​	pthread_mutex_lock函数以原子操作的方式给一个互斥锁加锁。如果目标互斥锁已经被锁上，则pthread_mutex_lock调用将阻塞，直到该互斥锁的占有者将其解锁。
>
> ​	pthread_mutex_trylock与pthread_mutex_lock函数类似，不过它始终立即返回，而不论被操作的互斥锁是否已经被加锁，相当于pthread_mutex_lock的非阻塞版本。当目标互斥锁未被加锁时，pthread_mutex_trylock对互斥锁执行加锁操作。当互斥锁已经被加锁时，pthread_mutex_trylock将返回错误码EBUSY。需要注意的是，这里讨论的pthread_mutex_lock和pthread_mutex_trylock的行为是针对普通锁而言的。后面我们将看到，对于其他类型的锁而言，这两个加锁函数会有不同的行为。
>
> ​	pthread_mutex_unlock函数以原子操作的方式给一个互斥锁解锁。如果此时有其他线程正在等待这个互斥锁，则这些线程中的某一个线程将获得它。
>
> ​	对于上面这些函数成功时返回0，失败则返回错误码。

### 9.5.2 互斥锁属性

> ​	pthread_mutexattr_t结构体定义了一套完整的互斥锁属性。线程库提供了一系列函数来操作pthread_mutex_attr_t类型的变量，以方便我们获取和设置互斥锁属性。这里我们列车其中一些主要的函数。
>
> ```c
> #include <pthread.h>
> /* 初始化互斥锁属性对象 */
> int pthread_mutexattr_init(pthread_mutexattr_t* attr);
> /* 销毁互斥锁属性对象 */
> int pthread_mutexattr_destroy(pthread_mutexattr_t* attr);
> /* 获取和设置互斥锁的pshared属性 */
> int pthread_mutexattr_getpshared(const pthread_mutexattr_t* attr, int* pshared);
> int pthread_mutexattr_setpshared(pthread_mutexattr_t* attr, int pshared);
> /* 获取和设置互斥锁的type属性 */
> int pthread_mutexattr_gettype(const pthread_mutexattr_t* attr, int* type);
> int pthread_mutexattr_settype(pthread_mutexattr_t* attr, int type);
> ```
>
> ​	我们只讨论互斥锁的两种常用属性：pshared和type。互斥锁属性pshared指定是否允许跨进程共享互斥锁，其可选值有两个：
>
> * PTHREAD_PROCESS_SHARED。互斥锁可以被跨进程共享。
> * PTHREAD_PROCESS_PRIVATE。互斥锁只能被和锁的初始化线程隶属于同一个进程的线程共享。
>
> 互斥锁属性type指定互斥锁的类型。Linux支持如下4种类型的互斥锁：
>
> * PTHREAD_MUTEX_NORMAL，普通锁。这是互斥锁的默认的类型。当一个线程对一个普通锁加锁以后，其余请求该锁的线程将形成一个等待队列，并在该锁解锁后按优先级获得它。这种锁类型保证了资源分配的公平性。但这种锁也很容易引发问题：**一个线程如果对一个已经加锁的普通锁再次加锁，将引发死锁**；对一个已经被其他线程加锁的普通锁解锁，或者对一个已经解锁的普通锁解锁，将导致不可预期的后果。
> * PTHREAD_MUTEX_ERRORCHECK，检错锁。一个线程如果对一个已经被加锁的检错锁再次加锁，则加锁操作返回EDEADLK。对一个已经被其他线程加锁的检错锁解锁，或者对一个已经解锁的检错锁再次解锁，则解锁操作返回EPERM。
> * PTHREAD_MUTEX_RECURSIVE，嵌套锁。这种锁允许一个线程在释放之前多次对它加锁而不发生死锁。不过其他线程如果要获得这个锁，则当前锁拥有者必须执行相应次数的解锁操作。对一个已经被其他线程加锁的嵌套锁解锁，或者对一个已经解锁的嵌套锁再次解锁，则解锁操作返回EPERM。
> * PTHREAD_MUTEX_DEFAULT，默认锁。一个线程如果对一个已经加锁的默认锁再次加锁，或者对一个已经被其他线程加锁的默认锁解锁，或者对一个已经解锁的默认锁再次解锁，将导致不可预期的结果。这种锁在实现的时候可能被映射为上面三种锁之一。

### 9.5.3 死锁举例

> ​	使用互斥锁的一个噩耗是死锁。死锁使得一个或多个线程被挂起而无法继续执行，而且这种情况还不容易被发现。前文提到，在一个线程中对一个已经加锁的普通锁再次加锁，将导致死锁。这种情况可能出现在设计得不够仔细的递归函数中。另外，如果两个线程按照不同的顺序来申请两个互斥锁，也容易产生死锁，如以下代码所示。
>
> ```c
> #include <pthread.h>
> #include <unistd.h>
> #include <stdio.h>
> 
> int a = 0;
> int b = 0;
> pthread_mutex_t mutex_a;
> pthread_mutex_t mutex_b;
> 
> void* another(void* arg)
> {
> 	pthread_mutex_lock(&mutex_b);
> 	printf("in child thread, got mutex b, waiting for mutex a\n");
> 	sleep(5);
> 	++b;
> 	pthread_mutex_lock(&mutex_a);
> 	b += a++;
> 	pthread_mutex_unlock(&mutex_a);
> 	pthread_mutex_unlock(&mutex_b);
> 	pthread_exit(NULL);
> }
> 
> int main()
> {
>     pthread_t id;
>     pthread_mutex_init(&mutex_a, NULL);
>     pthread_mutex_init(&mutex_b, NULL);
>     
>     pthread_mutex_lock(&mutex_a);
>     printf("in parent thread, got mutex a, waiting for mutex b\n");
>     sleep(5);
>     ++a;
>     pthread_mutex_lock(&mutex_b);
>     pthread_mutex_unlock(&mutex_a);
>     
>     pthread_join(id, NULL);
>     pthread_mutex_destroy(&mutex_a);
>     pthread_mutex_destroy(&mutex_b);
>     
>     return 0;
> }
> ```
>
> ​	代码中，主线程试图先占有互斥锁mutex_a，然后操作被该锁的保护的变量a，但存在完毕之后，主线程并没有立即释放互斥锁mutex_a，而是又申请互斥锁mutex_b，并在两个互斥锁的保护下，操作变量a和b，最后才一起释放这两个互斥锁；与此同时，子线程按照相反的顺序申请互斥锁mutex_a和mutex_b，并在两个锁的保护下操作变量a和b。我们用sleep函数来模拟连续两次调用pthread_mutex_lock之间的时间差，以确保代码中的两个线程各自先占有一个互斥锁（主线程占有mutex_a，子线程占有mutex_b），然后等待另外一个互斥锁（主线程等待mutex_b，子线程等待mutex_a）。这样，两个线程就僵持住了，谁都不能继续往下执行，从而形成死锁。如果代码中不加入sleep函数，则这段代码或许总能成功地运行，从而为程序留下了一个潜在的BUG。

## 9.6 条件变量

> ​	如果说互斥锁是用于同步线程对共享数据的访问的话，那么条件变量**则用于在线程之间同步共享数据的值**。条件变量提供了一种线程间通知机制：**当某个共享数据达到某个值的时候，唤醒等待这个共享数据的线程。**
>
> ​	条件变量的相关函数主要又如下5个：
>
> ```c
> #include <pthread.h>
> int pthread_cond_init(pthread_cond_t* cond, const pthread_condaattr_t* cond_attr);
> int pthread_cond_destroy(pthread_cond_t* cond);
> int pthread_cond_broadcast(pthread_cond_t* cond);
> int pthread_cond_signal(pthread_cond_t* cond);
> int pthread_cond_wait(pthread_cond_t* cond, pthread_mutex_t* mutex);
> ```
>
> ​	这些函数的对一个参数cond指向要操作的目标条件变量，条件变量的类型是pthread_cond_t结构体。
>
> ​	pthread_cond_init函数用于初始化条件变量。cond_attr参数指定条件变量的属性。如果将它设置为NULL，则表示使用默认属性。条件变量的属性不多，而且和互斥锁的属性类型相似，所以我们不再赘述。除了pthread_cond_init函数外，我们还可以使用如下方式来初始化一个条件变量：
>
> ```c
> pthread_cond_t codn = PTHREAD_COND_INITIALIZER;
> ```
>
> ​	PTHREAD_COND_INITIALIZER实际上只是把条件变量的各个字段都初始化为0。
>
> ​	pthread_cond_destroy函数用于销毁条件变量，以释放其占用的内核资源。**销毁一个正在被等待的条件变量将失败并返回EBUSY。**
>
> ​	pthread_cond_broadcast函数以广播的方式唤醒所有等待目标条件变量的线程。pthread_cond_signal函数用于唤醒一个等待目标条件变量的线程。至于哪个线程被唤醒，则取决于线程的优先级和调度策略。有时候我们可能像唤醒一个指定的线程，但pthread没有对该需求提供解决办法。不过我们可以间接的实现该需求：定义一个能够唯一表示目标线程的全局变量，在唤醒等待条件的线程前设置该变量为目标线程，然后采样广播方式唤醒所有等待条件变量的线程，这些线程被唤醒后都检查该变量以判断被唤醒的是否是自己，如果是就开始执行后续代码，如果不是则返回继续等待。
>
> ​	pthread_cond_wait函数用于等待目标条件变量。mutex参数是用于保护条件变量的互斥锁，以确保pthread_cond_wait操作的原子性。在调用pthread_cond_wait前，必须确保互斥锁mutex已经加锁，否则将导致不可预期的结果。pthread_cond_wait前，**必须确保互斥锁mutex已经加锁**，否则将导致不可预期的结果。pthread_cond_wait函数执行时，首先把调用线程放入条件变量的等待队列中，然后将互斥锁mutex解锁。可见，从pthread_cond_wiat开始执行到其调用线程被放入条件队列之间的这段时间内，pthread_cond_signal和pthread_cond_broadcast等函数不会修改条件变量。换言之，pthread_cond_wait函数不会错过目标条件变量的任何变化。当**pthread_cond_wait函数成功返回时，互斥锁mutex将再次被锁上**。
>
> ​	上面这些函数成功时返回0，失败则返回错误码。

## 9.7 线程同步机制包装类

> ​	为了充分复用代码，同时由于后文的需要，我们将前面讨论的3种线程同步机制分别封装成3个类，实现在locker.h文件中，如代码所示：
>
> ```cpp
> #ifndef LOCKER_H
> #define LOCKER_H
> 
> #include <exception>
> #include <pthread.h>
> #include <semaphore.h>
> 
> /* 封装信号量的类 */
> class sem
> {
> public:
>     /* 创建并初始化信号量 */
>     sem()
>     {
>         if(sem_init(&m_sem, 0, 0) != 0)
>         {
>             /* 构造函数没有返回值，可以通过抛出异常来报告错误 */
>             throw std::exception();
>         }
>     }
>     /* 销毁信号量 */
>     ~sem()
>     {
>         sem_destroy(&m_sem);
>     }
> 
>     /* 等待信号量 */
>     bool wait()
>     {
>         return sem_post(&m_sem) == 0;
>     }
> 
>     /* 增加信号量 */
>     bool post()
>     {
>         return sem_post(&m_sem) == 0;
>     }
> private:
>     sem_t m_sem;
> };
> 
> /* 封装互斥锁 */
> class locker
> {
> public:
>     /* 创建并初始化互斥锁 */
>     locker()
>     {
>         if(pthread_mutex_init(&mutex, NULL) != 0)
>         {
>             throw std::exception();
>         }
>     }
>     /* 销毁互斥锁 */
>     ~locker()
>     {
>         pthread_mutex_destroy(&m_mutex);
>     }
>     /* 获取互斥锁 */
>     bool lock()
>     {
>         return pthread_mutex_lock(&m_mutex) == 0;
>     }
> 
>     /* 释放互斥锁 */
>     bool unlock()
>     {
>         return pthread_mutex_unlock(&m_mutex) == 0;
>     }
> 
> private:
>     pthread_mutex_t m_mutex;
> };
> 
> /* 封装条件变量的类 */
> class cond
> {
> public:
>     /* 创建并初始化条件变量 */
>     cond()
>     {
>         if(pthread_mutex_init(&m_mutex, NULL) !=0 )
>         {
>             throw std::exception();
>         }
>         if(pthread_cond_init(&m_cond, NULL) != 0)
>         {
>             /* 构造函数中一旦出现问题，就应该立即释放已经成功分配了的资源 */
>             pthread_mutex_destroy(&m_mutex);
>             throw std::exception();
>         }
>     }
> 
>     /* 销毁条件变量 */
>     ~cond()
>     {
>         pthread_mutex_destroy(&m_mutex);
>         pthread_cond_destroy(&m_cond);
>     }
> 
>     /* 等待条件变量 */
>     bool wait()
>     {
>         int ret = 0;
>         pthread_mutex_lock(&m_mutex);
>         ret = pthread_cond_wait(&m_cond, &m_mutex);
>         pthread_mutex_unlock(&m_mutex);
>         return ret == 0;
>     }
>     /* 唤醒等待条件变量的线程 */
>     bool signal()
>     {
>         return pthread_cond_signal(&m_cond) == 0;
>     }
> private:    
>     pthread_mutex_t m_mutex;
>     pthread_cond_t m_cond;
> }
> 
> #endif
> ```

## 9.8 多线程环境

### 9.8.1 可重入函数

> ​	如果一个函数能被多个线程同时调用且不发生竞态条件，则我们称它是线程安全的（thread safe），或者说它是可重入函数。Linux库函数只有一小部分是不可重入的，比如4.1.4小节讨论的inet_ntoa函数，以及getservbyname和getservbyport函数。这些库函数之所以不可重入，主要是因为其内部使用了静态变量。不过Linux对很多不可重入的库函数提供了可重入版本，这些可重入版本的函数名是在原函数名尾部加上\_r。比如函数localtime对应的可重入函数是localtime_r。在多线程程序中调用库函数，一定要使用其可重入版本，否则可能导致预想不到的结果。

### 9.8.2 线程和进程

> ​	思考这样一个问题：如果一个多线程程序的某个线程调用了fork函数，那么创建的子进程是否将自动创建和父进程相同数量的线程呢？答案是“否”，正如我们期望的那样。子进程只拥有一个执行线程，它是fork的哪个线程的完整复制。并且子进程将自动继承父进程中的互斥锁（条件变量与之类似）的状态。也就是说，父进程中已经被加锁的互斥锁在子进程中也是被锁住的。这就引起一个问题：子进程可能不清楚从父进程继承而来的互斥锁的具体状态（是加锁状态还是解锁状态）。这个互斥锁可能被加锁了，但并不是由调用fork函数的那个线程锁住的，而是由其他线程锁住的。如果是这种情况，则子进程若再次对该互斥锁执行加锁操作就会导致死锁，如以下代码所示：
>
> ```c
> #include <pthread.h>
> #include <unistd.h>
> #include <stdio.h>
> #include <stdlib.h>
> #include <wait.h>
> 
> pthread_mutex_t mutex;
> /* 子线程运行的函数。它首先获得互斥锁mutex，然后暂停5s，再释放该互斥锁 */
> void* another(void* arg)
> {
> 	printf("in child thread, lock the mutex\n");
> 	pthread_mutex_lock(&mutex);
> 	sleep(5);
> 	pthread_mutex_unlock(&mutex);
> }
> 
> int main()
> {
> 	pthread_mutex_init(&mutex, NULL);
> 	pthread_t id;
> 	pthread_create(&id, NULL, another, NULL);
> 	/* 父进程中的主线程暂停1s，以确保在执行fork操作之前，子线程已经开始运行并获得了互斥变量mutex */
> 	sleep(1);
> 	int pid = fork();
> 	if(pid < 0)
> 	{
> 		pthread_join(id, NULL);
> 		pthread_mutex_destroy(&mutex);
> 		return 1;
> 	}
> 	else if(pid == 0)
> 	{
> 		printf("I am in the child, want to get the lock\n");
> 		/* 子进程从父进程继承了互斥锁mutex的状态，该互斥锁处于锁住的状态，这是由父进程中的子进程执行pthread_mutex_lock引起的，因此，下面这句加锁操作会一直阻塞，尽管从逻辑上来说它不应该阻塞的 */
> 		pthread_mutex_lock(&mutex);
> 		printf("I can not run to here, oop...\n");
> 		pthread_mutex_unlock(&mutex);
> 		exit(0);
> 	}
> 	else
> 	{
> 		wait(NULL);
> 	}
> 	pthread_join(id, NULL);
> 	pthread_mutex_destroy(&mutex);
> 	return 0;
> }
> ```
>
> ​	不过，pthread提供了一个专门的函数pthread_atfork，以确保fork调用后父进程和子进程都拥有一个清楚的锁状态。该函数的定义如下：
>
> ```c
> #include <pthread.h>
> int pthread_atfork(void (*prepare)(void), void(*parent)(void), void(*child)(void));
> ```
>
> ​	该函数将建立3个fork句柄来帮助我们清理互斥锁的状态。perpare句柄将在fork调用创建出子进程之前被执行。它可以用来锁住所有父进程中的互斥锁。parent句柄则是fork调用创建出子进程之后，而fork返回之前，在父进程中被执行。它的作用是释放所有在prepare句柄中锁住的互斥锁。child句柄是fork返回之前，在子进程中被执行。和parent句柄一样，child句柄也是用于释放所有在prepare句柄中被锁住的互斥锁。该函数成功时返回0，失败则返回错误码。
>
> ​	因此，如果要让代码正常工作，就应该在其中的fork调用前加入以下代码：
>
> ```c
> void prepare()
> {
> 	pthread_mutex_lock(&mutex);
> }
> void infork()
> {
> 	pthread_mutex_unlock(&mutex);
> }
> pthread_atfork(prepare, infork, infork);
> ```

### 9.8.3 线程和信号

> ​	每个线程都可以独立地设置信号掩码。我们在5.3.2小节讨论过设置进程信号掩码的函数sigprocmask，但在多线程环境下我们应该使用如下所示的pthread版本的sigprocmask函数来设置线程信号掩码：
>
> ```c
> #include <pthread.h>
> #include <signal.h>
> int pthread_sigmask(int how, const sigset_t* newmask, sigset_t* oldmask);
> ```
>
> ​	该函数的参数的含义与sigprocmask的参数完全相同。pthread_sigmask成功时返回0，失败则返回错误码。
>
> ​	由于进程中的所有线程共享该进程的信号，所以**线程库将根据线程掩码决定把信号发送给哪个具体的线程**。因此，如果我们在每个子线程中都单独设置信号掩码，就很容易导致逻辑错误。此外，**所有线程共享信号处理函数**。也就是说，当我们在一个线程中设置了某个信号的信号处理函数后，它将覆盖其它线程为同一个信号设置的信号处理函数。这两点都说明，我们应该定义一个专门的线程来处理所有的信号。这可以通过如下两个步骤来实现：
>
> 1) 在主线程创建出其它子线程之前就调用pthread_sigmask来设置好信号掩码，所有新创建的子线程都将自动继承这个信号掩码。这样做之后，实际上所有线程都不会响应被屏蔽的信号了。
>
> 2) 在某个线程中调用如下函数来等待信号并处理之：
>
>    ```c
>    #include <signal.h>
>    int sigwait(const sigset_t* set, int* sig);
>    ```
>
> set参数指定需要等待的信号的集合。我们可以简单地将其指定为第1步中创建的信号掩码，**表示在该线程中等待所有被屏蔽的信号**。参数sig指向的整数用于存储该函数返回的信号值。sigwait成功时返回0，失败则返回错误码。一旦sigwait正确返回，我们就可以对接收到的信号做处理了。很显然，如果我们使用了sigwait，就不应该再为信号设置信号处理函数了。这是因为当程序接收到信号时，二者中只能有一个起作用。
>
> ​	以下代码取自pthread_sigmask函数的man手册。它展示了如何通过上述两个步骤实现在一个线程中统一处理所有信号。
>
> ```c
> #include <pthread.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <signal.h>
> #include <errno.h>
> 
> #define handle_error_en(en, msg) \
> 	do {errno = en; perror(msg); exit(EXIT_FAILURE);} while(0)
> static void* sig_thread(void *arg)
> {
> 	sigset_t *set = (sigset_t*)arg;
> 	int s, sig;
> 	for(;;)
> 	{
> 		/* 第二个步骤，调用sigwait等待信号 */
> 		s = sigwait(set, &sig);
> 		if(s != 0)
> 			handle_error_en(s, "sigwait");
> 		printf("Signal handing thread got signal %d\n", sig);
> 	}
> }
> 
> int main(int argc, char* argv[])
> {
> 	pthread_t thread;
> 	sigset_t set;
> 	int s;
> 	
> 	/* 第一个步骤，在主线程中设置信号掩码 */
> 	sigemptyset(&set);
> 	sigaddset(&set, SIGQUIT);
> 	sigaddset(&set, SIGUSR1);
> 	s = pthread_sigmask(SIG_BLOCK, &set, NULL);
> 	if(s != 0)
> 		handle_error_en(s, "pthread_sigmask");
> 	s = pthread_create(&thread, NULL, &sig_thread, (void*)&set);
> 	if(s != 0)
> 		handle_error_en(s, "pthread_create");
> 	pause();
> }
> ```
>
> 最后，pthread还提供了下面的方法，使得我们可以明确地将一个信号发送给指定的线程：
>
> ```c
> #include <signal.h>
> int pthread_kill(pthread_t thread, in sig);
> ```
>
> ​	其中，thread参数指定目标线程，sig参数指定待发送的信号。如果sig为0，则pthread_kill不发送信号，但它仍然会执行错误检查。我们可以利用这种方式来检测目标线程是否存在。pthread_kill成功时返回0，失败则返回错误码。
>
> http://t.csdn.cn/bSWBH

# 10 进程池和线程池

> ​	在前面的章节中，我们是通过动态创建子进程（或子线程）来实现并发服务器的。这样做有如下缺点：
>
> * 动态创建进程（或线程）是比较耗费时间的，这将导致比较慢的客户响应。
>
> * 动态创建的子进程（或子线程）通常只用来为一个客户服务（除非我们做特殊的处理），这将导致系统产生大量的细微进程（或线程）。进程（或线程）间的切换将消耗大量CPU时间。
>
> * 动态创建的子进程是当前进程的完整映像。当前进程必须谨慎地管理其分配的文件描述符和堆内存等系统资源，否则子进程可能复制这些资源，从而使系统的可用资源急剧下降，进而影响服务器的性能。
>
>   第3章介绍过的进程池和线程池可以解决上述问题。

## 10.1 进程池和线程池概述

> ​	进程池和线程池相似，所以这里我们只以进程池为例进行介绍。如没有特殊声明，下面对进程池的讨论完全适用于线程池。
>
> ​	进程池是由服务器预先创建的一组子进程，这些子进程的数目在3~10个之间（当然，这只是典型情况）。比如8.5.5小节所描述的，httpd守护进程就是使用包含7个子进程的进程池来实现并发的。线程池的线程数量应该和CPU数量差不多。
>
> ​	进程池中所有子进程都运行着相同的代码，并具有相同的属性，比如优先级、PGID等。因为进程池在服务器启动之初就创建好了，所以每个子进程都相对“干净”，即它们没有打开不必要的文件描述符（从父进程继承而来），也不会错误地使用大块的对内存（从父进程复制得到）。
>
> ​	当有新的任务到来时，主进程将通过某种方式选择进程池中的某一个子进程来为之服务。相比于动态创建子进程，选择一个已经存在的子进程子进程的代价显然要小得多。至于主进程选择哪个子进程来为新任务服务，则有两种方式：
>
> * 主进程使用某种算法来主动选择子进程。最简单、最常用的算法是随机算法和Round Robin（轮流选取）算法，但更优秀、更智能的算法将使任务在各个工作进程中更均匀地分配，从而减轻服务器的整体压力。
> * 主进程和所有子进程通过一个共享的工作队列来同步，子进程都睡眠在该工作队列上。当有新的任务到来时，主进程将任务添加到工作队列中。这将唤醒正在等待任务的子进程，不过只有一个子进程将获得新任务的“接管权”，它可以从工作队列中取出任务并执行之，而其他的子进程将继续睡眠在工作队列上。
>
>   当选好子进程后，主进程还需要使用某种通知机制来告诉目标子进程有新任务需要处理，并传递必要的数据。最简单的方法是，在父进程和子进程之间预先建立好一条管道，然后通过该管道来实现所有的进程间通信（当然，要预先定义好一套协议来规范管道的使用）。在父线程和子线程之间传递数据就要简单得多，因为我们可以把这些数据定义为全局的，那么它们本身就是被所有线程共享的。
>
> ​	综合上面的论述，我们将进程池的一般模型描绘为下图所示的形式。
>
> ![image-20230517123320719](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230517123320719.png)

## 10.2 多处理客户

> ​	在使用进程池处理多客户任务时，首先要考虑的一个问题是：**监听socket和连接socket是否都由主进程来统一管理**。回忆第3章中介绍的几种并发模式，**其中半同步/半反应堆模式是由主进程统一管理这两种socket的**；而图3-11所示的**半同步/半异步模式，以及领导者/追随者模式，则是由主进程管理所有监听socket，而各个主进程分别管理属于自己的连接socket的**。对于前一种情况，主进程接受新的连接以得到连接socket，然后它需要将该socket传递给子进程（对于线程池而言，父进程将socket传递给子进程是很简单的，因为它们可以很容易地共享该socket。但对于进程池而言，我们必须使用8.9节介绍的方法来传递socket)。后一种情况的灵活性更大一些，因为子进程可以自己调用accept来接受新的连接，**这样父进程就无须向子进程传递socket，而只需要地通知一声：“我检测到了新的连接，你来接受它。”**
>
> ​	在（4.6.1）小节中我们讨论过常连接，即一个客户的多次请求可以复用一个TCP连接。那么，在设计进程池时还需要考虑：一个客户连接上的所有任务是否由一个子进程来处理。<font color=green><b>如果说客户任务是无状态的，那么我们可以考虑使用不同的子进程来为该客户的不同请求服务</b></font>，如下图所示。
>
> ![image-20230517125805238](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230517125805238.png)
>
> ​	但如果客户任务是存在上下文关系的，则最好一直用同一个子进程来为之服务，否则实现起来将比较麻烦，因为我们不得不在各子进程之间传递上下文数据。在4.3.4小节中，我们讨论了epoll的EPOLLONESHOT事件，这一事件能够确保一个客户连接在整个生命周期中仅被一个线程处理。

## 10.3 半同步/半异步进程池实现

> ​	综合前面的讨论，本节我们实现一个基于图8-11所示的半同步/半异步并发模式的进程池，如以下代码所示。为了避免在父、子进程之间传递文件描述符，我们将接受新连接的操作放到子进程中。很显然，对于这种模式而言，一个客户连接上的所有任务始终是由一个子进程来处理的。
>
> ````c++
> #ifndef PROCESSPOOL_H
> #define PROCESSPOOL_H
> 
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <signal.h>
> #include <sys/wait.h>
> #include <sys/stat.h>
> 
> /* 描述一个子进程的类，m_pid是目标子进程 */
> class process
> {
> public:
>     process() : m_pid(-1){}
> public:
>     pid_t m_pid;
>     int m_pipefd[2];
> };
> 
> /* 进程池类，将它定义为模板类是为了代码复用。其模板参数是处理逻辑任务的类 */
> template <typename T>
> class processpool
> {
> private:
>     /* 将构造函数定义为私有，因此我们只能通过后面的create静态函数来创建processpool实例 */
>     processpool(int listenfd, int process_number = 8);
> public:
>     /* 单体模式，以保证程序最多创建一个processpool实例，这是程序正确处理信号的必要条件 */
>     static processpool<T>* create(int listenfd, int process_number = 8)
>     {
>         if(!m_instance)
>         {
>             m_instance = new processpool<T>(listenfd, process_number);
>         }
>         return m_instance;
>     }
>     ~processpool()
>     {
>         delete [] m_sub_process;
>     }
>     /* 启动进程池 */
>     void run();
> private:
>     void setup_sig_pipe();
>     void run_parent();
>     void run_child();
> private:
>     /* 进程池允许的最大子进程数量 */
>     static const int MAX_PROCESS_NUMBER = 16;
>     /* 每个子进程最多能处理的客户数量 */
>     static const int USER_PER_PROCESS = 65536;
>     /* epoll 最多能处理的事件数 */
>     static const int MAX_EVENT_NUMBER = 10000;
>     /* 进程池中的进程总数 */
>     int m_process_number;
>     /* 子进程在池中的序号，从0开始 */
>     int m_idx;
>     /* 每个进程都有一个epoll内核事件表，用m_epollfd标识 */
>     int m_epollfd;
>     /* 监听socket */
>     int m_listenfd;
>     /* 子进程通过m_stop来决定是否停止运行 */
>     int m_stop;
>     /* 保存所有子进程的描述信息 */
>     process* m_sub_process;
>     /* 进程池静态实例 */
>     static processpool<T>* m_instance;
> };
> 
> template<typename T>
> processpool<T>* processpool<T>::m_instance = NULL;
> 
> /* 用于处理信号的管道，以实现统一事件源，后面称之为信号管道 */
> static int sig_pipefd[2];
> 
> static int setnonblocking(int fd)
> {
>     int old_option = fcntl(fd, F_GETFL);
>     int new_option = old_option | O_NONBLOCK;
>     fcntl(fd, F_SETFL, new_option);
>     return old_option;
> }
> 
> static void addfd(int epollfd, int fd)
> {
>     epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLIN | EPOLLET;
>     epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
>     setnonblocking(fd);
> }
> 
> /* 从epollfd标识的epoll内核事件表中删除fd上的所有注册事件 */
> static void removefd(int epollfd, int fd)
> {
>     epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, 0);
>     close(fd);
> }
> 
> static void sig_handler(int sig)
> {
>     int save_errno = errno;
>     int msg = sig;
>     send(sig_pipefd[1], (char*)&msg, 1, 0);
>     errno = save_errno;
> }
> 
> static void addsig(int sig, void (handler)(int), bool restart=true)
> {
>     struct sigaction sa;
>     memset(&sa, '\0', sizeof(sa));
>     sa.sa_handler = handler;
>     if(restart)
>     {
>         sa.sa_flags |= SA_RESTART;
>     }
>     sigfillset(&sa.sa_mask);
>     assert(sigaction(sig, &sa, NULL) != -1);
> }
> 
> /* 进程池构造函数，参数listenfd是监听socket，它必须在创建进程池之前被创建，否则子进程无法引用它。参数process_number
> 指定进程池中子进程的数量 */
> template<typename T>
> processpool<T>::processpool(int listenfd, int process_number) : m_listenfd(listenfd), m_process_number(process_number),
> m_idx(-1), m_stop(false)
> {
>     assert((process_number > 0) && (process_number <= MAX_PROCESS_NUMBER));
>     m_sub_process = new process[process_number];
>     assert(m_sub_process);
> 
>     /* 创建process_number个子进程，并建立它们和父进程之间的管道 */
>     for(int i = 0; i < process_number; ++i)
>     {
>         int ret = socketpair(PF_UNIX, SOCK_STREAM, 0, m_sub_process[i].m_pipefd);
>         assert(ret == 0);
> 
>         m_sub_process[i].m_pid = fork();
>         assert(m_sub_process[i].m_pid >= 0);
>         if(m_sub_process[i].m_pid > 0)
>         {
>             close(m_sub_process[i].m_pipefd[1]);
>             continue;
>         }
>         else
>         {
>             close(m_sub_process[i].m_pipefd[0]);
>             m_idx = i;
>             break;
>         }
>     }
> }
> 
> /* 统一事件源 */
> template<typename T>
> void processpool<T>::setup_sig_pipe()
> {
>     /* 创建epoll事件监听表和信号管道 */
>     m_epollfd = epoll_create(5);
>     assert(m_epollfd != -1);
> 
>     int ret = socketpair(PF_UNIX, SOCK_STREAM, 0, sig_pipefd);
>     assert(ret != -1);
> 
>     setnonblocking(sig_pipefd[1]);
>     addfd(m_epollfd, sig_pipefd[0]);
> 
>     /* 设置信号处理函数 */
>     addsig(SIGCHLD, sig_handler);
>     addsig(SIGTERM, sig_handler);
>     addsig(SIGINT, sig_handler);
>     addsig(SIGPIPE, SIG_IGN);
> }
> 
> /* 父进程中m_idx值为-1，子进程中m_idx值大于等于0，我们据此判断接下来要运行的是父进程代码还是子进程代码 */
> template<typename T>
> void processpool<T>::run()
> {
>     if(m_idx != -1)
>     {
>         run_child();
>         return ;
>     }
>     run_parent();
> }
> 
> template<typename T>
> void processpool<T>::run_child()
> {
>     setup_sig_pipe();
> 
>     /* 每个子进程都通过其在进程池中的序号值m_idx找到与父进程通信的管道 */
>     int pipefd = m_sub_process[m_idx].m_pipefd[1];
>     /* 子进程需要监听管道文件描述符pipefd，因为父进程将通过它来通知进程accept新连接 */
>     addfd(m_epollfd, pipefd);
> 
>     epoll_event events[MAX_EVENT_NUMBER];
>     T* users = new T[USER_PER_PROCESS];
>     assert(users);
>     int number = 0;
>     int ret = -1;
> 
>     while(!m_stop)
>     {
>         number = epoll_wait(m_epollfd, events, MAX_EVENT_NUMBER, -1);
>         if((number < 0) && (errno != EINTR))
>         {
>             printf("epoll failure\n");
>             break;
>         }
> 
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             if((sockfd == pipefd) && (events[i].events & EPOLLIN))
>             {
>                 int client = 0;
>                 /* 从父、子进程之间的管道读取数据，并将结果保存在变量client中。如果读
>                 取成功，则表示有新客户连接到来 */
>                 ret = recv(sockfd, (char*)&client, sizeof(client), 0);
>                 if(((ret < 0) && (errno != EAGAIN)) || ret == 0)
>                 {
>                     continue;
>                 }
>                 else
>                 {
>                     struct sockaddr_in client_address;
>                     socklen_t client_addrlength = sizeof(client_address);
>                     int connfd = accept(m_listenfd, (struct sockaddr*)&client_address, &client_addrlength);
>                     if(connfd < 0)
>                     {
>                         printf("errno is: %d\n", errno);
>                         continue;
>                     }
>                     addfd(m_epollfd, connfd);
>                     /* 模板类T必须实现init方法，以初始化一个客户连接，我们直接使用connfd来索引逻辑处理对象
>                     （T类型的对象），以提高程序效率 */
>                     users[connfd].init(m_epollfd, connfd, client_address);
>                 }
>             }
>             /* 下面处理子进程接收到的信号 */
>             else if((sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN))
>             {
>                 int sig;
>                 char signals[1024];
>                 ret = recv(sig_pipefd[0], signals, sizeof(signals), 0);
>                 if(ret <= 0)
>                 {
>                     continue;
>                 }
>                 else
>                 {
>                     for(int i = 0; i < ret; ++i)
>                     {
>                         switch(signal[i])
>                         {
>                             case SIGCHLD:
>                             {
>                                 pid_t pid;
>                                 int stat;
>                                 while((pid = waitpid(-1, &stat, WNOHANG)) > 0)
>                                 {
>                                     continue;
>                                 }
>                                 break;
>                             }
>                             case SIGTERM:
>                             case SIGINT:
>                             {
>                                 m_stop = true;
>                                 break;
>                             }
>                             default:
>                             {
>                                 break;
>                             }
>                         }
>                     }
>                 }
>             }
>             /* 如果是其它可读数据，那么必然是客户端请求到来。调用逻辑处理对象的process方法处理之 */
>             else if(events[i].events & EPOLLIN)
>             {
>                 users[sockfd].process();
>             }
>             else
>             {
>                 continue;
>             }
>         }
>     }
> 
>     delete [] users;
>     users = NULL;
>     close(pipefd);
>     close(m_listenfd); 
>     close(m_epollfd);
> }
> 
> template<typename T>
> void processpool<T>::run_parent()
> {
>     setup_sig_pipe();
> 
>     /* 父进程监听m_listenfd */
>     addfd(m_epollfd, m_listenfd);
> 
>     epoll_event events[MAX_EVENT_NUMBER];
>     int sub_process_counter = 0;
>     int new_conn = 1;
>     int number = 0;
>     int ret  = -1;
>     while(!m_stop)
>     {
>         number = epoll_wait(m_epollfd, events, MAX_EVENT_NUMBER, -1);
>         if((number < 0) && (errno != EINTR))
>         {
>             printf("epoll failure\n");
>             break;
>         }
>         for(int i = 0; i < number; i++)
>         {
>             int sockfd = events[i].data.fd;
>             if(sockfd == m_listenfd)
>             {
>                 /* 如果有新连接到来，就采用Round Roin方式将其分配给一个子进程处理 */
>                 int i = sub_process_counter;
>                 do
>                 {
>                     if(m_sub_process[i].m_pid != -1)
>                     {
>                         break;
>                     }
>                     i = (i + 1) % m_process_number;
>                 } while (i != sub_process_counter);
>                 
>                 if(m_sub_process[i].m_pid == -1)
>                 {
>                     m_stop = true;
>                     break;
>                 }
>                 sub_process_counter = (i + 1) % m_process_number;
>                 send(m_sub_process[i].m_pipefd[0], (char*)&new_conn, sizeof(new_conn), 0);
>                 printf("send request to child %d\n", i);
>             }
>             /* 下面处理父进程收到的信号 */
>             else if((sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN))
>             {
>                 int sig;
>                 char signals[1024];
>                 ret = recv(sig_pipefd[0], signals, sizeof(signals), 0);
>                 if(ret <= 0)
>                 {
>                     continue;
>                 }
>                 else
>                 {
>                     for(int i = 0; i < ret; ++i)
>                     {
>                         switch(signal[i])
>                         {
>                             case SIGCHLD:
>                             {
>                                 pid_t pid;
>                                 int stat;
>                                 while((pid = waitpid(-1, &stat, WNOHANG)) > 0)
>                                 {
>                                     for(int i = 0; i < m_process_number; ++i)
>                                     {
>                                         /* 如果进程池中第i个子进程退出了，则主进程关闭
>                                         相应的通信管道，并设置相应的m_pid为-1，以标记该子进程已经退出 */
>                                         if(m_sub_process[i].m_pid == pid)
>                                         {
>                                             printf("child %d join\n", i);
>                                             close(m_sub_process[i].m_pipefd[0]);
>                                             m_sub_process[i].m_pid = -1;
>                                         }
>                                     }
>                                 }
>                                 /* 如果子进程都已经退出了，则父进程也退出 */
>                                 m_stop = true;
>                                 for(int i = 0; i < m_process_number; ++i)
>                                 {
>                                     if(m_sub_process[i].m_pid != -1)
>                                     {
>                                         m_stop = false;
>                                     }
>                                 }
>                                 break;
>                             }
>                             case SIGTERM:
>                             case SIGINT:
>                             {
>                                 /* 如果父进程接收到终止信号，那么就杀死所有子进程，并等待它们全部结束。
>                                 当然，通知子进程结束更好方法是向父、子进程之间的通信管道发送特殊数据 */
>                                 printf("kill all the chlid now\n");
>                                 for(int i = 0; i < m_process_number; ++i)
>                                 {
>                                     int pid = m_sub_process[i].m_pid;
>                                     if(pid != -1)
>                                     {
>                                         kill(pid, SIGTERM);
>                                     }
>                                 }
>                                 break;
>                             }
>                             default:
>                             {
>                                 break;
>                             }
>                         }
>                     }
>                 }
>             }
>             else
>             {
>                 continue;
>             }
>         }
>     }
>     close(m_listenfd);  /* 由创建者关闭这个文件描述符 */
>     close(m_epollfd);
> }
> 
> #endif
> ````

## 10.4 用进程池实现简单CGI服务器

> 回忆1.2节，我们曾实现过一个非常简单的CGI服务器。下面我们将利用前面介绍的进程池来重新实现一个并发的CGI服务器，代码如下所示：
>
> ```cpp
> #include <sys/types.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <stdio.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <sys/epoll.h>
> #include <signal.h>
> #include <sys/wait.h>
> #include <sys/stat.h>
> 
> #include "processpool.h"
> 
> /* 用于处理客户CGI请求的类，它可以作为processpool类的模板参数 */
> class cgi_conn
> {
> public:
>     cgi_conn(){}
>     ~cgi_conn(){}
>     /* 初始化客户连接，清空读缓冲区 */
>     void init(int epollfd, int sockfd, const sockaddr_in& client_addr)
>     {
>         m_epollfd = epollfd;
>         m_sockfd = sockfd;
>         m_address = client_addr;
>         memset(m_buf, '\0', BUFFER_SIZE);
>         m_read_idx = 0;
>     }
> 
>     void process()
>     {
>         int idx = 0;
>         int ret = -1;
>         /* 循环读取和分析客户数据 */
>         while(true)
>         {
>             idx = m_read_idx;
>             ret = recv(m_sockfd, m_buf + idx, BUFFER_SIZE-1-idx, 0);
>             /* 如果读操作发生错误，则关闭客户连接。但如果是暂时无数据可读，则退出循环 */
>             if(ret < 0)
>             {
>                 if(errno != EAGAIN)
>                 {
>                     removefd(m_epollfd, m_sockfd);
>                 }
>                 break;
>             }
>             /* 如果对方关闭连接，则服务器也关闭连接 */
>             else if(ret == 0)
>             {
>                 removefd(m_epollfd, m_sockfd);
>                 break;
>             }
>             else
>             {
>                 m_read_idx += ret;
>                 printf("user content is: %s\n", m_buf);
>                 /* 如果遇到字符"\r\n"，则开始处理客户请求 */
>                 for(; idx < m_read_idx; ++idx)
>                 {
>                     if((idx >= 1) && (m_buf[idx-1]=='\r') && (m_buf[idx] == '\n'))
>                     {
>                         break;
>                     }
>                 }
>                 /* 如果没有遇到字符"\r\n"，则需要读取更多的客户数据 */
>                 if(idx == m_read_idx)
>                 {
>                     continue;
>                 }
>                 /*开始处理客户请求，"\r\n\0" ==> " "\0\n\0"*/
>                 m_buf[idx-1] = '\0';
> 
>                 char* file_name = m_buf;
>                 /* 判断客户要运行的CGI程序是否存在 */
>                 if(access(file_name, F_OK) == -1)
>                 {
>                     removefd(m_epollfd, m_sockfd);
>                     break;
>                 }
>                 /* 创建子进程来执行CGI程序 */
>                 ret = fork();
>                 if(ret == -1)
>                 {
>                     removefd(m_epollfd, m_sockfd);
>                     break;
>                 }
>                 else if(ret > 0)
>                 {
>                     /* 父进程只需关闭连接 */
>                     removefd(m_epollfd, m_sockfd);
>                     break;
>                 }
>                 else
>                 {
>                     /* 子进程将标准输出重定向到m_sockfd，并执行CGI程序 */
>                     close(STDOUT_FILENO);
>                     dup(m_sockfd);
>                     execl(m_buf, m_buf, NULL);
>                     exit(0);
>                 }
>             }
>         }
>     }
> private:
>     /* 读缓冲区的大小 */
>     static const int BUFFER_SIZE = 1024;
>     static int m_epollfd;
>     int m_sockfd;
>     sockaddr_in m_address;
>     char m_buf[BUFFER_SIZE];
>     /* 标记读缓冲区已经读入的客户数据的最后一个字节的下一个位置 */
>     int m_read_idx;
> };
> int cgi_conn::m_epollfd = -1;
> 
> /* 主函数 */
> int main(int argc, char* argv[])
> {
>     if(argc <= 2)
>     {
>         printf("usage: %s ip_address port_number\n", basename(argv[0]));
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi(argv[2]);
> 
>     int listenfd = socket(PF_INET, SOCK_STREAM, 0);
>     assert(listenfd >= 0);
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero(&address, sizeof(address));
>     address.sin_family = AF_INET;
>     inet_pton(AF_INET, ip, &address.sin_addr);
>     address.sin_port = htons(port);
> 
>     ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));
>     assert(ret != -1);
> 
>     ret = listen(listenfd, 5);
>     assert(ret != -1);
> 
>     processpool<cgi_conn>* pool = processpool<cgi_conn>::create(listenfd);
>     if(pool)
>     {
>         pool->run();
>         delete pool;
>     }
>     close(listenfd);
>     return 0;
> }
> ```

## 10.5 半同步/半反应堆线程池实现

> ​	本节我们实现一个基于图8-10所示的半同步/半反应堆并发模式的线程池，如以下代码所示。相比上章节的进程池实现，该线程的通用性要高很多，因为它使用一个工作队列完全解除了主线程和工作线程的耦合关系：主线程往工作队列中插入任务，工作线程通过竞争来取得任务并执行它。不过，如果要将该线程池应用到1实际服务器程序中，那么我们必须保证所有客户请求都是无状态的，因为同一个连接上的不同请求可能会由不同的线程处理。
>
> ```c++
> ```
>
> ​	值得一提的是，在C++程序中使用`pthread_create`函数时，该函数的第3个参数必须指向一个静态函数。而要在一个静态函数中使用类的动态成员（包括成员函数和成员变量），则只能通过如下两种方式来实现：
>
> * 通过类的静态对象来调用。比如单体模式中，静态函数可以通过类的全局唯一实例来访问动态成员函数。
>
> * 将类的对象作为参数传递给该静态函数，然后在静态函数中引用这个对象，并调用其动态方法。
>
>   代码中使用的是第2种方式：将线程参数设置为this指针，然后在worker函数中获取该指针并调用其动态方法`run`。

## 10.6 用线程池实现的简单Web服务器

> ​	在8.6节中，我们曾使用有限状态机实现过一个非常简单的解析HTTP请求的服务器。下面我们将利用前面介绍的线程池来重新实现一个并发的Web服务器。

### 10.6.1 http_conn类

> ​	首先，我们需要准备线程池的模板参数类，用以封装多逻辑任务的处理。这个类是http_conn.
>
> http_conn.h
>
> ```c++
> #ifndef HTTPCONNECTION_H
> #define HTTPCONNECTION_H
> 
> #include <unistd.h>
> #include <signal.h>
> #include <sys/types.h>
> #include <sys/epoll.h>
> #include <fcntl.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <assert.h>
> #include <sys/stat.h>
> #include <string.h>
> #include <pthread.h>
> #include <stdio.h>
> #include <stdlib.h>
> #include <sys/mman.h>
> #include <stdarg.h>
> #include <errno.h>
> #include <sys/uio.h>
> #include "locker.h"
> 
> class http_conn {
> public:
>     /* 文件名的最大长度 */
>     static const int FILENAME_LEN = 200;
>     /* 读缓冲区的大小 */
>     static const int READ_BUFFER_SIZE = 2048;
>     /* 写缓冲区的大小 */
>     static const int WRITE_BUFFER_SIZE = 1024;
>     /* HTTP请求方法，但我们仅支持GET */
>     enum METHOD {
>         GET = 0 , POST , HEAD , PUT , DELETE ,
>         TRACE , OPTIONS , CONNECT , PATCH
>     };
>     /* 解析客户请求时，主状态机所处的状态 */
>     enum CHECK_STATE {
>         CHECK_STATE_REQUESTLINE = 0 ,
>         CHECK_STATE_HEADER ,
>         CHECK_STATE_CONTENT
>     };
>     /* 服务器处理HTTP请求的可能结果 */
>     enum HTTP_CODE {
>         NO_REQUEST , GET_REQUEST , BAD_REQUEST ,
>         NO_RESOURCE , FORBIDDEN_REQUEST , FILE_REQUEST ,
>         INTERNAL_ERROR , CLOSED_CONNECTION
>     };
> 
>     /* 行的读取状态 */
>     enum LINE_STATUS {
>         LINE_OK = 0 ,
>         LINE_BAD ,
>         LINE_OPEN
>     };
> 
> public:
>     http_conn() {}
>     ~http_conn() {}
> 
> public:
>     /* 初始化新接受的连接 */
>     void init( int sockfd , const sockaddr_in& addr );
>     /* 关闭连接 */
>     void close_conn( bool real_close = true );
>     /* 处理客户请求 */
>     void process();
>     /* 非阻塞读操作 */
>     bool read();
>     /* 非阻塞写操作 */
>     bool write();
> 
> private:
>     /* 初始化连接 */
>     void init();
>     /* 解析HTTP请求 */
>     HTTP_CODE process_read();
>     /* 填充HTTP应答 */
>     bool process_write( HTTP_CODE ret );
> 
>     /* 下面这一组函数被process_read调用以分析HTTP请求 */
>     HTTP_CODE parse_request_line( char* text );
>     HTTP_CODE parse_headers( char* text );
>     HTTP_CODE parse_content( char* text );
>     HTTP_CODE do_request();
>     char* get_line() { return m_read_buf + m_start_line; }
>     LINE_STATUS parse_line();
> 
>     /* 下面这一组函数被process_write调用以填充HTTP应答 */
>     void unmap();
>     bool add_response( const char* format , ... );
>     bool add_content( const char* content );
>     bool add_status_line( int status , const char* title );
>     bool add_headers( int content_length );
>     bool add_content_length( int content_length );
>     bool add_linger();
>     bool add_blank_line();
> public:
>     /* 所有socket上的事件都被注册到同一个epoll内核事件表中，所以将epoll文件描述符设置为静态的 */
>     static int m_epollfd;
>     /* 统计用户数量 */
>     static int m_user_count;
> 
> private:
>     /* 该HTTP连接的socket和对方的socket地址 */
>     int m_sockfd;
>     sockaddr_in m_address;
> 
>     /* 读缓冲区 */
>     char m_read_buf[READ_BUFFER_SIZE];
>     /* 标识读缓冲区中已经读入的客户数据的最后一个字节的下一个位置 */
>     int m_read_idx;
>     /* 当前正在分析的字符在读缓冲区中的位置 */
>     int m_checked_idx;
>     /* 当前正在解析的行的起始位置 */
>     int m_start_line;
>     /* 写缓冲区 */
>     char m_write_buf[WRITE_BUFFER_SIZE];
>     /* 写缓冲区中待发送的字节数 */
>     int m_write_idx;
> 
>     /* 主状态机当前所处的状态 */
>     CHECK_STATE m_check_state;
>     /* 请求方法 */
>     METHOD m_method;
> 
>     /* 客户请求的目标文件的完整路径，其内容等于doc_root + m_url，doc_root是网站根目录 */
>     char m_real_file[FILENAME_LEN];
>     /* 客户请求的目标文件的文件名 */
>     char* m_url;
>     /* HTTP协议版本号，我们仅支持HTTP/1.1 */
>     char* m_version;
>     /* 主机名 */
>     char* m_host;
>     /* HTTP请求的消息体的长度 */
>     int m_content_length;
>     /* HTTP请求是否要求保持连接 */
>     bool m_linger;
> 
>     /* 客户请求的目标文件被mmap到内存的起始位置 */
>     char* m_file_address;
>     /* 目标文件的状态。通过它我们可以判断文件是否存在、是否为目录，是否可读，
>     并获取文件大小等信息 */
>     struct stat m_file_stat;
>     /* 我们将采用writev来执行写操作，所以定义下面两个成员，其中m_iv_count表示被写内存块的数量 */
>     struct iovec m_iv[2];
>     int m_iv_count;
> };
> 
> #endif 
> ```
>
> http_conn.cpp
>
> ```c++
> #include "http_conn.h"
> 
> /* 定义HTTP响应的一些状态信息 */
> const char* ok_200_title = "OK";
> const char* error_400_title = "Bad Request";
> const char* error_400_form = "Your request has bad syntax or is inherently impossible to satisfy.\n";
> const char* error_403_title = "Forbidden";
> const char* error_403_form = "You do not have permission to get file from this server.\n";
> const char* error_404_title = "Not Found";
> const char* error_404_form = "The requested file was not found on this server.\n";
> const char* error_500_title = "Internal Error";
> const char* error_500_form = "There was an unusual problem serving the requested file.\n";
> /* 网站的根目录 */
> const char* doc_root = "/var/www/html";
> 
> int setnonblocking( int fd ) {
>     int old_option = fcntl( fd , F_GETFL );
>     int new_option = old_option | O_NONBLOCK;
>     fcntl( fd , F_SETFL , new_option );
>     return old_option;
> }
> 
> void addfd( int epollfd , int fd , bool one_shot ) {
>     epoll_event event;
>     event.data.fd = EPOLLIN | EPOLLET | EPOLLRDHUP;
>     if (one_shot)
>     {
>         event.events |= EPOLLONESHOT;
>     }
>     epoll_ctl( epollfd , EPOLL_CTL_ADD , fd , &event );
>     setnonblocking( fd );
> }
> 
> void removefd( int epollfd , int fd ) {
>     epoll_ctl( epollfd , EPOLL_CTL_DEL , fd , 0 );
>     close( fd );
> }
> 
> void modfd( int epollfd , int fd , int ev ) {
>     epoll_event event;
>     event.data.fd = fd;
>     event.events = ev | EPOLLET | EPOLLONESHOT | EPOLLRDHUP;
>     epoll_ctl( epollfd , EPOLL_CTL_MOD , fd , &event );
> }
> 
> int http_conn::m_user_count = 0;
> int http_conn::m_epollfd = -1;
> 
> void http_conn::close_conn( bool real_close ) {
>     if (real_close && ( m_sockfd != -1 )) {
>         removefd( m_epollfd , m_sockfd );
>         m_sockfd = -1;
>         m_user_count--; /* 关闭一个连接时，将客户总量减1 */
>     }
> }
> 
> void http_conn::init( int sockfd , const sockaddr_in& addr ) {
>     m_sockfd = sockfd;
>     m_address = addr;
>     /* 如下两行是为了避免TIME_WAIT状态，仅用于调试，实际使用时应该去除 */
>     int reuse = 1;
>     setsockopt( m_sockfd , SOL_SOCKET , SO_REUSEADDR , &reuse , sizeof( reuse ) );
>     addfd( m_epollfd , sockfd , true );
>     m_user_count++;
> 
>     init();
> }
> 
> void http_conn::init() {
>     m_check_state = CHECK_STATE_REQUESTLINE;
>     m_linger = false;
> 
>     m_method = GET;
>     m_url = 0;
>     m_version = 0;
>     m_content_length = 0;
>     m_host = 0;
>     m_start_line = 0;
>     m_checked_idx = 0;
>     m_read_idx = 0;
>     m_write_idx = 0;
>     memset( m_read_buf , '\0' , READ_BUFFER_SIZE );
>     memset( m_write_buf , '\0' , WRITE_BUFFER_SIZE );
>     memset( m_real_file , '\0' , FILENAME_LEN );
> }
> 
> /* 主状态机， 其分析参考8.6节 */
> http_conn::LINE_STATUS http_conn::parse_line() {
>     char temp;
>     for (; m_checked_idx < m_read_idx; ++m_checked_idx) {
>         temp = m_read_buf[m_checked_idx];
>         if (temp == '\r') {
>             if (( m_checked_idx + 1 ) == m_read_idx) {
>                 return LINE_OPEN;
>             }
>             else if (m_read_buf[m_checked_idx + 1] == '\n') {
>                 m_read_buf[m_checked_idx++] = '\0';
>                 m_read_buf[m_checked_idx++] = '\0';
>                 return LINE_OK;
>             }
>             return LINE_BAD;
>         }
>         else if (temp == '\n') {
>             if (( m_checked_idx > 1 ) && ( m_read_buf[m_checked_idx - 1] == '\r' )) {
>                 m_read_buf[m_checked_idx - 1] = '\0';
>                 m_read_buf[m_checked_idx++] = '\0';
>                 return LINE_OK;
>             }
>             return LINE_BAD;
>         }
>     }
> 
>     return LINE_OPEN;
> }
> 
> /* 循环读取客户数据，直到无数据可读或者对方关闭连接 */
> bool http_conn::read() {
>     if (m_read_idx >= READ_BUFFER_SIZE) {
>         return false;
>     }
> 
>     int bytes_read = 0;
>     while (true) {
>         bytes_read = recv( m_sockfd , m_read_buf + m_read_idx , READ_BUFFER_SIZE - m_read_idx , 0 );
>         if (bytes_read == -1) {
>             if (errno == EAGAIN || errno == EWOULDBLOCK) {
>                 break;
>             }
>             return false;
>         }
>         else if (bytes_read == 0) {
>             return false;
>         }
> 
>         m_read_idx += bytes_read;
>     }
>     return true;
> }
> 
> /* 解析请求行 获得请求方法、目标URL，以及HTTP版本号 */
> http_conn::HTTP_CODE http_conn::parse_request_line( char* text ) {
>     m_url = strpbrk( text , " \t" );
>     if (!m_url) {
>         return BAD_REQUEST;
>     }
>     *m_url++ = '\0';
> 
>     char* method = text;
>     if (strcasecmp( method , "GET" ) == 0) {
>         m_method = GET;
>     }
>     else {
>         return BAD_REQUEST;
>     }
>     /* 跳过多余的空格 */
>     m_url += strspn( m_url , " \t" );
>     m_version = strpbrk( m_url , " \t" );
>     if (!m_version) {
>         return BAD_REQUEST;
>     }
>     *m_version++ = '\0';
>     m_version += strspn( m_version , " \t" );
>     if (strcasecmp( m_version , "HTTP/1.1" ) != 0) {
>         return BAD_REQUEST;
>     }
> 
>     if (strncasecmp( m_url , "http://" , 7 ) == 0) {
>         m_url += 7;
>         m_url = strchr( m_url , '/' );
>     }
> 
>     if (!m_url || m_url[0] != '/') {
>         return BAD_REQUEST;
>     }
> 
>     m_check_state = CHECK_STATE_HEADER;
>     return NO_REQUEST;
> }
> 
> /* 解析HTTP请求的一个头部信息 */
> http_conn::HTTP_CODE http_conn::parse_headers( char* text ) {
>     /* 遇到空行，表示头部字段解析完毕 */
>     if (text[0] == '\0') {
>         /* 如果HTTP请求有消息体，则还需要读取m_content_length字节的消息体，状态机转移到CHECK_STATE_CONTENT状态 */
>         if (m_content_length != 0) {
>             m_check_state = CHECK_STATE_CONTENT;
>             return NO_REQUEST;
>         }
> 
>         /* 否则说明我们已经获得了一个完整的HTTP请求 */
>         return GET_REQUEST;
>     }
> 
>     /* 处理Connection头部字段 */
>     else if (strncasecmp( text , "Connection:" , 11 ) == 0) {
>         text += 11;
>         text += strspn( text , " \t" );
>         if (strcasecmp( text , "keep-alive" ) == 0) {
>             m_linger = true;
>         }
>     }
>     /* 处理Content-Length头部字段 */
>     else if (strncasecmp( text , "Content-Length:" , 15 ) == 0) {
>         text += 15;
>         text += strspn( text , " \t" );
>         m_content_length = atol( text );
>     }
>     /* 处理Host头部字段 */
>     else if (strncasecmp( text , "Host:" , 5 ) == 0) {
>         text += 5;
>         text += strspn( text , " \t" );
>         m_host = text;
>     }
>     else {
>         printf( "oop! unknown header %s\n" , text );
>     }
> 
>     return NO_REQUEST;
> }
> 
> /* 主状态机 */
> http_conn::HTTP_CODE http_conn::process_read() {
>     LINE_STATUS line_status = LINE_OK;
>     HTTP_CODE ret = NO_REQUEST;
>     char* text = 0;
>     while (( ( m_check_state == CHECK_STATE_CONTENT ) && ( line_status == LINE_OK ) ) || ( ( line_status = parse_line() ) == LINE_OK )) {
>         text = get_line();
>         m_start_line = m_checked_idx;
>         printf( "got 1 http line: %s\n" , text );
> 
>         switch (m_check_state) {
>         case CHECK_STATE_REQUESTLINE: {
>             ret = parse_request_line( text );
>                 if (ret == BAD_REQUEST) {
>                     return BAD_REQUEST;
>                 }
>                 break;
>             } 
>         case CHECK_STATE_HEADER: {
>             ret = parse_headers( text );
>             if (ret == BAD_REQUEST) {
>                 return BAD_REQUEST;
>             }
>             else if (ret == GET_REQUEST) {
>                 return do_request();
>             }
>             break;
>         }
>         case CHECK_STATE_CONTENT: {
>             ret = parse_content( text );
>             if (ret == GET_REQUEST) {
>                 return do_request();
>             }
>             line_status = LINE_OPEN;
>             break;
>         }
>         default: {
>             return INTERNAL_ERROR;
>         }
>         }
>     }
>     return NO_REQUEST;
> }
> 
> /* 当得到一个完整、正确的HTTP请求时，我们就分析目标文件的属性。如果目标文件存在、对所有用户可读，
> 且不是目录，则使用mmap将其映射到内核地址m_file_address处，并告诉调用者获取文件成功 */
> http_conn::HTTP_CODE http_conn::do_request() {
>     strcpy( m_real_file , doc_root );
>     int len = strlen( doc_root );
>     strncpy( m_real_file + len , m_url , FILENAME_LEN - len - 1 );
>     if (stat( m_real_file , &m_file_stat ) < 0) {
>         return NO_RESOURCE;
>     }
> 
>     if (!( m_file_stat.st_mode & S_IROTH )) {
>         return FORBIDDEN_REQUEST;
>         BAD_REQUEST;
>     }
> 
>     if (S_ISDIR( m_file_stat.st_mode )) {
>         return BAD_REQUEST;
>     }
> 
>     int fd = open( m_real_file , O_RDONLY );
> 
>     m_file_address = (char*)mmap( 0 , m_file_stat.st_size , PROT_READ , MAP_PRIVATE , fd , 0 );
>     close( fd );
>     return FILE_REQUEST;
> }
> 
> /* 对内存映射区执行munmap操作 */
> void http_conn::unmap() {
>     if (m_file_address) {
>         munmap( m_file_address , m_file_stat.st_size );
>         m_file_address = nullptr;
>     }
> }
> 
> /* 写HTTP响应 */
> bool http_conn::write() {
>     int temp = 0;
>     int bytes_have_send = 0;
>     int bytes_to_send = m_write_idx;
>     if (bytes_to_send == 0) {
>         modfd( m_epollfd , m_sockfd , EPOLLIN );
>         init();
>         return true;
>     }
> 
>     while (1) {
>         temp = writev( m_sockfd , m_iv , m_iv_count );
>         if (temp <= -1) {
>             /* 如果TCP写缓冲没有空间，则等待下一轮EPOLLOUT事件。虽然在此期间，服务器无法立即接收到同一客户的下一个请求，但这
>             可以保证连接的完整性 */
>             if (errno == EAGAIN) {
>                 modfd( m_epollfd , m_sockfd , EPOLLOUT );
>                 return true;
>             }
>             unmap();
>             return false;
>         }
> 
>         bytes_to_send -= temp;
>         bytes_have_send += temp;
>         if (bytes_to_send <= 0) {
>             /* 发送HTTP响应成功，根据HTTP请求中的Connection字段决定是否关闭连接 */
>             unmap();
>             if (m_linger) {
>                 init();
>                 modfd( m_epollfd , m_sockfd , EPOLLIN );
>                 return true;
>             }
>             else {
>                 modfd( m_epollfd , m_sockfd , EPOLLIN );
>                 return false;
>             }
>         }
>     }
> }
> 
> /* 往写缓冲区中写入待发送的数据 */
> bool http_conn::add_response( const char* format , ... ) {
>     if (m_write_idx >= WRITE_BUFFER_SIZE) {
>         return false;
>     }
>     va_list arg_list;
>     va_start( arg_list , format );
>     int len = vsnprintf( m_write_buf + m_write_idx , WRITE_BUFFER_SIZE - 1 - m_write_idx,
>         format , arg_list );
>     if (len >= ( WRITE_BUFFER_SIZE - 1 - m_write_idx )) {
>         return false;
>     }
>     m_write_idx += len;
>     va_end( arg_list );
>     return true;
> }
> 
> bool http_conn::add_status_line( int status , const char* title ) {
>     return add_response( "%s %d %s\r\n" , "HTTP/1.1" , status , title );
> }
> 
> bool http_conn::add_headers( int content_len ) {
>     add_content_length( content_len );
>     add_linger();
>     add_blank_line();
> }
> 
> bool http_conn::add_content_length( int content_len ) {
>     return add_response( "Content-Length: %d\r\n" , content_len );
> }
> 
> bool http_conn::add_linger() {
>     return add_response( "Connection: %s\r\n" , ( m_linger == true ) ? "keep-alive" : "close" );
> }
> 
> bool http_conn::add_content( const char* content ) {
>     return add_response( "%s" , content );
> }
> 
> /* 根据服务器处理HTTP请求的结果，决定发回给客户端的内容 */
> bool http_conn::process_write( HTTP_CODE ret ) {
>     switch (ret) {
>     case INTERNAL_ERROR: {
>         add_status_line( 500 , error_500_title );
>         add_headers( strlen( error_500_form ) );
>         if (!add_content( error_500_form )) {
>             return false;
>         }
>         break;
>     }
>     case BAD_REQUEST: {
>         add_status_line( 404 , error_404_title );
>         add_headers( strlen( error_404_form ) );
>         if (!add_content( error_404_form )) {
>             return false;
>         }
>         break;
>     }
>     case FORBIDDEN_REQUEST: {
>         add_status_line( 403 , error_403_title );
>         add_headers( strlen( error_403_form ) );
>         if (!add_content( error_403_form )) {
>             return false;
>         }
>         break;
>     }
>     case FILE_REQUEST: {
>         add_status_line( 200 , ok_200_title );
>         if (m_file_stat.st_size != 0) {
>             add_headers( m_file_stat.st_size );
>             m_iv[0].iov_base = m_write_buf;
>             m_iv[0].iov_len = m_write_idx;
>             m_iv[1].iov_base = m_file_address;
>             m_iv[2].iov_len = m_file_stat.st_size;
>             m_iv_count = 2;
>             return true;
>         }
>         else {
>             const char* ok_string = "<html><body></body></html>";
>             add_headers( strlen( ok_string ) );
>             if (!add_content( ok_string )) {
>                 return false;
>             }
>         }
>     }
>     default: {
>         return false;
>     }
>     }
> 
>     m_iv[0].iov_base = m_write_buf;
>     m_iv[0].iov_len = m_write_idx;
>     m_iv_count = 1;
>     return true;
> }
> 
> /* 由线程池中的线程调用，这是处理HTTP请求的入口函数 */
> void http_conn::process() {
>     HTTP_CODE read_ret = process_read();
>     if (read_ret == NO_REQUEST) {
>         modfd( m_epollfd , m_sockfd , EPOLLIN );
>         return;
>     }
> 
>     bool write_ret = process_write( read_ret );
>     if (!write_ret) {
>         close_conn();
>     }
>     modfd( m_epollfd , m_sockfd , EPOLLOUT );
> }
> ```
>
> **main.py**
>
> ```c++
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <unistd.h>
> #include <errno.h>
> #include <string.h>
> #include <fcntl.h>
> #include <stdlib.h>
> #include <cassert>
> #include <sys/epoll.h>
> 
> #include "locker.h"
> #include "threadpool.h"
> #include "http_conn.h"
> 
> #define MAX_FD 65535
> #define MAX_EVENT_NUMBER 10000
> 
> extern int addfd( int epollfd , int fd , bool one_shot );
> extern int removefd( int epollfd , int fd );
> 
> void addsig( int sig , void( handler )( int ) , bool restart = true ) {
>     struct sigaction sa;
>     memset( &sa , '\0' , sizeof( sa ) );
>     sa.sa_handler = handler;
>     if (restart) {
>         sa.sa_flags |= SA_RESTART;
>     }
>     sigfillset( &sa.sa_mask );
>     assert( sigaction( sig , &sa , NULL ) != -1 );
> }
> 
> void show_error( int connfd , const char* info ) {
>     printf( "%s" , info );
>     send( connfd , info , strlen( info ) , 0 );
>     close( connfd );
> }
> 
> int main( int argc , char* argv[] ) {
>     if (argc <= 2) {
>         printf( "usage: %s ip_address port_number\n" , basename( argv[0] ) );
>         return 1;
>     }
>     const char* ip = argv[1];
>     int port = atoi( argv[2] );
> 
> 
>     /* 忽略SIGPIPE信号 */
>     addsig( SIGPIPE , SIG_IGN );
> 
>     /* 创建线程池 */
>     threadpool<http_conn>* pool = NULL;
>     try {
>         pool = new threadpool<http_conn>;
>     }
>     catch (...) {
>         return 1;
>     }
> 
>     /* 预先为每个可能的客户连接分配一个http_cnn对象 */
>     http_conn* users = new http_conn[MAX_FD];
>     assert( users );
>     int user_count = 0;
> 
>     int listenfd = socket( PF_INET , SOCK_STREAM , 0 );
>     assert( listenfd >= 0 );
>     struct linger tmp = { 1, 0 };
>     setsockopt( listenfd , SOL_SOCKET , SO_LINGER , &tmp , sizeof( tmp ) );
> 
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero( &address , sizeof( address ) );
>     address.sin_family = AF_INET;
>     inet_pton( AF_INET , ip , &address.sin_addr );
>     address.sin_port = htons( port );
> 
>     ret = bind( listenfd , (struct sockaddr*)&address , sizeof( address ) );
>     assert( ret >= 0 );
> 
>     ret = listen( listenfd , 5 );
>     assert( ret >= 0 );
> 
>     epoll_event events[MAX_EVENT_NUMBER];
>     int epollfd = epoll_create( 5 );
>     assert( epollfd != -1 );
>     addfd( epollfd , listenfd , false );
>     http_conn::m_epollfd = epollfd;
> 
>     while (true) {
>         int number = epoll_wait( epollfd , events , MAX_EVENT_NUMBER , -1 );
>         if (( number < 0 ) && ( errno != EINTR )) {
>             printf( "epoll failure\n" );
>             break;
>         }
> 
>         for (int i = 0; i < number; i++) {
>             int sockfd = events[i].data.fd;
>             if (sockfd == listenfd) {
>                 struct sockaddr_in client_address;
>                 socklen_t client_addrlength = sizeof( client_address );
>                 int connfd = accept( listenfd , (struct sockaddr*)&client_address , &client_addrlength );
>                 if (connfd < 0) {
>                     printf( "errno is: %d\n" , errno );
>                     continue;
>                 }
>                 if (http_conn::m_user_count >= MAX_FD) {
>                     show_error( connfd , "Internal server busy" );
>                     continue;
>                 }
>                 /* 初始化客户连接 */
>                 users[connfd].init( connfd , client_address );
>             }
>             else if (events[i].events & ( EPOLLRDHUP | EPOLLHUP | EPOLLERR )) {
>                 /* 如果有异常，直接关闭客户连接 */
>                 users[sockfd].close_conn();
>             }
>             else if (events[i].events & EPOLLIN) {
>                 /* 根据读的结果，决定将任务添加到线程池，还是关闭连接 */
>                 if (users[sockfd].read()) {
>                     pool->append( users + sockfd );
>                 }
>                 else {
>                     users[sockfd].close_conn();
>                 }
>             }
>             else if (events[i].events & EPOLLOUT) {
>                 /* 根据写的结果，决定是否关闭连接 */
>                 if (!users[sockfd].write()) {
>                     users[sockfd].close_conn();
>                 }
>             }
>             else {}
>         }
>     }
>     close( epollfd );
>     close( listenfd );
>     delete[] users;
>     delete pool;
>     return 0;
> }
> ```

# 11 服务器调制、调试和测试

> ​	在前面的章节中，我们已经细致地探讨了服务器编程的诸多方面。现在我们要从系统的角度来优化、改进服务器，这包括3个方面的内容：系统调制、服务器调试和压力测试。
>
> ​	Linux平台的一个优秀特性是内核微调，即我们可以通过修改文件的方式来调整内核参数。11.2节将讨论与服务器性能相关的部分内核参数。这些内核参数中，系统或进程能打开的最大文件描述符尤其重要，所以在11.1节单独讨论之。
>
> ​	在服务器的开发过程中，我们可能碰到各种意想不到的错误。一种调试方法是用tcpdump抓包，正如之前章节介绍的那样。不过这种方法主要用于分析程序的输入和输出。对于服务器的逻辑错误，更方便的调试方法是使用gdb调试器。我们将在11.3节讨论如何用gdb调试多进程和多线程程序。
>
> ​	编写压力测试工具通常被认为是服务器开发的一个部分。压力测试工具模拟现实世界中的高并发的客户请求，以测试服务器在高压状态下的稳定性。我们将在11.4节给出一个简单的压力测试程序。

## 11.1 最大文件描述符数

> ​	文件描述符是服务器程序的宝贵资源，几乎所有的系统调用都是和文件描述符打交道。系统分配给应用程序的文件描述符数量是由限制的，所以我们必须总是关闭那些已经不再使用的文件描述符，以释放它们占用的资源。比如作为守护进程运行的服务器程序就应该总是关闭标准输入、标准输出和标准错误这3个文件描述符。
>
> ​	Linux对应用程序能打开的最大文件描述符数量有两个层次的限制：用户级限制和系统级限制。用户级限制是指**目标用户运行的所有进程总共能打开的文件描述符数**；系统级的限制是**所有用户总共能打开的文件描述符数**。
>
> ​	下面这个命令是最常用的查看用户级文件描述符限制的方法：
>
> ```shell
> $ ulimit -n
> ```
>
> ​	我们可以通过如下方式将用户级文件描述符限制设定为`max-file-number`：
>
> ```shell
> $ ulimit -SHn max-file-number
> ```
>
> ​	不过这种设置是临时的，只是当前的`session`中有效。为永久修改用户级文件描述符数限制，可以在`/etc/security/limits.conf`文件中加入如下两项：
>
> ```shell
> hard nofile max-file-number
> soft nofile max-file-number
> ```
>
> 第一行是指系统的硬限制，第二行是软限制。我们在7.4节讨论过这两种资源限制。
>
> ​	如果要修改系统级文件描述符数限制，则可以使用如下命令：
>
> ```shell
> $ sysctl -w fs.file-max=max-file-number
> ```
>
> ​	不过该命令也是临时更改系统限制。要永久更改系统级文件描述符限制，则需要在`/etc/sysctl.conf`文件中添加如下一项：
>
> ```shell
> fs.file-max=max-file-number
> ```
>
> ​	然后通过执行`sysctl -p`命令使更改生效。

## 11.2 调整内核参数

> ​	几乎所有的内核模块，包括内核核心模块和驱动程序，都在`/proc/sys`文件系统下提供了某些配置文件以供用户调整模块的属性和行为。**通常一个配置文件对应一个内核参数，文件名就是参数的名字，文件的内容是参数的值。**我们可以通过命令`sysctl -a`查看所有这些内核参数。本节将讨论其中和网络编程关系较为紧密的部分内核参数。

### 11.2.1 /proc/sys/fs目录下的部分文件

> ​	`/proc/sys/fs`目录下的内核参数都与文件系统相关。对于服务器程序来说，其中最重要的是如下两个参数：
>
> * `/proc/sys/fs/file-max`，系统级文件描述符限制。直接修改这个参数和11.1节讨论的修改方法有相同的效果（**不过这是临时修改**）。一般修改`/proc/sys/fs/file-max`后，应用程序需要把`/proc/sys/fs/inode-max`设置为新`/proc/sys/fs/file-max`值的3~4倍，否则可能导致`i`节点数不够用。
> * `/proc/sys/fs/epoll/max_user_watches`，**一个用户能够往epoll内核事件表中注册的事件的总量。**它是指该用户打开的所有epoll实例总共能监听的事件数目，而不是单个epoll实例能监听的事件数目。往epoll内核事件表中注册一个事件，在32位系统上大概消耗90字节的内核空间，在64位系统上则消耗160字节的内核空间。所以，这个内核参数限制了epoll使用的内核内存总量。

### 11.2.2 /proc/sys/net目录下的目录的部分文件

> ​	内核中网络模块的相关参数都位于`/proc/sys/net`目录下，其中和TCP/IP协议相关的参数主要位于如下三个子目录中：`core`、`ipv4`和`ipv6`。在前面的章节中，我们已经介绍过这些子目录下的很多参数的含义。
>
> * `/proc/sys/net/core/somaxconn`，指定listen监听队列里，能够建立完整连接从而进入ESTABLISHED状态的socket的最大数目。
>
> * `/proc/sys/net/ipv4/tcp_max_syn_backlog`，指定listen监听队列里，能够转移至ESTABLISHED或者SYN_RCVD状态的socket的最大数目。
>
> * `/proc/sys/net/ipv4/tcp_syncookies`，指定是否打开TCP同步标签（syncookie）。同步标签通过启动cookie来防止一个监听socket因不停地重复接收来自同一个地址的连接请求（同步报文段），而导致listen监听队列溢出（所谓的SYN风暴）。
>
>   除了通过直接文件的方式来修改这些系统参数外，我们也可以使用`sysctl`命令来修改它们。这两种修改方式都是临时的。永久的修改方法是在`/etc/sysctl.conf`文件中加入相应的网络参数及其数值，并执行`sysctl -p`使之生效，就像修改系统最大允许打开的文件描述符那样。

## 11.3 gdb调试

### 11.3.1 用gdb调试多进程程序

> ​	如果一个进程通过`fork`系统调用创建了子进程，gdb会继续调试原来的进程，子进程则正常运行。那么如何调试子进程呢？常用的方法有如下两种。
>
>  1. 单独调试子进程
>
>     子进程从本质上也是一个进程，因此我们可以用通用的gdb调试方法来调试它。举例来说，如果要调试代码清单15-2描述的CGI进程池服务器的某一个子进程，则我们可以先运行服务器，然后找到目标子进程的PID，再将其附加（attach）到gdb调试器上，具体操作如下所示：
>
>     ![image-20230606110542017](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230606110542017.png)
>
>     ![image-20230606110757724](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230606110757724.png)
>
>     2. 使用调试器选项`follow-fork-mode`
>
>        gdb调试器的选项`follow-fork-mode`允许我们选择在fork系统调用后是继续调试父进程还是调试子进程。其用法如下：
>
>        ```shell
>        (gdb)set follow-fork-mode mode
>        ```
>
>        ​	其中，mode的可选值是parent和child，分别表示调试父进程和子进程。还是使用前面的例子，这次考虑使用`follow-fork-mode`选项来调试子进程，如下所示：
>
>        ![image-20230606111757390](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230606111757390.png)

### 11.3.2 用gdb调试多线程程序

> gdb有一组命令可辅助多线程程序的调试。下面我们仅列举一些其中常用的一些：
>
> * `info threads`，显示当前可调试的所有线程。gdb会为每个线程分配一个ID，我们可以使用这个ID来操作对应的线程。ID前面有"*"号的线程是当前被调试的线程。
>
> * `thread ID`，调试目标ID指定的线程。
>
> * `set scheduler-locking[off|on|step]`。调试多线程程序时，默认除了被调试的线程在执行外，其他线程也在继续执行，但有的时候我们希望只让被调试的线程运行。这可以通过这个命令来实现。该命令设置`scheduler-locking`的值：`off`表示不锁定任何线程，即所有线程都可以继续执行，这是默认值；`on`表示只有当前被调试的线程会继续执行；`step`表示在单步执行的时候只有当前线程会执行。
>
>   举例来说，如果要依次调试代码15-6所描述的Web服务器（名为websrv）的父线程和子线程，则可以采用如下所示的方法：
>
>   ![image-20230606112837604](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230606112837604.png)
>
>   ![image-20230606112858051](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230606112858051.png)
>
>   ​	最后，关于调试进程池和线程池程序的一个不错的方法，是先将池中的进程个数或线程个数减少至1，以观察程序的逻辑是否正确，比如以上做法；然后逐步增加进程或线程的数量，以调试进程或线程的同步是否正确。

## 11.4 压力测试

> ​	压力测试程序有很多种实现方式，比如I/O复用方式，多线程、多进程并发编程方式，以及这些方式的结合使用。不过，单纯的I/O复用方式的施压程度是最高的，因为线程和进程的调度本身也是要占用一定CPU时间的。因此，我们将使用epoll来实现一个通用的服务器压力测试程序。
>
> ```c++
> #include <stdlib.h>
> #include <stdio.h>
> #include <assert.h>
> #include <unistd.h>
> #include <sys/types.h>
> #include <sys/epoll.h>
> #include <fcntl.h>
> #include <sys/socket.h>
> #include <netinet/in.h>
> #include <arpa/inet.h>
> #include <string.h>
> 
> /* 每个客户说连接不停地向服务器发送这个请求 */
> static const char* request = "GET http://localhost/index.html HTTP/1.1\r\nConnection: keep-alive\r\n\r\nxxxxxxxxxxxxxxxxx";
> 
> int setnonblocking( int fd ) {
>     int old_option = fcntl( fd , F_GETFL );
>     int new_option = old_option | O_NONBLOCK;
>     fcntl( fd , F_SETFL , new_option );
>     return old_option;
> }
> 
> void addfd( int epoll_fd , int fd ) {
>     epoll_event event;
>     event.data.fd = fd;
>     event.events = EPOLLOUT | EPOLLET | EPOLLERR;
>     epoll_ctl( epoll_fd , EPOLL_CTL_ADD , fd , &event );
>     setnonblocking( fd );
> }
> 
> /* 向服务器写入len字节的数据 */
> bool write_nbytes( int sockfd , const char* buffer , int len ) {
>     int bytes_write = 0;
>     printf( "write out %d bytes to socket %d\n" , len , sockfd );
>     while (1) {
>         bytes_write = send( sockfd , buffer , len , 0 );
>         if (bytes_write == -1) {
>             return false;
>         }
>         else if (bytes_write == 0) {
>             return false;
>         }
> 
>         len -= bytes_write;
>         buffer = buffer + bytes_write;
>         if (len <= 0) {
>             return true;
>         }
>     }
> }
> 
> /* 从服务器读取数据 */
> bool read_once( int sockfd , char* buffer , int len ) {
>     int bytes_read = 0;
>     memset( buffer , '\0' , len );
>     bytes_read = recv( sockfd , buffer , len , 0 );
>     if (bytes_read == -1) {
>         return false;
>     }
>     else if (bytes_read == 0) {
>         return false;
>     }
>     printf( "read in %d bytes from socket %d with content: %s\n" , bytes_read , sockfd , buffer );
> 
>     return true;
> }
> 
> /* 向服务器发起num个TCP连接，我们可以通过改变num来调整测试压力 */
> void start_conn( int epoll_fd , int num , const char* ip , int port ) {
>     int ret = 0;
>     struct sockaddr_in address;
>     bzero( &address , sizeof( address ) );
>     address.sin_family = AF_INET;
>     inet_pton( AF_INET , ip , &address.sin_addr );
>     address.sin_port = htons( port );
> 
>     for (int i = 0; i < num; i++) {
>         sleep( 1 );
>         int sockfd = socket( PF_INET , SOCK_STREAM , 0 );
>         printf( "create 1 sock\n" );
>         if (sockfd < 0) {
>             continue;
>         }
> 
>         if (connect( sockfd , (struct sockaddr*)&address , sizeof( address ) ) == 0) {
>             printf( "build connection %d\n" , i );
>             addfd( epoll_fd , sockfd );
>         }
>     }
> }
> 
> void close_conn( int epoll_fd , int sockfd ) {
>     epoll_ctl( epoll_fd , EPOLL_CTL_DEL , sockfd , 0 );
>     close( sockfd );
> }
> 
> int main( int argc , char* argv[] ) {
>     assert( argc == 4 );
>     int epoll_fd = epoll_create( 100 );
>     start_conn( epoll_fd , atoi( argv[3] ) , argv[1] , atoi( argv[2] ) );
>     epoll_event events[10000];
>     char buffer[2048];
>     while (1) {
>         int fds = epoll_wait( epoll_fd , events , 10000 , 2000 );
>         for (int i = 0; i < fds; i++) {
>             int sockfd = events[i].data.fd;
>             if (events[i].events & EPOLLIN) {
>                 if (!read_once( sockfd , buffer , 2048 )) {
>                     close_conn( epoll_fd , sockfd );
>                 }
>                 struct epoll_event event;
>                 event.events = EPOLLOUT | EPOLLET | EPOLLERR;
>                 event.data.fd = sockfd;
>                 epoll_ctl( epoll_fd , EPOLL_CTL_MOD , sockfd , &event );
>             }
>             else if (events[i].events & EPOLLOUT) {
>                 if (!write_nbytes( sockfd , request , strlen( request ) )) {
>                     close_conn( epoll_fd , sockfd );
>                 }
>                 struct epoll_event event;
>                 event.events = EPOLLIN | EPOLLET | EPOLLERR;
>                 event.data.fd = sockfd;
>                 epoll_ctl( epoll_fd , EPOLL_CTL_MOD , sockfd , &event );
>             }
>             else if (events[i].events & EPOLLERR) {
>                 close_conn( epoll_fd , sockfd );
>             }
>         }
>     }
> }
> ```
>
> ​	![image-20230607121606921](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230607121606921.png)

# 12 系统监测工具

> ​	Linux 提供了很多有用的工具，以方便开发人员和测评服务器程序。娴熟的网络程序员在开发服务器程序的整个过程，都将不断地使用这些工具中的一个或者多个来监测服务器行为。其中某些工具更是黑客们常用的利器。
>
> ​	本章将讨论几个常用的工具：`tcpdump`、`nc`、`strace`、`lsof`、`vmstat`、`ifstat`和`mpstat`。这些工具都支持很多选项，不过我们的讨论仅限于其中最常用、最实用的那些。

## 12.1 tcmdump

> ​	`tcpdump`是一款经典的网络抓包工具。即使在今天，我们拥有像`Wireshark`这样更易于使用和掌握的抓包工具，`tcpdump`仍然是网络程序员的必备利器。
>
> ​	`tcpdump`给使用者提供了大量的选项，用以过滤数据包或者定制输出格式。我们把常见的选项总结如下：
>
> * `-n`，使用`IP`地址表示主机，而不是主机名；使用数字表示端口，而不是服务名称。
>
> * `-i`，指定要监听的网卡接口。`-i any`表示抓取所有网卡接口上的数据包。
>
> * `-v`，输出一个稍微详细的信息，例如，显示IP数据包中的`TTL`和`TOS`信息。
>
> * `-t`，不打印时间戳。
>
> * `-e`，显示以太网帧头部信息。
>
> * `-c`，仅抓取指定数量的数据包。
>
> * `-x`，以十六进制显示数据包，但不显示包中以太网帧的头部信息。
>
> * `-X`，与`-x`类似，不过还打印每个十六进制字节对应的`ASCII`字符。
>
> * `-XX`，与`-X`类似相同，不过还打印以太网帧的头部信息。
>
> * `-s`，设置抓包时的抓取长度。当数据包的长度超过抓取长度，`tcpdump`抓取到的将是被截断的数据包。在4.0以及之前的版本中，默认的抓取包长度是``68`字节。这对于`IP`、`TCP`和`UDP`等协议就已经足够了，但对于`DNS`、`NFS`这样的协议，68字节通常不能容纳一个完整的数据包。比如我们在1.6.3小节抓取`DNS`数据包时，就使用了`-s`选项。不过4.0版本之后的版本，默认的抓包长度被修改为`65535`字节，因此我们不用再担心抓包长度的问题了。
>
> * `-S`，以绝对值来显示TCP报文段的序号，而不是相对值。
>
> * `-w`，将`tcpdump`的输出以特殊的格式定向到某个文件。
>
> * `-r`，从文件读取数据包信息并显示之。
>
>   除了使用选项外，`tcpdump`还支持用表达式来进一步过滤数据包。`tcpdump`表达式的操作数分为3种：类型（type）、方向（dir）和协议（proto）。下面依次介绍之。
>
> * 类型，解释其后面紧跟着的参数的含义。`tcpdump`支持的类型包括`host`、`net`、`port`和`portrange`。它们分别指定主机名（或IP地址），用CIDR方法表示的网络地址，端口号以及端口范围。比如，要抓取整个`1.2.3.0/255.255.255.0`网络上的数据包，可以使用如下命令：
>
>   * ```shell
>     tcpdump net 1.2.3.0/24
>     ```
>
> * 方向，`src`指定数据包的发送端，`dst`指定数据包的目的端。比如要抓取进入端口13589的数据包，可以使用如下命令：
>
>   * ```shell
>     tcpdump dst port 13579
>     ```
>
> * 协议，指定目标协议。比如要抓取所有`ICMP`数据包，可以使用如下命令：
>
>   * ```shell
>     tcpdump icmp
>     ```
>
>   当然，我们还可以使用逻辑操作符来组织上述操作以创建更复杂的表达式。`tcpdump`支持的逻辑操作符和编程语言中的逻辑操作符完全相同，包括`and`（或者`&&`）、`or`（或者`||`）、`not`（或者`!`）。比如要抓取主机`ernest-laptop`和所有非`Kongming20`的主机之间交换的`IP`数据包，可以使用如下命令：
>
> ```shell
> tcpdump ip host ernest-laptop and not Kongming20
> ```
>
> ​	如果表达式比较复杂，那么我们可以使用括号将它们分组。不过在使用括号时，我们要么使用"`\`"对它转义，要么用单引号"`'`"将其括住，以避免它被`shell`所解释。比如要抓取来自主机`10.0.2.4`，目标端口时`3389`或`22`的数据包，可以使用如下命令：
>
> ```shell
> tcpdump 'src 10.02.4 and (dst port 3389 or 22)'
> ```
>
>   此外，`tcpdump`还允许直接使用数据包中的部分协议字段的内容来过滤数据包。比如仅抓取`TCP`同步报文段，可使用如下命令：
>
> ```shell
> tcpdump 'tcp[13] & 2 != 0'
> ```
>
> ​	这是因为`TCP`头部的第14个字节的第2个位正是同步标志。该命令也可以表示为：
>
> ```shell
> tcpdump 'tcp[tcpflags] & tcp-syn != 0'
> ```
>
> ​	最后，`tcpdump`的具体输出格式除了与选项有关外，还与协议有关。前文我们讨论过`IP`、`TCP`、`ICMP`、`DNS`等协议的`tcpdump`的输出格式。关于其他协议的`tcpdump`，可以阅读`tcpdump`的`man`手册。

## 12.2 lsof

> ​	`lsof`（list open file）是一个列出当前系统打开文件描述符的工具。通过它我们可以了解感兴趣的进程打开了哪些文件描述符，或者我们感兴趣的文件描述符被哪些进程打开了。
>
> ​	`lsof`命令常用的选项包括：
>
> * `-i`，显示`socket`文件描述符。该选项的使用方法是：
>
>   * ```shell
>     lsof -i [46] [protocol][@hostname|ipaddr][:service|port]
>     ```
>
>       其中，4表示`IPv4`，6表示`IPv6`协议；`protocol`指定传输层协议，可以是`TCP`或者`UDP`；`hostname`指定主机名；`ipaddr`指定主机的IP地址；`port`指定端口号。比如，要显示所有连接到主机`192.168.1.108`的`ssh`服务的`socket`文件描述符，可以使用命令：
>
>     ```shell
>     lsof -i@192.168.1.108:22
>     ```
>
>     如果`-i`选项不指定任何参数，则`lsof`命令将显示所有`socket`文件描述符。
>
> * `-u`，显示指定用户启动的所有进程打开的文件描述符。
>
> * `-c`，显示指定的命令打开的文件描述符。比如要查看`websrv`程序打开了哪些文件描述符，可以使用如下命令：
>
>   * ```shell
>     lsof -c websrv
>     ```
>
> * `-p`，显示指定进程打开的所有文件描述符。
>
> * `-t`，仅显示打开了目标文件描述符的进程的`PID`。
>
>   我们还可以直接将文件名作为`lsof`命令的参数，以查看哪些进程打开了该文件。
>
> ​	下面介绍一个实例：查看`websrv`服务器打开了哪些文件描述符。
>
> ![image-20230610130253339](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230610130253339.png)
>
> `lsof`命令的输出内容相当丰富，其中每行内容都包含如下字段：
>
> * `COMMAND`，执行程序所使用的终端命令（默认仅显示前9个字符）。
> * `PID`，文件描述符所属进程的`PID`。
> * `USER`，拥有该文件描述符的用户的用户名。
> * `FD`，文件描述符的描述。其中`cwd`表示进程的工作目录，`rtd`表示用户的根目录，`txt`表示进程允许的程序代码，`mem`表示映射到内存中的文件（本例中都是动态库）。有的`FD`是以"数字 + 访问权限"表示的，其中数字是文件描述符的具体数值，访问权限包括`r`（可读）、`w`（可读可写）和`u`（可读可写）。在本例中`0u`、`1u`、`2u`分别表示标准输入、标准输出和标准错误输出；`3u`表示处于`LISTEN`状态的监听`socket`；`4u`表示`epoll`内核事件表对应的文件描述符。
> * `TYPE`，文件描述符的类型。其中`DIR`是目录，`REG`是普通文件，`CHR`是字符设备，`IPv4`是`IPv4`类型的`socket`文件描述符，`0000`是未知类型。更多文件描述符的类型可参考`lsof`的`man`手册。
> * `DEVICE`，文件所属设备。对于字符设备和块设备，其表示方法是“主设备号，次设备号”。由上图可见，测试机器上的程序文件和动态库都存放在设备"8, 3"中。其中，"8"表示这是一个`SCSI`硬盘；"3"表示这是该硬盘上的第3分区，即`sda3`。`websrv`程序的标准输入、标准输出和标准错误输出对应的设备是"136,3"。其中，"136"表示这是一个伪终端；"3"表示它是第3个伪终端，即`/dev/pts/3`。关于设备编号的更多细节，可参考[文档](http://www.kernel.org/pub/linux/docs/lanana/device-list/devices-2.6.txt)。对于`FIFO`类型的文件，比如管道和`socket`，该字段将显示一个内核引用目标文件的地址，或者是其`i节点号`。
> * `SIZE/OFF`，文件大小或者偏移值。如果该字段显示为"`0t*`"或者"`0x*`"，就表示这是一个偏移值，否则就表示这是一个文件大小。对字符设备或在`FIFO`类型的文件定义文件大小没有意义，所以该字段将显示一个偏移值。
> * `NODE`，文件的`i节点号`,对于`socket`，则显示为协议类型，比如"TCP"。
> * `NAME`，文件的名字。
>
>  如果我们使用`telnet`命令向`websrv`服务器发起一个连接，则再次执行代码`lsof`命令时，其输出将多出如下一行：
>
> ```shell
> websrv 6346 shuang 5u IPv4 44288 0t0 TCP localhost:13579->localhost:48215(ESTABLISHESD)
> ```
>
> ​	该输出表示服务器带开了一个`IPv4`类型的`socket`，其值是5，其它处于`ESTABLISHED`状态。该`socket`对应的连接的本端`socket`地址是（`127.0.0.1 13579`），远端`socket`地址则是(`127.0.0.1, 48215`)。

## 12.3 nc

> ​	`nc`（netcat）命令短小精干、功能强大，有着“瑞士军刀”的美誉。它主要功能被用来快速构建网络连接。我们可以让它以服务器方式运行，监听某个端口并接收客户连接，因此它可用来调试客户端程序。我们也可以使之以客户端方式运行，因此它可以用来调试服务器程序，此时它有点像`telnet`程序。
>
> ​	`nc`命令常用的选项包括：
>
> * `-i`，设置数据包传送的时间间隔。
>
> * `-l`，以服务器方式运行，监听指定的端口。`nc`命令默认以客户端方式运行。
>
> * `-k`，重复接受并处理某个端口上的所有连接，必须与`-l`选项一起使用。
>
> * `-n`，使用IP地址表示主机，而不是主机名；使用数字表示端口号，而不是服务名称。
>
> * `-p`，当`nc`命令以客户端方式运行时，强制使用指定的端口号。
>
> * `-s`，设置本地主机发送出的数据包的`IP`地址。
>
> * `-C`，将`CR`和`LF`两个字符作为行结束符。
>
> * `-U` ，使用`UNIX`本地域协议通信。
>
> * `-u`，使用`UDP`协议。`nc`命令默认使用的传输层协议时`TCP`协议。
>
> * `-w`，如果`nc`客户端在指定的时间内未检测到任何输入，则退出。
>
> * `-X`，当时`nc`客户端和代理服务器通信时，该选项指定它们之间使用的通信协议。目前`nc`支持的代理协议包括`"4"(SOCKS v.4), "5"(SOCKS v.5)`和`(HTTPS proxy)`。`nc`默认使用的代理协议是`SOCKS v.5`。
>
> * `-x`，指定目标代理服务器的IP地址和端口号。比如，要从Kongming20连接到ernet-laptop上的`squid`代理服务器，并通过它来访问`www.baidu.com`的Web服务，可以使用如下命令：
>
>   * ```shell
>     nc -x ernest-laptop:1080 -X connect www.baidu.com 80
>     ```
>
> * `-z`，扫描目标机器上的某个或某些服务是否开启（端口扫描）。比如，要扫描机器ernest-laptop上的端口号在`20~50`之间的服务，可以使用如下命令：
>
>   * ```shell
>     nc -z ernest-laptop 20-50
>     ```
>
>   举例来说，我们可以使用如下方式来连接`websrv`服务器并向它发送数据：
>
> ![image-20230610202504428](https://pic1.xuehuaimg.com/proxy/https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20230610202504428.png)
>
> ​	这里我们使用`-C`选项，这样每次我们按下回车键向服务器发送一行数据时，`nc`客户端程序都会给服务器额外发送一个`<CR><LF>`，而正是`websrv`服务器期望的`HTTP`行结束符。发送完第三行数据之后，我们得到了服务器的响应，内容正是我们所期望的：服务器没有找到被请求的资源文件a.html。可见`nc`命令是一个很方便的快速测试工具，通过它我们可以很快找出服务器的逻辑错误。

