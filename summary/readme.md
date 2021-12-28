# 剖析 b-validate 实现原理

### 一、前言

&emsp;&emsp;b-validate 是一个小巧且功能丰富的类型校验库，它的特点如下：

- 链式调用
- 支持自定义校验规则
- 支持 schema 校验
- 支持异步校验

&emsp;&emsp;它基础的使用方式如下：

```JavaScript
bv(123)
  .number
  .min(2)
  .max(10)
  .collect((error) => {
    console.log(error); // { value: 123, type: 'number', message: '123 is not less than 10' }
    // if no error, error equal to null
  });

const error = bv('b-validate').string.isRequired.match(/validater/).end;
// { value: 'b-validate', type: 'string', message: '`b-validate` is not match pattern /validater/' }
```

### 二、Base Class

&emsp;&emsp;b-validate 所有数据结构的校验类是基于 Base Class 来扩展：

```JavaScript
class Base {
  constructor(obj, options) {
    if (isObject(options) && isString(obj) && options.trim) {
      this.obj = obj.trim();
    } else if (isObject(options) && options.ignoreEmptyString && obj === "") {
      this.obj = undefined;
    } else {
      this.obj = obj;
    }
    // 自定义校验提示信息
    this.message = options.message;
    this.type = options.type;
    // 存储校验结果
    this.error = null;
  }

  get end() {
    return this.error;
  }

  addError(message) {
    if (!this.error && message) {
      this.error = {
        value: this.obj,
        type: this.type,
        message: this.message || `${this._not ? "[NOT MODE]:" : ""}${message}`,
      };
    }
  }

  validate(expression, errorMessage) {
    const _expression = this._not ? expression : !expression;
    if (_expression) {
      this.addError(errorMessage);
    }
    return this;
  }

  collect(callback) {
    callback && callback(this.error);
  }
}
```

&emsp;&emsp;校验流程非常的容易理解，就是**通过调用各种规则输出最终的 this.error 信息**。

```JavaScript
// 省略部分代码
class ArrayValidater extends Base {
  constructor(obj, options) {
    super(obj, {
      ...options,
      type: 'array'
    });
    this.validate(
      options && options.strict ? isArray(this.obj) : true,
      `Expect array type but got \`${this.obj}\``
    );
  }

  length(num) {
    return this.obj ? this.validate(
      this.obj.length === num,
      `Expect array length ${num} but got ${this.obj.length}`
    ) : this;
  }
  
  minLength(num) {
    return this.obj ? this.validate(
      this.obj.length >= num,
      `Expect min array length ${num} but got ${this.obj.length}`
    ) : this;
  }
}
```

&emsp;&emsp;针对于特定数据结构，在子类的构造函数中首先会校验其类型，然后就是扩展各种特定的校验规则方法。

### 三、链式调用

&emsp;&emsp;b-validate 采用了最通常的方式实现链式调用：**基于 this 作用域**。

```JavaScript
// 省略部分代码
class Base {
  constructor(obj, options) {
  }

  get not() {
    this._not = !this._not;
    // 返回当前实例，以提供链式调用
    return this;
  }

  get isRequired() {
    if (isEmptyValue(this.obj) || isEmptyArray(this.obj)) {
      this.error = {
        value: this.obj,
        type: this.type,
        requiredError: true,
        message:
          this.message ||
          `${this._not ? "[NOT MODE]:" : ""}${this.type} is required`,
      };
    }
    // 返回当前实例，以提供链式调用
    return this;
  }
}
```