# lab1

## **Question 1**: 

If you add a background command after the pipeline command, can the pipeline work?

### Solution

![image-20221004234233382](C:/Users/17622/AppData/Roaming/Typora/typora-user-images/image-20221004234233382.png)

由上图可知，background command会导致pipeline不能正常工作，shell提前打印出了`ccs-lab@`字样，并且在执行完background command后依然在等待用户输入，我实现的background如下

```c
case CMD_BACK: {
	struct Cmd_Back *t = (struct Cmd_Back *) cmd;
	Cmd_t back = t->back;

	if(fork() == 0){
      	Cmd_run(back);
     }
  	//wait(0);  //no care child
	break;
}
```

而在pipe的实现机制下

```c
case CMD_PIPE: {
	struct Cmd_Pipe *t = (struct Cmd_Pipe *) cmd;
	Cmd_t left = t->left;
	Cmd_t right = t->right;
	int fd[2];
	pipe(fd);

    if(fork() == 0) {
       	close(1);
       	dup(fd[1]);
        close(fd[0]);
       	close(fd[1]);
       	Cmd_run(left);
	}
    wait(0);

    if(fork() == 0){
      	close(0);
       	dup(fd[0]);
        close(fd[0]);
        close(fd[1]);
     	Cmd_run(right);
   	}
    close(fd[0]);
    close(fd[1]);
    wait(0);
    break;
}
```

分析其原因，执行right节点命令时，fork出子进程来执行`grep`，由于此时执行的是background command，父进程并未等待子进程退出(`grep程序还在运行`)，可以直接返回到父进程，最终返回到main函数中，此时进入新一轮循环，`css-lab@`被打印出来了，但此时cpu又调度了grep进程，grep执行完后，又返回到main里，这时shell会等待用户输入命令，所以出现了一个空行，等待命令输入

## **Question 2**:

Why `fork` and `exec` are not combined in a single call?

### Solution

`fork`和`exec`分离，可以使得在fork后exec前，设置一些必要的参数，如设置argv[]参数、打开或关闭文件描述符

例如在实现`redirect`过程中，执行`left节点`命令之前，先关闭`stdout/stdin`，再打开要重定向的文件，然后直接left节点命令，即可实现重定向

## **Question 3**: 

What are the advantages of pipes over temporary files in this situation?

+ use pipe

```bash
echo hello world | wc
```

+ without pipe

```bash
echo hello world > /tmp/xyz ; wc < /tmp/xyz
```

### Solution

代码简洁明了，不需要暂时文件来存储中间信息，便于编写组合命令，实现复杂命令

## **Exercise 1**: 

Please complete the corresponding part in:

```c
Cmd_t Cmd_Back_new(Cmd_t back);
Cmd_t Cmd_Pipe_new(Cmd_t left, Cmd_t right);
Cmd_t Cmd_Redir_new(Cmd_t left, Cmd_t right, int fd);
```

Hint: you can refer to the data structure defined in the `ast.h`.

### Soution

```c
Cmd_t Cmd_Back_new(Cmd_t back){
    Cmd_Back cmd;
    NEW(cmd);
    cmd->type = CMD_BACK;
    cmd->back = back;
    return (Cmd_t)cmd;
}

Cmd_t Cmd_Pipe_new(Cmd_t left, Cmd_t right){
    Cmd_Pipe cmd;
    NEW(cmd);
    cmd->type = CMD_PIPE;
    cmd->left = left;
    cmd->right = right;
    return (Cmd_t)cmd;
}

Cmd_t Cmd_Redir_new(Cmd_t left, Cmd_t right, int fd){
    Cmd_Redir cmd;
    NEW(cmd);
    cmd->type = CMD_REDIR;
    cmd->left = left;
    cmd->right = right;
    cmd->fd = fd;
    return (Cmd_t)cmd;
}
```

## **Exercise 2**: 

Please complete the `void Cmd_print(Cmd_t cmd)`. Then call `void Cmd_print(Cmd_t cmd)` in `main.c` and comment out `Cmd_run(root)`.

Finally, compile and run the project in Clion and use the following test cases to check whether your answer is correct.

```bash
cat hello.txt | grep -n hello > c.txt ; ls -l 
```

The result will be as shown in the figure below.

![img](https://csslab-ustc.github.io/courses/sysprog/labs/lab1/figures/print1.PNG)

 ```bash
 sleep 10 &    
 ```

The result will be as shown in the figure below.

![image-20220924143217317](C:/Users/17622/AppData/Roaming/Typora/typora-user-images/image-20220924143217317.png)

### Solution

```c
case CMD_BACK: {
   	struct Cmd_Back *t = (struct Cmd_Back *) cmd;
   	Cmd_t back = t->back;
    
   	Cmd_print(back);
    printf("& ");
   	break;
}

case CMD_PIPE: {
	struct Cmd_Pipe *t = (struct Cmd_Pipe *) cmd;
	Cmd_t left = t->left;
	Cmd_t right = t->right;

	Cmd_print(left);
	printf("| ");
	Cmd_print(right);
	break;
}

case CMD_REDIR: {
	struct Cmd_Redir *t = (struct Cmd_Redir *) cmd;
	Cmd_t left = t->left;
  	Cmd_t right = t->right;

	Cmd_print(left);
   	printf("> ");
	Cmd_print(right);
  	break;
}
```

运行效果如下：

![image-20221004233932251](C:/Users/17622/AppData/Roaming/Typora/typora-user-images/image-20221004233932251.png)

## **Exercise 3**: 

Please complete the `void Cmd_run(Cmd_t cmd)`. Regarding redirection commands, you only need to implement output redirection. Commands like this:

```bash
cat hello.txt > c.txt 
```

Then call `void Cmd_run(Cmd_t cmd)` in `main.c` and comment out `Cmd_print(root)`.

Finally, compile and run the project in Clion and use the following test cases to check whether your answer is correct.

```bash
cat hello.txt | grep -n hello > c.txt ; cat c.txt ; ls
```

Before entering the command, create a file called `hello.txt` in the current working directory. The content of the file is as follows:

![img](https://csslab-ustc.github.io/courses/sysprog/labs/lab1/figures/contents.PNG)

The result will be as shown in the figure below.

![img](https://csslab-ustc.github.io/courses/sysprog/labs/lab1/figures/test1.PNG)

```bash
sleep 10 &  
```

If you enter the command `sleep 10`, you cannot use the shell for ten seconds. However, when you add an extra `&` to your command, you'll be able to use the shell immediately. The result will be as shown in the figure below.

![img](https://csslab-ustc.github.io/courses/sysprog/labs/lab1/figures/test2.PNG)

### Thinking

BACK设计思路：

+ 父进程不等候子进程的退出

PIPE设计思想：

+ 先fork一个子进程用来运行left，使得文件描述符1指向管道的写端，当left执行时会输出到管道里，第二次fork，子进程也会指向这个管道，让文件描述符0指向管道的读端，当right执行时会从管道中读入，最后父进程关闭管道

REDIR设计思想：

+ 获取right的文件名，关闭文件描述符0/1，然后再打开right文件，运行left即可

### Solution

```c
case CMD_BACK: {
    struct Cmd_Back *t = (struct Cmd_Back *) cmd;
	Cmd_t back = t->back;
	if(fork() == 0){
     	Cmd_run(back);
    }
    //wait(0);		// don't care child
    break;
}

case CMD_PIPE: {
	struct Cmd_Pipe *t = (struct Cmd_Pipe *) cmd;
	Cmd_t left = t->left;
	Cmd_t right = t->right;
	int fd[2];
	pipe(fd);

    if(fork() == 0) {	// run left
       	close(1);
       	dup(fd[1]);
        close(fd[0]);
       	close(fd[1]);
       	Cmd_run(left);
	}
    wait(0);

    if(fork() == 0){	// run right
      	close(0);
       	dup(fd[0]);
        close(fd[0]);
        close(fd[1]);
     	Cmd_run(right);
   	}
    close(fd[0]);
    close(fd[1]);
    wait(0);
    break;
}

case CMD_REDIR: {
    struct Cmd_Redir *t = (struct Cmd_Redir *) cmd;
    Cmd_t left = t->left;
    Cmd_t right = t->right;

    Cmd_Atom redir = (Cmd_Atom) right; 	// get redir filename
    char *filename;
   	struct node *node = redir->node;
    filename = node->data;

    close(t->fd);  // close fd 0/1
    open(filename, O_CREAT | O_RDWR, 0777); // new file obtain fd 0/1
    Cmd_run(left);
    break;
}
```

运行结果如下：

![image-20221004234347458](C:/Users/17622/AppData/Roaming/Typora/typora-user-images/image-20221004234347458.png)

## Challenge:

Implement the input redirection.

### Solution

+ 修改`parser.y`

```c
%left '|' ';'
%left '>'
%left '&'
%left '<'

%%

line            :  command '\n'	    		{ root = $1; $$ = 0; return 0; }
		        | '\n'       		{ root = 0; $$ = 0; return 0; }
                ;
                
command	        :  basic_command		    { $$ = $1;}
		        |  command ';' command  	{ $$ = Cmd_Seq_new($1, $3);}
		        |  command '&'			    { $$ = Cmd_Back_new($1);}
		        |  command '|' command 		{ $$ = Cmd_Pipe_new($1, $3);}
		        |  command '>' command		{ $$ = Cmd_Redir_new($1, $3, 1);}
		        |  command '<' command		{ $$ = Cmd_Redir_new($1, $3, 0);}
		        ;
```

+ 修改`scanner.l`

```c
<<EOF>>                                 {return 0;}
"&"|";"|"|"             			    {return yytext[0];}
">"                                     {return yytext[0];}
"<"                                     {return yytext[0];}
"\n"						            {return yytext[0]; }
[ ]*					                {}
{ID}+				    		        {yylval = strdup(yytext); return T_ARG; }
.						                {printf("lex err:[%s]", yytext); return 0; }
```

+ 原`Redir`代码无需修改，因为`yyparse`会根据`<`和`>`调用不同的构造函数，区别在于传入的`fd`不同

```c
(yyval.cmd) = Cmd_Redir_new((yyvsp[-2].cmd), (yyvsp[0].cmd), 1);
(yyval.cmd) = Cmd_Redir_new((yyvsp[-2].cmd), (yyvsp[0].cmd), 0);
```

编译运行即可，效果如下：

![image-20221004234434766](C:/Users/17622/AppData/Roaming/Typora/typora-user-images/image-20221004234434766.png)
