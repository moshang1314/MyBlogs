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
> ​	使用splice函数时，fd_in和fd_out必须至少有一个是管道文件描述符。splice函数调用成功时返回移动字节的数量。它可能返回0，表示没有数据需要移动，这发生在从管道中读取数据（fd_in是管道文件描述符）而该管道没有被写入任何数据时。splice函数失败时返回-1并设置errno。常见的errno如下表所示：
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
> ​	该函数的参数的含义与splice相同（但fd_in和fd_out必须都是管道文件描述符）。tee函数成功时返回在两个文件描述符之间复制的数据数量（字节数）。返回0表示没有复制任何数据。tee失败时返回-1并设置errno。
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

> ​	fcntl函数，正如其名字（file control）描述的那样，提供了对文件描述符的各种控制操作。另外一个常见的控制文件描述符属性和行为的系统调用是ioctl，而且ioctl比fcntl能够执行更多的控制。但是，对于控制文件描述符常用的属性和行为，fcntl函数是由POSIX规范指定的首选方法。fcntl函数的定义如下：
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
> ![image-20221103203931844](G:\images\Linux\2f34169d42b3ef51c6840ebb263c9a01f.png)

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
> ​	需要指出的是，一个进程拥有两个用户ID：UID和EUID。EUID存在的目的是方便资源访问：它使得运行程序的用户拥有该程序的有效用户的权限。比如su程序，任何用户都可以使用它来修改自己的账户信息，但修改账户时su程序不得不访问/etc/passwd文件，而访问该文件是需要root权限的。那么以普通用户身份启动的su程序如何能访问/etc/passwd文件呢？窍门就在EUID。用ls命令可以查看到，su程序的所有者是root，并且它被设置了set-user-id标志。这个标志表示，任何普通用户运行su程序时，其有效用户就是该程序的所有者root。那么根据有效用户的含义，任何运行su程序的普通用户都能访问/etc/passwd文件。**有效用户为root的进程称为特权进程（privileged processes）。**EGID的含义与EUID类似：给运行程序的组用户提供有效组的权限。
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
> ​	该函数不能由进程组的首领进程调用，否组将产生一个错误。对于非组首领的进程，调用该函数不仅创建新会话，而且有如下额外效果：
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
> ![image-20221106155128589](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221106155128589.png)

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
> ​	rlim_t是一个整数类型，它描述了资源级别。rlim_cur成员指定资源的软限制，rlim_max成员指定资源的硬限制。软限制是一个建议性的、最好不要超越的限制，如果超越的话，系统可能向进程发送信号以终止其运行。例如，当进程CPU时间超过其软限制时，系统将向进程发送SIGXCPU信号；当文件尺寸超过其软限制时，系统将向进程发送SIGXFSZ信号。**硬限制一般是软限制的上限。**普通程序可以减少硬限制，而只有root身份运行的程序才能增加硬限制。此外，我们可以使用ulimit命令修改当前shell环境下的资源限制（软限制或/和硬限制），这种修改将对该shell启动的所有后续程序有效。我们也可以通过修改配置文件来改变系统软限制和硬限制。下表列举了部分比较重要的资源限制类型。
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
> ![image-20221107194354685](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107194354685.png)
>
> ​	采用C/S模型的TCP服务器和TCP客户端的工作流程如下图8-2所示。
>
> ​	C/S模型的逻辑很简单。服务器启动后，首先创建一个（或多个）监听socket，并调用bind函数将其绑定到服务器感兴趣的端口上，然后调用listen函数等待客户连接。服务器稳定运行之后，客户端就可以调用connect函数服务器发起连接了。**由于客户连接请求是随机到达的异步事件，服务器需要使用某种I/O模型来监听这一事件。**I/O模型有多种，图8-2中，服务器使用的是I/O复用技术之一的select系统调用。当监听到连接请求后，**服务器就调用accept函数接受它，并分配一个逻辑单元为新的连接服务。**逻辑单元可以是新创建的子进程、子线程或其它。图8-2中，服务器给客户端分配的逻辑单元是由fork系统调用创建的子进程。逻辑单元读取客户端请求，处理该请求，然后将处理结果返回给客户端。客户端接收到服务器的反馈结果之后，可以继续向服务器发送请求，也可以立即主动关闭连接。如果客户端主动关闭连接，则服务器执行被动关闭连接。至此，双方的通信结束。需要注意的是，服务器在处理一个客户请求的同时还会继续监听其它客户请求，否则就变成了效率低下的串行服务器了（必须先处理完前一个客户的请求，才能继续处理下一个客户请求）。图8-2中，服务器同时监听多个客户请求是通过select系统调用实现的。
>
> ![image-20221107195914814](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107195914814.png)
>
> ​	C/S模型非常适合资源相对集中的场合，并且它的实现也很简单，但其缺点也很明显：服务器是通信的中心，当访问量过大时，可能所有客户都将得到很慢的响应。下面讨论的P2P模型解决了这个问题。

### 3.1.2 P2P模型

> ​	P2P（Peer to Peer，点对点）模型比C/S模型更符合网络通信的实际情况。它摒弃了以服务器为中心的格局，让网络上所有主机重新回归对等的地位。P2P模型如图8-3a所示。
>
> ​	P2P模型使得每台机器在消耗服务的同时也给别人提供服务，这样资源就能够充分、自由地共享。云计算机群可以看作P2P模型的一个典范。但P2P模型的缺点也很明显：当用户之间传输请求过多时，网络的负载将加重。
>
> ​	图8-3a所示的P2P模型存在一个显著的问题，即主机之间很难相互发现。所以实际使用的P2P模型通常带有一个专门的发现服务器，如图8-3b所示。这个发现服务器通常还提供查找服务（甚至还可以提供内容服务），使每个客户都能尽快地找到自己需要的资源。
>
> ![image-20221107201943967](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107201943967.png)
>
> ​	从编程角度来讲，P2P模型可以看作是C/S模型的扩展：每台主机既是客户端，又是服务器。

## 3.2 服务器编程框架

> ​	虽然服务器程序种类繁多，但其基本框架都一样，不同之处在于逻辑处理。如8-4所示：
>
> ![image-20221107202326885](https://cdn.jsdelivr.net/gh/moshang1314/myBlog@main/image/image-20221107202326885.png)
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
> ​	I/O复用是最常使用的I/O通知机制。它指的是，**应用程序通过I/O复用函数向内核注册一组事件，内核通过I/O复用函数把其中就绪的事件通知给应用程序。**Linux上常用的I/O复用函数是select、poll和epoll_wait，我们将在后续的章节讨论它们。需要指出的是，I/O复用函数本身是阻塞的，它们能提高程序效率的原因在于它们具有同时监听多个I/O事件的能力。
>
> ```
> #include 
> 
> ```
>
> **fjlsgj**





