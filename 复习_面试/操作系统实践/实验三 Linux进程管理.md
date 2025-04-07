# 1.设计目的
(1)通过对Linux 进程控制的相关系统调用的编程应用，进一步加深对进程概念的理解，明确进程和程序的联系与区别，理解进程并发执行的具体含义。
(2)通过Linux管道通信机制、消息队列通信机制、共享内存通信机制的应用，加深对不同类型的进程通信方式的理解。
(3)通过对 Linux 的 POSIX 信号量及 IPC 信号量的应用,加深对信号量同步机制的理解。
(4)请根据自身情况，进一步阅读分析相关系统调用的内核源码实现。

# 2.设计要求

### 模拟shell
#### 实验要求
```
编写三个不同的程序cmd1.c、cmd2.c及cmd3.c，每个程序的功能自定，分别编译成可执行文件cmd1、cmd2及cmd3。然后再编写一个程序，模拟shell 程序的功能:能根据用户输入的字符串(表示相应的命令名)，为相应的命令创建子进程并让它去执行相应的程序而父进程则等待子进程结束，然后再等待接收下一条命令。如果接收到的命令为exit，则父进程结束，退出模拟 shell; 如果接收到的命令是无效命令,则显示“Command not found”继续等待输入下一条命令。
```
**cmd1**
```
#include <stdio.h>
int main()
{
    printf("[*] Hello world! I'm cmd1.\n");
    return 0;
}
```
**cmd2**
```
#include <stdio.h>

int main()
{
    printf("[*] Hello world! I'm cmd3.\n");
    return 0;
}
                                                                                                                                                                                 
```
**cmd3**
```
#include <stdio.h>

int main()
{
    printf("[*] Hello world! I'm cmd3.\n");
    return 0;
}
```
**shell**
```
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/wait.h>

#define MAX_CMD_LEN 128  // 最大命令长度
#define PROMPT "mysh> "  // 提示符样式
void execute_command(const char *cmd) {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork失败");
        return;
    } else if (pid == 0) {
        // 子进程执行命令（自动添加./前缀）
        execlp(cmd, cmd, (char*)NULL);  // 使用execlp自动搜索PATH

        perror("执行失败");
        exit(EXIT_FAILURE);
    } else {
        // 父进程等待命令执行完成
        int status;
        waitpid(pid, &status, 0);  // 精确等待特定子进程
    }
}

int main() {
    char cmd[MAX_CMD_LEN];

    while (1) {
        printf(PROMPT);
        fflush(stdout);

        if (fgets(cmd, sizeof(cmd), stdin) == NULL) {
            printf("\n");  // 处理Ctrl+D的优雅退出
            break;
        }

        // 去除换行符
        cmd[strcspn(cmd, "\n")] = '\0';

        // 处理空输入
        if (strlen(cmd) == 0) continue;

        // 退出命令处理
        if (strcmp(cmd, "exit") == 0) {
            printf("安全退出shell\n");
            break;
        }

        // 执行外部命令
        execute_command(cmd);
    }

    return 0;
}
```
**实验效果**

![[file-20250328105710624.png]]模拟shell
### 实验详解

#### fork()

`fork()` : 创建一个普通进程，调用一次返回两次，子进程中返回 `0` ，父进程中返回 **`子进程pid`** 。清楚哪些代码块是子进程可以达到的，那些代码块是父进程可以达到的，对接下来的实验至关重要。

#### execl()

`execl(const char *path, const char *arg, ... NULL)` : 第一个参数 `path` 字符指针所指向要执行的文件路径， 接下来的参数代表执行该文件时传递的参数列表：`argv[0]` , `argv[1]` ... 最后一个参数须用空指针 `NULL` 作结束。


## 管道通信
由父进程创建一个管道，然后再创建三个子进程，并由这三个子进程利用管道与父进
程之间进行通信：子进程发送信息，父进程等三个子进程全部发完消息后再接收信息。通
信的具体内容可根据自己的需要随意设计，要求能试验阻塞型读写过程中的各种情况，测
试管道的默认大小，并且要求利用POSIX信号量机制实现进程间对管道的互斥访问。运行
程序，观察各种情况下，进程实际读写的字节数以及进程阻塞唤醒的情况。


### 父子进程通信

```
* ====================== pipe_sync.c ====================== */
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <string.h>

#define SEM_W "sem_write"
#define SEM_R "sem_read"
#define ENABLE_NUM 3

void write_to_pipe(int fd[2], int sid);
int read_from_pipe(int fd[2]);
void P(sem_t *sem_ptr);
void V(sem_t *sem_ptr);
void catch_INT(int sig);
void destroy_sem();

int main(void)
{
    int fd[2];
    pid_t pid;
    int ret = -1, child_num = ENABLE_NUM, enable_val = -1, sid = 0, i;
    sem_t *write_psx, *read_psx;
    write_psx = sem_open(SEM_W, O_CREAT, 0666, 1);
    read_psx = sem_open(SEM_R, O_CREAT, 0666, 0);

    if (pipe(fd) < 0) {
        printf("✖ 管道创建失败\n");
        exit(1);
    }
    for (i = 0; i < child_num; i++) {
        pid = fork();
        sid++;
        if (pid < 0) {
            printf("✖ 进程创建失败\n");
            destroy_sem();
            exit(0);
        } else if (pid == 0) {
            break;
        }
    }
    if (pid == 0) {
        P(write_psx);
        write_to_pipe(fd, sid);
        V(read_psx);
        V(write_psx);
        exit(0);
    } else {
        signal(SIGINT, catch_INT);
        while (1) {
            P(read_psx);
            P(read_psx);
            P(read_psx);
            read_from_pipe(fd);
        }
    }
    return 0;
}

void write_to_pipe(int fd[2], int sid)
{
    int value;
    char buf[10];
    close(fd[0]);
    memset(buf, sid + '0', sizeof(char)*10);
    printf("├── 子进程[%d] 发送 %dB\n", sid, (int)sizeof(buf));
    write(fd[1], buf, sizeof(buf));
}

int read_from_pipe(int fd[2])
{
    int ret;
    char buf[1024];
    close(fd[1]);
    memset(buf, '\0', sizeof(char)*1024);
    ret = read(fd[0], buf, 1024);
    while (ret > 0) {
        printf("└── 父进程接收: %dB → %s\n", ret, buf);
        memset(buf, '\0', 1024);
        ret = read(fd[0], buf, 1024);
    }
    return ret;
}

void P(sem_t *sem_ptr)
{
    sem_wait(sem_ptr);
}

void V(sem_t *sem_ptr)
{
    sem_post(sem_ptr);
}

void catch_INT(int sig)
{
    printf("\n════ 捕获 CTRL+C 信号 ════\n");
    destroy_sem();
    exit(0);
}

void destroy_sem()
{
    sem_unlink(SEM_W);
    sem_unlink(SEM_R);
}

```

![[file-20250407105927787.png]]

### 管道默认大小

```
/* ====================== pipe_max.c ====================== */
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

void write_to_pipe(int fd[2]);

int main(void)
{
    int fd[2];
    pid_t pid;
    int ret = -1;

    if (pipe(fd) < 0) {
        printf("✖ 管道创建失败\n");
        exit(1);
    }

    pid = fork();
    if (pid < 0) {
        printf("✖ 进程创建失败\n");
        exit(1);
    } else if (pid == 0) {
        write_to_pipe(fd);
    } else {
        wait(NULL);
    }
    return 0;
}

void write_to_pipe(int fd[2])
{
    int count, ret;
    char buffer[1024];
    close(fd[0]);
    memset(buffer, '*', 1024);

    ret = write(fd[1], buffer, sizeof(buffer));
    count = 1;
    printf("┌── 初始写入: %dKB\n", count);
    while (1) {
    ret = write(fd[1], buffer, sizeof(buffer));
        if(ret == -1){
            printf("└── 写入终止 (总写入量: %dKB)\n", count);
            break;
        }
        count++;
        printf("├── 持续写入: %dKB\n", count);
    }
}
```




![[file-20250407105154710.png]]![[file-20250407105214501.png]]



## 消息队列通信
### 实验要求
```
**利用 linux 的消息队列通信机制实现两个线程间的通信：**  
编写程序创建三个线程：sender1 线程、sender2 线程和 receive 线程，三个线程的功能描述如下：  
① sender1 线程：运行函数 sender1()，它创建一个消息队列，然后，等待用户通过终端输入一串字符，将这串字符通过消息队列发送给 receiver 线程；可循环发送多个消息，直到用户输入 `exit` 为止，表示它不再发消息，最后向 receiver 线程发送消息 `end1` ，并且等待 receiver 的应答，等到应答消息后，将接收到的应答信息显示在终端屏幕上，结束程序的运行。  
② sender2 线程：运行函数 sender2()，它创建一个消息队列，然后，等待用户通过终端输入一串字符，将这串字符通过消息队列发送给 receiver 线程；可循环发送多个消息，直到用户输入 `exit` 为止，表示它不再发消息，最后向 receiver 线程发送消息 `end2` ，并且等待 receiver 的应答，等到应答消息后，将接收到的应答信息显示在终端屏幕上，结束程序的运行。  
③ receiver 线程运行 receive()，它通过消息队列接收来自 sender1 和 sender2 两个线程的消息，将消息显示在终端屏幕上，当收到内容为 `end1` 的消息时，就向 sender1 发送一个应答消息 `over1` ；当收到内容为 `end2` 的消息时，就向 sender2 发送一个应答消息 `over2` ；消息收完后删除消息队列。使用合适的信号量机制实现三个线程之间的同步与互斥。
```
msg_queue.c
```
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define MSG_TYPE_NORMAL 1
#define MSG_TYPE_END1    2
#define MSG_TYPE_END2    3
#define MSG_TYPE_OVER1   4
#define MSG_TYPE_OVER2   5

typedef struct {
    long mtype;
    char mtext[256];
} Message;

sem_t sem_sender1, sem_sender2, sem_receiver;
pthread_mutex_t mutex;
int msgid;
int sender1_done = 0, sender2_done = 0;

void* sender1(void* arg) {
    Message msg;
    while(1) {
        // 获取用户输入
        printf("[Sender1] 输入消息: ");
        fgets(msg.mtext, 256, stdin);
        msg.mtext[strcspn(msg.mtext, "\n")] = '\0'; // 去除换行符

        // 发送消息
        if(strcmp(msg.mtext, "exit") == 0) {
            msg.mtype = MSG_TYPE_END1;
            msgsnd(msgid, &msg, sizeof(msg.mtext), 0);
            sem_post(&sem_receiver);
            break;
        }
        msg.mtype = MSG_TYPE_NORMAL;
        msgsnd(msgid, &msg, sizeof(msg.mtext), 0);
        sem_post(&sem_receiver);
    }

    // 等待应答
    sem_wait(&sem_sender1);
    msgrcv(msgid, &msg, sizeof(msg.mtext), MSG_TYPE_OVER1, 0);
     printf("[Sender2] 收到应答: %s\n", msg.mtext);

    pthread_mutex_lock(&mutex);
    sender2_done = 1;
    pthread_mutex_unlock(&mutex);

    return NULL;
}

void* receiver(void* arg) {
    Message msg;
    while(1) {
        sem_wait(&sem_receiver);

        // 接收所有类型的消息
        if(msgrcv(msgid, &msg, sizeof(msg.mtext), -6, IPC_NOWAIT) == -1) {
            continue;
        }

        switch(msg.mtype) {
            case MSG_TYPE_NORMAL:
                printf("[Receiver] 收到消息: %s\n", msg.mtext);
                break;

            case MSG_TYPE_END1: {
                Message response = {MSG_TYPE_OVER1, "over1"};
                msgsnd(msgid, &response, sizeof(response.mtext), 0);
                sem_post(&sem_sender1);
                printf("[Receiver] 收到结束信号1，发送应答\n");
                break;
            }

            case MSG_TYPE_END2: {
			Message response = {MSG_TYPE_OVER2, "over2"};
                msgsnd(msgid, &response, sizeof(response.mtext), 0);
                sem_post(&sem_sender2);
                printf("[Receiver] 收到结束信号2，发送应答\n");
                break;
            }
        }

        // 检查是否全部完成
        pthread_mutex_lock(&mutex);
        if(sender1_done && sender2_done) {
            pthread_mutex_unlock(&mutex);
            break;
        }
        pthread_mutex_unlock(&mutex);
    }

    // 清理消息队列
    msgctl(msgid, IPC_RMID, NULL);
    printf("[Receiver] 消息队列已清理\n");
    return NULL;
}

int main() {
    // 初始化消息队列
    msgid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

    // 初始化同步对象
    sem_init(&sem_sender1, 0, 0);
    sem_init(&sem_sender2, 0, 0);
    sem_init(&sem_receiver, 0, 0);
    pthread_mutex_init(&mutex, NULL);

    // 创建线程
    pthread_t t1, t2, t3;
    pthread_create(&t1, NULL, sender1, NULL);
    pthread_create(&t2, NULL, sender2, NULL);
    pthread_create(&t3, NULL, receiver, NULL);
    // 等待线程结束
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);

    // 清理资源
    sem_destroy(&sem_sender1);
    sem_destroy(&sem_sender2);
    sem_destroy(&sem_receiver);
    pthread_mutex_destroy(&mutex);

    return 0;
}
```


![[file-20250407111043590.png]]

## 共享内存通信
### 实验要求
```
**利用 linux 的共享内存通信机制实现两个进程间的通信：**  
编写程序 sender，它创建一个共享内存，然后等待用户通过终端输入一串字符，并将这串字符通过共享内存发送给 receiver；最后，它等待 receiver 的应答，等到应答消息后，将接收到的应答信息显示在终端屏幕上，删除共享内存，结束程序的运行。编写 receiver 程序，它通过共享内存接收来自 sender 的消息，将消息显示在终端屏幕上，然后再通过该共享内存向 sender 发送一个应答消息 `over` ，结束程序的运行。使用 **有名信号量** 或 **System V** 信号量实现两个进程对共享内存的互斥及同步使用。
```
init.h
```
/* ================ init.h ================ */
#ifndef _INIT_H_
#define _INIT_H_

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/sem.h>
#include <unistd.h>
#include <signal.h>

#define SHM_SIZE 1024
#define SHM_KEY 0x1234
#define SEM_KEY 0x5678

union semun {
    int val;
    struct semid_ds *buf;
    unsigned short *array;
};

// 信号量操作宏
#define SEM_MUTEX 0  // 互斥信号量
#define SEM_SYNC  1  // 同步信号量

void P(int semid, int semnum);
void V(int semid, int semnum);
int init_sem(int semid, int semnum, int value);

#endif
```
init.c
```
/* ================ init.c ================ */
#include "init.h"

void P(int semid, int semnum) {
    struct sembuf sb = {semnum, -1, 0};
    semop(semid, &sb, 1);
}

void V(int semid, int semnum) {
    struct sembuf sb = {semnum, 1, 0};
    semop(semid, &sb, 1);
}

int init_sem(int semid, int semnum, int value) {
    union semun arg;
    arg.val = value;
    return semctl(semid, semnum, SETVAL, arg);
}
```
receiver.c
```
/* ================ receiver.c ================ */
#include "init.h"

int shmid, semid;
char *shmptr;

void cleanup() {
    shmdt(shmptr);
    printf("[Receiver] 资源已释放\n");
}

void sig_handler(int sig) {
    printf("\n[Receiver] 收到终止信号\n");
    cleanup();
    exit(0);
}

int main() {
    signal(SIGINT, sig_handler);

    // 获取共享内存
    if ((shmid = shmget(SHM_KEY, SHM_SIZE, 0666)) == -1) {
        perror("shmget");
        exit(1);
    }

    // 连接共享内存
    if ((shmptr = shmat(shmid, NULL, 0)) == (char*)-1) {
        perror("shmat");
        exit(1);
    }

    // 获取信号量集
    if ((semid = semget(SEM_KEY, 2, 0666)) == -1) {
        perror("semget");
        cleanup();
        exit(1);
    }

    while(1) {
        // 等待发送方通知
        P(semid, SEM_SYNC);

        // 获取互斥锁
        P(semid, SEM_MUTEX);

        // 读取共享内存
        printf("[Receiver] 收到消息: %s\n", shmptr);
        if(strcmp(shmptr, "exit") == 0) {
            // 发送应答
            strncpy(shmptr, "over", SHM_SIZE);
            V(semid, SEM_SYNC);
            V(semid, SEM_MUTEX);
            break;
        }

        // 释放互斥锁
        V(semid, SEM_MUTEX);
    }

    cleanup();
    return 0;
}
```
sender.c
```
/* ================ sender.c ================ */
#include "init.h"

int shmid, semid;
char *shmptr;

void cleanup() {
    // 断开共享内存连接
    shmdt(shmptr);
    printf("[Sender] 资源已释放\n");
}

void sig_handler(int sig) {
    printf("\n[Sender] 收到终止信号\n");
    cleanup();
    exit(0);
}

int main() {
    signal(SIGINT, sig_handler);

    // 创建/获取共享内存
    if ((shmid = shmget(SHM_KEY, SHM_SIZE, IPC_CREAT|0666)) == -1) {
        perror("shmget");
        exit(1);
    }

    // 连接共享内存
    if ((shmptr = shmat(shmid, NULL, 0)) == (char*)-1) {
        perror("shmat");
        exit(1);
    }

    // 创建信号量集
    if ((semid = semget(SEM_KEY, 2, IPC_CREAT|0666)) == -1) {
        perror("semget");
        cleanup();
        exit(1);
    }

    // 初始化信号量
    init_sem(semid, SEM_MUTEX, 1);  // 互斥信号量初始为1
    init_sem(semid, SEM_SYNC, 0);   // 同步信号量初始为0

    char message[SHM_SIZE];
    while(1) {
        // 获取用户输入
        printf("[Sender] 输入消息: ");
        fgets(message, SHM_SIZE, stdin);
        message[strcspn(message, "\n")] = '\0';

        // 获取互斥锁
        P(semid, SEM_MUTEX);

        // 写入共享内存
        strncpy(shmptr, message, SHM_SIZE);

        // 通知接收方
        V(semid, SEM_SYNC);

        // 释放互斥锁
        V(semid, SEM_MUTEX);

        if(strcmp(message, "exit") == 0) {
            // 等待应答
            P(semid, SEM_SYNC);
            printf("[Sender] 收到应答: %s\n", shmptr);
            break;
        }
    }

    // 清理资源
    shmctl(shmid, IPC_RMID, NULL);  // 删除共享内存
    semctl(semid, 0, IPC_RMID);     // 删除信号量集
    cleanup();
    return 0;
}
```
![[file-20250407113515131.png]]![[file-20250407113533645.png]]