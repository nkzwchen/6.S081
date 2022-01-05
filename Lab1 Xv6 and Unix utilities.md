# Lab: Xv6 and Unix utilities

## sleep

具体程序如下:

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
int main(int argc,char* argv[]){
	if (argc < 2){
		fprintf(2, "no input argument ^.^\n");
		exit(1);
	}	
	sleep(atoi(argv[1]));
	exit(0);
}

```

##  pingpong

具体程序如下:

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
int main(int argc,char* argv[]){
	int p[2];
        int q[2];
        char c = 'z';
        pipe(p);
        pipe(q);
        if (fork() == 0){
        		read(p[0], &c, sizeof(char));
        		fprintf(2,"%d: received ping\n",getpid());
        		write(q[1], &c, sizeof(char));
        		
	}
        else{
        		write(p[1], &c, sizeof(char));
        		read(q[0], &c, sizeof(char));
        		fprintf(2,"%d: received pong\n",getpid());		
        		
        }
        	
        exit(0);
}

```
## primes

具体程序如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
int main(int argc,char* argv[]){
        int p[36];
        int prime;
        int number;
        int count = 0;
        pipe(p + count);
        if(fork() == 0){
		while(1){
			count += 2;
			if (count > 2)
				close(p[count - 4]);
			close(p[count - 1]);
			if (!read(p[count - 2] , &prime , sizeof(int))){
				close(p[count - 2]);
				exit(0);
			}
			fprintf(2,"prime %d\n" , prime);
			pipe(p + count);
			if (fork() == 0)
				continue;
			close(p[count]);
			while (read(p[count -2] , &number , sizeof(int)))
				if(number % prime != 0)
					write(p[count + 1], &number, sizeof(int));
			close(p[count + 1]); 
			close(p[count - 2]);
			wait((int*) 0);
			exit(0);    	
		}
	}
        prime = 2;
        fprintf(2,"prime %d\n" , prime);
        close(p[count]);
        for (int i = 3; i < 36 ; i++)
        	if (i % prime != 0)
        		write(p[count + 1], &i, sizeof(int)); 
        close(p[count + 1]);
        wait((int*) 0);
        exit(0);
}

```

## xargs

具体程序如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"
#define MAXSIZE 16
#define MAXSTRINGSIZE 64
int get_parameter(char* buf, char**  argv){
	int parameter_start[MAXSIZE];
	int parameter_end[MAXSIZE];
	int parameter_number = 0;
	int Case = 0;
	int pointer = -1;
	char* p = buf;
	while(*p != '\0' && *p != '\n'){
	        pointer++;
	       if (*p != ' ' ){
			if (Case == 0){
				parameter_start[parameter_number] = pointer;
				parameter_number++;
				Case = 1;
			}
	        }
		else if (Case == 1){
			parameter_end[parameter_number - 1] = pointer;
			Case = 0;
		}
		p++;
	}
	if (Case == 1)
		parameter_end[parameter_number - 1] = pointer + 1;
	for (int i = 0; i < parameter_number; i++){

		int length = parameter_end[i] - parameter_start[i];
		argv[i] = (char*)malloc ((length + 1) * sizeof(char));
		memcpy(argv[i], buf + parameter_start[i], length * sizeof(char) );
		argv[i][length] = '\0'; 
 	}
 	argv[parameter_number + 1] = 0;
 	return parameter_number;
} 
int main(int argc,char* argv[]){
	char buf[MAXSTRINGSIZE + 1];
	while(1){
		memset(buf, '\0', MAXSTRINGSIZE + 1);
		gets(buf,MAXSTRINGSIZE);
		if(strlen(buf) == 1 || buf[0] == '\0' || buf[0] == '\n')
			break;
		if (fork()==0){
			char* argv1[MAXSIZE + 2];
			for(int i = 1; i < argc ; i++){
				argv1[i - 1] = (char*)malloc(strlen(argv[i])*sizeof(char));
				strcpy(argv1[i - 1],argv[i]);
			}
			get_parameter(buf, argv1 + argc - 1);
			exec(argv[1], argv1);
			exit(getpid());
		}
			wait((int*)0);
	}
	wait((int*)0);
        exit(0);
}



```
