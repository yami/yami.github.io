# Content-Md5

Content-Md5 is base64 encoded string of 16 bytes (128 bits) MD5 hash value. So the Content-Md5 string length should be 24 (due to padding).

Following is Python code for generating Content-Md5 string:
 
```python
>>> import base64,hashlib
>>> hash = hashlib.md5()
>>> hash.update("0123456789")
>>> base64.b64encode(hash.digest())
```
where content is '0123456789'. The resulted Content-Md5 is 'eB5eJF1ptWaXm4bijSPyxw=='.
