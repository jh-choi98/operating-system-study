# Process

An instance of a program running
When we execute our program, it becomes a rpocess which performs all the tasks mentioned in the program
When a program is loaded into the memory and it becomes a process, it can be divided into 4 sections - stack, heap, text and data

- Stack: the process Stack contains the temporary data such as method/function parameters, return address and local variables
- Heap: dynamically allocated memory to a process during its runtime
- Text: includes the current activity represented by the value of Program Counter and the contents of the processor's registers
- Data: contains the global and static variables

## User view of processes

### Creating processes

- int fork(void);
  - creates new rpocess that is eact copy of current one
  - returns rpocess ID of new process in parent
  - returns 0 in child
- int waitpid(int pid, int \*stat, int opt);
  - waits for a child process to terminate
  - pid - process to wait for or -1 for any
  - stat - will contain exit value or signal
  - opt - usually 0 or WNOHANG
    - WNOHANG: Child process가 아직 종료되지 않았더라도 waitpid가 즉시 반환되도록 한다. CP 종료시에는 CP's pid를, CP 종료 전에는 0을 반환
      (원래 waitpid는 CP가 종료될 때까지 차단되는데, WNOHANG을 통해 함수는 차단되지 않고 다른 process를 수행하게 한다.)
  - Returns process ID or -1 on error

### Deleting processes

- void exit(int status);
  - current process ceases to exist
  - status shows up in waitpid
  - By convention, status of 0 is success, non-zero error
- int kill(int pid, int sig);
  - sends signal sig to process pid
  - SIGTERM kills process by default
  - SIGKILL kills process always (stronger)

### Running programs

- int execve(char \*prog, char **argv, char **envp);

  - execute a new program
  - prog: full pathname of program to run
  - argv: argument vector that gets passed to main
  - envp: environment variables ex. PATH, HOME
  - Generally called through a wrapper functions

    - int execvp(char \*prog, char \*\*argv);
      - Searches PATH for prog, uses current environment (process gets replaced)
    - int execlp(char *prog, char *arg, ...);
      - Lists arguments one at a time, finishing with NULL

    Example - minish.c:

        Parent Process(PID 5)
        pid_t pid; char \*\*av;
        void doexec() {
        execvp(av[0], av);
        perror(av[0]);
        exit(1);
        }

        for([infinite]) {
        parse_intput(&av, stdin);
        switch(pid = fork()) {
        case -1:
        perror("fork"); break;
        case 0:
        doexec(); // CP에서는 명령어 실행
        default:
        waitpid(pid, NULL, 0); break; // PP에서는 CP가 종료될 때까지 기다림
        }
        }

### File descriptor

An integer by OS to manage files or I/O devices. It abstracts the connection btw a program and the file system
A FD is a unique number assigned by OS to identify an open file. When a program opens a file, OS assigns it a FD. The program then can use this descriptor to perform read or write operations on the file.

Examle:

    int fd = open("example.txt", o_RDONLY); // opens teh file and gets a fd
    read(fd, buffer, size); // uses the fd to read the file
    close(fd); // closes the fd

File descriptors serve as an interface btw OS and programs, allowing efficient management of file or I/O device access

- int dup2(int oldfd, int newfd);

  - closes newfd, if it was a valid descriptor
  - makes newfd an exact copy of oldfd
  - two file descriptors will share same offset

    - offset: 파일에서 읽기나 쓰기가 이뤄질 위치
    - 한쪽으로 파일을 읽거나 쓰면 나머지쪽의 offset도 변경

    Example - redirsh.c:
