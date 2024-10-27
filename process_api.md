1. int fork(void)
   - creates a new process that is an exact copy of current one (not 100% exact though)
   - returns process ID of new process in parent
   - returns 0 in child

```c
// Calling fork() (p1.c)
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("hello (pid: %d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("child (pid:%d) \n", (int) getpid());
    } else {
        // parent goes down this path (main)
        printf("parent of %d (pid:%d)\n", rc, (int) getpid());
    }
    return 0;
}
/*
prompt> ./p1
hello (pid:29146)
child (pid:29147)
parent of 29147 (pid:29146)
prompt>

- The child just comes into life as if it had called fork() itself. DOES NOT start from main()
- NOT deterministic: the child or the parent might run at that point (자식이 먼저 실행될지, 부모가 먼저 실행될지 예측할 수 없다.)
=> Non-determinism

*/
```

2. int waitpid(int pid, int \*stat, int opt);
   - waits for a child process to terminate => makes output deterministic
   - pid: process to wait for , or -1 for any
   - stat: will contain exit value, or signal
   - opt: usually 0 or WNOHANG
   - returns process ID or -1 on error

```c
// Calling fork() and wait() (p2.c)
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("hello (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (main)
        int rc_wait = waitpid(rc, NULL, 0);
        printf("parent of %d (rc_wait: %d) (pid:%d)\n", rc, rc_wait, (int) getpid());
    }
    return 0;
}
/*
prompt> ./p2
hello (pid:29266)
child (pid:29267)
parent of 29267 (rc_wait:29267) (pid:29266)
prompt>


wait() 함수는 부모 프로세스가 자식 프로세스가 종료될 때까지 기다리도록 한다. 따라서 자식 프로세스가 먼저 실행되고 종료된 후에 부모 프로세스가 실행되도록 보장한다.

- useful when you wanna keep running copies of the same program
*/
```

3. int execvp(char \*prog, char \*\*argv);
   - executes a new program
   - searches PATH for prog and use current environment
   - prog: full pathname of program to run
   - argv: argument vector that gets passed to main
   - useful when you wanna run a program that is different from the calling program

```c
// Calling fork(), wait(), and exec() (p3.c)
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    printf("hello (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("child (pid:%d)\n", (int) getpid());
        char *myargs[3];
        myargs[0] = strdup("wc"); // program: wc
        myargs[1] = strdup("p3.c"); // arg: input file
        myargs[2] = NULL; // mark end of array
        execvp(myargs[0], myargs); // runs word count
        printf("this shouldn't print out");
    } else {
        // parent goes down this path (main)
        int rc_wait = waitpid(rc, NULL, 0);
        printf("parent of %d (rc_wait: %d) (pid:%d)\n", rc, rc_wait, (int) getpid());
    }
    return 0;
}

/*
prompt> ./p3
hello (pid:29383)
child (pid:29384)
      29     107    1030 p3.c
parent of 29384 (rc_wait:29384) (pid:29383)
prompt>


The child process calls execvp() in order to run the program wc, which is the word counting program. In fact, it runs wc on the source file p3.c

Note that fork() and exec() are not normal. exec() loads code from the executable and overwrites its current code segment. Thus, it does not create a new process; rather, it transforms the currently running program (formerly p3) into a different running program (wc)
 */
```

```c
// All of the above with redirection
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child: redirect standard output to a file
        close(STDOUT_FILENO);
        open("./p4.output", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);
        // now exec wc
        char *myargs[3];
        myargs[0] = strdup("wc");   // program: wc
        myargs[1] = strdup("p4.c"); // arg: file to count
        myargs[2] = NULL;           // mark end of array
        execvp(myargs[0], myargs); // runs word count
    } else {
        // parent goes down this path (main)
        int rc_wait = waidpid(rc, NULL, 0);
    }
    return 0;
}
```

4. int kill(int pid, int sig);

   - sends signal sig to process pid
   - SIGTERM most common value, kills process by default
   - SIGKILL stronger, kills process always

5. void exit(int status);
   - current process ceases to exist
   - status shows up in waitpid (shifted)
   - by convention, status of 0 is success, non-zero error
