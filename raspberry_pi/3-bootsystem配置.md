config.txt

```bash
enable_uart=1  #enable pl uart 开启串口
uart_2ndstage=1   #enable FW debug info 开启树莓派固件调试

```





cmdline.txt

```bash
quiet #删除quiet 显示启动阶段打印
console=serial0,115200 #添加该字段配置输出串口
```


