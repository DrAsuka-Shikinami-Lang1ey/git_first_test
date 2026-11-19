### 验证 RA2L1 使用内部 Data Flash 和 VEE 中间件实现的模拟 EEPROM 功能，包括数据保存、读回校验、同一记录更新及断电重启后的数据恢复
### 表达式
g_hsi_ee_state
g_hsi_ee_command
g_hsi_ee_write_value
g_hsi_ee_read_value
g_hsi_ee_record_valid
g_hsi_ee_verify_ok
g_hsi_ee_write_request_count
g_hsi_ee_write_complete_count
g_hsi_ee_error_count
g_hsi_ee_last_error

#### **测试一：启动读取**

操作：

1. 启动程序，连续运行约一秒。
2. 暂停，记录 EE 状态、读回值和错误码。

预期现象：

- 如果从未保存过记录：`state=1`、`record_valid=false`、`last_error=FSP_SUCCESS`。
- 如果已有记录：`state=3`、`record_valid=true`，读回上次保存值。


**测试二：保存并立即读回**

操作：

1. 确认 `state` 为 `1` 或 `3`。
2. 暂停后设置：
    
    ```
    g_hsi_ee_write_value = 1234
    g_hsi_ee_command = 1
    ```
    
3. 继续运行约一秒，再暂停。

预期现象：

|表达式|预期结果|
|---|---|
|`g_hsi_ee_command`|自动回到 `0`|
|`g_hsi_ee_state`|`3`|
|`g_hsi_ee_read_value`|`1234`|
|`g_hsi_ee_record_valid`|`true`|
|`g_hsi_ee_verify_ok`|`true`|
|写入请求／完成计数|分别增加 `1`|
|错误计数|不增加|
|最近错误码|


**测试三：更新同一条记录**

操作：

重复上面的步骤，将保存值改为 `5678`。

预期现象：

- 读回值变为 `5678`。
- `verify_ok=true`。
- 写入完成计数再增加 `1`。
- 错误计数不增加。


这项证明的是：**同一个 ID 能更新，并读取最新值。**

**测试四：单独读取，不触发写入**

操作：

1. 记下当前写入完成计数。
2. 将待写入变量改成另一个数，但只发读取命令：
    
    ```
    g_hsi_ee_write_value = 9999
    g_hsi_ee_command = 2
    ```
    
3. 继续运行，再查看结果。

预期现象：

- 读回值仍然是已保存的 **5678**。
- 写入请求／完成计数均不增加。
- `record_valid=true`。
- `verify_ok=false` 是正常的，因为此次只是读取，没有进行“写入后比较”。

**测试五：断电保持

操作：

1. 确认 `5678` 已保存成功。
2. 结束调试，将板子实际断电。
3. 等待约 5 秒后重新上电。
4. 重新连接调试，避免擦除 Data Flash。
5. **不发保存命令**，运行后查看读回结果。

预期现象：

```
g_hsi_ee_state                = 3
g_hsi_ee_read_value           = 5678
g_hsi_ee_record_valid         = true
g_hsi_ee_write_complete_count = 0
g_hsi_ee_error_count          = 0
```