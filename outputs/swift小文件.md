# swift小文件合并

1. 把部分container看成volume，将小文件放在里面。
2. 使用一个数据库存储小文件的原始URI与真实URI的映射

## 过程

写：io请求->决定是大小文件->选择container->获取地址->发回响应

读：io请求->数据库查询->地址转换->查询->发回响应