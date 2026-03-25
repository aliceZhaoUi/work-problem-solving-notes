# 遇到的问题

上传csv file时，文件内容为每行一个号码，使用 `event.target.result` 拿到content时，content中的换行格式会根据原文件的格式有所不同，比如有的文件的换行符格式为纯 CR (`\r`)，而有的是CRLF (`\r\n`)。而原本的后端代码被写死了，只能处理 CRLF (`\r\n`)，所以上传 CR (`\r`) 格式时会报错。

## csv file format sample

```text
1234567899
0123456789
9876543210
1234567890
```

## 问题根因

前端的换行符为自动接收文件的格式（numbers = event.target.result），或者为const lines = numbers.split('\r\n'); // 只能处理 CRLF

后端（PostgreSQL）插入数据时也写死了 \r\n: regexp_split_to_table(csvContent, '\r\n');

如果用户上传的文件是纯 CR (\r)，前端无法正确分行，整个文件被当成一行处理，后端同样无法正确拆分插入。

---
## 解决思路

1. 前端负责标准化换行符，无论用户上传什么格式，统一转成 \r\n 再发给后端
2. 后端不需要改动，继续按 \r\n 处理

---
## 解决方法

submitInfoFOTA（原本只做了 MDN 处理，补充换行兼容）：

// 改前
const lines = numbers.split('\r\n');

// 改后
const lines = numbers.split(/\r\n|\r/);

submitInfoSU（原本直接传原始内容，补充标准化）：

// 改前
let csvContent = numbers;

// 改后
let csvContent = numbers.split(/\r\n|\r/).join('\r\n');

正则 /\r\n|\r/ 的关键点： \r\n 写在左边优先匹配，不会把 CRLF 拆成两个空行。

---
##调试过程

| 方法 | 结论 |
| includes('\r\n') | 不可靠，\r\n 本身包含 \r 和 \n，三个都返回 true 并不能区分格式 |
| charCodeAt() | 可靠，直接检查原始字节，13=\r，10=\n，看前后组合确认格式 |

最终通过 charCodeAt 确认：
- 原始 numbers： index 10 = 13，后一个 = 55（'7'）→ 纯 CR
- 处理后 csvContent： index 10 = 13，后一个 = 10（\n）→ 已转为 CRLF

##具体检测方法：
换行符在计算机里就是特定的数字（ASCII码）：

| 字符 | charCode | 含义 |
| \r | 13 | Carriage Return |
| \n | 10 | Line Feed |

所以代码做的事情是：

1. 遍历字符串，找到第一个 charCode 是 13 或 10 的位置（即第一个换行符）
2. 看它的前一个和后一个字符的 charCode

然后通过组合判断：
- 13 后面跟 10 → \r\n（CRLF）
- 13 后面不是 10 → 纯 \r（CR）
- 10 前面不是 13 → 纯 \n（LF）

具体检测代码：
if (code === 13 || code === 10) {
    console.log(`index ${i}: ${code}, 前一个: ${numbers.charCodeAt(i-1)}, 后一个: ${numbers.charCodeAt(i+1)}`);
    break;
}

---
## 总结

- includes() 不适合精确检测换行符类型，因为 \r\n 本身包含 \r 和 \n
- charCodeAt() 是检测换行符的可靠方式
- 处理用户上传文件时，应在前端统一标准化换行符，不能假设用户文件的格式
- 正则 /\r\n|\r|\n/ 可兼容所有三种换行符格式（CRLF / CR / LF）