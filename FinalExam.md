# UnixProgramming

## 20181029

- p8-1.c
  Parent process는 세 개의 child process를 만들고, 1초 간격으로 각 child process에게 SIGUSR1 signal을 보낸 후, child process들이 종료 할 때까지 대기 하였다가, 종료한 child process의 id와 exit status를 출력 후 종료 합니다. 각 child process는 생성 후 parent로부터 signal이 올 때까지 대기 하였다가 (pause() 사용), signal을 받으면 자신의 process id를 세 번 출력 한 후 종료 합니다. 이와 같이 동작하는 프로그램을 아래 코드를 사용하여 작성 하시오.

  ```c
  #define BUFSIZE 512
  
  void catchusr(int);
  
  void do_child(int i){
          int pid;
          static struct sigaction act;
  
          act.sa_handler = catchusr;
          sigaction(SIGUSR1, &act, NULL);
  
          printf("%d-th child is created...\n", i);
  
          pause();
  
          pid = getpid();
  
          for(i=0;i<3;i++){
                  printf("[child process id] : %d ...\n", getpid());
          }
  
          exit(0);
  }
  
  void catchusr(int signo){
          printf(".....catch SIGUSR1 = %d\n", signo);
  }
  
  int main(int argc, char **argv){
  
          int i, k, status;
          pid_t pid[3];
  
          for(i=0;i<3;i++){
                  pid[i] =fork();
                  if(pid[i] == 0){
                          do_child(i);
                  }
          }
  
          for(i=0;i<3;i++){
                  sleep(1);
                  kill(pid[i], SIGUSR1);
          }
  
          for(i=0;i<3;i++){
                  k = wait(&status);
                  printf("child #%d is terminated...%d\n", k, WEXITSTATUS(status));
          }
  
          exit(0);
  }
  ```

- P8-2.c
  Parent process는 다섯 개의 child process들을 만들고, 대기하였다가 child process가 모두 종료 한 후 종료 합니다. 각 child process는 자신의 순서가 될 때까지 대기 하였다가, 1초씩 쉬면서 (sleep (1); 사용) 자신의 process id(getpid(); 사용)를 5회 출력하는 작업을 한 후 종료 합니다. child process의 id 츨력 순서는 생성 순서의 역순이며, 이와 같은 순서 동기화 작업은 signal을 보내서 진행 합니다.

  - 생성 순서의 역순으로 작업 수행해야하니까, 첫 번째로 만들어진 child 이외의 child들은 pause로 기다림

  ```c
  void catchusr(int);
  
  void do_child(int i, int *pid_array){
          int j, pid;
          static struct sigaction act;
  
          act.sa_handler = catchusr;
          sigaction(SIGUSR1, &act, NULL);
  
          printf("%d-th child is created ... pid = %d\n", i, getpid());
  
          //생성 순서의 역순으로 작업 수행해야하니까, 첫 번째로 만들어진 child 이외의 child들은 pause로 기다림
          if(i<4)
                  pause();
  
          pid = getpid();
  
          for(j=0;j<5;j++){
                  printf("[child process id] : %d ...\n", getpid());
                  sleep(1);
          }
  
          if(i>0){
                  kill(pid_array[i-1], SIGUSR1);
          }
  
          exit(0);
  }
  
  void catchusr(int signo){
          printf(".....catch SIGUSR1 = %d\n", signo);
  }
  
  int main(int argc, char **argv){
  
          int i, status;
          pid_t pid[5];
  
          for(i=0;i<5;i++){
                  pid[i] =fork();
                  if(pid[i] == 0){
                          do_child(i, pid);
                  }
          }
  
          for(i=0;i<5;i++){
                  wait(&status);
          }
  
          exit(0);
  }
  ```

- P8-3.c
  10개의 정수를 입력으로 받아 그 합을 출력하는 프로그램을 작성 합니다. 단, 10초간 사용자로부터 의 입력이 없으면, 경고 메시지를 출력하고 다시 입력을 기다립니다.

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  
  #define BUFSIZE 512
  
  void catchalarm(int);
  
  int main(int argc, char **argv){
  
          int i, num, sum = 0;
          static struct sigaction act;
  
          act.sa_handler = catchalarm;
          sigaction(SIGALRM, &act, NULL);
  
          for(i=0;i<10;i++){
                  do{
                          alarm(10);
                  }while(scanf("%d", &num) < 0);
                  alarm(0);
                  sum += num;
                  printf("sum = %d\n", sum);
          }
          exit(0);
  }
  
  void catchalarm(int signo){
          printf("[Warning]No input for 10 seconds ... %d\n", signo);
  }
  ```

## 20181031

- 원격 서버의 디렉토리 로컬에 저장. scp 명령

  - -r 옵션 주면 디렉토리 복사 가능
  - -r 옵션 안 주면 파일만 복사 가능

  ```
  $ scp -P 1074 -r s13011022@203.250.148.93:/home/account/class/tspark/s13011022 ./unix
  
  [참고]
  http://faq.hostway.co.kr/?mid=Linux_ETC&page=8&document_srl=1426
  https://skylit.tistory.com/90
  ```


## 20181105

- p9-1
  메모리 매핑을 이용한 두 개의 프로그램을 작성 합니다.

  - (a) reader 프로그램은 “temp" 파일을 메모리 매핑 한 후, scanf()로 10개의 정수를 읽어서 매핑된 파일에 저장하는 작업을 10회 실행 합니다.

    ```c
    #define BUFSIZE 512
    
    int main(int argc, char **argv){
            int i, fd;
            int *addr;
    
            fd = open("temp", O_RDWR | O_CREAT, 0600);
            addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
        
            ftruncate(fd, sizeof(int)*10);
    
            for(i=0;i<10;i++){
                    scanf("%d", addr+i);
            }
            exit(0);
    }
    ```

  - (b) writer 프로그램은 “temp" 파일을 메모리 매핑 한 후, 매핑된 파일에 있는 정수를 출력하는 작업을 10회 실행합니다 (printf() 사용). 단, 출력 프로그램은 5초간 sleep() 한 후 5회의 출력 작업을 연속 진 행 하고 다시 5초간 sleep() 한 후 5회의 출력 작업을 진행 합니다.

    ```c
    #define BUFSIZE 512
    
    int main(int argc, char **argv){
            int i, j, fd;
            int *addr;
    
            fd = open("temp", O_RDWR | O_CREAT, 0600);
            addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    
        	//굳이 sleep해주는건 동기화 때문에
            sleep(5);
    
            for(i=0;i<10;i++){
                    sleep(5);
                    for(j=0;j<5;j++){
                            printf("%d ", *(addr+i));
                    }
                    printf("\n");
            }
            exit(0);
    }
    ```

  - (c) 두 프로그램이 모두 종료 한 후 ”temp" 파일의 크기를 확인합니다.

    ```c
    40으로 동일
    ```

- P9-2
  메모리 매핑을 이용한 두 개의 프로그램을 작성 합니다.

  - (a) reader 프로그램은 “temp" 파일을 메모리 매핑 한 후, 외부 입력을 읽어서 (read() 시스템 콜 사용) 매핑된 메모리에 저장하는 작업을 3회 실행 합니다.

    ```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <sys/wait.h>
    #include <fcntl.h>
    #include <unistd.h>
    #include <dirent.h>
    #include <string.h>
    #include <time.h>
    #include <ftw.h>
    #include <stdlib.h>
    #include <sys/mman.h>
    
    #define BUFSIZE 512
    
    int main(int argc, char **argv){
    
            int i, fd, len = 0;
            char *addr;
    
            fd = open("temp", O_RDWR | O_CREAT, 0600);
    
            addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
    
            ftruncate(fd, BUFSIZE);
            for(i=0;i<3;i++){
                    len = len + read(0, addr + len, BUFSIZE);
                    printf("[read] : %d\n", len);
                    if(len > BUFSIZE)
                            break;
            }
    
            exit(0);
    }
    ```

  - (b) writer 프로그램은 “temp" 파일을 메모리 매핑 한 후, 매핑된 메모리의 내용을 출력하는 작업을 3회 실행합니다 (write() 시스템 콜 사용). 단. 3초간 sleep() 하면서 출력 작업을 진행합니다.

    ```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <sys/wait.h>
    #include <fcntl.h>
    #include <unistd.h>
    #include <dirent.h>
    #include <string.h>
    #include <time.h>
    #include <ftw.h>
    #include <stdlib.h>
    #include <sys/mman.h>
    
    #define BUFSIZE 512
    
    int main(int argc, char **argv){
    
            int i, fd, len = 0;
            char *addr;
    
            fd = open("temp", O_RDWR | O_CREAT, 0600);
    
            addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
            printf("[test]%s\n", addr);
    
            for(i=0;i<3;i++){
                    sleep(3);
                    len = len + write(1, addr + len, BUFSIZE);
                    write(1, "-------\n", 8);
                    if(len > BUFSIZE)
                            break;
            }
    
            exit(0);
    }
    ```

- p9-3.c
  Parent process는 세 개의 child process들을 만들고, 모든 child process가 종료 한 후 종료합니다. 각 child process는 자신의 순서가 될 때까지 대기 하였다가, 1초씩 쉬면서 자신의 process id를 5 회 출력 한 후 종료합니다. child process의 id 츨력 순서는 생성 순서의 역순이며, 이와 같은 순서 동 기화 작업은 매핑된 파일을 이용하여 진행 합니다.

  - 일일이 메모리 맵핑하지 말고, 메모리 매핑한 배열을 child 함수 인자로 넘겨줘서 실행하면 더 간단하게 코드 작성 가능

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  
  #define BUFSIZE 512
  
  void do_child(int i, int *addr){
          int fd, child, j;
  
          child = *(addr + 0);
  
          while(i != child){
                  sleep(2);
                  printf(".........................pid : %d waiting\n", i);
  
                  child = *(addr + 0);
          }
  
          for(j=0;j<5;j++){
                  sleep(1);
                  printf("%dth child pid = %d\n", i, getpid());
          }
          *(addr + 0) = child - 1;
  
          exit(0);
  }
  
  int main(int argc, char **argv){
  
          int i, status, fd;
          pid_t pid[3];
          int *addr;
  
          fd = open("temp", O_RDWR | O_CREAT | O_TRUNC, 0600);
          addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
          ftruncate(fd, sizeof(int));
          *(addr + 0) = 2;
  
          for(i=0;i<3;i++){
                  pid[i] =fork();
                  if(pid[i] == 0){
                          do_child(i, addr);
                  }
          }
  
          for(i=0;i<3;i++){
                  wait(&status);
          }
  
          exit(0);
  }
  ```

  - 교수님 답안

    - 일일이 메모리 맵핑하지 말고, 메모리 매핑한 배열을 child 함수 인자로 넘겨줘서 실행하면 더 간단하게 코드 작성 가능

    ```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <sys/wait.h>
    #include <fcntl.h>
    #include <unistd.h>
    #include <dirent.h>
    #include <string.h>
    #include <time.h>
    #include <ftw.h>
    #include <stdlib.h>
    #include <sys/mman.h>
    
    #define BUFSIZE 512
    
    void do_child(int i, int *turn){
            int j, pid;
    
            if(i<2){
                    do{
                    }while(i!=*turn);
            }
    
            pid = getpid();
    
            for(j=0;j<5;j++){
                    printf("child %d ... \n", pid);
                    sleep(1);
            }
    
            if(i>0)
                    *turn = i-1;
    
            exit(0);
    }
    
    int main(int argc, char **argv){
            int i, status, fd;
            pid_t pid[3];
            int *addr;
    
            fd = open("temp", O_RDWR | O_CREAT | O_TRUNC, 0600);
            addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
            ftruncate(fd, sizeof(int));
            *addr = 2;
    
            for(i=0;i<3;i++){
                    pid[i] =fork();
                    if(pid[i] == 0){
                            do_child(i, addr);
                    }
            }
    
            for(i=0;i<3;i++){
                    wait(&status);
            }
    
            exit(0);
    }
    ```

## 20181107

- p10-1.c
  parent process는 세 개의 child process를 만들고, parent process에서 child process로의 단방향 통신이 가능 한 pipe를 만듭니다. parent process는 외부 입력으로 정수를 12개 입력 받아, 세 child process에게 순서대로 보냅니다. child process는 자신이 받은 정수를 자신의 프로세스 id와 함께 출력 합니다. parent process는 모든 정수를 전달 한 후 정수 -1을 전달하고, -1을 전달 받으면 child process는 종료 합니다. 모든 child process의 종료를 확인 한 후 parent process는 종료 합니다.

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  
  #define BUFSIZE 512
  
  void do_child(int id, int p[3][2]){
      int i, in, pid = getpid();
  
      for(i=0;i<3;i++){
          close(p[i][1]);
          if(id != i)
              close(p[i][0]);
      }
  
      while(1){
          read(p[id][0], &in, sizeof(int));
          if(in == -1)
              exit(0);
          else
              printf("id : %d, pid : %d, input : %d\n", id, pid, in);
      }
  }
  
  int main(int argc, char **argv){
      int i, in, pid, p[3][2];
  
      for(i=0;i<3;i++){
          pipe(p[i]);
      }
  
      for(i=0;i<3;i++){
          if(fork() == 0){
              do_child(i, p);
          }
      }
  
      for(i=0;i<3;i++){
          close(p[i][0]);
      }
  
      for(i=0;i<12;i++){
          scanf("%d", &in);
          write(p[i%3][1], &in, sizeof(int));
      }
  
      in = -1;
  
      for(i=0;i<3;i++){
          write(p[i][1], &in, sizeof(int));
      }
  
      for(i=0;i<3;i++){
          wait(0);
      }
  
      exit(0);
  }
  ```

- p10-2.c
  parent process는 세 개의 child process를 만들고, 모든 child process가 종료 한 후 종료 합니다. 각 child process는 자신의 순서가 될 때까지 대기 하였다가, 1초씩 쉬면서 (sleep (1); 사용) 자신의 process id(getpid(); 사용)를 5회 출력하는 작업을 한 후 종료 합니다. child process의 id 츨력 순서는 생성 순서의 역순이며, 이와 같은 순서 동기화 작업은 pipe를 이용하여 진행 합니다.

  - signal에서의 pause / kill은,,, pipe에서 read / write라고 볼 수 있다.
  - pipe에서 기본으로 blocking read로 설정이 되기 때문에 write가 오기 전까지 read는 block이 되기 때문에 기다릴 수 있다

  ```c
  #define BUFSIZE 512
  
  void do_child(int id, int pipe[2][2]){
          char buffer = 'a';
          int i, pid = getpid();
  
          if(id < 2){
                  read(pipe[id][0], &buffer, 1);
          }
  
          for(i=0;i<5;i++){
                  sleep(1);
                  printf("id : %d, pid : %d\n", id, pid);
          }
  
          if(id > 0){
                  write(pipe[id-1][1], &buffer, 1);
          }
  
          exit(0);
  }
  
  int main(int argc, char **argv){
          int i, status, fd, p[2][2];
          pid_t pid[3];
  
          for(i=0;i<2;i++){
                  pipe(p[i]);
          }
  
          for(i=0;i<3;i++){
                  pid[i] =fork();
                  if(pid[i] == 0){
                          do_child(i, p);
                  }
          }
  
          for(i=0;i<3;i++){
                  wait(&status);
          }
  
          exit(0);
  }
  ```


## 20181112

- p11-1
  fifo를 이용하여 통신하는 두 개의 프로그램을 작성합니다. 프로그램 A는 외부 입력으로 정수를 입력 받아 프로그램 B에 전달합니다. 프로그램 B는 전달 받은 정수에 +8을 한 뒤 프로그램 A에 돌려줍니다. 프로그램 A는 돌려받은 정수 값을 출력 합니다. -1이 입력되면 두 프로그램은 종료합니다.

  - O_WRONLY, O_RDONLY 명시해주는 순서가 중요함

    ```c
    //A
    fd[0] = open(fifo[0], O_WRONLY);
    fd[1] = open(fifo[1], O_RDONLY);
    ```

    ```c
    //B
    fd[0] = open(fifo[0], O_RDONLY);
    fd[1] = open(fifo[1], O_WRONLY);
    ```

  - fifo f[0]을 이용한 파이프 fd[0]을 A에서 WRONLY로 만들었으면 RDONLY로 짝을 맞춰줘야하는데, 아니게 되면 서로를 계속해서 기다리게 되는 deadlock 상태가 발생하게 된다. 두 개만 있을 때는 적어서 큰 문제가 안 생기는데, 더 많은 프로그램이 개입을 하게되면 빈번하게 문제가 발생하게 된다.

  p11-1A.c

  ```c
  int main(int argc, char **argv){
          char fifo[2][3] = {"f1", "f2"};
          int i, in, fd[2];
  
          for(i=0;i<2;i++){
                  mkfifo(fifo[i], 0600);
          }
  
          fd[0] = open(fifo[0], O_WRONLY);
          fd[1] = open(fifo[1], O_RDONLY);
  
          for(;;){
                  scanf("%d", &in);
                  write(fd[0], &in, sizeof(int));
                  if(in == -1)
                          exit(0);
                  read(fd[1], &in, sizeof(int));
                  printf("final A received : %d\n", in);
          }
  
          exit(0);
  }
  ```

  p11-1B.c

  ```c
  int main(int argc, char **argv){
          char fifo[2][3] = {"f1", "f2"};
          int i, in, fd[2];
  
          fd[0] = open(fifo[0], O_RDONLY);
          fd[1] = open(fifo[1], O_WRONLY);
  
          for(;;){
                  read(fd[0], &in, sizeof(int));
                  printf("first B received : %d\n", in);
                  if(in == -1)
                          exit(0);
                  in = in + 8;
                  write(fd[1], &in, sizeof(int));
          }
  
          exit(0);
  }
  ```

- p11-2
  server process는 세 개의 client process들과 데이터를 주고받기 위한 fifo를 만듭니다. 각 client는 미리 정해진 이름의 FIFO로 접속하여, 표준 입력으로 입력된 정수를 server process에게 전송합니다. server process는 client process로부터 전송된 정수 값에 +8을 한 후, 해당 client에게 다시 보냅니다. client process는 돌려받은 정수 값을 표준 출력으로 출력합니다. client process는 정수 데이터의 입/출력 작업을 5회 반복한 후 종료 합니다. client process로 부터의 입력을 blocking으로 기다리기 위해 select 문장을 사용합니다.

  - O_RDONLY 할 때, O_RDWR을 해야하는지 아니면 O_RDONLY를 해야하는지 아직 모르겠음
  - p11-2server.c (학교 리눅스 서버 버전). 
    - 교수님 수업 시간때 말씀하신거 기반으로 작성한 코드라 이게 정답인듯함. client프로그램 전부 닫히면 전부 0을 return 하여 select는 3을 반환하게 되는거고, 근데 read한 값이 0바이트인 경우 모든 프로그램이 종료한 경우라고 하셔서 이렇게 작성했지만, 혹시나 잘못된 경우를 생각해봐야함

  ```c
  int main(int argc, char **argv) {
      char fifo[6][15] = {"p2_f1", "p2_f2", "p2_f3", "p2_f4", "p2_f5", "p2_f6"};
      int i, in, fd[6], select_check, nread, nread_count[3] = {0};
      fd_set set, master;
  
      for(i=0;i<6;i++){
          mkfifo(fifo[i], 0600);
      }
  
      fd[0] = open(fifo[0], O_RDONLY);
      fd[1] = open(fifo[1], O_RDONLY);
      fd[2] = open(fifo[2], O_RDONLY);
  
      fd[3] = open(fifo[3], O_WRONLY);
      fd[4] = open(fifo[4], O_WRONLY);
      fd[5] = open(fifo[5], O_WRONLY);
  
      FD_ZERO(&master);
  
      for (i=0;i<3;i++){
          FD_SET(fd[i], &master);
      }
  
      while (set=master, (select_check = select(fd[2]+1, &set, NULL, NULL, NULL)) > 0) {
          for (i=0;i<3;i++) {
              if (FD_ISSET(fd[i], &set)) {
                  if((nread = read(fd[i], &in, sizeof(int))) > 0){
                      printf("received : %d\n", in);
                      in = in + 8;
                      write(fd[i+3], &in, sizeof(int));
                  }
                  else if(select_check == 3 && nread == 0)
                      return 0;
              }
          }
      }
      exit(0);
  }
  ```

  - p11-2server.c (내 컴퓨터 버전)

  ```c
  int main(int argc, char **argv) {
      char fifo[6][15] = {"p2_f1", "p2_f2", "p2_f3", "p2_f4", "p2_f5", "p2_f6"};
      int i, in, fd[6], select_check, nread, nread_count[3] = {0};
      fd_set set, master;
  
      for(i=0;i<6;i++){
          mkfifo(fifo[i], 0600);
      }
  
      fd[0] = open(fifo[0], O_RDONLY);
      fd[1] = open(fifo[1], O_RDONLY);
      fd[2] = open(fifo[2], O_RDONLY);
  
      fd[3] = open(fifo[3], O_WRONLY);
      fd[4] = open(fifo[4], O_WRONLY);
      fd[5] = open(fifo[5], O_WRONLY);
  
      FD_ZERO(&master);
  
      for (i=0;i<3;i++){
          FD_SET(fd[i], &master);
      }
  
      while (set=master, (select_check = select(fd[2]+1, &set, NULL, NULL, NULL)) > 0) {
          for (i=0;i<3;i++) {
              if (FD_ISSET(fd[i], &set)) {
                  if((nread = read(fd[i], &in, sizeof(int))) > 0){
                      printf("received : %d\n", in);
                      in = in + 8;
                      write(fd[i+3], &in, sizeof(int));
                  }
                  else if(nread == 0)
                      nread_count[i] = 1;
              }
          }
          if(nread_count[0] == 1 && nread_count[1] == 1 && nread_count[2] == 1)
              return 0;
      }
      exit(0);
  }
  ```

  - p11-2client.c

  ```c
  int main(int argc, char **argv){
      char fifo[6][15] = {"p2_f1", "p2_f2", "p2_f3", "p2_f4", "p2_f5", "p2_f6"};
      int i, in, fd[6];
      int id = atoi(argv[1]);
  
      fd[id] = open(fifo[id], O_WRONLY);
      fd[id+3] = open(fifo[id+3], O_RDONLY);
  
      scanf("%d", &in);
      write(fd[id], &in, sizeof(int));
      read(fd[id+3], &in, sizeof(int));
      for(i=0;i<5;i++){
          sleep(1);
          printf("received : %d\n", in);
      }
      exit(0);
  }
  ```

- p11-3
  네 개의 프로세스가 동기화를 하며 자신의 프로세스 id를 5회 출력하는 프로그램을 작성합니다. 이 프로그램은 main() 함수의 arguments로 동기화에 참여하는 전체 프로세스 중 자신의 출력 순서를 입력받습니다. 프로그램이 시작되면, 순서대로 자신의 프로세스 id를 출력합니다. 동기화 작업은 fifo를 사용하여 수행합니다.

  - 네 개의 프로그램을 각각 실행한 후에 메인함수 인자로 0, 1, 2, 3을 주면 순서대로 실행 후 종료를 수행한다.

  ```c
  int main(int argc, char **argv){
      char buf, f[3][3] = {"f1", "f2", "f3"};
      int i, k ,in ,fd[2];
  
      for(i=0;i<3;i++){
          mkfifo(f[i], 0600);
      }
  
      k = atoi(argv[1]);
  
      if(k > 0)
          fd[0] = open(f[k-1], O_RDONLY);
      if(k < 3)
          fd[1] = open(f[k], O_WRONLY);
  
      if(k > 0)
          read(fd[0], &buf, 1);
  
      for(i=0;i<5;i++){
          printf("pid : %d\n", getpid());
          sleep(1);
      }
  
      if(k < 3)
          write(fd[1], "a", 1);
  
      exit(0);
  }
  ```

## 20181114

- p12-1
  server process는 세 개의 client process들과 데이터를 주고받기 위해 message queue를 만듭니다. 각 client는 message queue를 이용하여, 표준 입력으로 입력된 정수를 server process에게 전송합니다. server process는 client process로부터 전송된 정수 값에 +8을 한 후, 해당 client에게 다시 보냅니다. client process는 돌려받은 정수 값을 표준 출력으로 출력합니다. client process는 정수 데이터의 입/출력 작업을 5회 반복 한 후 종료합니다. 각 client process는 main() 함수의 arguments로 자신의 id를 입력받습니다.

  - p12-1server.c

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  #include <sys/ipc.h>
  #include <sys/msg.h>
  
  struct q_entry{
      long mtype;
      int data;
  };
  
  int main(int argc, char **argv){
      int qid, i, in;
      struct q_entry msg;
      key_t key;
  
      key = ftok("key", 1);
      qid = msgget(key, 0600|IPC_CREAT);
  
      if(qid == -1){
          perror("msgget");
          exit(0);
      }
  
      for(i=0;i<15;i++){
          msgrcv(qid, &msg, sizeof(int), -3, 0);
          msg.mtype += 3;
          msg.data += 8;
          msgsnd(qid, &msg, sizeof(int), 0);
      }
      
      //메세지큐 기능 다 사용하고 삭제해주는 시스템 콜 추가
      msgctl(qid, IPC_RMID, 0);
  
      exit(0);
  }
  ```

  - p12-1client.c

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  #include <sys/ipc.h>
  #include <sys/msg.h>
  
  struct q_entry{
      long mtype;
      int data;
  };
  
  int main(int argc, char **argv){
      int qid, i, in, id;
      struct q_entry msg;
      key_t key;
  
      id = atoi(argv[1]);
  
      key = ftok("key", 1);
      qid = msgget(key, 0600|IPC_CREAT);
  
      if(qid == -1){
          perror("msgget");
          exit(0);
      }
  
      for(i=0;i<5;i++){
          scanf("%d", &in);
          msg.mtype = id;
          msg.data = in;
          msgsnd(qid, &msg, sizeof(int), 0);
          msgrcv(qid, &msg, sizeof(int), id+3, 0);
          printf("%d\n", msg.data);
      }
  
      exit(0);
  }
  ```

- p12-2.c
  네 개의 프로세스가 동기화를 하며 자신의 프로세스 id를 5회 출력하는 프로그램을 작성합니다. 이 프로그램은 main() 함수의 arguments로 동기화에 참여하는 전체 프로세스 중 자신의 출력 순서를 입력받습니다. 프로그램이 시작되면, 순서대로 자신의 프로세스 id를 출력합니다. 동기화 작업은 message queue를 사용하여 수행합니다.

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  #include <sys/ipc.h>
  #include <sys/msg.h>
  
  struct q_entry{
      long mtype;
      int data;
  };
  
  int main(int argc, char **argv){
      int i, id, qid;
      key_t key;
      struct q_entry msg;
  
      key = ftok("key", 3);
      qid = msgget(key, IPC_CREAT|0600);
  
      id = atoi(argv[1]);
  
      if(id > 1)
          msgrcv(qid, &msg, sizeof(int), id, 0);
  
      for(i=0;i<5;i++){
          sleep(1);
          printf("id = %d pid = %d\n", id, getpid());
      }
  
      if(id < 4){
          msg.mtype = id + 1;
          msg.data = id;
          msgsnd(qid, &msg, sizeof(int), 0);
      }
  
      exit(0);
  }
  ```

## 20181119

- p13-1
  메모리 매핑을 이용한 두 개의 프로그램을 작성합니다. 두 프로그램 모두 "data" 파일을 메모리 매핑합니다. 한 프로그램은 외부 입력을 읽어서 매핑된 메모리에 저장하는 작업을 3회 실행합니다. 다른 프로그램은 매핑된 메모리의 내용을 출력하는 작업을 3회 실행합니다. 두 프로그램 사이의 읽기-쓰기 동기화를 위해 semaphore를 사용합니다.

  p13-1a.c

  ```c
  union semun{
          int val;
          struct semid_ds *buf;
          ushort *array;
  };
  
  int main(int argc, char **argv){
          int i, len = 0, status, fd, semid;
          char *addr;
          key_t key;
          union semun arg;
          struct sembuf p_buf;
  
          key = ftok("key", 3);
          semid = semget(key, 1, 0600|IPC_CREAT|IPC_EXCL);
  
          if(semid == -1){
                  semid = semget(key, 1, 0600);
          }
          else{
                  arg.val = 0;
                  semctl(semid, 0, SETVAL, arg);
          }
  
          fd = open("temp", O_RDWR | O_CREAT | O_TRUNC, 0600);
          addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
  
          ftruncate(fd, BUFSIZE);
  
          for(i=0;i<3;i++){
                  len = len + read(0, addr + len, BUFSIZE);
                  len = len-1;
                  *(addr+len) = '\0';
  
                  p_buf.sem_num = 0;
                  p_buf.sem_op = 1;
                  p_buf.sem_flg = 0;
                  semop(semid, &p_buf, 1);
              
                  printf("semval : %d\n", semctl(semid, 0, GETVAL, arg));
          }
  
          exit(0);
  }
  ```

  p13-1b.c

  ```c
  union semun{
          int val;
          struct semid_ds *buf;
          ushort *array;
  };
  
  int main(int argc, char **argv){
          int i, len = 0, fd, semid;
          char *addr;
          key_t key;
          union semun arg;
          struct sembuf p_buf;
  
          key = ftok("key", 3);
          semid = semget(key, 1, 0600|IPC_CREAT|IPC_EXCL);
  
          if(semid == -1){
                  semid = semget(key, 1, 0600);
          }
          else{
                  arg.val = 0;
                  semctl(semid, 0, SETVAL, arg);
          }
  
          fd = open("temp", O_RDWR | O_CREAT, 0600);
  
          addr = mmap(NULL, BUFSIZE, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
  
          for(i=0;i<3;i++){
                  p_buf.sem_num = 0;
                  p_buf.sem_op = -1;
                  p_buf.sem_flg = 0;
                  semop(semid, &p_buf, 1);
  
                  printf("semval : %d\n", semctl(semid, 0, GETVAL, arg));
  
                  len = len + write(1, addr + len, strlen(addr+len));
                  write(1, "-------\n", 8);
          }
  
          exit(0);
  }
  ```

- p13-2.c
  네 개의 프로세스가 동기화를 하며 자신의 프로세스 id를 5회 출력하는 프로그램을 작성합니다. 이 프로그램은 main()함수의  arguments로 동기화에 참여하는 전체 프로세스 중 자신의 출력 순서를 입력받습니다. 프로그램이 시작되면, 순서대로 자신의 프로세스 id를 출력합니다. 동기화 작업은 semaphore를 사용하여 수행합니다.

  ```c
  union semun{
      int val;
      struct semid_ds *buf;
      ushort *array;
  };
  
  int main(int argc, char **argv){
      int i, id, pid, semid;
      key_t key;
      union semun arg;
      ushort buf[3] = {0};
      struct sembuf p_buf;
  
      id = atoi(argv[1]);
      key = ftok("key", 3);
      semid = semget(key, 3, 0600|IPC_CREAT|IPC_EXCL);
  
      if(semid == -1){
          semid = semget(key, 3, 0600);
      }
      else{
          arg.array = buf;
          semctl(semid, 0, SETALL, arg);
      }
  
      if(id>1){
          p_buf.sem_num = id-2;
          p_buf.sem_op = -1;
          p_buf.sem_flg = 0;
          semop(semid, &p_buf, 1);
      }
  
      pid = getpid();
  
      for(i=0;i<5;i++){
          sleep(1);
          printf("pid : %d\n", pid);
      }
  
      if(id<4){
          p_buf.sem_num = id-1;
          p_buf.sem_op = 1;
          p_buf.sem_flg = 0;
          semop(semid, &p_buf, 1);
      }
  
      exit(0);
  }
  ```

- p13-3.c
  2번 문제에서 더 적은 semaphore를 사용하여 코드 작성하시오.

  ```c
  union semun{
      int val;
      struct semid_ds *buf;
      ushort *array;
  };
  
  int main(int argc, char **argv){
      int i, id, pid, semid;
      key_t key;
      union semun arg;
      struct sembuf p_buf;
  
      id = atoi(argv[1]);
      key = ftok("key", 1);
      semid = semget(key, 3, 0600|IPC_CREAT|IPC_EXCL);
  
      if(semid == -1){
          semid = semget(key, 3, 0600);
      }
      else{
          arg.val = 0;
          semctl(semid, 0, SETVAL, arg);
      }
  
      if(id>1){
          p_buf.sem_num = 0;
          p_buf.sem_op = -id;
          p_buf.sem_flg = 0;
          semop(semid, &p_buf, 1);
      }
  
      pid = getpid();
  
      for(i=0;i<5;i++){
          sleep(1);
          printf("pid : %d\n", pid);
      }
  
      if(id<4){
          p_buf.sem_num = 0;
          p_buf.sem_op = id + 1;
          p_buf.sem_flg = 0;
          semop(semid, &p_buf, 1);
      }
  
      exit(0);
  }
  ```

- p13-4.c
  세 개의 프로세스가 동기화를 하며 자신의 프로세스 id를 5회 씩 3번 출력하는 프로그램을 작성합니다. 이 프로그램은 main() 함수의 arguments로 동기화에 참여하는 전체 프로세스 중 자신의 출력 순서를 입력받습니다. 프로그램이 시작되면, 1 → 2 → 3 → 1 → 2 → 3 → 1 → 2 → 3 순서대로 자신의 프로세스 id를 출력합니다.

  - (a) 동기화 작업은 message queue를 사용하여 수행 합니다.

  ```c
  struct q_entry{
      long mtype;
      int data;
  };
  
  int main(int argc, char **argv){
      int i, j, id, qid;
      key_t key;
      struct q_entry msg;
      
      key = ftok("key", 3);
      qid = msgget(key, IPC_CREAT|0600);
      
      id = atoi(argv[1]);
      
      for(i=4;i<7;i++){
          //id = 1,2,3
          //i = 4,5,6
          if(id > 1)
              msgrcv(qid, &msg, sizeof(int), id, 0);
          
          if(id == 1 && i > 4)
              msgrcv(qid, &msg, sizeof(int), i, 0);
          
          for(j=0;j<3;j++){
              sleep(1);
              printf("id = %d pid = %d\n", id, getpid());
          }
          
          if(id < 3){
              msg.mtype = id + 1;
              msg.data = id;
              msgsnd(qid, &msg, sizeof(int), 0);
          }
          else if(id == 3 && i < 6){
                  msg.mtype = i + 1;
                  msg.data = id;
                  msgsnd(qid, &msg, sizeof(int), 0);
          }
      }
      
      if(id == 3)
          msgctl(qid, IPC_RMID, 0);
      
      exit(0);
  }
  ```

  - (b) 동기화 작업은 semaphore를 사용하여 수행 합니다.

  ```c
  
  ```

## 20181121

- p14-1
  공유 메모리를 이용하는 두 개의 프로그램을 작성합니다. 프로그램 A는 scanf() 명령으로 10개의 정수를 입력 받아 공유 메모리에 저장 하는 작업을 10회 반복 실행합니다. 프로그램 B는 공유 메모리 에 저장된 내용을 printf() 명령으로 출력하는 작업을 10회 반복 실행합니다. 이때, 프로그램 B는 프로 그램 A가 정수를 쓴 후 읽어야 합니다. 이러한 동기화 작업은 semaphore를 사용합니다.

  - write

    ```c
    union semun{
            int val;
            struct semid_ds *buf;
            ushort *array;
    };
    
    int main(int argc, char **argv){
        key_t semkey, shmkey;
        int semid, shmid, i, n;
        int *buf;
        union semun arg;
        struct sembuf p_buf;
        
        semkey = ftok("semkey", 3);
        semid = semget(semkey, 1, 0600|IPC_CREAT|IPC_EXCL);
        
        if(semid == -1){
            semid = semget(semkey, 1, 0600);
        }
        else{
            arg.val = 0;
            semctl(semid, 0, SETVAL, arg);
        }
    
        shmkey = ftok("shmfile", 1);
        shmid = shmget(shmkey, 10*sizeof(int), IPC_CREAT|0600);
    
        buf = (int *)shmat(shmid, 0, 0);
    
        for(i=0;i<10;i++){
            scanf("%d", (buf+i));
            
            //signal
            p_buf.sem_num = 0;
            p_buf.sem_op = 1;
            p_buf.sem_flg = 0;
            semop(semid, &p_buf, 1);
        }
    
        semctl(semid, IPC_RMID, 0);
        shmdt(buf);
        shmctl(shmid, IPC_RMID, 0);
    
        return 0;
    }
    ```

  - read

    ```c
    union semun{
            int val;
            struct semid_ds *buf;
            ushort *array;
    };
    
    int main(int argc, char **argv){
        key_t semkey, shmkey;
        int semid, shmid, i, n;
        int *buf;
        union semun arg;
        struct sembuf p_buf;
        
        semkey = ftok("semkey", 3);
        semid = semget(semkey, 1, 0600|IPC_CREAT|IPC_EXCL);
        
        if(semid == -1){
            semid = semget(semkey, 1, 0600);
        }
        else{
            arg.val = 0;
            semctl(semid, 0, SETVAL, arg);
        }
    
        shmkey = ftok("shmfile", 1);
        shmid = shmget(shmkey, 10*sizeof(int), IPC_CREAT|0600);
    
        buf = (int *)shmat(shmid, 0, 0);
    
        for(i=0;i<10;i++){
            //wait
            p_buf.sem_num = 0;
            p_buf.sem_op = -1;
            p_buf.sem_flg = 0;
            semop(semid, &p_buf, 1);
            
            printf("%d\n", *(buf+i));
        }
    
        semctl(semid, IPC_RMID, 0);
        shmdt(buf);
        shmctl(shmid, IPC_RMID, 0);
    
        return 0;
    }
    ```

- p14-2
  1번의 프로그램을 공유 메모리 자체 정보에 의해 동기화 작업이 이루어지도록 수정 합니다.
  semaphore를 사용하지 않기 때문에 busy waiting으로 block을 할 수밖에 없어서, 비효율적인 프로그램이 된다.

  - write

    ```c
    struct databuf{
        int flag;
        int data;
    };
    
    int main(int argc, char **argv){
        key_t key;
        int shmid, i, n;
        struct databuf *buf;
    
        key = ftok("shmfile", 1);
        shmid = shmget(key, 10* sizeof(struct databuf), IPC_CREAT|0600);
    
        buf = (struct databuf *)shmat(shmid, 0, 0);
    
        for(i=0;i<10;i++){
            scanf("%d", &(buf+i)->data);
            (buf+i)->flag = 1;
        }
    
        shmdt(buf);
        shmctl(shmid, IPC_RMID, 0);
    
        return 0;
    }
    ```

  - read

    ```c
    struct databuf{
        int flag;
        int data;
    };
    
    int main(int argc, char **argv){
        key_t key;
        int shmid, i, n;
        struct databuf *buf;
    
        key = ftok("shmfile", 1);
        shmid = shmget(key, 10* sizeof(struct databuf), IPC_CREAT|0600);
    
        buf = (struct databuf *)shmat(shmid, 0, 0);
    
        for(i=0;i<10;i++){
            while((buf+i)->flag == 0);
            printf("%d\n", (buf+i)->data);
        }
    
        shmdt(buf);
        shmctl(shmid, IPC_RMID, 0);
    
        return 0;
    }
    ```

- p14-3
  Server process는 세 개의 client process들과 데이터를 주고받기 위해 공유 메모리를 만듭니다. 각 client는 공유 메모리 공간을 이용하여 표준 입력으로 입력된 정수를 server process에게 전송합니다. Server process는 client process로부터 전송된 정수 값에 +8을 한 후, 해당 client에게 다시 보냅니다. Client process는 돌려받은 정수 값을 표준 출력으로 출력합니다. Client process는 정수 데이터의 입/출력 작업을 5회 반복 한 후 종료합니다.

  - server랑 client 모두 busy waiting으로 기다리고 있음. 그래서 비효율적인 방법인데, shared memory는 생성할 때만 OS로 switching 일어나고 이후에는 context switching이 안 일어나고 프로세스에서 작업을 하기 때문에 그나마 효율적임

  - 그래서 세마포를 사용해주면 더 효과적인 코드가 될 수 있음

  - shmget할 때 EXCL 옵션을 안 붙이는 이유

    - Flag 값을 사용하여 데이터가 왔나, 안 왔나를 확인하기 때문에 처음에 0으로 모두 다 설정돼있어야함. 0보다 큰 값이 들어가 있으면 안 됨
    - shared memory를 만들면 기본으로 0으로 초기화되기 때문에 EXCL 옵션으로 다시 재설정을 해줄 필요가 없음
    - 혹시나 1로 모든 flag 값을 설정하고 1이 아닌 값이 들어오게 되면 데이터가 온 것으로 간주를 할 것이라면 EXCL 옵션을 사용하여 모든 flag값을 재설정해줄 필요가 있음.

  - server

    ```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <sys/wait.h>
    #include <fcntl.h>
    #include <unistd.h>
    #include <dirent.h>
    #include <string.h>
    #include <time.h>
    #include <ftw.h>
    #include <stdlib.h>
    #include <sys/mman.h>
    #include <sys/ipc.h>
    #include <sys/msg.h>
    #include <sys/sem.h>
    #include <sys/shm.h>
    
    #define BUFSIZE 512
    
    struct databuf{
        int flag;
        int data;
    };
    
    int main(int argc, char **argv){
        int i, j, shmid;
        struct databuf *buf;
        key_t shmkey;
        
        shmkey = ftok("shmkey", 3);
        shmid = shmget(shmkey, 6*sizeof(struct databuf), 0600|IPC_CREAT);
        buf = (struct databuf *)shmat(shmid, 0 ,0);
        
        for(i=0;i<15;i++){
            for(j=0;;j=(j+1)%3)
                if((buf+j)->flag == 1)
                    break;
            (buf+j+3)->data = (buf+j)->data + 8;
            (buf+j)->flag = 0;
            (buf+j+3)->flag = 1;
        }
        
        shmctl(shmid, IPC_RMID, 0);
    
        exit(0);
    }
    ```

  - client

    ```c
    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <sys/wait.h>
    #include <fcntl.h>
    #include <unistd.h>
    #include <dirent.h>
    #include <string.h>
    #include <time.h>
    #include <ftw.h>
    #include <stdlib.h>
    #include <sys/mman.h>
    #include <sys/ipc.h>
    #include <sys/msg.h>
    #include <sys/sem.h>
    #include <sys/shm.h>
    
    #define BUFSIZE 512
    
    struct databuf{
        int flag;
        int data;
    };
    
    int main(int argc, char **argv){
        int i, j, id, shmid;
        struct databuf *buf;
        key_t shmkey;
        
        id = atoi(argv[1]);
        shmkey = ftok("shmkey", 3);
        shmid = shmget(shmkey, 6*sizeof(struct databuf), 0600|IPC_CREAT);
        buf = (struct databuf *)shmat(shmid, 0, 0);
        
        for(i=0;i<5;i++){
            scanf("%d", &((buf+id)->data));
            (buf+id)->flag = 1;
            while((buf+id+3)->flag == 0);
            printf("id : %d, input + 8 = %d\n", id, (buf+id+3)->data);
            (buf+id+3)->flag = 0;
        }
    
        exit(0);
    }
    ```

- p14-4

  p14-3 문제 busy waiting 사용하지 말고 세마포 사용하여 만들어보기

  - 세마포 2개 만들어서 client에서 server로 보내는 동기화를 위한 세마포 하나, server에서 client로 보내는 동기화를 위한 세마포 하나, 의 두 개 세마포를 사용
  - 이전 busy waiting이랑 다르게 shared memory 구조체 말고 그냥 int 메모리 공간으로 사용.
  - flag 없이 int size 하나만 사용하여 작업
  - semget할 때 ```semid = semget(semkey, 2, 0600|IPC_CREAT|IPC_EXCL);``` 두 번째 인자는 세마포 집합 내의 세마포 갯수이니까 주의할 것

  - server

    ```c
    int main(int argc, char **argv){
        int i, semid, shmid;
        key_t semkey, shmkey;
        union semun arg;
        struct sembuf p_buf;
        ushort sembuf[2] = {0};
        int *shmbuf;
        
        semkey = ftok("semkey", 3);
        semid = semget(semkey, 2, 0600|IPC_CREAT|IPC_EXCL);
        
        if(semid == -1){
            semid = semget(semkey, 1, 0600);
        }
        else{
            arg.array = sembuf;
            semctl(semid, 0, SETALL, arg);
        }
        
        shmkey = ftok("shmkey", 3);
        shmid = shmget(shmkey, sizeof(int), 0600|IPC_CREAT);
        shmbuf = (int *)shmat(shmid, 0 ,0);
        
        for(i=0;i<15;i++){
            //0 wait
            p_buf.sem_num = 0;
            p_buf.sem_op = -1;
            p_buf.sem_flg = 0;
            semop(semid, &p_buf, 1);
            
            *(shmbuf+0) = *(shmbuf+0) + 8;
            
            //1 signal
            p_buf.sem_num = 1;
            p_buf.sem_op = 1;
            p_buf.sem_flg = 0;
            semop(semid, &p_buf, 1);
        }
        
        semctl(semid, IPC_RMID, 0);
        shmctl(shmid, IPC_RMID, 0);
    
        exit(0);
    }
    ```

  - client

    ```c
    int main(int argc, char **argv){
        int i, semid, shmid;
        key_t semkey, shmkey;
        union semun arg;
        struct sembuf p_buf;
        ushort sembuf[2] = {0};
        int *shmbuf;
        
        semkey = ftok("semkey", 3);
        semid = semget(semkey, 2, 0600|IPC_CREAT|IPC_EXCL);
        
        if(semid == -1){
            semid = semget(semkey, 1, 0600);
        }
        else{
            arg.array = sembuf;
            semctl(semid, 0, SETALL, arg);
        }
        
        shmkey = ftok("shmkey", 3);
        shmid = shmget(shmkey, sizeof(int), 0600|IPC_CREAT);
        shmbuf = (int *)shmat(shmid, 0 ,0);
        
        for(i=0;i<5;i++){
            scanf("%d", (shmbuf+0));
            
            //0 signal
            p_buf.sem_num = 0;
            p_buf.sem_op = 1;
            p_buf.sem_flg = 0;
            semop(semid, &p_buf, 1);
            
            //1 wait
            p_buf.sem_num = 1;
            p_buf.sem_op = -1;
            p_buf.sem_flg = 0;
            semop(semid, &p_buf, 1);
            
            printf("[input + 8] ==> %d\n", *(shmbuf+0));
        }
    
        exit(0);
    }
    ```

## 20181126

- p15-1.c
  Parent process는 표준 입력으로 정수를 하나 입력 받아, “data" 파일에 쓴 후, 세 개의 child process들을 만듭니다. 각 child process는 "data" 파일의 정수 값을 읽고 5초간 기다렸다 +10 한 값을 씁니다. 세 child의 덧셈이 정확히 되도록 file locking을 써서 동기화를 합니다.

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  #include <sys/ipc.h>
  #include <sys/msg.h>
  #include <sys/sem.h>
  #include <sys/shm.h>
  
  #define BUFSIZE 512
  
  void do_child(int fd){
      int in;
      struct flock lock;
      
      lock.l_whence = SEEK_SET;
      lock.l_start = 0;
      lock.l_len = 4;
      lock.l_type = F_WRLCK;
      fcntl(fd, F_SETLKW, &lock);
      
      lseek(fd, 0, SEEK_SET);
      read(fd, &in, sizeof(int));
      sleep(2);
      
      in = in + 10;
      lseek(fd, 0, SEEK_SET);
      write(fd, &in, sizeof(int));
      
      lock.l_type = F_UNLCK;
      fcntl(fd, F_SETLK, &lock);
      
      exit(0);
  }
  
  int main(int argc, char **argv){
      int fd, i, num;
      struct flock lock;
      pid_t pid[3];
      
      fd=open("data", O_RDWR|O_CREAT, 0600);
      scanf("%d", &num);
      write(fd, &num, sizeof(int));
  
      for(i=0;i<3;i++){
          pid[i] = fork();
          if(pid[i] == 0){
              do_child(fd);
          }
          else{
              wait(0);
          }
      }
      
      lseek(fd, 0, SEEK_SET);
      read(fd, &num, sizeof(int));
      printf("%d\n", num);
      
      return 0;
  }
  ```

- p15-2.c
  네 개의 프로세스가 동기화를 하며 자신의 프로세스 id를 5회 출력하는 프로그램을 작성 합니다. 이 프로그램은 "turn1" file을 이용하여 동기화에 참여하는 전체 프로세스 중 자신의 출력 순서를 결정 합니 다. 둘 이상의 프로세스가 동시에 자신의 id를 출력하지 않도록 하는 동기화 작업은 file locking을 사용 합니다. 네 프로세스의 출력 순서가 정해져 있지 않습니다. 동시에 출력을 하지 않도록만 하면 됩니다.

  - 되긴 잘되는데, 아직 완벽한건 아닌 것 같음

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <sys/wait.h>
  #include <fcntl.h>
  #include <unistd.h>
  #include <dirent.h>
  #include <string.h>
  #include <time.h>
  #include <ftw.h>
  #include <stdlib.h>
  #include <sys/mman.h>
  #include <sys/ipc.h>
  #include <sys/msg.h>
  #include <sys/sem.h>
  #include <sys/shm.h>
  
  #define BUFSIZE 512
  
  int main(int argc, char **argv){
      int fd, i, in = 5;
      struct flock lock;
      
      fd=open("turn1", O_RDWR|O_CREAT, 0600);
      
      lock.l_whence = SEEK_SET;
      lock.l_start = 0;
      lock.l_len = 4;
      lock.l_type = F_WRLCK;
      fcntl(fd, F_SETLKW, &lock);
      
      lseek(fd, 0, SEEK_SET);
      read(fd, &in, sizeof(int));
      
      for(i=0;i<5;i++){
          sleep(1);
          printf("pid = %d\n", getpid());
      }
      
      lock.l_type = F_UNLCK;
      fcntl(fd, F_SETLK, &lock);
      
      return 0;
  }
  ```

## asdf

