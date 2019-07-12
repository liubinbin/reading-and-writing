# python异常

## 异常

![image.png](http://upload-images.jianshu.io/upload_images/9085642-650dafa1af8ae7ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

BaseException是所有异常的基本，派生了KeyboardInterrupt，Exception，SystemExit。

KeyboaryInterrupt 是通过键盘的ctrl+c终止进程。

一般情况下使用Exception就可以了。



## 捕获异常

```python
try:
    pass
except Exception as e:    
	exstr = traceback.format_exc()
	print(exstr)
```



import traceback
import sys
try:    

​	a = 1
except:    

​	traceback.print_exc()    

​        sys.exc_info() 



## 使用预定义清理语句

```
with open("myfile.txt") as f:
    for line in f:
        print(line)
```

## 异常使用参考

http://www.pythondoc.com/pythontutorial3/errors.html