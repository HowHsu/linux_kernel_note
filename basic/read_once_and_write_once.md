read_once和write_once的作用是防止多线程情景下编译器的不适当优化。
原因是编译器优化是以单线程考虑的。如：
```c
	a = 1;
	do something non-related with a;
	a = 1;
```
在单线程程序中，第三行是没有必要的。因为a始终没有变，所以编译器会将该语句去除。
问题是多线程情况下这就有问题了。a在两个a = 1中间可能被其他线程访问，所以第三行是很必要的。
为了防止编译器优化第三行，WRITE_ONCE就出场了。
```c
	a = 1;
	do something non-related with a;
	WRITE_ONCE(a, 1);
```
改成这样就能保证第三行不被编译器优化。
那么为什么READ/WRITE_ONCE能够达到这种效果呢？
其实是它们是利用了编译器的一个装饰器volatile。
volatile意思是善变的，当定义变量时带上该标记，表示该变量经常被改变，不要信任寄存器中的暂存值，而要从内存读取。这能够阻止编译器对其进行一系列优化（包括不限于调整顺序、去除等）
来看READ_ONCE的代码：
```c
#define READ_ONCE(x)					\
({							\
	union { typeof(x) __val; char __c[1]; } __u =	\
		{ .__c = { 0 } };			\
	__read_once_size(&(x), __u.__c, sizeof(x));	\
	__u.__val;					\
})

static __always_inline void __read_once_size(const volatile void *p, void *res, int size)
{
	switch (size) {
	case 1: *(__u8_alias_t  *) res = *(volatile __u8_alias_t  *) p; break;
	case 2: *(__u16_alias_t *) res = *(volatile __u16_alias_t *) p; break;
	case 4: *(__u32_alias_t *) res = *(volatile __u32_alias_t *) p; break;
	case 8: *(__u64_alias_t *) res = *(volatile __u64_alias_t *) p; break;
	default:
		barrier();
		__builtin_memcpy((void *)res, (const void *)p, size);
		barrier();
	}
}
```
可以看到，READ_ONCE()定义了union变量--u， 其中--c用来方便寻址。然后在--read_once_size()中，
将需要copy的x的值copy到--u中。这里的关键是copy时将p强转为volatile类型，这保证了这些语句不被编译器优化。
特别的，当x的size大于64位时，将调用普通的memcpy进行拷贝。这个情况下没有volatile保护，所以需要在其前后加上barrier():
```c
#define barrier() __asm__ __volatile__("": : :"memory")
```
可见其就是插入了一句汇编，其中"memory"是告诉编译器本段汇编代码修改了内存，之后的代码不要信任本句之前的内存数据，这样编译器就不会把barrier()之前和之后的内存访问代码连起来看从而优化掉。如：
```c
int a = 5, b = 6;
barrier();
a = b;
```
假设int a = 5, b = 6;的时候b在ebx中。
第三行a = b不会编译成类似：
```asm
mov %ebx, (mem addr of a)
```
而是老老实实访问内存：
```asm
mov (mem addr of b), %ebx
mov %ebx, (mem addr of a)
```
因为barrier()中的"memory"告诉了编译器内存已经被改变了，请无效化寄存器和cache。

——volatile--意味着不要优化这段汇编，所以在语句的前后加上barrier()，那么中间的语句就不会被编译器优化了。
这里我的疑问是为什么要在内存拷贝的前后都加上barrier()，按理说只要在前面加上barrier()即可。
READ_ONCE()的最后一个语句是--u.--val; 也就是READ_ONCE()返回要读的value。

WRITE_ONCE()的语义和READ_ONCE()相反，代码是类似的，不再累述。


关于gcc内连汇编的资料见：http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html#s2
关于barrier()的更多资料参考：https://zhuanlan.zhihu.com/p/96001570
