---  
layout: post  
title:  "Porcupine Testing linearizability注解"  
date:   2023-06-14 00:00:00 +0530  
---  
  
## Testing linearizability  
Porcupine测试基本方法参考: [Testing linearizability](https://github.com/anishathalye/porcupine#testing-linearizability)  
本文为原文的一个注解  
  
我们对一个寄存器state进行读写，初始值为0. 对其按照如下进行定义:  
写: state为当前的状态值，input为写入的新值。写完成后，state更新为新值  
读: state为当前的状态值，output为读取的值。读取的值应等于当前的值  
  
```go  
type registerInput struct {  
    op    bool // false = put, true = get  
    value int  
}  
  
// a sequential specification of a register  
registerModel := porcupine.Model{  
    Init: func() interface{} {  
        return 0  
    },  
    // step function: takes a state, input, and output, and returns whether it  
    // was a legal operation, along with a new state  
    // state: 当前的值; input: 此次写操作的新值; output: 此次读操作读取的值  
    Step: func(state, input, output interface{}) (bool, interface{}) {  
        regInput := input.(registerInput)  
        if regInput.op == false {  
            return true, regInput.value // always ok to execute a put  
        } else {  
            readCorrectValue := output == state  
            return readCorrectValue, state // state is unchanged  
        }  
    },  
}  
```  
  
假设我们发起了3个并发的请求。`|`表示请求的起止时间.  
  
```  
C0:  |-------- put('100') --------|  
C1:     |--- get() -> '100' ---|  
C2:        |- get() -> '0' -|  
```  
  
得到如下的执行历史:  
  
```go  
events := []porcupine.Event{  
    // C0: put('100')  
    {Kind: porcupine.CallEvent, Value: registerInput{false, 100}, Id: 0, ClientId: 0},  
    // C1: get()  
    {Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 1, ClientId: 1},  
    // C2: get()  
    {Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 2, ClientId: 2},  
    // C2: Completed get() -> '0'  
    {Kind: porcupine.ReturnEvent, Value: 0, Id: 2, ClientId: 2},  
    // C1: Completed get() -> '100'  
    {Kind: porcupine.ReturnEvent, Value: 100, Id: 1, ClientId: 1},  
    // C0: Completed put('100')  
    {Kind: porcupine.ReturnEvent, Value: 0, Id: 0, ClientId: 0},  
}  
```  
  
调用CheckEvents()进行验证  
  
```go  
ok := porcupine.CheckEvents(registerModel, events)  
// returns true  
```  
  
如上执行历史共有六种执行序列，其中一个执行序列满足一致性要求:  
C2:get(0) -> C0:put(100) -> C1:get(100)  
  
Step:  
  func(0, read, ignore, 0)     返回(true, 0)  
  func(0, write, 100, ignore)  返回(true, 100)  
  func(100, read, ignore, 100) 返回(true, 100)  
  
  
另外一个执行历史:  
  
```  
C0:  |---------------- put('200') ----------------|  
C1:    |- get() -> '200' -|  
C2:                           |- get() -> '0' -|  
```  
  
```go  
events := []porcupine.Event{  
    // C0: put('200')  
    {Kind: porcupine.CallEvent, Value: registerInput{false, 200}, Id: 0, ClientId: 0},  
    // C1: get()  
    {Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 1, ClientId: 1},  
    // C1: Completed get() -> '200'  
    {Kind: porcupine.ReturnEvent, Value: 200, Id: 1, ClientId: 1},  
    // C2: get()  
    {Kind: porcupine.CallEvent, Value: registerInput{true, 0}, Id: 2, ClientId: 2},  
    // C2: Completed get() -> '0'  
    {Kind: porcupine.ReturnEvent, Value: 0, Id: 2, ClientId: 2},  
    // C0: Completed put('200')  
    {Kind: porcupine.ReturnEvent, Value: 0, Id: 0, ClientId: 0},  
}  
```  
  
如上执行历史存在如下三种执行序序列，三种执行序列均不满足线性一致性:  
C0:put(200) -> C1:get(200) -> C2:get(0)  
Step:  
  func(0, write, 200, ignore)  返回(true, 200)  
  func(200, read, ignore, 200) 返回(true, 200)  
  func(200, read, ignore, 100) 返回(false, 200)  // 不满足  
  
C1:get(200) -> C0:put(200) -> C2:get(0)  
Step:  
  func(0, read, ignore, 200)     返回(false, 0)  // 不满足  
  
C1:get(200) -> C2:get(0) -> C0:put(200)  
Step:  
  func(0, read, ignore, 200)     返回(false, 0)  // 不满足  
