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

### 四、schema 实现

&emsp;&emsp;schema 与之前调用方式的不同点在于：

- 批量处理类型校验
- 通过 schema 约定减少手动链式调用的繁琐

```JavaScript
export class Schema {
  constructor(schema, options = {}) {
    this.schema = schema;
    this.options = options;
  }

  validate(values, callback) {
    if (!isObject(values)) {
      return;
    }
    const promises = [];
    let errors = null;
    // 根据 key 设置相应的 error 信息
    function setError(key, error) {
      if (!errors) {
        errors = {};
      }
      if (!errors[key] || error.requiredError) {
        errors[key] = error;
      }
    }
    if (this.schema) {
      Object.keys(this.schema).forEach((key) => {
        if (isArray(this.schema[key])) {
          for (let i = 0; i < this.schema[key].length; i++) {
            const rule = this.schema[key][i];
            const type = rule.type;
            const message = rule.message;
            // 省略部分代码
          }
        }
      });
    }
    // 省略部分代码
    callback && callback(errors);
  }
}
```

&emsp;&emsp;在有 schema 约定的前提下，内部执行代码可以自动遍历每一条规则，通过 collect 方法获取到相应的 this.error 信息，最终调用 setError 方法整合成数组输出。

### 五、总结

&emsp;&emsp;本文重点如下：

- 了解类型校验的核心流程
- 基于继承来扩展不同数据结构的校验规则类
- 基于 this 作用域实现链式调用
- 通过 schema 的设计优化批量数据校验场景下的校验流程

&emsp;&emsp;对于数据类型校验库感兴趣的同学，千万不要错过 **joi** 这个库。

&emsp;&emsp;最后，**如果本文对您有帮助，欢迎点赞、收藏、分享**。