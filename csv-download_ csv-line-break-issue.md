# 遇到的问题

上传csv file时，文件内容为每行一个号码，使用 `event.target.result` 拿到content时，content中的换行格式会根据原文件的格式有所不同，比如有的文件的换行符格式为纯 CR (`\r`)，而有的是CRLF (`\r\n`)。而原本的后端代码被写死了，只能处理 CRLF (`\r\n`)，所以上传 CR (`\r`) 格式时会报错。

## csv file format sample

```text
1234567899
0123456789
9876543210
1234567890