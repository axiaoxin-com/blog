---
title: "在 JavaScript 中循环访问对象——如何在 JS 中遍历对象"
author: "axiaoxin"
date: 2023-01-12T14:20:07+08:00
subtitle: ""
image: ""
tags: ["javascript"]
slug: ""
---

在 JavaScript 中，当您听到“循环”这个词时，您可能会想到使用各种循环方法，例如 `for` 循环、`forEach()`、`map()` 等。

不幸的是，对于对象，这些方法不起作用，因为对象是不可迭代的。

这并不意味着我们不能循环遍历一个对象，但这意味着我们无法像对数组那样直接循环遍历：

```javascript
let arr = [24, 33, 77];
arr.forEach((val) => console.log(val)); // ✅✅✅

for (val of arr) {
  console.log(val); // ✅✅✅
}

let obj = { age: 12, name: "John Doe" };
obj.forEach((val) => console.log(val)); // ❌❌❌

for (val of obj) {
  console.log(val); // ❌❌❌
}
```

在本文中，您将了解如何在 JavaScript 中循环遍历对象。您可以使用两种方法，其中一种早于 ES6 的引入。

## 如何使用 `for...in` 循环遍历 JavaScript 中的对象

在 ES6 之前，每当我们想要循环遍历一个对象时，我们都依赖于 `for...in` 方法。

`for...in` 循环遍历原型链（prototype chain）中的属性。这意味着每当我们使用 `for...in` 循环遍历一个对象时，我们都需要使用 `hasOwnProperty` 检查该属性是否属于该对象：

```javascript
const population = {
  male: 4,
  female: 93,
  others: 10
};

// 遍历该对象
for (const key in population) {
  if (population.hasOwnProperty(key)) {
    console.log(`${key}: ${population[key]}`);
  }
}
```

为了避免循环的压力和困难以及使用 hasOwnProperty 方法，ES6 和 ES8 引入了对象静态方法。它们将对象属性转换为数组，允许我们直接使用数组方法。

##  如何使用对象静态方法在JavaScript中循环遍历对象

对象由具有键值对的属性组成，即每个属性始终有一个对应的值。

通过对象静态方法，我们可以提取 `keys()`、`values()` 或同时提取键和值的 `entries()`，这样我们就可以像处理实际数组一样灵活地处理它们。

我们有三个对象静态方法，它们是：

- `Object.keys()`
- `Object.values()`
- `Object.entries()`


### 如何使用 `Object.keys()` 方法循环遍历 JavaScript 中的对象

ES6 中引入了 `Object.keys()` 方法。它将我们要循环的对象作为参数，返回一个包含所有属性名（键）的数组。

```javascript
const population = {
  male: 4,
  female: 93,
  others: 10
};

let genders = Object.keys(population);

console.log(genders); // ["male","female","others"]
```

现在，我们可以应用任何数组循环方法来遍历数组并检索每个属性的值：

```javascript
let genders = Object.keys(population);

genders.forEach((gender) => console.log(gender));
```

这将返回：

```text
"male"
"female"
"others"
```

我们还可以使用括号符号来使用键来获取值，例如 `population[gender]` 如下所示：

```javascript
genders.forEach((gender) => {
  console.log(`There are ${population[gender]} ${gender}`);
})
```

这将返回：

```text
"There are 4 male"
"There are 93 female"
"There are 10 others"
```

在我们继续之前，让我们使用此方法通过循环计算所有人口的总和，从而了解总人口：

```javascript
const population = {
  male: 4,
  female: 93,
  others: 10
};

let totalPopulation = 0;
let genders = Object.keys(population);

genders.forEach((gender) => {
  totalPopulation += population[gender];
});

console.log(totalPopulation); // 107
```

### 如何使用 `Object.values()` 方法遍历 JavaScript 中的对象

`Object.values()` 方法与 `Object.keys()` 方法非常相似，是在 ES8 中引入的。此方法将我们要循环的对象作为参数，返回一个包含所有值的数组。

```javascript
const population = {
  male: 4,
  female: 93,
  others: 10
};

let numbers = Object.values(population);

console.log(numbers); // [4,93,10]
```

现在，我们可以应用任何数组循环方法来遍历数组并检索每个属性的值：

```javascript
let numbers = Object.values(population);

numbers.forEach((number) => console.log(number));
```

这将返回：

```text
4
93
10
```

由于我们可以直接循环，因此可以高效地执行总计算：

```javascript
let totalPopulation = 0;
let numbers = Object.values(population);

numbers.forEach((number) => {
  totalPopulation += number;
});

console.log(totalPopulation); // 107
```

### 如何使用 `Object.entries()` 方法循环遍历 JavaScript 中的对象

ES8 还引入了 `Object.entries()` 方法。从基本意义上讲，它所做的是输出一个数组的数组，其中每个内部数组都有两个元素，即键和值。

```javascript
const population = {
  male: 4,
  female: 93,
  others: 10
};

let populationArr = Object.entries(population);

console.log(populationArr);
```

这将输出：

```javascript
[
  ['male', 4]
  ['female', 93]
  ['others', 10]
]
```

返回一个数组的数组，每个内部数组都有 `[key, value]`。您可以使用任何数组方法来循环遍历：

```javascript
for (array of populationArr){
  console.log(array);
}

// Output:
// ['male', 4]
// ['female', 93]
// ['others', 10]

```

我们可以解构数组，这样我们就得到了键和值：

```javascript
for ([key, value] of populationArr){
  console.log(key);
}
```

## 总结

在本文中，您了解到循环访问对象的最佳方法是根据你的需要使用对象静态方法，在循环之前首先将对象转换为数组。。

祝您编码愉快！
