# 功能概览

Tracing 模块用于对组件与任意函数进行追踪（tracing），包含两部分：**Log** 与 **Report**。

- **Log**：以 DashScope Log 的格式输出日志。
- **Report**：使用 OpenTelemetry SDK 上报追踪信息。

# 用法

## Logging（日志）

1. 配置环境变量（默认启用）

```shell
export TRACE_ENABLE_LOG=true
```

2. 为任意函数添加装饰器，例如：

```python
from agentscope_runtime.engine.tracing import trace, TraceType

@trace(trace_type=TraceType.LLM, trace_name="llm_func")
def llm_func():
    pass
```

输出示例：

```text
{"time": "2025-08-13 11:23:41.808", "step": "llm_func_start", "model": "", "user_id": "", "code": "", "message": "", "task_id": "", "request_id": "", "context": {}, "interval": {"type": "llm_func_start", "cost": 0}, "ds_service_id": "test_id", "ds_service_name": "test_name"}
{"time": "2025-08-13 11:23:41.808", "step": "llm_func_end", "model": "", "user_id": "", "code": "", "message": "", "task_id": "", "request_id": "", "context": {}, "interval": {"type": "llm_func_end", "cost": "0.000"}, "ds_service_id": "test_id", "ds_service_name": "test_name"}
```

3. 自定义日志（前置条件：**函数包含 kwargs 参数**）

```python
from agentscope_runtime.engine.tracing import trace, TraceType

@trace(trace_type=TraceType.LLM, trace_name="llm_func")
def llm_func(**kwargs):
    trace_event = kwargs.pop("trace_event", None)
    if trace_event:
        # 自定义字符串 message
        trace_event.on_log("hello")

        # 结构化 step message
        trace_event.on_log(
            "",
            **{
                "step_suffix": "mid_result",
                "payload": {
                    "output": "hello",
                },
            },
        )
```

输出示例：

```text
{"time": "2025-08-13 11:27:14.727", "step": "llm_func_start", "model": "", "user_id": "", "code": "", "message": "", "task_id": "", "request_id": "", "context": {}, "interval": {"type": "llm_func_start", "cost": 0}, "ds_service_id": "test_id", "ds_service_name": "test_name"}
{"time": "2025-08-13 11:27:14.728", "step": "", "model": "", "user_id": "", "code": "", "message": "hello", "task_id": "", "request_id": "", "context": {}, "interval": {"type": "", "cost": "0"}, "ds_service_id": "test_id", "ds_service_name": "test_name"}
{"time": "2025-08-13 11:27:14.728", "step": "llm_func_mid_result", "model": "", "user_id": "", "code": "", "message": "", "task_id": "", "request_id": "", "context": {"output": "hello"}, "interval": {"type": "llm_func_mid_result", "cost": "0.000"}, "ds_service_id": "test_id", "ds_service_name": "test_name"}
{"time": "2025-08-13 11:27:14.728", "step": "llm_func_end", "model": "", "user_id": "", "code": "", "message": "", "task_id": "", "request_id": "", "context": {}, "interval": {"type": "llm_func_end", "cost": "0.000"}, "ds_service_id": "test_id", "ds_service_name": "test_name"}
```

## Reporting（上报）

1. 配置环境变量（默认关闭）

```shell
export TRACE_ENABLE_LOG=false
export TRACE_ENABLE_REPORT=true
export TRACE_AUTHENTICATION={YOUR_AUTHENTICATION}
export TRACE_ENDPOINT={YOUR_ENDPOINT}
```

2. 为**非流式**函数添加装饰器，例如：

```python
from agentscope_runtime.engine.tracing import trace, TraceType

@trace(trace_type=TraceType.LLM,
       trace_name="llm_func")
def llm_func(args: str):
    return args + "hello"
```

3. 为**流式**函数添加装饰器，例如：

```python
from agentscope_runtime.engine.tracing import trace, TraceType
from agentscope_runtime.engine.tracing.message_util import (
    get_finish_reason,
    merge_incremental_chunk,
)

@trace(trace_type=TraceType.LLM,
       trace_name="llm_func",
       get_finish_reason_func=get_finish_reason,
       merge_output_func=merge_incremental_chunk)
def llm_func(args: str):
    for i in range(10):
        yield i
```

其中 `get_finish_reason` 与 `merge_incremental_chunk` 为自定义处理函数（可选）。默认使用 `message_util.py` 中提供的同名实现。

- `get_finish_reason`：用于获取 `finish_reason`，以判断流式输出是否结束。示例：

```python
from openai.types.chat import ChatCompletionChunk
from typing import List, Optional

def get_finish_reason(response: ChatCompletionChunk) -> Optional[str]:
    finish_reason = None
    if hasattr(response, 'choices') and len(response.choices) > 0:
        if response.choices[0].finish_reason:
            finish_reason = response.choices[0].finish_reason

    return finish_reason
```

- `merge_output`：用于合并输出，以构造最终输出信息。示例：

```python
from openai.types.chat import ChatCompletionChunk
from typing import List, Optional

def merge_incremental_chunk(
    responses: List[ChatCompletionChunk],
) -> Optional[ChatCompletionChunk]:
    # get usage or finish reason
    merged = ChatCompletionChunk(**responses[-1].__dict__)

    # if the responses has usage info, then merge the finish reason chunk to usage chunk
    if not merged.choices and len(responses) > 1:
        merged.choices = responses[-2].choices

    for resp in reversed(responses[:-1]):
        for i, j in zip(merged.choices, resp.choices):
            if isinstance(i.delta.content, str) and isinstance(
                j.delta.content,
                str,
            ):
                i.delta.content = j.delta.content + i.delta.content
        if merged.usage and resp.usage:
            merged.usage.total_tokens += resp.usage.total_tokens

    return merged
```

4. 设置 `request_id` 与通用属性（common attributes）

`request_id` 用于绑定不同请求的上下文；通用属性为公共 span attributes，该请求下的所有 span 都会携带这些属性。

- **自动设置 request_id**：当用户没有在请求处理开始时手动调用 `TracingUtil.set_request_id`，系统会在 root span 中自动生成并设置一个唯一的 `request_id`。
- **手动设置**：在**未使用 `@trace` 装饰器**的函数中设置 `request_id` 与通用属性（例如：请求信息解析完毕后立刻设置）。示例：

```python
from agentscope_runtime.engine.tracing import TracingUtil

common_attributes = {
    "gen_ai.user.id": "user_id",
    "bailian.app.id": "app_id",
    "bailian.app.owner_id": "app_id",
    "bailian.app.env": "pre",
    "bailian.app.workspace": "workspace"
}
TracingUtil.set_request_id("request_id")
TracingUtil.set_common_attributes(common_attributes)
```

5. 自定义上报（前置条件：**函数包含 kwargs 参数**）

```python
import json
from agentscope_runtime.engine.tracing import trace, TraceType

@trace(trace_type=TraceType.LLM, trace_name="llm_func")
def llm_func(**kwargs):
    trace_event = kwargs.pop("trace_event", None)
    if trace_event:
        # Set string attribute
        trace_event.set_attribute("key", "value")
        # Set dict attribute
        trace_event.set_attribute("func_7.key", json.dumps({'key0': 'value0', 'key1': 'value1'}))
```

