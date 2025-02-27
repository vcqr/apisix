---
title: 批处理器
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

批处理器可用于聚合条目（日志/任何数据）并进行批处理。
当 `batch_max_size` 设置为零时，处理器将立即执行每个条目。将批处理的最大值设置为大于 1 将开始聚合条目，直到达到最大值或超时。

## 配置

创建批处理器的唯一必需参数是函数。当批处理达到最大值或缓冲区持续时间超过时，函数将被执行。

|名称           |必选项    |描述|
|-------        |-----          |------|
|name           |可选的       |标识批处理者的唯一标识符|
|batch_max_size |可选的       |每批的最大大小，默认为 `1000`|
|inactive_timeout|可选的      |如果不活动，将刷新缓冲区的最大时间（以秒为单位），默认值为 `5`|
|buffer_duration|可选的       |必须先处理批次中最旧条目的最大期限（以秒为单位），默认是 `5`|
|max_retry_count|可选的       |从处理管道中移除之前的最大重试次数；默认为 `0`|
|retry_delay    |可选的       |如果执行失败，应该延迟进程执行的秒数；默认为 `1`|

以下代码显示了如何在你的插件中使用批处理器：

```lua
local bp_manager_mod = require("apisix.utils.batch-processor-manager")
...

local plugin_name = "xxx-logger"
local batch_processor_manager = bp_manager_mod.new(plugin_name)
local schema = {...}
local _M = {
    ...
    name = plugin_name,
    schema = batch_processor_manager:wrap_schema(schema),
}

...


function _M.log(conf, ctx)
    local entry = {...} -- data to log

    if batch_processor_manager:add_entry(conf, entry) then
        return
    end
    -- create a new processor if not found

    -- entries is an array table of entry, which can be processed in batch
    local func = function(entries)
        -- serialize to json array core.json.encode(entries)
        -- process/send data
        return true
        -- return false, err_msg if failed
    end
    batch_processor_manager:add_entry_to_new_processor(conf, entry, ctx, func)
end
```

批处理器的配置将通过该插件的配置设置。
举个例子：

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
      "plugins": {
            "http-logger": {
                "uri": "http://mockbin.org/bin/:ID",
                "batch_max_size": 10,
                "max_retry_count": 1
            }
       },
      "upstream": {
           "type": "roundrobin",
           "nodes": {
               "127.0.0.1:1980": 1
           }
      },
      "uri": "/hello"
}'
```

如果你的插件只使用一个全局的批处理器，
你可以直接使用它：

```lua
local entry = {...} -- data to log
if log_buffer then
    log_buffer:push(entry)
    return
end

local config_bat = {
    name = config.name,
    retry_delay = config.retry_delay,
    ...
}

local err
-- entries is an array table of entry, which can be processed in batch
local func = function(entries)
    ...
    return true
    -- return false, err_msg if failed
end
log_buffer, err = batch_processor:new(func, config_bat)

if not log_buffer then
    core.log.warn("error when creating the batch processor: ", err)
    return
end

log_buffer:push(entry)
```

注意：请确保批处理的最大值（条目数）在函数执行的范围内。
刷新批处理的计时器基于 `inactive_timeout` 配置运行。因此，为了获得最佳使用效果，
保持 `inactive_timeout` 小于 `buffer_duration`。
