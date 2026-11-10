如果以后某个时间又想让 apt 临时走代理，不用永久配置，可以临时这样：

``` bash
sudo apt -o Acquire::http::Proxy="http://127.0.0.1:7897" \
         -o Acquire::https::Proxy="http://127.0.0.1:7897" \
         update
```

这样只有这一条命令走代理，不影响以后。