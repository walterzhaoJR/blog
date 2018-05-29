# 0.背景介绍
* 今天在工作中要使用pthread_create函数开启一个线程，并且需要传递参数，其中遇到了一些问题，现在来总结一下。
#1.pthread_create介绍
* 函数原型：
```linux
int pthread_create(pthread_t *tidp,const pthread_attr_t *attr, (void*)(*start_rtn)(void*),void *arg);
```
  * 第一个参数为指向线程标识符的指针（例如：pthread_t p_thread）
  * 第二个参数用来设置线程属性
  * 第三个参数是线程运行函数的起始地址
  * 第四个参数是运行函数的参数
* 在Linux系统中如果希望开启一个新的线程，可以使用pthread_create函数，它实际的功能是确定调用该线程函数的入口点，在线程创建以后，就开始运行相关的线程函数。
```linux
头文件 #include<pthread.h>
```
* pthread_create的返回值表征进程是否创建成功。其中0：成功，-1：失败。
* 编译的时候，需要添加编译条件 -pthread。例如：g++ pthread_test.cpp -o pthread_test -pthread

#2.pthread_create开启线程，传递参数的几种情况
##2.1不传递参数
```linux
#include <iostream>
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <string>

using namespace std;

pthread_t pid;

void* thread_no_argv(void* argv)
{ 	
	while(1)
	{
		cout << "i am new thread"<< endl;
		sleep(1);
	}
}

int main()
{
	pthread_create(&pid,NULL,thread_no_argv,NULL);
	pthread_detach(pid);//回收线程资源

	while(1)
	{
	}
	return 0;
}
```

##2.2 传递一个简单类型
```linux
#include <iostream>
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <string>

using namespace std;

pthread_t pid;

void* thread_int_argv(void* argv)
{ 	
	int i = *(int*)argv;//进行类型装换，具体的类型具体转换
	while(1)
	{
		cout << "i am new thread"<< endl;
		cout << "i:" << i << endl;
		sleep(1);
	}
}

int main()
{
	int i = 10;

	pthread_create(&pid,NULL,thread_int_argv,(void*)&i);//最后一个参数为void*类型，传递变量的地址，其实其他的类型也遵循这个原则
	pthread_detach(pid);

	while(1)
	{
	}
	return 0;
}
```
##2.3传递一个结构体对象
```linux
#include <iostream>
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <string>

using namespace std;

pthread_t pid;

typedef struct 
{
	int i;
	std::string str;
}my_adt;

void* thread_struct_argv(void* argv)
{ 	
	my_adt* adt = (my_adt*)argv;

	while(1)
	{
		cout << "i am new thread"<< endl;
		cout << "i:" << adt->i << endl;
		cout << "str:" << adt->str << endl;
		sleep(1);
	}
}

int main()
{
	my_adt adt;
	adt.i = 22;
	adt.str = "walter";

	pthread_create(&pid,NULL,thread_struct_argv,(void*)&adt);
	pthread_detach(pid);

	while(1)
	{
	}
	return 0;
}
```
##2.4传递一个类对象
```linux
#include <iostream>
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <string>
#include <iterator>
#include <vector>

using namespace std;

pthread_t pid;

class A
{
public:
	int get_i()
	{
		return m_i;
	}

	void set_i(int i)
	{
		m_i = i;
	}

	std::string get_str()
	{
		return m_str;
	}

	void set_str(std::string str)
	{
		m_str = str;
	}

private:
	int m_i;
	std::string m_str;
};


class B
{
public:
	A get_a()
	{
		return m_a;
	}

	void set_a(A a)
	{
		m_a = a;
	}

	std::vector<std::string > get_iplist()
	{
		return m_iplist;
	}

	void set_iplist(std::vector<std::string> iplist)
	{
		m_iplist = iplist;
	}

private:
	A m_a;
	std::vector<std::string> m_iplist;
};

void* thread_object_argv(void* argv)
{ 	
	B *b = (B*)argv;

	A a = b->get_a();
	std::vector<std::string> iplist = b->get_iplist();

	cout << "a.i:" << a.get_i() << endl;
	cout << "a.str:" << a.get_str() << endl;
	cout << "iplist.size:" << iplist.size();
	for(std::vector<std::string>::iterator iter = iplist.begin();iter != iplist.end();iter++)
	{
		cout << "iplist:" << std::string(*iter) << endl;
	}


	while(1)
	{
		sleep(1);
	}
}

int main()
{
	A a;
	a.set_i(99);
	a.set_str("zx");

	std::vector<std::string> iplist;
	iplist.push_back("1.1.1.1");
	iplist.push_back("2.2.2.2");

	B *b = new B();//注释1
	b->set_a(a);
	b->set_iplist(iplist);


	pthread_create(&pid,NULL,thread_object_argv,(void*)b);
	pthread_detach(pid);

	while(1)
	{
	}
	return 0;
}
```
* 注释1：这个地方最好使用new一个对象的方式，这样可以为复杂的数据结构分配内存，否则的话就可能会出现内存分配的异常：std::bad_alloc。

# 总结
* 上面介绍了几种常用的传递参数的情况，在实际的情况中灵活变通使用。
* 如果上面介绍的情况有什么问题，还请大家指出了，或者一起讨论，或者还有什么更好的方法，也希望和我交流。
