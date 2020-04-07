# socketProject

### 服务器端

规定服务器端最大客户连接数为512，不同的client采用不同的线程处理，同时为了解决多线程之间共享内存带来的数据读写问题，我们维护一个size == 512的数组，该数组元素包含基本信息如下：

~~~c
struct Client {
    char name[25];  //客户端的姓名
    int online;  //客户是否在线
    pthread_t tid;  //该客户对应线程的id
    int fd;  //发送消息使用的套接字文件的描述符
};
~~~

1. 根据server.conf配置文件，配置服务端的端口号
2. 创建一个处于监听状态的套接字文件
3. accept()接受请求，客户端connect()之后发送一个试探性的消息，服务器端接受这个试探性的消息，若未接收到，说明连接有问题，关闭连接，处理下一个请求。
4. 若连接没有问题，检测用户是否已经在线，若已经在线，拒绝此次连接。
5. 到此说明连接正常，服务器端向客户端发送欢迎消息。
6. 从数组找到第一个有效空位置，记录该连接的基本信息（由于存在用户的上线与下线，故已下线用户的连接信息会被覆盖，所以每次都要遍历数组找到合适的位置）
7. 创建子线程，同时服务端显示: Login : name显示该用户已登录。
8. 此时用户端可以向服务器端发送消息，我们规定发送的消息是一个报文，包含基本元素如下：

~~~c
struct Msg {
    char from[25];  //发送方的姓名 （姓名为主键，不可重复）
    int flag;  //发送信息的类型
    char message[512];  //发送的信息内容
};
/* 我们对文件的类型进行规定
 * flag == 0 公聊消息 (服务器转发给所有的用户)
 * flag == 1 私聊消息 转发给指定的用户
 * flag == 2 从服务器端发来的通知消息，或者是client端第一次的测试连接所发送的消息类型
 * flag == 3 断开连接的通知消息 （对方已登录，此时服务器向客户发送该消息；用户主动端开连接不需要）
 */
~~~

​		对于不同类型的报文数据，服务器采用不同的应对机制，只有客户主动断开连接，服务器才会关闭连接，子线程结束生命。

9.  处理客户端的其他请求，如：查询当前会议室在线人数 /etc.
10.  争取实现图片和文件的发送。

---

### 客户端

1. 查询本地的配置文件，得到将要连接的服务器的IP地址和端口号
2. 创建一个套接字连接，成功后向服务器端发送试探性的连接消息，若发送失败，程序结束；若成功，接受服务器端返回的确认消息，若报文类型为3，说明此前已经成功登录，本次登录被拒绝，程序结束。
3. 由于客户端需要同时进行消息的接受和发送，故采用多进程通信；此处规定子进程负责发送消息，父进程负责接受消息。
4. 发送消息简单，此处不介绍（当用户Ctrl C结束会话时，子进程结束）。
5. 接收消息，我们将所有接受到的消息添加到 chat.log 日志文件中，同时通过tail -f chat.log命令 不断的显示文件中的内容，起到消息界面的作用。 （故客户端需要打开两个terminal，一个发送消息，另一个tail -f chat.log显示所有的公聊信息和他人私发给自己的消息）
6. 若子进程结束先于父进程结束，为了防止僵尸进程的出现，我们在父进程结束，即将关闭连接前 wait(NULL) 回收子进程的PCB。
7. 若父进程先结束，子进程变为孤儿进程，被init进程收养，对计算机无影响，不做处理。

---

1. 为了更加人性化，我们对于接受到的不同信息，采用不同的颜色标识