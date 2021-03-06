## atexit
只有在进程正常退出时，其定义的退出函数才会执行，exit()退出或者main的return退出，如果是因为收到信号而退出，则不会被调用

### atexit 函数定义
```
// glibc-2.26/stdlib/atexit.c 文件
/* Register FUNC to be executed by `exit'.  */
int
#ifndef atexit
attribute_hidden
#endif
atexit (void (*func) (void))
{
  return __cxa_atexit ((void (*) (void *)) func, NULL,
		       &__dso_handle == NULL ? NULL : __dso_handle);
}

// glibc-2.26/stdlib/cxa_atexit.c 文件
/* Register a function to be called by exit or when a shared library
   is unloaded.  This function is only called from code generated by
   the C++ compiler.  */
int
__cxa_atexit (void (*func) (void *), void *arg, void *d)
{
  return __internal_atexit (func, arg, d, &__exit_funcs);
}

// glibc-2.26/stdlib/cxa_atexit.c 文件
int
attribute_hidden
__internal_atexit (void (*func) (void *), void *arg, void *d,
		   struct exit_function_list **listp)
{
  struct exit_function *new = __new_exitfn (listp);  // 得到一个新节点来初始化

  if (new == NULL)
    return -1;

#ifdef PTR_MANGLE
  PTR_MANGLE (func);
#endif
  new->func.cxa.fn = (void (*) (void *, int)) func;
  new->func.cxa.arg = arg;
  new->func.cxa.dso_handle = d;
  atomic_write_barrier ();
  new->flavor = ef_cxa;
  return 0;
}

上面可见atexit函数是把定义的退出函数注册到 __exit_funcs 内部
__exit_funcs 的定义如下:

//这里可以看 __exit_funcs 的定义 : 
struct exit_function_list
{
    struct exit_function_list *next;
    size_t idx;
    struct exit_function fns[32];
};
static struct exit_function_list initial;
struct exit_function_list *__exit_funcs = &initial;
```

### atexit注册的函数调用时机
```
在exit() 函数内部
// glibc-2.26/stdlib/exit.c 文件
void
exit (int status)
{
  __run_exit_handlers (status, &__exit_funcs, true, true);
}

__run_exit_handlers 函数内部会遍历__exit_funcs，逐一调用注册的退出函数
```

### 为什么进程收到信号退出不会执行atexit注册的函数
```
进程由接收到信号退出是由内核内部完成的，而atexit注册的函数是在用户程序中注册的，在C库中被调用的
所以信号退出进程不会执行注册退出的函数
```

### atexit运用
```
#include<stdio.h>
#include<stdlib.h>

static void callback1(void)
{
    printf("callback1\n");
}

static void callback2(void)
{
    printf("callback2\n");
}

static void callback3(void)
{
    printf("callback3\n");
}

int main(void)
{
    atexit(callback1);
    atexit(callback2);
    atexit(callback3);
    printf("main exit\n");
    return 0;
}

输出:
main exit
callback3
callback2
callback1
```

### atexit的局限
```
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

static void callback1(void)
{
    printf("callback1\n");
}

int main(void)
{
    atexit(callback1);
    while(1)
    {
	sleep(1);
    }
    printf("main exit\n");
    return 0;
}

测试:
killall atexit1
输出:
已终止
```
