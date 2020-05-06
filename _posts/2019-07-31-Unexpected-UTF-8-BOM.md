---

title: Unexpected-UTF-8-BOM
date: 2019-07-31 14:29:43
categories: 
 - Python
---
# Unexpected-UTF-8-BOM

问题描述：
json.loads(text,encoding=’utf8’) 报Unexpected UTF-8 BOM (decode using utf-8-sig)错误，将encoding改为’utf-8-sig’仍然报错。
原因分析：
text包含BOM字符
解决方案：
将BOM头去掉，代码如下：

```python
if text.startswith(u'\ufeff'):    
    text = text.encode('utf8')[3:].decode('utf8')
```