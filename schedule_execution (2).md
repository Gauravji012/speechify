# How `/schedule_execution` Endpoint Works Internally

## Overview

The `/schedule_execution` endpoint is a critical component of the orchestration service that allows scheduling plan executions for future execution. This document explains the internal flow from HTTP request to Redis scheduling.

---

## 1. Entry Point - Route Handler

The endpoint is defined in `ScheduleExecution.lua`:

```lua
local function request_handler(request_context)
    local operator_name = request_context.claims.operator_name
    local body = request_context.body
    body.execution_invoker = request_context.claims.portal_id or (request_context.headers or {})[cons.CALLER_SERVICE_HEADER_NAME]

    local resp, err = orchestration.schedule_execution(operator_name, body)

    if err then
        return nil, err
    end

    return {
        body = resp
    }
end
```

**What it does:**
- Extracts `operator_name` from JWT claims
- Sets `execution_invoker` (portal_id or caller service header)
- Calls `orchestration.schedule_execution()` with operator and body
- Returns the response or error

---

## 2. Main Scheduling Function

The `schedule_execution` function in `OrchestrationManager.lua`:

```lua
local function schedule_execution(operator_name, body)
    local start_at = body.start_at or ngx.time()
    local delay = body.delay or 0

    local schedule_execution_ttl, validate_start_time_err = execution_scheduler.validate_start_time(
        operator_name, start_at, delay, body.plan_id, body.input, body.ignore_ttl
    )
    if validate_start_time_err then
        return nil, validate_start_time_err
    end
    local reservation_data = {
        operator_name = operator_name,
        plan_id = body.plan_id,
        input = body.input,
        master_plan_execution_id = body.master_plan_execution_id,
        _ttl = schedule_execution_ttl,
        execution_invoker = body.execution_invoker,
        master_rollback_execution_id = body.master_rollback_execution_id
    }
    local total_delay = delay + (start_at - ngx.time())

    local reservation, err = schedule_execution_handler(
        reservation_data,
        total_delay
    )

    if err then
        return nil, err
    end

    return reservation
end
```

**Key Steps:**
1. **Time Calculation**: Calculates `start_at` (defaults to current time) and `delay`
2. **TTL Validation**: Validates TTL constraints via `execution_scheduler.validate_start_time()`
3. **Reservation Data**: Builds `reservation_data` with plan info, input, and metadata
4. **Delay Calculation**: Computes `total_delay` (delay + time until start_at)
5. **Handler Invocation**: Calls `schedule_execution_handler()` with reservation data and delay

---

## 3. Schedule Execution Handler - Core Logic

The `schedule_execution_handler` function performs the main orchestration work:

```lua
local function schedule_execution_handler(reservation_data, delay, skip_add_reservation_and_schedule_execution)
    local reservation_type = cons.RESERVATION.TYPES.START
    local plan_id = reservation_data.plan_id
    local operator_name = reservation_data.operator_name
    local input = reservation_data.input
    local master_plan_execution_id = (ngx.req.get_headers() or {})['Execution-Id']

    -- Fetch and validate plan
    local full_plan, plan_err = _M.get_plan_from_cache(operator_name, plan_id, nil, nil, nil, true)
    if plan_err then
        return nil, plan_err
    end
    local plan = full_plan.data
    local plan_version = full_plan.version

    -- Validate plan access
    local _, plan_running_err = _is_running_plan_allowed(plan.access, plan_id)
    if plan_running_err then
        return nil, plan_running_err
    end

    -- Validate input
    local _, validation_err = input_validation.handle_input_validation(operator_name, plan, input, plan_version)
    if validation_err then
        return nil, validation_err
    end

    -- Generate execution ID
    local execution_id = uuid.generate_uuid()
    ctx.refresh(execution_id, {
        plan_id = plan_id,
        operator_name = operator_name,
        plan_owner = plan.plan_owner or cons.DEFAULT_PLAN_OWNER,
        async_query = plan.async_writes or false,
    })

    -- Setup related executions
    local related_executions = {
        sub_plan_execution_id = {},
        master_plan_execution_id = reservation_data.master_plan_execution_id or master_plan_execution_id or "unset",
        on_pause_handler_execution_id = {},
        on_resume_handler_execution_id = {},
        master_rollback_execution_id = reservation_data.master_rollback_execution_id,
    }

    local context = {
        run_plan_callback_reservation_id = reservation_data.run_plan_callback_reservation_id,
        related_executions = cjson.encode(related_executions),
        execution_invoker = reservation_data.execution_invoker,
    }

    -- Resolve rollback definition
    local _, rollback_definition_err = _resolve_rollback_definition(operator_name, plan, input, execution_id, context)
    if rollback_definition_err then
        return nil, rollback_definition_err
    end

    -- Create execution flow
    local schedule_execution_flow = execution_flow.create_flow(execution_id)
    
    -- Prepare mapper IDs
    local mapper_prep_result, mapper_prep_err = mapper_ids_util.prepare_and_insert_mapper_ids_at_beginning(
        operator_name, plan_id, execution_id, plan, input, schedule_execution_flow
    )
    if mapper_prep_err then
        return nil, mapper_prep_err
    end
    context.mapper_ids = cjson.encode(mapper_prep_result.mapper_ids or {})

    -- Insert plan execution map (if needed)
    if config.integration_test_pipeline or not plan.async_writes then
        schedule_execution_flow.insert_plan_execution_map_sharded(operator_name, plan_id, execution_id)
    end

    -- Start execution
    schedule_execution_flow.start_execution(
        operator_name, plan_id, cons.EXECUTION_STATUS.PENDING, execution_id, context, delay, current_ms(), plan.start_at
    )

    reservation_data.execution_id = execution_id
    reservation_data.state = plan.start_at

    -- Add reservation and schedule
    if not skip_add_reservation_and_schedule_execution then
        schedule_execution_flow.add_reservation_and_schedule_execution(reservation_data, reservation_type, delay, nil, true)
    end

    -- Execute flow (batched operations)
    local schedule_flow_response, flow_err = schedule_execution_flow.execute_flow()
    if flow_err then
        return nil, flow_err
    end

    -- Finalize mapper IDs
    local _, finalize_err = mapper_ids_util.finalize_mapper_ids_at_beginning(
        operator_name, plan_id, execution_id, mapper_prep_result.mapper_ids, mapper_prep_result.lock_keys, plan.sync_mapper_processing
    )
    if finalize_err then
        return nil, finalize_err
    end

    -- Return response
    local schedule_resp = schedule_flow_response.schedule_func_responses[1]
    return {
        execution_data = {
            execution_id = execution_id,
            reservation_id = (schedule_resp or {}).reservation_id,
            reservation_type = (schedule_resp or {}).reservation_type,
        }
    }
end
```

**What it does:**

1. **Plan Fetching**: Fetches and validates the plan from cache
2. **Access Validation**: Validates plan access permissions
3. **Input Validation**: Validates input against plan schema
4. **Execution ID Generation**: Generates unique `execution_id`
5. **Context Setup**: Sets up execution context (related executions, rollback definitions)
6. **Flow Creation**: Creates ExecutionFlow for batched database operations
7. **Mapper Preparation**: Prepares mapper IDs for tracking
8. **Execution Record**: Inserts execution record with PENDING status
9. **Reservation**: Adds reservation and schedules execution in Redis
10. **Flow Execution**: Executes the flow (batched DB operations + Redis scheduling)
11. **Finalization**: Finalizes mapper IDs
12. **Response**: Returns execution_id, reservation_id, and reservation_type

---

## 4. ExecutionFlow - Batching Pattern

`ExecutionFlow` is a sophisticated batching mechanism that collects database operations and defers Redis scheduling:

```lua
local function create_flow(execution_id)
    local flow = {
        query_descriptors = {},
        errors = {},
        scheduled_functions = {},
        kafka_event_data = {}
    }

    for function_name, func in pairs(_M.get_execution_operations()) do
        flow[function_name] = function(...)
            local response, error = func(...)
            if error then
                table.insert(flow.errors, error)
                return nil, error
            end
            response = response or {}
            pl.tablex.insertvalues(flow.query_descriptors, response.query_descriptors or {})
            pl.tablex.insertvalues(flow.scheduled_functions, response.scheduled_functions or {})
            pl.tablex.insertvalues(flow.kafka_event_data, response.kafka_event_data or {})
            return response
        end
    end
    
    flow.execute_flow = function()
        -- Validate errors
        if #flow.errors > 0 then
            return nil, flow.errors
        end

        -- Execute batched database queries
        local num_queries = #flow.query_descriptors
        local response, query_error
        if num_queries > 1 then
            response, query_error = orch_da.execute_batched_queries(flow.query_descriptors)
        else
            response, query_error = orch_da.execute_query(flow.query_descriptors)
        end
        
        if query_error then
            return nil, query_error
        end

        -- Send Kafka events
        for _, event_data in pairs(flow.kafka_event_data) do
            local _, kafka_err = orch_da.send_kafka_event(event_data)
            if kafka_err then
                table.insert(flow.errors, kafka_err)
            end
        end

        -- Execute scheduled functions (Redis scheduling)
        local schedule_func_responses = {}
        for _, scheduled_function_call in ipairs(flow.scheduled_functions or {}) do
            local schedule_response, schedule_err = scheduled_function_call.func(unpack(scheduled_function_call.args))
            if schedule_err then
                return nil, schedule_err
            end
            table.insert(schedule_func_responses, schedule_response)
        end
        
        EXECUTION_FLOW[execution_id] = nil
        return { batch_response = response, schedule_func_responses = schedule_func_responses }
    end
    
    EXECUTION_FLOW[execution_id] = flow
    return EXECUTION_FLOW[execution_id]
end
```

**How it works:**

- **Collection Phase**: Collects DB queries into `query_descriptors` and scheduling functions into `scheduled_functions`
- **Execution Phase**: On `execute_flow()`:
  - Executes batched database queries first
  - Then executes scheduling functions (Redis operations)
  - This ensures data consistency before scheduling

---

## 5. Reservation and Redis Scheduling

The reservation is added and scheduled in Redis:

```lua
local function schedule_execution(execution_id, reservation_type, operator_name, reservation_id, delay, is_callback_run_plan, is_postponed)
    local scheduler = schedule_fn_map.SCHEDULE
    return scheduler(execution_id, reservation_type, operator_name, reservation_id, delay, is_callback_run_plan, is_postponed)
end
```

The `SCHEDULE` function:

```lua
SCHEDULE = function(execution_id, reservation_type, operator_name, reservation_id, delay, is_callback_run_plan, is_postponed)
    local score = delay or 0
    if not is_postponed then
        score = score + ngx.time()
    end
    local poller_id = is_callback_run_plan and POLLER_ID.ORCHESTRATION_CALLBACK_RUN_PLAN or RESERVATION_TYPE_TO_POLLER_ID[reservation_type]
    local message = {
        execution_id = execution_id,
        reservation_id = reservation_id,
        reservation_type = reservation_type,
        schedule_type = reservation_type,
        integration_tests = routes_util.get_cached_integration_tests(),
    }
    local _, err = task_scheduler.schedule(
        operator_name,
        poller_id,
        message,
        score
    )
    return nil, err
end
```

**What it does:**

- **Score Calculation**: Calculates Redis score: `delay + current_time` (Unix timestamp in seconds)
- **Poller Selection**: Selects appropriate poller_id based on reservation type
- **Scheduling**: Calls TaskScheduler which uses Redis ZADD to schedule the task

---

## 6. Redis Storage (TaskScheduler)

TaskScheduler stores the task in Redis using a Lua script for atomicity:

```lua
local schedule_request_script = [==[
    local scheduled_task_key = KEYS[1]
    local request_data_key = KEYS[2]
    local entry_key = ARGV[1]
    local execution_time_in_seconds = ARGV[2]
    local entry_value = ARGV[3]

    redis.call("ZADD", scheduled_task_key, execution_time_in_seconds, entry_key)
    redis.call("HSET", request_data_key, entry_key, entry_value)
]==]
```

**Redis Operations:**

- **ZADD**: Adds entry to sorted set with execution timestamp as score
- **HSET**: Stores task details in a hash for later retrieval
- **Atomicity**: Lua script ensures both operations happen atomically

---

## Summary Flow

The complete flow from HTTP request to Redis scheduling:

1. **HTTP Request** → Route handler extracts operator and invoker
2. **Validation** → Validates TTL constraints and timing
3. **Plan Fetch** → Retrieves and validates plan from cache
4. **Execution Setup** → Generates execution_id, prepares context
5. **Flow Creation** → Creates ExecutionFlow for batched operations
6. **Database Writes** → Batched inserts (execution, reservation, mappers)
7. **Redis Scheduling** → ZADD to sorted set with execution timestamp
8. **Response** → Returns execution_id, reservation_id, reservation_type

---

## Key Design Patterns

### **Batching Pattern**
- ExecutionFlow batches database operations for efficiency
- Reduces database round trips

### **Deferred Execution**
- Scheduling functions are deferred until after database commits
- Ensures data consistency

### **Atomic Operations**
- Redis Lua scripts ensure atomic scheduling
- Prevents race conditions

### **Error Handling**
- Errors are collected and propagated throughout the flow
- Ensures proper cleanup on failure

### **Scalability**
- Sorted sets enable efficient polling of scheduled tasks
- Supports high-volume scheduling

---

## Response Format

The endpoint returns:

```json
{
    "execution_data": {
        "execution_id": "uuid-string",
        "reservation_id": "uuid-string",
        "reservation_type": "START"
    }
}
```

These IDs can be used to:
- Track execution status
- Query execution history
- Cancel/halt executions if needed

---

## Conclusion

This design enables scheduling plan executions at specific times, with the system polling Redis sorted sets to find tasks ready to execute. The batching pattern ensures efficiency, while the deferred execution ensures data consistency before scheduling.

