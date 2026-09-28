---
date: "2024-11-17T00:00:00+08:00"
title: "OP-TEE: TAs"
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

- {{< anchor id="entry-open-session" >}}`entry_open_session()`
    - Open the session based on the UUID.
    - [`tee_ta_open_session()`](/posts/optee-ta/#tee-ta-open-session)
- {{< anchor id="tee-ta-open-session" >}}`tee_ta_open_session()`
    - [`tee_ta_init_session()`](/posts/optee-ta/#tee-ta-init-session)
    - Call `ts_ctx->ops->enter_open_session()` callback,
        - For pseudo TAs, the callback is: [`pseudo_ta_enter_open_session()`](/posts/optee-pseudo-ta/#hl-1-4)
        - For user TAs, the callback is [`user_ta_enter_open_session()`](/posts/optee-user-ta/#hl-1-8)
- {{< anchor id="tee-ta-init-session" >}}`tee_ta_init_session()`
    - Look for already loaded TA
        - `tee_ta_init_session_with_context()`
    - If the TA for this UUID is not loaded yet:
        - Look for secure partition
            - `stmm_init_session()`
        - Look for pseudo TA
            - [`tee_ta_init_pseudo_ta_session()`](/posts/optee-pseudo-ta/#hl-0-2)
        - Look for user TA
            - [`tee_ta_init_user_ta_session()`](/posts/optee-user-ta/#hl-0-3)
            - If [`tee_ta_init_user_ta_session()`](/posts/optee-user-ta/#hl-0-3) returns `TEE_SUCCESS`, call [`tee_ta_complete_user_ta_session()`](/posts/optee-user-ta/#tee-ta-complete-user-ta-session)

---

- {{< anchor id="entry-invoke-command" >}}`entry_invoke_command()`
    - Call `tee_ta_get_session()` to get the opened session from `arg->session`.
    - Call `tee_ta_invoke_command()` on the opened session.
        - Call `ts_ctx->ops->enter_invoke_cmd()`:
        - For pseudo TAs, the callback is: [`pseudo_ta_enter_invoke_cmd()`](/posts/optee-pseudo-ta/#hl-1-5)
        - For user TAs, the call back is: [`user_ta_enter_invoke_cmd()`](/posts/optee-user-ta/#hl-1-9)
