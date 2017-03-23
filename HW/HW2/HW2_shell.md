# 6.828 HW2 shell

这个作业要求实现一个简易的shell，`sh.c`实现了递归下降分析，我们需要在此基础上完成exec、redirection、pipe等操作。

## Exec

`man 3 exec`，不难写出如下代码

```c
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    //fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    ret = execvp(ecmd->argv[0],ecmd->argv);
    if(ret<0)
      fprintf(stderr,"exec failed\n");
    break;
```

`execvp`会在PATH环境变量下搜索命令。

## I/O redirection

这里要注意`open(2)`的第三个参数。

`int open(const char *pathname, int flags, mode_t mode);`

如果flags中设置了O_CREAT，并且创建了新文件，那么必须设置mode，否则新文件的权限可能是随机值（这点我 不太确定）。

```c
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    //fprintf(stderr, "redir not implemented\n");
    // Your code here ...
    ret = close1(rcmd->fd);
    ret = open(rcmd->file,rcmd->mode,S_IRUSR|S_IWUSR|S_IRGRP);
    if(ret!=rcmd->fd)
      fprintf(stderr,"redir open failed\n");
    runcmd(rcmd->cmd);
    break;
```

## PIPE

关于pipe命令符的实现，xv6 book中已经讲解地很清楚了。

```c
  case '|':
    pcmd = (struct pipecmd*)cmd;
    //fprintf(stderr, "pipe not implemented\n");
    // Your code here ...
    if(pipe(p)<0)
      fprintf(stderr,"pipe failed");
    if(fork1()==0){
      close1(1);
      dup1(p[1]);
      close1(p[0]);
      close1(p[1]);
      runcmd(pcmd->left);
    }
    close1(0);
    dup1(p[0]);
    close1(p[0]);
    close1(p[1]);
    runcmd(pcmd->right);
    break;
```

## Challenge exercises

我实现了lists of commands和background supporting，sub shells留待以后完成。

先来看lists of commands的实现。

```c
struct listcmd{
  int type;         //;
  struct cmd* cmd;
  struct cmd* next;
};

//runcmd
  case ';':
    lcmd = (struct listcmd*)cmd;
    if(fork1()==0){
      runcmd(lcmd->cmd);
    }
    wait(&r);
    runcmd(lcmd->next);
    break;

struct cmd* listcmd(struct cmd* subcmd, struct cmd* next){
  struct listcmd *cmd;
  cmd = malloc(sizeof(*cmd));
  cmd->type = ';';
  cmd->cmd = subcmd;
  cmd->next = next;
  return (struct cmd*)cmd;
}

struct cmd* parselist(char **ps, char *es){
  struct cmd *cmd;
  cmd = parsepipe(ps,es);
  if(peek(ps,es,";")){
    gettoken(ps,es,0,0);
    cmd = listcmd(cmd,parselist(ps,es));
  }
  return cmd;
}
```

listcmd的实现其实与pipecmd大同小异，将命令串成列表即可。

background的实现更简单。

```c
struct backcmd{
  int type;   //&
  struct cmd* cmd;
};
//run cmd
case '&':
    bcmd = (struct backcmd*)cmd;
    if(fork1()==0){
      runcmd(bcmd->cmd);
    }

struct cmd* backcmd(struct cmd*subcmd){
  struct backcmd *cmd;
  cmd = malloc(sizeof(*cmd));
  cmd->type = '&';
  cmd->cmd = subcmd;
  return (struct cmd*)cmd;
}

struct cmd* parseback(char **ps, char *es){
  struct cmd *cmd;
  cmd = parselist(ps,es);
  if(peek(ps,es,"&")){
    gettoken(ps,es,0,0);
    cmd = backcmd(cmd);
  }
  return cmd;
}

```

