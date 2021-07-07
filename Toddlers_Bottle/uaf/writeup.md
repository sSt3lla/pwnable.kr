Initallialy when I ssh I get two files, the raw binary, and the .cpp source code (this means we don't need to reverse)

uaf@pwnable:~$ ls
flag  uaf  uaf.cpp

We download the files with the command
scp -P2222 uaf@pwnable.kr:uaf.cpp ./
scp -P2222 uaf@pwnable.kr:uaf ./

I have basically never done any kind of c++ class programming, so looking at the source code was a new experience.
However most programming languages have alot in common with each other, and the progam is quite small, so it should be easy to learn.

Before we start looking for the bug, lets go over the theory of a UAF (Use After Free)

As it names implied, a UAF happens when you use memory after you have freeded it.
the simpliest example possible

#include <stdio.h>
#include <stdlib.h>

int main(){
    //We get 16 bytes
    char *ptr = malloc(0x10);

    //We give back the memory
    free(ptr); 

    //Uh oh, we are now accessing memory we shouldn't
    important_func(ptr)
    
    return 0; 
}

In normal operations, after you free some memory, that is it, the pointer do it never gets used again.
But either due to some programming error or bad design, a UAF happens when a pointer gets passed to free, then gets used again, which will result in undefined behavior.

Now let use look into the program to find a bug

#include <fcntl.h>
#include <iostream> 
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};

class Man: public Human{
public:
	Man(string name, int age){
		this->name = name;
		this->age = age;
        }
        virtual void introduce(){
		Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
	Human* m = new Man("Jack", 25);
	Human* w = new Woman("Jill", 21);

	size_t len;
	char* data;
	unsigned int op;
	while(1){
		cout << "1. use\n2. after\n3. free\n";
		cin >> op;

		switch(op){
			case 1:
				m->introduce();
				w->introduce();
				break;
			case 2:
				len = atoi(argv[1]);
				data = new char[len];
				read(open(argv[2], O_RDONLY), data, len);
				cout << "your data is allocated" << endl;
				break;
			case 3:
				delete m;
				delete w;
				break;
			default:
				break;
		}
	}

	return 0;	
}

Theres alot of stuff going on.
Looking the top, there is a bunch of class stuff, which we will get into later, but purely logic wise,
we can see that its a simple menu program with 3 options

❯ ./uaf
1. use
2. after
3. free

From the start
Create 2 classes

Get input:
Option number 1, it calls the introduce functions on the classes
Options number 2, it creates list and reads some input into it
Option number 3, it delete's (free's) the two classes

Now in the case of the bug, when we delete (free) the objects, we can still interact with them.
This is via calling the functions from the classes

It is kind of funny that to use a UAF, you do it in reverse order 

Free
After   (Do something with the freeded data)
Use     (Now let the program use the data in an incorrect way)

Now that we have the bug secured, lets first look at getting a crash (hopefully by using our bug)
Lets first try to free, then use 

❯ ./uaf
1. use
2. after
3. free
3
1. use
2. after
3. free
1
[1]    7708 segmentation fault  ./uaf

Beautiful, we have achivied segmentation fault
Although we can't say that this is useful, as any programmer could make this from a mistake, lets try and 
instead get a segmentation fault on a controlled address

Now we need to get a deeper understanding