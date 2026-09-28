---
date: "2024-11-21T00:00:00+08:00"
title: "OP-TEE: Pseudo TAs"
author: "Frank Chang"
categories:
  - "Security"
tags:
  - "OP-TEE"
series:
  - "OP-TEE Code Trace"
---

> ⚠️ The code is based on: [https://gitlab.com/riseproject/riscv-optee/optee_os/-/tree/dev-optee-mpxy](https://gitlab.com/riseproject/riscv-optee/optee_os/-/tree/dev-optee-mpxy)
>
> Commit ID: **`75df9ba41a404aec897399ead0ff0aebcbff48ca`**

- {{< anchor id="tee-ta-init-pseudo-ta-session" >}}`tee_ta_init_pseudo_ta_session()`
    - Look up the pseudo TA based on UUID.
    - Create pseudo TA context for the UUID, assign `ctx->ts_ctx.ops` to `pseudo_ta_ops`. `pseudo_ta_ops` will be the global callbacks for the pseudo TAs:

        ```c
        // core/kernel/pseudo_ta.c
        TEE_Result tee_ta_init_pseudo_ta_session(const TEE_UUID *uuid,
        			struct tee_ta_session *s)
        {

        	.....

          ctx->ref_count = 1;
        	ctx->flags = ta->flags;
        	stc->pseudo_ta = ta;
        	ctx->ts_ctx.uuid = ta->uuid;
        	ctx->ts_ctx.ops = &pseudo_ta_ops;

        	.....

        }
        ```

        ```c
        // core/kernel/pseudo_ta.c

        static const struct ts_ops pseudo_ta_ops = {
        	.enter_open_session = pseudo_ta_enter_open_session,
        	.enter_invoke_cmd = pseudo_ta_enter_invoke_cmd,
        	.enter_close_session = pseudo_ta_enter_close_session,
        	.destroy = pseudo_ta_destroy,
        };
        ```


---

- {{< anchor id="pseudo-ta-enter-open-session" >}}`pseudo_ta_enter_open_session()`
    - Call `stc->pseudo_ta->open_session_entry_point()` callback, if defined.
        - E.g. If the opened session is for pseudo TA: `rtc.pta`, `open_session_entry_point()` callback is: `open_session()`:

            ```c
            // core/pta/rtc.c

            pseudo_ta_register(.uuid = PTA_RTC_UUID, .name = PTA_NAME,
            		   .flags = PTA_DEFAULT_FLAGS | TA_FLAG_CONCURRENT |
            			    TA_FLAG_DEVICE_ENUM,
            		   .open_session_entry_point = open_session,
            		   .invoke_command_entry_point = invoke_command);
            ```


---

- {{< anchor id="pseudo-ta-enter-invoke-cmd" >}}`pseudo_ta_enter_invoke_cmd()`
    - Call `stc->pseudo_ta->invoke_command_entry_point()` callback.
        - E.g. If the opened session is for pseudo TA: `device.pta`, `invoke_command_entry_point()` callback is: `invoke_command()`, which will call `get_devices()` to retrieve the UUIDs of the pseudo TAs:

            ```c
            // core/pta/device.c

            pseudo_ta_register(.uuid = PTA_DEVICE_UUID, .name = PTA_NAME,
            		   .flags = PTA_DEFAULT_FLAGS,
            		   .invoke_command_entry_point = invoke_command);
            ```
