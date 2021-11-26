1. Lua redis() command arguments must be strings or integers   
```lua
-- 参数，之间要有空格
redis-cli -a 123456 --eval .\lua\init.lua 11 , 111

https://zhuanlan.zhihu.com/p/77484377
http://redisdoc.com/script/eval.html#redis
```

2. redis规定 lua 不允许使用全局变量