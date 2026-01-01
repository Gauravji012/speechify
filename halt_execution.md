# How `/halt_execution` Endpoint Works Internally

## Overview

The **`/halt_execution`** endpoint is responsible for gracefully stopping an orchestration plan execution and performing all necessary cleanup operations. It's a **POST** endpoint located at `/api/v3/orchestration/halt_execution`.

---

## Endpoint Entry Point

The route handler is defined in `services_root/services/orchestration/routes/HaltExecution.lua`:

```lua
local function request_handler(request_context)
    local plan_execution_id = request_context.body.plan_execution_id
    local operator_name = request_context.claims.operator_name
    local input = request_context.body.input
    local plan_version = request_context.body.plan_version
    local halt_all_child_executions = request_context.body.halt_all_child_executions
    local portal_id = request_context.claims.portal_id or "unknown"

    local resp, err = orchestration.halt_execution(
            operator_name, plan_execution_id, nil, nil, input, plan_version, halt_all_child_executions, portal_id
    )
    if err then return nil, err end
    return {
        body = {
            halt_execution_resp = resp.halt_execution_resp or {},
            callback_for_halt = resp.halt_execution_callback
        }
    }
end
```

The route handler extracts the necessary parameters from the request and calls the core `orchestration.halt_execution()` function.

---

## Core `halt_execution` Function Flow

The main `halt_execution` function in `OrchestrationManager.lua` performs the following operations in sequence:

### 1. **Optional Child Execution Halt** (if requested)

```lua
local halt_child_execution_resp
if halt_all_child_executions then
    halt_child_execution_resp = _M.halt_child_executions(operator_name, execution_id, portal_id)
end
```

If the `halt_all_child_executions` flag is set to `true`, the function first recursively finds and halts all active child executions before proceeding with the main execution.

### 2. **Fetch Execution and Validate State**

```lua
local execution, err_fetch = orch_da.get_execution(operator_name, execution_id)
if err_fetch then
    return nil, err_fetch
end

local state = execution.last_state
local plan_id = execution.plan_id

ctx.refresh(execution_id, {
    operator_name = operator_name,
    plan_id = plan_id,
    state = state,
    current_execution_status = execution.status,
})

if not cons.EXECUTION_STATUS_TRANSITION[cons.EXECUTION_STATUS.STOPPED][execution.status] then
    logger:error("Can't halt execution %s, blocked by status transition rule. Current status: %s, desired status: %s",
        execution_id, execution.status, cons.EXECUTION_STATUS.STOPPED)
    return nil, {
        code = errors.ERROR.ORCHESTRATION.INVALID_STATE_TRANSITION,
        module = errors.MODULE_NAME.ORCHESTRATION,
    }
end
```

**State Transition Validation**: An execution can only be halted if it's not already in a terminal state (SUCCESS or FAILED). The transition rules are:
- ✅ **Can halt from**: EXECUTING, PENDING, WAITING_FOR_CALLBACK, WAITING_FOR_HUMAN_TASK, etc.
- ❌ **Cannot halt from**: SUCCESS, FAILED

### 3. **Log History Event**

```lua
history.log_history(
    execution_id,
    cons.EVENT_TYPE.EXECUTION.HALTED,
    {
        previous_status = execution.status
    }
)
```

Records the halt event in the execution history for audit purposes.

### 4. **Update Execution Status to STOPPED**

```lua
if is_batch_operation then
    local _, err = orch_da.update_execution(operator_name, execution_id,
        {
            status = cons.EXECUTION_STATUS.STOPPED,
            batch_operations = {
                cjson.encode({
                    batch = {
                        name = name,
                        type = cons.BATCH_OPERATIONS.TYPE.HALT,
                        create_time = current_ms(),
                    },
                })
            }
        }
    )
    if err then
        return nil, err
    end
else
    local _, err = orch_da.update_execution(operator_name, execution_id, { status = cons.EXECUTION_STATUS.STOPPED })
    if err then
        return nil, err
    end
end
```

Updates the execution status in the database. For batch operations, it also records batch operation metadata.

### 5. **Handle Paused Executions**

```lua
--remove from paused_executions table
if execution.status == cons.EXECUTION_STATUS.WAITING_FOR_RESTART then
    local _, paused_execution_err = orch_da.remove_paused_execution_by_execution_id(execution_id)
    if paused_execution_err then
        logger:error("[ORCHESTRATION] Error occurred while removing paused execution %s, from paused_executions table", execution_id)
        return nil, {
            module = errors.MODULE_NAME.ORCHESTRATION,
            code = errors.ERROR.ORCHESTRATION.PAUSED_EXECUTION_REMOVE_FAILED,
        }
    end
end
```

If the execution was waiting for restart, it's removed from the paused executions table.

### 6. **Deschedule Active Reservations**

```lua
local deschedule_reservations
if(execution.status ~= client_constants.EXECUTION_STATUS.STOPPED
    and execution.status ~= client_constants.EXECUTION_STATUS.SUCCESS
    and execution.status ~= client_constants.EXECUTION_STATUS.FAILED) then
    local reservations, err_fetch_res = reservation_util.get_fresh_reservations_for_exec(execution_id)
    if err_fetch_res then
        return nil, err_fetch_res
    end
    deschedule_reservations = reservations

    if not is_batch_operation then
        for _, reservation in ipairs(reservations) do
            if cons.FRESH_RESERVATION_TYPE_CONTAINS_REDIS_ENTRY[reservation.type] then
                local _, err_halt = reservation_util.halt_reservation_redis(
                    operator_name, execution_id, reservation.reservation_id, reservation.type, (reservation.data or {}).is_callback_run_plan
                )
                if err_halt then
                    return nil, err_halt
                end
            end
        end
    end
end
```

For executions that aren't already in terminal states, this step:
- Fetches all active reservations (scheduled tasks)
- Halts Redis-based reservations immediately (prevents scheduled tasks from running)
- For batch operations, defers Redis halting to batch processing

### 7. **Cleanup Pending Reservations**

```lua
local _, err = execution_util.cleanup_pending_reservations(execution_id, state)
if err then
    return nil, err
end
```

Marks all pending reservations as STALE in the database so they won't be processed.

The `cleanup_pending_reservations` function:

```lua
local function cleanup_pending_reservations(execution_id, state)
    local final_err
    local pending_res_list, err = reservation_util.get_fresh_reservations_for_exec(execution_id)
    if err then
        return nil, err
    end

    local resolved_reservations = {}
    for _, reservation in ipairs(pending_res_list) do
        local _, cleanup_err = reservation_util.cleanup_reservation(reservation.execution_id, reservation.reservation_id, reservation.type)
        final_err = final_err or cleanup_err
        if not cleanup_err then
            table.insert(resolved_reservations, reservation)
        end
    end

    history.log_history(
        execution_id,
        cons.EVENT_TYPE.RESERVATION.CLEANED,
        {
            execution_id = execution_id,
            state = state,
            resolved_reservations = resolved_reservations
        }
    )

    return nil, final_err
end
```

### 8. **Unlock Execution**

```lua
local _, unlock_err = orchestration_locking.handle_execution_unlocking(execution, true)
if unlock_err then
    return nil, unlock_err
end
```

Releases any locks held by the execution to prevent deadlocks.

### 9. **Parent Execution Callback** (if applicable)

```lua
local callback_for_halt, callback_err
local related_executions = cjson.decode(execution.context.related_executions) or {}

if related_executions.master_plan_execution_id and related_executions.master_plan_execution_id ~= "unset" and not disable_parent_callback then
    callback_for_halt, callback_err = _send_callback_for_parent_execution(operator_name, execution_id, execution, plan_version, related_executions, input)
    if callback_err then
        return nil, callback_err
    end
end
```

If this execution is a child of a parent execution, it notifies the parent that the child was halted. This allows the parent to continue or handle the halt appropriately.

### 10. **Return Response**

```lua
return {
    halt_execution_resp = halt_child_execution_resp,
    halt_execution_callback = callback_for_halt
}
```

Returns:
- **`halt_execution_resp`**: Results of halting child executions (if requested)
- **`halt_execution_callback`**: Information about the parent callback (if applicable)

---

## Key Design Points

### **1. Idempotency**
The function is safe to call multiple times - it validates state transitions before proceeding.

### **2. Cascading Halts**
When `halt_all_child_executions` is true, the function recursively finds all active child executions and halts them first.

### **3. Reservation Cleanup**
The function ensures that:
- Active Redis reservations are halted immediately
- Pending reservations are marked as STALE
- No scheduled tasks will run after the halt

### **4. Parent Notification**
If the halted execution is a child of a parent execution, the parent is notified via callback so it can handle the halt appropriately.

### **5. Lock Management**
All execution locks are released to prevent deadlocks.

### **6. History Tracking**
All halt events are logged in the execution history for auditability.

---

## Error Handling

The function returns errors for:
- ❌ **Invalid state transitions** (already SUCCESS/FAILED)
- ❌ **Database fetch/update failures**
- ❌ **Reservation cleanup failures**
- ❌ **Lock release failures**
- ❌ **Parent callback failures**

---

## Summary

The `halt_execution` function ensures that an execution is fully stopped and cleaned up, preventing orphaned tasks or inconsistent state. It handles:
- ✅ State validation
- ✅ Child execution halting
- ✅ Reservation cleanup
- ✅ Lock release
- ✅ Parent notification
- ✅ History logging

This comprehensive approach ensures the orchestration system maintains consistency even when executions are forcefully stopped.

