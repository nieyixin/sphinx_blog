# Subprocess踩坑记录
记录一个subprocess的踩坑记录，原因在于没有仔细看官方文档，踩了一些坑，导致在某些场景下会出现bug

## 背景
在业务中，使用subprocess去执行`ps -ef | grep xxx`命令，去查看是否存在某个进程，在python代码中是这样写的
```
process = 'xxx'
ps = subprocess.Popen(['/usr/bin/ps', 'ef'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
grep = subprocess.Popen(['grep', process], stdin=ps.stdout, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
result = str(grep.stdout.read(), encoding='utf-8')
if process in result:
    return True
return False
```
这里使用了`subprocess.Popen`方法，将ps命令的输出直接作为输入传给grep命令，最后再判断grep命令的输出是否包含我们想要的进程

## 原理
`subprocess.Popen`在执行命令时，会fork一个子进程执行命令，主进程与子进程之间通过管道进行通信。当命令执行完成后，会将子进程销毁。此时可以获取到命令的stdout与stderr

可以通过`subprocess.poll`方法来判断执行是否完成，没结束返回None，结束后返回状态码；也可以通过`subprocess.wait(timeout)`来等待子进程结束

还可以通过`subprocess.communicate(timeout)`方法来获取返回结果，该方法会等子进程终止

## 踩坑

这里踩了两个坑，主要是不熟悉subprocess的用法导致的

### 未使用try except语句
`subprocess.Popen`方法可能是有异常的，例如你传入错误的命令，subprocess找不到该命令，会抛出`ValueError`，这里没有用try except语句，可能会导致程序异常而未捕获

### 未等待子进程终止
上文ps与grep会分别fork一个进程执行对应命令，而此处代码并未等待子进程终止就去获取结果   

这里可以使用`subprocess.communicate(timeout)`方法去安全的获取输出。该方法会在timeout时间内等待子进程终止，并获取输出   

如果进程在timeout秒后未终止，一个`TimeoutExpired`异常将被抛出。捕获此异常并重新等待将不会丢失任何输出

针对上述问题对代码进行修改
```
process = 'xxx'
try:
    ps = subprocess.Popen(['/usr/bin/ps', 'ef'], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    grep = subprocess.Popen(['grep', process], stdin=ps.stdout, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    ps.stdout.close() # Allow ps to receive a SIGPIPE if grep exits.
    stdout, stderr = grep.communicate(timeout=2)
    if stderr:
        return False
    result = str(stdout, encoding='utf-8')
    if process in result:
        return True
    return False
except:
    return False
```

主要有以下几点修改：
1. 使用try except从句捕获异常
2. 使用communicate替代直接使用stdout.read()，保证子进程都已终止
3. 启动grep之后再执行ps.stdout.close()，这能确保grep先于ps退出时，ps能够收到`SIGPIPE`信号，这个信号通常在进程间通信时产生，当管道破裂时，写进程会收到该信号。这说明当grep退出时，管道已经破裂，此时ps作为写进程就会收到该信号，不再写数据