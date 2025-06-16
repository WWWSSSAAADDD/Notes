# Notes
Used to take notes

## Mystring::operator>> 实现要点

### 1. 前导空白跳过

* 使用 `input.get(ch)` 逐字符获取。由于它是**非格式化输入**，不会自动跳过空白，需手动跳过:

  ```cpp
  char ch;
  while (input.get(ch) && std::isspace(static_cast<unsigned char>(ch))) {
      // skip
  }
  if (!input) return input; // 若遇 EOF/出错
  ```

### 2. 动态扩容机制

* 初始分配缓冲区 `capacity = 256`，设置 `length = 0`。
* 将第一个有效字符 `buf[length++] = ch` 写入。
* 在循环中持续读取:

  ```cpp
  while (input.get(ch) && !std::isspace(static_cast<unsigned char>(ch))) {
      if (length + 1 >= capacity) {
          // 翻倍扩容
          Mysize newCap = capacity * 2;
          char* tmp = new char[newCap];
          std::memcpy(tmp, buf, capacity);
          delete[] buf;
          buf = tmp;
          capacity = newCap;
      }
      buf[length++] = ch;
  }
  ```
* 留出 1 字节用于字符串结束符 `\0`。

### 3. 遇到空白与 EOF 的处理

* **空白**: 若循环因空白终止，需将该空白字符 `unget()` 回流:

  ```cpp
  if (input && std::isspace(static_cast<unsigned char>(ch))) {
      input.unget();
  }
  ```
* **EOF / 流错误**: `input.get(ch)` 返回 `false`，且 `!input` 为真，直接跳出并返回流。

  * 可选：使用无参数 `input.get()` 返回 `int`，与 `EOF` 比较:

    ```cpp
    int c;
    while ((c = input.get()) != EOF && !std::isspace(c)) {
        // 处理 c
    }
    ```

### 4. 构造与清理

* 在缓冲区末尾添加结束符:

  ```cpp
  buf[length] = '\0';
  ```
* 用 `buf` 构造 `Mystring`：

  ```cpp
  str = Mystring(buf);
  ```
* 释放临时缓冲区:

  ```cpp
  delete[] buf;
  ```

---

### 5. 使用 std::isspace 时的类型安全

* `std::isspace` 接受的参数类型是 `int`，其有效值必须来自 `unsigned char` 或特殊值 `EOF`。
* 如果直接将带符号的 `char`（可能为负值或大于 `UCHAR_MAX`）传入，会触发未定义行为，因为函数内部会将参数用作索引访问字符分类表，超出范围会越界。
* 使用 `static_cast<unsigned char>(ch)` 强制转换，确保输入落在 `[0, UCHAR_MAX]` 范围，然后提升为 `int`，这样即可安全调用。

---

以上流程无需依赖 `std::string` 或 `gcount()`，保证了任意长度单词的安全读取与动态扩容。
