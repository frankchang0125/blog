---
date: "2024-12-24T00:00:00+08:00"
title: "Linux Kernel: TEE"
author: "Frank Chang"
categories:
  - "Security"
tags:
  - "Linux Kernel"
  - "OP-TEE"
  - "TEE"
series:
  - "OP-TEE Code Trace"
---

> ⚠️ The code is based on: [https://gitlab.com/riseproject/riscv-optee/linux/-/tree/dev-optee-mpxy](https://gitlab.com/riseproject/riscv-optee/linux/-/tree/dev-optee-mpxy)
>
> Commit ID: **`df5dc01764820f113312f7a39f221b49985bbd7a`**

- Kernel provides a **TEE bus infrastructure** where a **Trusted Application** is represented as a **device** identified via Universally Unique Identifier (UUID) and **client drivers** register a table of supported device UUIDs.

---

```c
// include/linux/tee_core.h

/**
 * struct tee_device - TEE Device representation
 * @name:	name of device
 * @desc:	description of device
 * @id:		unique id of device
 * @flags:	represented by TEE_DEVICE_FLAG_REGISTERED above
 * @dev:	embedded basic device structure
 * @cdev:	embedded cdev
 * @num_users:	number of active users of this device
 * @c_no_user:	completion used when unregistering the device
 * @mutex:	mutex protecting @num_users and @idr
 * @idr:	register of user space shared memory objects allocated or
 *		registered on this device
 * @pool:	shared memory pool
 */
struct tee_device {
	char name[TEE_MAX_DEV_NAME_LEN];
	const struct tee_desc *desc;
	int id;
	unsigned int flags;

	struct device dev;
	struct cdev cdev;

	size_t num_users;
	struct completion c_no_users;
	struct mutex mutex;	/* protects num_users and idr */

	struct idr idr;
	struct tee_shm_pool *pool;
};
```

```c
// include/linux/tee_core.h

/**
 * struct tee_desc - Describes the TEE driver to the subsystem
 * @name:	name of driver
 * @ops:	driver operations vtable
 * @owner:	module providing the driver
 * @flags:	Extra properties of driver, defined by TEE_DESC_* below
 */
#define TEE_DESC_PRIVILEGED	0x1
struct tee_desc {
	const char *name;
	const struct tee_driver_ops *ops;
	struct module *owner;
	u32 flags;
};
```

```c
// include/linux/tee_core.h

/**
 * struct tee_driver_ops - driver operations vtable
 * @get_version:	returns version of driver
 * @open:		called when the device file is opened
 * @release:		release this open file
 * @open_session:	open a new session
 * @close_session:	close a session
 * @system_session:	declare session as a system session
 * @invoke_func:	invoke a trusted function
 * @cancel_req:		request cancel of an ongoing invoke or open
 * @supp_recv:		called for supplicant to get a command
 * @supp_send:		called for supplicant to send a response
 * @shm_register:	register shared memory buffer in TEE
 * @shm_unregister:	unregister shared memory buffer in TEE
 */
struct tee_driver_ops {
	void (*get_version)(struct tee_device *teedev,
			    struct tee_ioctl_version_data *vers);
	int (*open)(struct tee_context *ctx);
	void (*release)(struct tee_context *ctx);
	int (*open_session)(struct tee_context *ctx,
			    struct tee_ioctl_open_session_arg *arg,
			    struct tee_param *param);
	int (*close_session)(struct tee_context *ctx, u32 session);
	int (*system_session)(struct tee_context *ctx, u32 session);
	int (*invoke_func)(struct tee_context *ctx,
			   struct tee_ioctl_invoke_arg *arg,
			   struct tee_param *param);
	int (*cancel_req)(struct tee_context *ctx, u32 cancel_id, u32 session);
	int (*supp_recv)(struct tee_context *ctx, u32 *func, u32 *num_params,
			 struct tee_param *param);
	int (*supp_send)(struct tee_context *ctx, u32 ret, u32 num_params,
			 struct tee_param *param);
	int (*shm_register)(struct tee_context *ctx, struct tee_shm *shm,
			    struct page **pages, size_t num_pages,
			    unsigned long start);
	int (*shm_unregister)(struct tee_context *ctx, struct tee_shm *shm);
};
```

```c
// include/linux/tee_core.h

/**
 * struct tee_shm_pool - shared memory pool
 * @ops:		operations
 * @private_data:	private data for the shared memory manager
 */
struct tee_shm_pool {
	const struct tee_shm_pool_ops *ops;
	void *private_data;
};
```

```c
// include/linux/tee_core.h

/**
 * struct tee_shm_pool_ops - shared memory pool operations
 * @alloc:		called when allocating shared memory
 * @free:		called when freeing shared memory
 * @destroy_pool:	called when destroying the pool
 */
struct tee_shm_pool_ops {
	int (*alloc)(struct tee_shm_pool *pool, struct tee_shm *shm,
		     size_t size, size_t align);
	void (*free)(struct tee_shm_pool *pool, struct tee_shm *shm);
	void (*destroy_pool)(struct tee_shm_pool *pool);
};
```

---

```c
// include/linux/tee_drv.h

/**
 * struct tee_context - driver specific context on file pointer data
 * @teedev:	pointer to this drivers struct tee_device
 * @data:	driver specific context data, managed by the driver
 * @refcount:	reference counter for this structure
 * @releasing:  flag that indicates if context is being released right now.
 *		It is needed to break circular dependency on context during
 *              shared memory release.
 * @supp_nowait: flag that indicates that requests in this context should not
 *              wait for tee-supplicant daemon to be started if not present
 *              and just return with an error code. It is needed for requests
 *              that arises from TEE based kernel drivers that should be
 *              non-blocking in nature.
 * @cap_memref_null: flag indicating if the TEE Client support shared
 *                   memory buffer with a NULL pointer.
 */
struct tee_context {
	struct tee_device *teedev;
	void *data;
	struct kref refcount;
	bool releasing;
	bool supp_nowait;
	bool cap_memref_null;
};
```

---

- `do_initcalls()`
    - …
        - [`tee_init()`](/posts/linux-kernel-tee/#tee-init)

---

- {{< anchor id="tee-init" >}}`tee_init()`
    - Call `class_register()` to register `tee_class`.

        ```c
        // drivers/tee/tee_core.c

        static const struct class tee_class = {
        	.name = "tee",
        };
        ```

    - Call `alloc_chrdev_region()` to register a range of char device numbers for TEE devices.
        - 0 ~ `TEE_NUM_DEVICES`.
    - Call `bus_register()` to register `tee_bus_type`.

        ```c
        // drivers/tee/tee_core.c

        const struct bus_type tee_bus_type = {
        	.name		= "tee",
        	.match		= tee_client_device_match,
        	.uevent		= tee_client_device_uevent,
        };
        ```

- {{< anchor id="tee-device-register" >}}`tee_device_register()`
    - Call `cdev_device_add()` to add the TEE character device to the system.
- {{< anchor id="tee-open" >}}`tee_open()`
    - [`teedev_open()`](/posts/linux-kernel-tee/#teedev-open)
    - Assign `filp->private_data` to the allocated `struct tee_context`.
- {{< anchor id="teedev-open" >}}`teedev_open()`
    - Allocate `struct tee_context` for the TEE device.
    - Assign `tee_context->teedev` to TEE device.
    - Call `teedev->desc->ops->open()`.
        - E.g. [`optee_smc_open()` ](/posts/linux-kernel-optee/#optee-smc-open)
- {{< anchor id="tee-client-open-context" >}}`tee_client_open_context()`
    - Call `class_find_device()` to look up the TEE devices that match with passed in `match()` (e.g. `optee_ctx_match()`) under TEE class (`tee_class`), then call [`teedev_open()`](/posts/linux-kernel-tee/#teedev-open) to open the TEE devices.
- {{< anchor id="tee-client-open-session" >}}`tee_client_open_session()`


    - Call `ctx->teedev->desc->ops->open_session()`.
        - E.g. [`optee_open_session()`](/posts/linux-kernel-optee/#optee-open-session)
- {{< anchor id="tee-client-invoke-func" >}}`tee_client_invoke_func()`
    - Call `teedev->desc->ops->invoke_func()`
        - E.g. [`optee_invoke_func()`](/posts/linux-kernel-optee/#optee-invoke-func)

---

- {{< anchor id="tee-ioctl" >}}`tee_ioctl()`
    - If `cmd`:
        - `TEE_IOC_OPEN_SESSION`:
            - [`tee_ioctl_open_session()`](/posts/linux-kernel-tee/#tee-ioctl-open-session)
        - …
        - {{< anchor id="tee-ioc-suppl-recv" >}}`TEE_IOC_SUPPL_RECV`:
            - [`tee_ioctl_supp_recv()`](/posts/linux-kernel-tee/#tee-ioctl-supp-recv)
        - {{< anchor id="tee-ioc-suppl-send" >}}`TEE_IOC_SUPPL_SEND`:
            - [`tee_ioctl_supp_send()`](/posts/linux-kernel-tee/#tee-ioctl-supp-send)
- {{< anchor id="tee-ioctl-open-session" >}}`tee_ioctl_open_session()`
- {{< anchor id="tee-ioctl-supp-recv" >}}`tee_ioctl_supp_recv()`
    - Call `ctx->teedev->desc->ops->supp_recv()`.
        - E.g. [`optee_supp_recv()` - TEE supplicant waits for OP-TEE’s request](/posts/linux-kernel-optee/#optee-supp-recv)
- {{< anchor id="tee-ioctl-supp-send" >}}`tee_ioctl_supp_send()`
    - Call `ctx->teedev->desc->ops->supp_send()`.
        - E.g. [`optee_supp_send()` - TEE supplicant handles the RPC and sends the response to OP-TEE. ](/posts/linux-kernel-optee/#optee-supp-send)
