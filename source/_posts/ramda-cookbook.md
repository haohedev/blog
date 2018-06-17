---
title: ramda-cookbook
date: 2018-06-15 12:57:06
tags: 
    - JavaScript
categories: FunctionalPrograming
---

## 背景
在学习`函数式编程`的时候，无意中看到了[JS 函数式编程指南](https://legacy.gitbook.com/book/llh911001/mostly-adequate-guide-chinese/details)，使得对`函数式编程`的概念有了初步的认识。恰巧当前工作的重心在`node`这块开发，在学习`ES6`语法的同时，也强迫自己用`函数式编程`的思路去写代码。  
工欲善其事必先利其器，选择一个优秀的库，就等于迈向了成功的大门。在看了很多评价`lodash`、`underscore`、`ramda`的文章之后，感觉被`ramda`的`pointfree`的风格吸引住了，很庆幸，有[先驱者](https://adispring.coding.me/)为`ramda`库的一系列文章做了翻译，使得更容易理解`ramda`的设计初衷与使用姿势。正是因为这一些列的文章，让我对`函数式编程`有了更高层的认识，犹如醍醐灌顶、豁然开朗。  
虽然没有选择`lodash`和`underscore`, 但一有时间我一定会去阅读他们的文档，理解设计理念。同时也建议像我一样初学`JavaScript`的同学，多利用现有开源库会让你事半功倍，写出更健壮的代码。

## 初衷
在工作中因为经常用到`ramda`库，会对常用的一些方法进行封装。在看到[ramda cookbook](https://github.com/ramda/ramda/wiki/Cookbook)的时候，犹如发现了宝藏一般，里面的内容覆盖了大部分常用的函数。通过看其实现的代码，对`ramda`的一些函数有了更深的理解。  

虽然[ramda cookbook](https://github.com/ramda/ramda/wiki/Cookbook)中的函数比较全，但有时仍不免要自己实现一些特定需求的函数，因此才有了这篇文章。  
> 本篇文章会随着需求的增加不定时更新。  



<!-- more -->

## 实现


### MapKeys
`cookbook`中有`mapkeys`的实现，但不支持嵌套的遍历，因此按照递归的思路，实现了如下一个版本:  

```javascript
const mapKeys = R.curry((fn, obj) => {
    const go = obj_ => R.chain(([k, v]) => {
        const recursionWhenValueWasObject = (value) =>
            R.equals('Object', R.type(value)) ? R.fromPairs(R.map(R.identity, go(value))) : value;
        return [[R.call(fn, k), recursionWhenValueWasObject(v)]]
    }, R.toPairs(obj_))
    return R.fromPairs(go(obj));
});
```

用例：
```javascript
mapKeys(R.toUpper, { a: 1, b: 2, c: { d: 3} });
//=> { A: 1, B: 2, C: { D: 3 } }
```


### renameKeys
理由同上， 为了支持遍历

```javascript
const renameKeys = R.curry((keysMap, obj) =>{
    let rename = key => R.defaultTo(key, R.prop(key, keysMap));
    return mapKeys(rename, obj);
});
```

用例：
```javascript
const input = { name: { lastName: 'Elisia' }, age: 22, type: 'human' }
renameKeys({ lastName: 'firstName', type: 'kind', foo: 'bar' })(input)
//=> { name: { firstName: 'Elisia' }, age: 22, kind: 'human' }
```


### flattenObj
原实现遇到数组的时候，不会展开数组

```javascript
const flattenObj = obj => {
    const go = obj_ => R.chain(([k, v]) => {
        if (R.contains(R.type(v), ["Object", "Array"])){
            return R.map(([k_, v_]) => [`${k}.${k_}`, v_], go(v))
        } else {
            return [[k, v]]
        }
    }, R.toPairs(obj_))

    return R.fromPairs(go(obj))
}
```

用例：
```javascript
flattenObj({a:1, b:{c:3}, d:{e:{f:6}, g:[{h:8, i:9}, 0]}})
//=> {"a": 1, "b.c": 3, "d.e.f": 6, "d.g.0.h": 8, "d.g.0.i": 9, "d.g.1": 0}
```  

---
> **本文作者：** 郝赫   
> **本文链接：** https://zyzz.xyz/ramda-cookbook/   
> **版权声明：** 本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 进行许可。转载请注明出处！  
> ![license](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png)