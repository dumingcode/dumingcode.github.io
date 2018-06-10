---
title: iview使用async-validator校验数字范围
tags: [nodejs]
---
# iview使用async-validator校验数字范围
## 自定义验证
在实际开发中使用`iview input`组件被要求实现一个功能：校验输入的数字小于xx或者大于xxx，看了一下`async-validator API`，自定义实现的validator能够通过rule对象的options变量传入参数。自定义校验函数如下：
```
//验证输入的数字比min小
export const validateNumberMin = (rule, value, callback) => {
        let val = Number(value)
        if (val != '' && rule.options.min != '' && (val < rule.options.min)) {
            return callback(new Error('number'))
        } else {
            callback()
        }
    }
    //验证输入的数字比max大
export const validateNumberMax = (rule, value, callback) => {
    let val = Number(value)
    if (val != '' && rule.options.max != '' && (val > rule.options.max)) {
        return callback(new Error('number'))
    } else {
        callback()
    }
}
```
实际调用validator函数：
```
{
            validator: validateNumberMin,
            message: "应大于等于100",
            trigger: "blur",
            options: { min: 100 }
}
{
            validator: validateNumberMax,
            message: "应小于等于200",
            trigger: "blur",
            options: { max: 100 }
}
```

注意：options实际上是rule对象中的一个变量。