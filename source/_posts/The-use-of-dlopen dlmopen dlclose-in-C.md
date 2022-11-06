---
title: 加载动态链接库——dlopen dlsym dlclose
date: 2018-09-29
tags: [C,函数详解]
toc: true
---
## DLOPEN&nbsp;DLMOPEN&nbsp;DLCLOSE

### NAME
&nbsp;&nbsp;&nbsp;&nbsp;**dlclose, dlopen, dlmopen** - 打开／关闭共享对象
<!--more-->

### SYNOPSIS
```C
#include <dlfcn.h>

void *dlopen(const char *filename, int flags);

int dlclose(void *handle);

#define _GNU_SOURCE
#include <dlfcn.h>

void *dlmopen (Lmid_t lmid, const char *filename, int flags);
```


### DESCRIPTION


#### dlopen()
&nbsp;&nbsp;&nbsp;&nbsp;这个函数加载由以null结尾的字符串文件名命名的动态共享对象（共享库）文件，并为加载的对象返回不透明的“句柄”。此句柄与 dlopen API 中的其他函数一起使用，例如`dlsym()`，`dladdr()`，`dlinfo()`和`dlclose()`。

如果 <u>filename</u> 为 NULL，则返回的句柄用于主程序。如果 <u>filename</u> 包含斜杠（“/”），则它被解释为（相对或绝对）路径名。否则，动态链接器将按如下方式搜索对象（有关详细信息，请参阅`ld.so(8)`）：
- （仅限ELF）如果调用程序的可执行文件包含 DT_RPATH 标记，并且不包含 DT_RUNPATH 标记，则会搜索 DT_RPATH 标记中列出的目录。
- 如果在程序启动时，环境变量 LD_LIBRARY_PATH 被定义为包含以冒号分隔的目录列表，则会搜索这些目录。 （作为安全措施，set-user-ID 和 set-group-ID程序将忽略此变量。）
- （仅限ELF）如果调用程序的可执行文件包含　DT_RUNPATH　标记，则搜索该标记中列出的目录。
- 检查缓存文件/etc/ld.so.cache（由ldconfig（8）维护）以查看它是否包含filename的条目。
- 搜索目录 /lib和 /usr/lib（按此顺序）。

&nbsp;&nbsp;&nbsp;&nbsp;如果 <u>filename</u> 指定的对象依赖于其他共享对象，则动态链接器也会使用相同的规则自动加载这些对象。 （如果这些对象依次具有依赖性，则此过程可以递归地发生）

<u>flags</u> 参数必须包括以下两个值中的一个：
- RTLD_LAZY
执行延迟绑定。仅在执行引用它们的代码时解析符号。如果从未引用该符号，则永远不会解析它（只对函数引用执行延迟绑定;在加载共享对象时，对变量的引用总是立即绑定）。自 glibc 2.1.1，此标志被**LD_BIND_NOW**环境变量的效果覆盖。
- RTLD_NOW
如果指定了此值，或者环境变量**LD_BIND_NOW**设置为非空字符串，则在`dlopen()`返回之前，将解析共享对象中的所有未定义符号。如果无法执行此操作，则会返回错误。

<u>flags</u> 也可以通过以下零或多个值进行或运算设置：
- RTLD_GLOBAL
此共享对象定义的符号将可用于后续加载的共享对象的符号解析。
- RTLD_LOCAL
这与**RTLD_GLOBAL**相反，如果未指定任何标志，则为默认值。此共享对象中定义的符号不可用于解析后续加载的共享对象中的引用。
- RTLD_NODELETE (since glibc 2.2)
在`dlclose()`期间不要卸载共享对象。因此，如果稍后使用`dlopen()`重新加载对象，则不会重新初始化对象的静态变量。
- RTLD_NOLOAD (since glibc 2.2)
不要加载共享对象。这可用于测试对象是否已经驻留（如果不是，则`dlopen()`返回 NULL，如果是驻留则返回对象的句柄）。此标志还可用于提升已加载的共享对象上的标志。例如，以前使用**RTLD_LOCAL**加载的共享对象可以使用**RTLD_NOLOAD | RTLD_GLOBAL**重新打开。
- RTLD_DEEPBIND (since glibc 2.3.4)
将符号的查找范围放在此共享对象的全局范围之前。这意味着自包含对象将优先使用自己的符号，而不是全局符号，这些符号包含在已加载的对象中。


#### dlmopen()
&nbsp;&nbsp;&nbsp;&nbsp;这个函数除了以下几点与`dlopen()`有所不同外，都执行同样的任务。
&nbsp;&nbsp;&nbsp;&nbsp;`dlmopen()`与`dlopen()`的主要不同之处主要在于它接受另一个参数 <u>lmid</u>，它指定应该被加载的共享对象的链接映射列表（也称为命名空间）。对于命名空间，<u>Lmid_t</u> 是个不透明的句柄。
<u>lmid</u> 参数要么是已经存在的命名空间的ID（这个命名空间可以通过`dlinfo RTLD_DI_LMID`请求获得）或者是以下几个特殊值中的其中一个：
- LM_ID_BASE
在初始命名空间中加载共享对象（即应用程序的命名空间）。
- LM_ID_NEWLM
创建新的命名空间并在该命名空间中加载共享对象。该对象必须已正确链接到引用               所有其他需要的共享对象，因为新的命名空间最初为空。

如果 <u>filename</u> 是 NULL，那么 <u>lmid</u> 的值只能是**LM_ID_BASE**。


#### dlclose()
&nbsp;&nbsp;&nbsp;&nbsp;`dlclose()`减少指定句柄 <u>handle</u> 引用的动态加载共享对象的引用计数。如果引用计数减少为０，那么这个动态加载共享对象将被真正卸载。所有在`dlopen()`被调用的时候自动加载的共享对象将以相同的方式递归关闭。

&nbsp;&nbsp;&nbsp;&nbsp;`dlclose()`成功返回并不保证与句柄相关的符号将从调用方的地址空间中删除。除了显式通过`dlopen()`调用产生的引用之外，一些共享对象作为依赖项可能已被隐式加载（和引用计数）。只有当所有引用都已被释放才可以从地址空间中删除共享对象。

### RETURN VALUE
&nbsp;&nbsp;&nbsp;&nbsp;执行成功时，`dlopen()`和`dlmopen()`返回一个非空句柄。
&nbsp;&nbsp;&nbsp;&nbsp;执行失败时（文件找不到、不可读、错误的格式或者在加载的时候出现错误），`dlopen()`和`dlmopen()`返回 NULL。
&nbsp;&nbsp;&nbsp;&nbsp;对于`dlclose()`成功执行，将返回０值，失败时，返回一个非０值。

以上这些函数产生的错误，其错误信息都可以通过`dlerror()`获知。


### NOTES


#### dlmopen() 与 命名空间
&nbsp;&nbsp;&nbsp;&nbsp;链接映射列表定义了通过动态链接器解析的符号的孤立命名空间。在命名空间内，被依赖的共享对象根据通常的规则被隐式加载，符号引用同样以通常的规则被解析。但是这种方案受限于已经被（显式和隐式）加载进命名空间的对象的定义。

&nbsp;&nbsp;&nbsp;&nbsp;`dlmopen()`函数允许对象隔离加载——在新的命名空间中加载共享对象而不暴露其余的应用于新对象提供的符号。注意使用**RTLD_LOCAL**标志不足以达到此目的，因为它防止一个共享对象的符号对**任何其他**共享对象可用。在某些情况下，我们可能想使得由一些动态加载共享对象提供的符号对于其他共享对象可用，而不将这些符号暴露给整个应用。这可以通过使用单独的命名空间和**RTLD_GLOBAL**标志来实现。

&nbsp;&nbsp;&nbsp;&nbsp;`dlmopen()`函数可以提供比**RTLD_LOCAL**标志更好的隔离效果。特别是，当共享对象是通过**RTLD_LOCAL**标志加载的，并且其依赖的共享对象是通过**RTLD_GLOBAL**加载的，那么有可能升级为**RTLD_GLOBAL**。因此，明确控制了所有共享对象的依赖的这种情况外，**RTLD_LOCAL**是不足以隔离加载的共享对象，。

&nbsp;&nbsp;&nbsp;&nbsp;`dlmopen()`函数的一种用法是多次加载同样的对象。不使用`dlmopen()`函数来实现这个功能的话，需要创建共享对象的一个副本。而如果使用`dlmopen()`函数来实现的话，可以通过将相同的共享对象文件加载到不同的命名空间来实现。
       glibc实现最多支持16个命名空间。

#### 初始化和终结功能
&nbsp;&nbsp;&nbsp;&nbsp;共享对象可以使用**__attribute __((constructor))**和**__attribute __((destructor))**函数属性。构造函数在`dlopen()`返回之前执行，而析构函数在`dlclose()`返回之前执行。共享对象可以导出多个构造函数和析构函数并且优先顺序可以和每个函数相关联来决定它们的执行顺序。

## DLSYM


### NAME
&nbsp;&nbsp;&nbsp;&nbsp;**dlsym, dlvsym** - 获取共享对象或可执行文件中符号的地址


### SYNOPSIS
```C
#include <dlfcn.h>

void *dlsym(void *handle, const char *symbol);

#define _GNU_SOURCE
#include <dlfcn.h>

void *dlvsym(void *handle, char *symbol, char *version);
```


### DESCRIPTION
&nbsp;&nbsp;&nbsp;&nbsp;`dlsym()`接受由`dlopen()`返回的动态加载的共享对象的“句柄”，并返回该符号加载到内存中的地址。如果未找到符号，则在加载该对象时，在指定对象或`dlopen()`自动加载的任何共享对象中，`dlsym()`将返回NULL。（`dlsym()`通过这些共享对象的依赖关系树进行宽度优先搜索。）
&nbsp;&nbsp;&nbsp;&nbsp;因为符号本身可能是 NULL（所以`dlsym()`返回 NULL 并不意味着错误），因此判断是否错误的正确做法是调用`dlerror()`清除任何旧的错误条件，然后调用`dlsym()`，并且再次调用`dlerror()`，保存其返回值，判断这个保存的值是否是 NULL。
&nbsp;&nbsp;&nbsp;&nbsp;可以在句柄中指定两个特殊的伪句柄：
- RTLD_DEFAULT
使用默认共享对象搜索顺序查找所需符号的第一个匹配项。搜索将包括可执行文件中的全局符号及其依赖项，以及使用**RTLD_GLOBAL** 标志动态加载的共享对象中的符号。
- RTLD_NEXT
在当前对象之后的搜索顺序中查找下一个所需符号。这允许人们在另一个共享对象中提供一个函数的包装器，因此，例如，预加载的共享对象中的函数定义（参见ld.so（8）中的**LD_PRELOAD**）可以找到并调用在另一个共享对象中提供的“真实”函数（或者就此而言，在存在多个预加载层的情况下，函数的“下一个”定义）。

&nbsp;&nbsp;&nbsp;&nbsp;`dlvsym()`除了比`dlsym()`多提供了一个额外的参数外，其余与`dlsym()`相同。


### RETURN VALUE
&nbsp;&nbsp;&nbsp;&nbsp;执行成功，这些函数将会返回　<u>symbol</u> 关联的地址。执行失败，它们将返回 NULL。错误的原因可以通过`dlerror()`进行诊断。


## EXAMPLE


```C
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <gnu/lib-names.h>  /* Defines LIBM_SO (which will be a string such as "libm.so.6") */
int
main(void)
{
  void *handle;
  double (*cosine)(double);
  char *error;

  handle = dlopen(LIBM_SO, RTLD_LAZY);
  if (!handle) {
     fprintf(stderr, "%s\n", dlerror());
     exit(EXIT_FAILURE);
  }

  dlerror();    /* Clear any existing error */

  cosine = (double (*)(double)) dlsym(handle, "cos");

  /* According to the ISO C standard, casting between function
    pointers and 'void *', as done above, produces undefined results.
    POSIX.1-2003 and POSIX.1-2008 accepted this state of affairs and
    proposed the following workaround:

        *(void **) (&cosine) = dlsym(handle, "cos");

    This (clumsy) cast conforms with the ISO C standard and will
    avoid any compiler warnings.

    The 2013 Technical Corrigendum to POSIX.1-2008 (a.k.a.
    POSIX.1-2013) improved matters by requiring that conforming
    implementations support casting 'void *' to a function pointer.
    Nevertheless, some compilers (e.g., gcc with the '-pedantic'
    option) may complain about the cast used in this program. */

  error = dlerror();
  if (error != NULL) {
     fprintf(stderr, "%s\n", error);
     exit(EXIT_FAILURE);
  }

  printf("%f\n", (*cosine)(2.0));
  dlclose(handle);
  exit(EXIT_SUCCESS);
}
```