---
title: async-await 陷阱
date: 2018-06-15 09:54:10
tags:
categories: JavaScript
---

> 在使用`async / await`时可能会犯什么错误？这里总结了一些常见的问题。

## 过于串行化
虽然`await`可以让你的代码看起来像同步，但请记住它们仍然是异步的，必须小心避免过于串行化。

``` javascript
async getBooksAndAuthor(authorId) {
  const books = await bookModel.fetchAll();
  const author = await authorModel.fetch(authorId);
  return {
    author,
    books: books.filter(book => book.authorId === authorId),
  };
}
```
上面的代码虽然在逻辑上看起来正确。但还是存在问题的。

<!-- more -->

1. `await bookModel.fetchAll()` 会一直等待直到 `fetchAll()` 返回.
2. 然后`await authorModel.fetch(authorId)` 才会被调用.

值得注意的是`authorModel.fetch(authorId)`实际上并不依赖于`bookModel.fetchAll()`返回的结果，他们完全可以并行执行！然而，在这里使用await，使得两个调用变为顺序执行，并且总执行时间将比并行版本长得多。
  
  
正确的做法应该是：

``` javascript
async getBooksAndAuthor(authorId) {
  const bookPromise = bookModel.fetchAll();
  const authorPromise = authorModel.fetch(authorId);
  const book = await bookPromise;
  const author = await authorPromise;
  return {
    author,
    books: books.filter(book => book.authorId === authorId),
  };
}
```

如果你想要一个一个地提取物品清单，你必须依靠`promises`

``` javascript
async getAuthors(authorIds) {
  // 错误的做法， 这会导致串行化调用
  // const authors = _.map(
  //   authorIds,
  //   id => await authorModel.fetch(id));
// 正确的做法：
  const promises = _.map(authorIds, id => authorModel.fetch(id));
  const authors = await Promise.all(promises);
}
```

总而言之，你需要异步考虑工作流程，然后利用await进行同步编写代码。在复杂的工作流程中，直接使用`promise`可能更容易。
