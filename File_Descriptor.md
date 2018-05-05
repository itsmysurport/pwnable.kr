파일 기술자(File Descriptor) 문제 풀이
=====================================
## 1. 파일 디스크립터란?
> 리눅스 혹은 유닉스의 계열의 시스템에서 프로세스(Process)가 파일(File)을 다룰 때 사용하는 개념으로 "파일 기술자" 혹은 파일 디스크립터가 있다.  
즉, **파일 디스크립터는 리눅스 / 유닉스에서 프로세스가 특정 파일에 접근할 때 사용하는 추상적인 값이다.**

---
## 2. 파일 디스크립터
기본적으로, 파일 디스크립터의 0, 1, 2번은 예약되어 있다.

| 0  | stdin(표준 입력 : 키보드)  |
|:-:|:-:|
| 1  | stdout(표준 출력 : 모니터)  |
| 2  | stderr(표준 오류 : 모니터)  |

---
## 3. pwnable.kr 'fd' 문제
<br>
fd 문제의 내용은 이렇다.  
>  Mommy! what is a file descriptor in Linux?

> try to play the wargame your self but if you are ABSOLUTE beginner, follow this tutorial link:
https://youtu.be/971eZhMHQQw

>ssh fd@pwnable.kr -p2222 (pw:guest)

먼저, 터미널로 ssh(**S** ecure **SH** ell) 로 fd@pwnable.kr에 접속한다.

접속한 다음에는 그 안에 무슨 파일이 있는지를 먼저 살펴보기 위해서 **ls 명령어** 를 사용해서 무슨 파일이 있는지를 먼저 확인하자.
```
fd@ubuntu:~$ ls
fd  fd.c  flag
```
안에는 **fd.c 파일**과 fd.c를 **프로그램으로 만든 fd 파일**, 우리가 정답을 맞추기 위해 넣어줘야 하는 flag 값이 **flag 파일** 에 들어있다.

---
### **3-1 fd.c 파일 분석**
먼저 fd.c의 파일을 분석을 해야 fd의 프로그램이 어떻게 실행되는지 알 수 있고, 어떻게 프로그램을 실행해야 flag 값을 출력하는지 알 수 있다.

**fd.c 파일 분석 방법은 다음과 같다.**

#### <a> fd@ubuntu:~$ cat fd.c

> cat 명령어는 텍스트로 된 파일일 경우 그 내용을 정상적으로 출력하지만 바이너리 파일일 경우에는 출력은 하지만 알아볼 수 없다. 그리고 2개 이상의 파일이름이 지정되면 모든 파일이 연결되어 보여진다.

fd.c 파일을 cat 명령어로 열어보자.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}

	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}
```

---
### **3-2 fd.c 파일 상세 분석 **

이 코드에서 가장 중요한 것은 **system("/bin/cat flag");**
이 있는 if문 안이다.<br><br>
if문 안의 조건식에서 strcmp(배열 비교) 함수로 LETMEWIN과 buf 배열을 비교해 서로 같아야지 참이 되어 if문이 실행된다.

#### **즉, buf 안의 내용이 LETMEWIN이 되어야 한다.**
그렇다면, buf 안의 내용을 직접 입력해야 하는데, 직접 입력하기 위해서는 read의 함수를 이용 해야 한다. 그렇다면 read 함수란 무엇인가?

### > ** read() 함수의 원형 **
```c
ssize-t read(int fd, void* buf, size_t nbytes);
```
해석을 해보면 **fd(File Descriptor)가 바라보고 있는 파일에서 nbytes 길이 만큼 buf에 읽어(read) 들인다** 는 뜻이다.

우리는 키보드로 직접 LETMEWIN을 읽어들이기 위해서, 파일 디스크립터에 미리 예약된 **0번(stdin : 표준 출력 함수)** 을 사용해야 한다.
그럼 다시 코드로 돌아가 보자, **len = read(fd, buf, 32);** 의 뜻은 fd가 바라보고 있는 파일에서 32바이트 만큼 buf에 읽어서 len에 넣는것이다. 그리고 우리는 fd에 0을 넣어야한다.

더 위의 코드로 올라가보면
```c
int fd = atoi( argv[1] ) - 0x1234;
```
라는 코드가 있다.  여기서 atoi 함수는 ()안의 값을 **문자열에서 정수로 바꿔 주는 역할** 을 한다.<br><br>
그렇다면 우리는 *argv[1]* 즉, 프로그램을 실행시킬 때 인자값을 0x1234를 10진수로 입력하면 atoi 함수가 우리가 입력한 문자열을 정수로 바꿔 줄 것이고, fd 값은 0이 되면서 read 함수에서 성공적으로 입력을 buf에 문자열을 입력할 수 있게 된다.

---
### ** 3-3 fd 프로그램 실행 **
리눅스에서 파일을 실행하는 명령어는 **./파일.확장자** 이다. 우리는 또한 전달인자를 0x1234에 10진수인 4660을 값으로 줘야하므로 다음과 같이 입력한다.
```
fd@ubuntu:~$ ./fd 4660
```
이렇게 입력한 후 엔터를 눌러 본다면 커서가 깜빡이는 것을 볼 수 있다.<br>
프로그램 실행 중 read 함수에서 fd가 가르키는 것이 0(stdin) 이어서 입력 대기 중이라는 뜻이다.<br>
그럼 LETMEWIN을 입력 해보도록 하자.

#### 결과는 다음과 같다.

```
fd@ubuntu:~$ ./fd 4660
LETMEWIN
good job :)
mommy! I think I know what a file descriptor is!!
```

flag의 값이 나왔다. flag는
### **mommy! I think I know what a file descriptor is!!**
였다.
