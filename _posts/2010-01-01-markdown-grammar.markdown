---  
layout: post  
title:  "markdown语法"  
date:   2010-01-01 00:00:00 +0530  
---  
  
<style>  
.tablelines table, .tablelines td, .tablelines th {  
  border: 1px solid black;  
  }  
</style>  
  
## 标题一  
### 标题二  
  
**加粗**  
*斜体*  
  
点列表方式一:  
- 一级点列表  
  - 二级点列表  
  
点列表方式二:  
  * 点列表  
  
1. 数字列表  
  
[链接: Google](https://google.com)  
  
图片引用:  
![事务时间戳](https://chenghua-root.github.io/images/transaction-timestamp.png)  
  
表格:  
  
  | 名词 | 含义|  
  | -----  | ----  |  
  | x86 | x86系统，32位x86系统 |  
  | x86-64 | 64位x86系统 |  
{: .tablelines}  
  
代码:
```
struct free_area {  
  struct list_head free_list[MIGRATE_TYPES];  
  unsigned long nr_free;  
};
```
  
sed操作:  
```  
行尾删除空格: sed -i 's/[ \t]*$//' filename  
行尾添加空格: sed -i 's/$/&  /' filename  
```  
