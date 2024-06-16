---
title: "`container_of` macro from the Linux"
date: 2024-06-13T00:11:11+08:00
slug: 2024-06-13-container_of
type: posts
draft: false
categories:
  - default
tags:
  - Linux
  - C
  - Macro
---

While working on my code, I came across the `container_of` macro, which I've seen before but never really delved into. Its implementation was mostly as I expected, but some details caught my interest.

```
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:    the pointer to the member.
 * @type:   the type of the container struct this is embedded in.
 * @member: the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({          \
	const typeof(((type *)0)->member)*__mptr = (ptr);    \
		     (type *)((char *)__mptr - offsetof(type, member)); })
```

For those who might be wondering why we don't use a simpler version like this:

```
#define container_of(ptr, type, member) ({          \
		     (type *)((char *)ptr - offsetof(type, member)); })
```

This simpler version seems like it would do the job, so why add extra complexity? To answer this, let's look at another version of `container_of` from the Linux:

```
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	static_assert(__same_type(*(ptr), ((type *)0)->member) ||	\
		      __same_type(*(ptr), void),			\
		      "pointer type mismatch in container_of()");	\
	((type *)(__mptr - offsetof(type, member))); })
```

The Linux implementation includes type checking. This ensures at compile time that the `ptr` you pass is a pointer of type `member`. Here's an example to illustrate this:

```
#include <stdio.h>
#include <assert.h>

#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

#define container_of(ptr, type, member) ({          \
	const typeof(((type *)0)->member)*__mptr = (ptr);    \
		     (type *)((char *)__mptr - offsetof(type, member)); })

struct container {
  int a;
  long b;
};

int main() {
  struct container cont;
  int *wrong_ptr = (int *)&(cont.b);

  struct container *ptr = container_of(wrong_ptr, struct container, b);
  assert(ptr == &cont);

  return 0;
}
```

Compiling this with GCC gives us the following warning:

```
$ gcc test.c
test.c:19:27: warning: incompatible pointer types initializing 'const typeof (((struct container *)0)->b) *' (aka 'const long *') with an expression of type 'int *' [-Wincompatible-pointer-types]
  struct container *ptr = container_of(wrong_ptr, struct container, b);
                          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test.c:7:36: note: expanded from macro 'container_of'
        const typeof(((type *)0)->member)*__mptr = (ptr);    \
                                          ^        ~~~~~
1 warning generated.
```

This warning helps prevent potential errors, such as a user passing an incorrect `member`. However, you may notice that the `static_assert` version implementation allows the check to pass if the `*ptr` type is `void`, even if it doesn't match the `member` type. Why is this?

According to ChatGPT, this design choice enhances generality and flexibility. Though I haven't encountered a situation requiring this feature, I won't delve deeper into it here.
