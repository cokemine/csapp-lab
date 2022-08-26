# Shell Lab

任务要求：实现一个自己的 Shell `tsh`，需要补全的代码在 `tsh.c`中

## Preparation

可用辅助函数，这部分 assignment 没有， [这篇文章](https://zhuanlan.zhihu.com/p/58362429) 对这些函数的功能进行了整理，在补全代码时可以直接使用这些函数

- `int parseline(const char *cmdline,char **argv)`：获取参数列表`char **argv`，返回是否为后台运行命令（`true`）。
- `void clearjob(struct job_t *job)`：清除`job`结构。
- `void initjobs(struct job_t *jobs)`：初始化`jobs`链表。
- `void maxjid(struct job_t *jobs)`：返回`jobs`链表中最大的`jid`号。
- `int addjob(struct job_t *jobs,pid_t pid,int state,char *cmdline)`：在`jobs`链表中添加`job`
- `int deletejob(struct job_t *jobs,pid_t pid)`：在`jobs`链表中删除`pid`的`job`。
- `pid_t fgpid(struct job_t *jobs)`：返回当前前台运行`job`的`pid`号。
- `struct job_t *getjobpid(struct job_t *jobs,pid_t pid)`：返回`pid`号的`job`。
- `struct job_t *getjobjid(struct job_t *jobs,int jid)`：返回`jid`号的`job`。
- `int pid2jid(pid_t pid)`：将`pid`号转化为`jid`。
- `void listjobs(struct job_t *jobs)`：打印`jobs`。
- `void sigquit_handler(int sig)`：处理`SIGQUIT`信号。

## eval

eval 函数是要求实现的第一个函数，但是是细节最多的函数，大部分代码在书中都有，可以直接拿来用，注释也直接复制了。

其实最重要的是两个加锁的地方，还有一个是要保证子进程有自己的进程组，避免杀死子进程的时候父进程也被杀死了，这部分 assignment 有提到。

1. `addjob` 前后要加锁，这个比较好理解，要保证 `addjob` 函数的完整性，避免在执行 `addjob`函数中（未执行完）就被信号打断，导致我们的全局变量 `jobs`变得不安全。
2. 另一个是书中提到的避免子进程与父进程的竞争关系，因为有可能的一种情况是：还未执行 `addjob` 就收到 `SIGCHLD` 信号执行 `deletejob` ，结果最后添加了一个已经结束了的进程。所以要先阻塞掉父进程的 `SIGCHLD` 信号，确保执行顺序。

```c
void eval(char *cmdline) 
{
    char *argv[MAXARGS]; /* Argument list execve() */
    char buf[MAXLINE];   /* Holds modified command line */
    int bg;              /* Should the job run in bg or fg? */
    pid_t pid;           /* Process id */

    sigset_t mask_all, mask_one, prev_one;
    sigfillset(&mask_all);
    sigemptyset(&mask_one);
    sigaddset(&mask_one, SIGCHLD);

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);
    if (argv[0] == NULL)
        return;   /* Ignore empty lines */

    if (!builtin_cmd(argv)) {
        sigprocmask(SIG_BLOCK, &mask_one, &prev_one); /* Block SIGCHLD */
        
        if ((pid = fork()) == 0) {   /* Child runs user job */
            sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */
            setpgid(0, 0); /* 子进程拥有自己的进程组 */
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found.\n", argv[0]);
                exit(1);
            }
        }

        sigprocmask(SIG_BLOCK, &mask_all, NULL); /* Parent process */
        int state = bg ? BG : FG;
        addjob(jobs, pid, state, cmdline); /* Add the child to the job list */
        sigprocmask(SIG_SETMASK, &prev_one, NULL); /* Unblock SIGCHLD */

        /* Parent waits for foreground job to terminate */
        if (!bg) {
            waitfg(pid);
        }
        else {
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
        }
    }
    return;
}
```

## builtin_cmd

实现 `quit` 、`jobs`、`fg`、`bg` 四个内置命令。书中有一个实现了 `quit` 命令的模板，可以根据书中的例子进行修改

```c
int builtin_cmd(char **argv) 
{
    if (!strcmp(argv[0], "quit")) {
        exit(0);        
    }
    if (!strcmp(argv[0], "&")) {
        return 1;
    }
    if(!strcmp(argv[0], "jobs")) {
        listjobs(jobs);
        return 1;
    }
    if(!strcmp(argv[0], "fg") || !strcmp(argv[0], "bg")) {
        do_bgfg(argv);
        return 1;
    }
    return 0;     /* not a builtin command */
}
```

## do_bgfg

实现 `bg` / `fg` 命令，因为大部分函数如 `getjobjid` 已经有了，所以这部分主要是个命令解析的工作。

```c
void do_bgfg(char **argv) 
{
    struct job_t *job = NULL;
    int id;
    int state = strcmp(argv[0], "bg") ? FG : BG;

    if(argv[1] == NULL){
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }

    // jid
    if(sscanf(argv[1], "%%%d", &id) > 0){
        job = getjobjid(jobs, id); 
        if(job == NULL){
            printf("%%%d: No such job\n", id);
            return;
        }
    }
    // pid
    if (sscanf(argv[1], "%d", &id) > 0) {
    	job = getjobpid(jobs, id);
    	if (job == NULL) {
    		printf("(%d): No such process\n", id);
    		return ;
    	}
    }

    if (job == NULL) {  // 没有得到 job 则格式不对为非法输入
        printf("%s: argument must be a PID or %%jobid\n", argv[0]);
        return;
    }

    kill(-(job->pid), SIGCONT); //重启进程
    job->state = state;
    state == BG 
        ? printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline)
        : waitfg(job->pid);

    return;
}
```

## waitfg

根据提示，这里使用 `sleep` 即可

> In `waitfg`, use a busy loop around the `sleep` function.  

```c
void waitfg(pid_t pid)
{
    while(pid == fgpid(jobs)) {
        sleep(1);
    }
    return;
}
```

## sigchld_handler

负责回收子进程。书上的 handler 加上了锁，于是这边我也加上锁，但是我对这里保持疑问，估计是因为为了保证原子性才加的锁，但是这里应该不会被任何信号中断。

```c
void sigchld_handler(int sig) 
{
    sigset_t mask_all, prev_all;
    pid_t pid;
    int status;

    sigfillset(&mask_all);

    /* 
      WNOHANG | WUNTRACED立即返回，如果等待集合中的子进程都没有被停止或终止，则返回值为 0；如果有一个停止或终止，则返回值为该子进程的 PID。
    */
    while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
        sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        if (WIFSTOPPED(status)) {       // 暂停信号
            printf("Job [%d] (%d) stopped by signal %d\n", pid2jid(pid), pid,
                   WSTOPSIG(status));
            struct job_t* job = getjobpid(jobs, pid);
            job->state = ST;
        } else if (WIFSIGNALED(status)) {  // 被一个未被捕获的信号终止
                printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(pid),
                       pid, WTERMSIG(status));
                deletejob(jobs, pid);
        } else if (WIFEXITED(status)) { // 正常退出
            deletejob(jobs, pid);
        }
        sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    return;
}
```

## sigint_handler

捕获 `SIGINT` 信号，我们输入的 `SIGINT` 要发送给前台任务。

```c
void sigint_handler(int sig) 
{
    pid_t pid = fgpid(jobs); // 获取前台任务
    if(pid != 0){
        kill(-pid, sig); // 向子进程组发送信号
    }
    return;
}
```

## sigtstp_handler  

同上

```c
void sigtstp_handler(int sig) 
{
    pid_t pid = fgpid(jobs); // 获取前台任务
    if(pid != 0){
        kill(-pid, sig); // 向子进程组发送信号
    }
    return;
}
```

