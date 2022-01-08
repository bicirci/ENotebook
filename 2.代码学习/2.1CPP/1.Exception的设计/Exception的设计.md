
# Exception的设计  

## 1. 背景  

Exception是为了简化C语言的错误处理，不用一轮一轮的往上传递错误码，而是提供公共的exception封装，使得上层应用可以直接读取到底层错误发生的原因。  

## 2. 基本原则  

对于一个程序，存在正常退出，和exception退出两种方式。编程时需要保证这两种方式退出时都要完成对资源的释放。常用的推荐处理方式是用shared_ptr或者RAII。  

## 3. 三种Exception的安全等级  

1. No Fail Guanartee  
No Fail 是指function绝对不会抛出exception。只有在：  
    * function内的包含的其他function都是No Fail的  
    * function内包含的其他function已经各自处理好对应的exception（和上一条有重叠）  
    * 该function内部可以处理所有的Exception类型  

2. Strong Guarantee  
Strong Guarantee 保证如果程序是因exception结束，那么该function内部造成的一切对state的修改都将回退成原状。也就是要么成功，要么失败但对运行没有丝毫影响。  

3. Basic/Weak Guarantee  
仅保证如果出现exception，那么系统不会出现资源泄漏。  

## 4. 类的常规处理  

1. 尽量使用smart_ptr和RAII来进行资源的分配。要清楚如果在constructor执行期间，如果发生了exception，那么deconstructor是不会被呼叫的。  

2. 在base的constructor产生的exception不会像函数一样再抛给derived class， 而是直接抛到调用函数去了。  

3. 保证deconstructor是No fail的，必须禁止向上抛出exception。（如果是上层class在deconstruct途中，当前class的deconstruct抛出了异常，有可能会造成上层class后续的资源释放受到影响）  
