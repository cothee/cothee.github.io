# the use of pipe #
  the head file of the pipe() function is **<unistd.h>**:
```cpp
#include <unistd.h>

int pipe(int fd[2]);
```
## 1. the use of pipe in inter-process communications ##
&ensp;&ensp;Pipe is often used for communications between processes which have genetic relationship with each other. the following is an example given by UNP(II):
```cpp
#include <unistd.h>      /*pipe() && fork() & write() & read() & STDOOUT_FILENO*/
#include <sys/types.h>  /* pid_t ssize_t*/
#include <sys/wait.h>  /* waitpid() */
#include <stdio.h>    /*fgets(), stdin*/
#include <string.h>  /*strlen*/ 

#define BUFFERSIZE 20

void client(int readfd, int writefd);
void server(int readfd, int writefd);
    
int main(int argc, char *argv[]) {
 int pipe1[2], pipe2[2];
 pid_t child_pid;
 
 pipe(pipe1);  /* create two pipes */
 pipe(pipe2);
 
 if( (child_pid = fork()) == 0) {    /*child process for server */
  close(pipe1[1]);  /* close write fd*/
  close(pipe2[0]);  /* close read fd */
   
  server(pipe1[0], pipe2[1]);  /*read from pipe1[0], and write to pipe2[1]*/
  exit(); 
 }
 
 close(pipe1[1]);
 close(pipe2[0]);
 
 client(pipe1[1], pipe2[0]); /*write to pipe1[0], and read from pipe2[1]*/
 
 waitpid(child_pid, NULL, 0);
 
}

void client(int readfd, int writefd) {
 size_t len;
 ssize_t n;
 char buff[BUFFERSIZE];
 
 fgets(buff, BUFFERSIZE, stdin);
 
 len = strlen(buff);

 if (buff[len-1] == '\n'){
  len--;
 }
 
 write(writefd, buff, len);
 
 while ((n = read(readfd, buff, BUFFERSIZE)) > 0) {
  write(writefd, buff, n);
 }
}

void server(int readfd, int writefd) {
 size_t len;
 ssize_t n;
 int fd;
 char buff[BUFFERSIZE+1];
 
 if (n = read(readfd, buff, BUFFERSIZE) == 0) {
  printf("server read error!");
  eixt(1);
 }
 buff[n] = '\0';
 
 if ((fd = open(buff,O_RDONLY)) < 0) {
  snprintf(buff + n, sizeof(buff) - n, "can not open: %s\n", stderror(errno));
  n = strlen(buff);
  write(writefd, buff, n);
 }
 else {
  while ((n = read(fd, buff, BUFFERSIZE)) > 0){
   write(writefd, buff, n);
  }
  close(fd);
 }

}
```

## 2. the use of pipe( combing with select) in controlling the executing time of a process ##
&ensp;&ensp; We can alose use pipe combing with select() to controlling the executing time of a child process, 
I first saw this kind of usage in one of our team projects named [Qconf](https://github.com/Qihoo360/QConf/blob/master/agent/qconf_script.cc#L50).
I'll just paste the codes here straightforward, and then give my understanding of these codes.
```cpp
int execute_script(const string &script, const long mtimeout)
{
 if (script.empty()) return QCONF_ERR_PARAM;

 int     rv = 0;
 int     pfd[2];
 pid_t   pid;
 fd_set  set;
 struct  timeval timeout;
 timeout.tv_sec = mtimeout / 1000;
 timeout.tv_usec = mtimeout % 1000 * 1000;

 struct sigaction act, oact;
 sigemptyset(&act.sa_mask);
 act.sa_handler = sig_child;
 act.sa_flags = SA_RESTART;
 sigaction(SIGCHLD, &act, &oact);

 if (pipe(pfd) < 0) {
  LOG_ERR("Failed to create pipe! error:%d", errno);
  return QCONF_ERR_OTHER; 
 }

 if ((pid = fork()) < 0) {
  LOG_ERR("Failed to fork process! error:%d", errno);
  return QCONF_ERR_OTHER; 
 } 
 else if (0 == pid) {
  setpgrp();
  close(pfd[0]);
  execl("/bin/sh", "sh", "-c", script.c_str(), (char *)NULL);
  _exit(127);  // won't be here, if execl was successful
 }

 close(pfd[1]);
 do {
     FD_ZERO(&set); 
     FD_SET(pfd[0], &set);
     rv = select(pfd[0] + 1, &set, NULL, NULL, &timeout);
     if (-1 == rv) {
      if (EINTR == errno) continue;
      else {
       LOG_ERR("Failed to select from pipe, error:%d!", errno);
        break;
      }
     }
     else if(0 == rv) {
      LOG_FATAL_ERR("Script execute timeout! script:%s", script.c_str());
      break;
     }
     else {
      close(pfd[0]); // sub process has exit
      return QCONF_OK;
     }
    } while (true);

    close(pfd[0]);
    kill(-pid, SIGKILL);
    if (0 == rv) 
      return QCONF_ERR_SCRIPT_TIMEOUT;
      
    return QCONF_ERR_OTHER;
}

static void sig_child(int sig) {
 pid_t pid;
 int status;
 while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
  if (WIFEXITED(status)) {
   if (WEXITSTATUS(status)) {
    LOG_ERR("Script not exit with success, ret:%d", WEXITSTATUS(status));
   }
  }
  else {
   LOG_ERR("Child process didnot terminate normally, pid:%d", pid);
  }
 }
}

```
continue ……


