---
title: "rdma-core | ibv_get_device_list"
date: 2024-06-19T08:11:52+08:00
slug: 2024-06-19-ibv_get_device_list
type: posts
draft: false
categories:
  - default
tags:
  - RDMA
---

## Related resources
- The implementation of ibv_get_device_list is available in the [rdma-core repository](https://github.com/linux-rdma/rdma-core/blob/4b429243850b000a3650eff6707676ce9aa45941/libibverbs/device.c#L54).
- You can find the specification in the man ibv_get_device_list manual or on [RDMAmojo](https://www.rdmamojo.com/2012/05/31/ibv_get_device_list/).
## API definition

```c
#   define LATEST_SYMVER_FUNC(_public_sym, _uniq, _ver_str, _ret, ...)         \
	_MAKE_SYMVER_FUNC(_public_sym, _uniq, "@" _ver_str, _ret, __VA_ARGS__)

LATEST_SYMVER_FUNC(ibv_get_device_list, 1_1, "IBVERBS_1.1",
		   struct ibv_device **,
		   int *num)
```

First, the implementation declares it is of version “IBVERBS_1.1”. The function accepts a pointer to an integer, num, as its only argument. It will be set to the number of devices found and can be NULL if you want to ignore the device count. The function returns a pointer to an array of pointers to ibv_device structures.

## Thread-safe

```c
static pthread_mutex_t dev_list_lock = PTHREAD_MUTEX_INITIALIZER;

LATEST_SYMVER_FUNC(ibv_get_device_list, 1_1, "IBVERBS_1.1",
		   struct ibv_device **,
		   int *num)
{
  struct ibv_device **l = NULL;
  // ...
  static bool initialized;

  pthread_mutex_lock(&dev_list_lock);
  if (!initialized) {
		if (ibverbs_init())
			goto out;
		initialized = true;
	}

  // ...
out:
	pthread_mutex_unlock(&dev_list_lock);
	return l;
}
```

It uses a mutex lock to prevent resource contention during the device list retrieval process. Note that `dev_list_lock` is a global variable and `initialized` is a static boolean.
## Workhorse
```c
static struct list_head device_list = LIST_HEAD_INIT(device_list);

LATEST_SYMVER_FUNC(ibv_get_device_list, 1_1, "IBVERBS_1.1",
		   struct ibv_device **,
		   int *num)
{
  // ... 
  
  num_devices = ibverbs_get_device_list(&device_list);
  if (num_devices < 0) {
      errno = -num_devices;
      goto out;
  }

  // ...
}
```

`ibverbs_get_device_list` is the workhorse. It finishes the following jobs:
1. Fetches the list of IB devices and uverbs through netlink or by reading the file system directly, and links the fetched devices with `sysfs_list`.
2. Removes entries from the `sysfs_list` that are already present in the `device_list` (global), and removes entries from the `device_list` that are not present in the `sysfs_list`.
3. Matches every entry in the `sysfs_list` to a driver and adds the new matched entries to `device_list`. Once matched to a driver, the entry in `sysfs_list` is removed.

After this, we get the currently available `device_list`.

![fig1](/images/ibv_get_device_list_fig1.png)

## Garbage collection
```c
LATEST_SYMVER_FUNC(ibv_get_device_list, 1_1, "IBVERBS_1.1",
		   struct ibv_device **,
		   int *num)
{
  // ... 

  list_for_each(&device_list, device, entry) {
      l[i] = &device->device;
      ibverbs_device_hold(l[i]);
      i++;
  }
  
  // ...
}
```

`ibverbs_device_hold` atomically fetches and increments the device's `refcount` by 1. Note that the `refcount` will be used when removing the entry in `sysfs_list`.