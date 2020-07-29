---
title: 浅析 async-validator 源码
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd,validator'
abbrlink: ed4731db
date: 2018-01-24 06:00:00
updated: 2018-01-24 06:00:00
---

在async-validator源码中，src/rule文件夹下各代码文件实现了最基础的数据校验能力，因此也可以被称为基础校验规则。

类型校验
要分析async-validator模块的实现，容笔者先从类型校验说起。

在async-validator模块中，单个校验器被定义为validator函数。该validator函数接受rule、value、callback、source、options作为参数。source、options这两个参数姑且按下不论。参数rule是以对象形式配置的校验规则，value是待校验的数据，callback是由开发者手动执行的回调函数。

容易猜想的是，在实现类型校验的过程中，async-validator模块采用了{ type: ‘array’ }的形式配置rule。其中type可接受的值包含’integer’, ‘float’, ‘array’, ‘regexp’, ‘object’, ‘method’, ‘email’, ‘number’, ‘date’, ‘url’, ‘hex’这几种类型。在其背后的实现机制中，async-validator模块没有采用switch语句在单一的函数中完成类型判断，而是通过types对象设定每种类型的校验方法，接着通过调用typestype方法实现类型校验，这样能使代码的结构变得清晰。

```javascript
// src/rule/type.js
const types = {
  integer(value) {
    return types.number(value) && parseInt(value, 10) === value;
  },
  float(value) {
    return types.number(value) && !types.integer(value);
  },
  array(value) {
    return Array.isArray(value);
  },
  regexp(value) {
    if (value instanceof RegExp) {
      return true;
    }
    try {
      return !!new RegExp(value);
    } catch (e) {
      return false;
    }
  },
  date(value) {
    return typeof value.getTime === 'function' &&
      typeof value.getMonth === 'function' &&
      typeof value.getYear === 'function';
  },
  number(value) {
    if (isNaN(value)) {
      return false;
    }
    return typeof (value) === 'number';
  },
  object(value) {
    return typeof (value) === 'object' && !types.array(value);
  },
  method(value) {
    return typeof (value) === 'function';
  },
  email(value) {
    return typeof (value) === 'string' && !!value.match(pattern.email) && value.length < 255;
  },
  url(value) {
    return typeof (value) === 'string' && !!value.match(pattern.url);
  },
  hex(value) {
    return typeof (value) === 'string' && !!value.match(pattern.hex);
  },
};
```

从上面的代码中，可以发现types并不包含’boolean’, ‘string’，那么是不是async-validator模块遗漏了这两种数据类型呢？不是的。async-validator通过if-else if语句校验了这两种数据类型（没有else语句，意味着不满足条件的视为校验通过）。

```javascript
// src/rule/type.js
const custom = ['integer', 'float', 'array', 'regexp', 'object',
  'method', 'email', 'number', 'date', 'url', 'hex'];
const ruleType = rule.type;
if (custom.indexOf(ruleType) > -1) {
  if (!types[ruleType](value)) {
    errors.push(util.format(options.messages.types[ruleType], rule.fullField, rule.type));
  }
  // straight typeof check
} else if (ruleType && typeof (value) !== rule.type) {
  errors.push(util.format(options.messages.types[ruleType], rule.fullField, rule.type));
}
```

奇怪的是，在async-validator模块中，既然type的种类包含了’email’(邮箱), ‘url’(网址)，为什么没有包含’phone’(手机号)呢？或者，为什么async-validator模块不对开发者开放注册type的方法呢？那样开发者可以在types对象中添加方法，使得类型校验更为多样，不是吗？实际上，在async-validator模块的设计中，这个type只是单条校验规则，还不是校验器。作为不可变动的底层，它没有提供向types对象注入新类型的方法。关于可注册的校验器类型，将在下文予以分析。

### 其他基本校验

在async-validator模块中，其他基本校验包含’required’(非空校验), ‘whitespace’(空白字符校验), ‘range’(范围校验), ‘enum’(枚举校验), ‘pattern’(正则校验)。’required’, ‘whitespace’校验自不必多说，笔者将着重介绍’range’, ‘enum’, ‘pattern’校验。

关于’range’校验，其能力包含校验数值的大小、字符串或数组的长度，因此rule对象相应添加了min、max、len属性，其中len属性的优先级高于min、max属性。在async-validator源码的实现中，通过先将字符串和数组形式的value转换成待校验的value.length，再通过if语句协调len与min、max校验规则的优先级。具体可参详src/rule/range.js文件的源码。

关于’enum’校验，其能力为校验数据value是否某个枚举数组的一份子，相应rule对象添加了enum属性。若value在rule.enum中，则校验通过；否则校验失败。

关于’pattern’校验，其能力为校验数据value是否匹配正则字符串或正则表达式，相应rule对象添加了pattern属性。该pattern属性可配置为正则字符串或正则表达式。

介绍完了前述校验规则，笔者再度提问，为什么’range’校验中并不包含日期对象的起止时间校验，’enum’校验也不包含数组value尽在rule.enum配置数据中的校验？若说前一条会使’range’校验的实现过于复杂，后一条针对checkbox、select元素却较为常见。实际上，日期对象的range校验在date校验器中实现，笔者将在下文予以描述。

### 校验器

#### 内置校验器

在async-validator源码中，src/validator文件夹下各代码文件将基础校验规则整合成单个校验器（字面上是单个校验器，实际是多个基础校验规则构成的组合校验器）。下面将予以简单的介绍，源码不再赘述。

array校验器校验rule为{ type: ‘array’, required: true, min: 1 }的数据value非空、且为数组类型、且长度匹配校验条件。required为false时，跳过非空校验，后同。

boolean校验器校验rule为{ type: ‘boolean’, required: true }的数据value非空、且为布尔类型。

date校验器校验rule为{ type: ‘date’, required: true, min: new Date(‘2017 12 10’).getTime() }的数据value非空、且为date日期对象，且该日期对象匹配起止时间校验。在date校验器的实现中，通过value.getTime()方法将其转化为毫秒数，然后再作range基础校验。

enum校验器校验rule为{ type: ‘enum’, required: true, enum: [‘male’, ‘female’] }的数据value非空，且其值为rule.enum中的一个。

float校验器校验rule为{ type: ‘float’, required: true, min: 1 }的数据value非空、且为浮点型数值、且长度匹配校验条件。

integer校验器校验rule为{ type: ‘integer’, required: true, min: 1 }的数据value非空、且为整型数值、且长度匹配校验条件。

method校验器校验rule为{ type: ‘method’, required: true }的数据value非空、且为函数。

number校验器校验rule为{ type: ‘number’, required: true, min: 1 }的数据value非空、且为数值、且长度匹配校验条件。

object校验器校验rule为{ type: ‘object’, required: true }的数据value非空、且为对象。

pattern校验器校验rule为{ pattern: /\s/, required: true }的数据value非空、且匹配正则表达式或正则字符串。

regexp校验器校验rule为{ type: ‘regexp’, required: true }的数据value非空、且为正则表达式或正则字符串。

required校验器校验rule为{ required: true }的数据value非空。

string校验器校验rule为{ type: ‘string’, pattern: /\s/, required: true, min: 1, whitespace: true }的数据value非空、且为字符串、且其长度匹配校验条件、且匹配正则表达式或正则字符串、且不是空字符串。

type校验器校验rule为{ type: ‘email | url | hex’, required: true }的数据value非空、且是’email’或’url’或’hex’中的一种。

#### 自定义校验

前述内置校验器无法满足丰富的业务需求，为此，在async-validator模块的实现中，可以通过配置rule = { validator: function(rule, value, callback, source, options){} }或rule = function(rule, value, callback, source, options){}的方式设置自定义校验器。

上文未提及到的是，错误文案可作为callback回调函数的参数传入。并且，该callback函数必须得到调用，其中的因由，将在下文予以描述。

与此同时，在自定义校验器函数中，可以通过首参获得开发者配置的rule对象，或者说，开发者可以通过rule向自定义校验器传入额外的参数。比如，在async-validator模块结合react使用的场景中，笔者在组件内部定义了一个validateId的方法，可以通过配置rule = { vaidator: this.validateId, ctx: this }的校验规则，从而在validateId方法的内部，通过rule.ctx就可以获得该组件实例。虽然，当需要访问组件实例时，常规的配置方式是{ validator: this.validateId.bind(this) }。这个例子旨在说明，跟随外部环境变动的数据可以通过rule对象传入自定义校验器函数内部，不必通过bind方法实现。

假使多个字段共用同一个自定义校验器时，要怎样才能区分当前的校验字段是哪一个呢？在async-validator源码的实现中，开发者配置的rule数据会添加field、fullField属性，其中field就是当前校验的字段名。fullField是当前校验字段的全名，这在深度校验的时候用到。

值得说明的是，参数source是整个待校验的数据，value也是source的一个属性。这样，async-validator模块就为开发者提供了关联校验的能力。假使有字段min、max需要校验，对min字段，需要校验其数值不大于max字段的值，开发者就可以通过source[max]属性获取到max字段的值，从而实现关联校验。

关于参数options，笔者将在后文予以分析。

#### 注册校验器

前文已经提到，async-validator模块并未提供注入基础校验规则的方法，相应的，它提供了注册单个校验器的能力。通过调用AsyncValidator.register(type, validator)静态方法，开发者即可以注册一个类型为type的validator校验器。

```javascript
// src/index.js
Schema.property = {
  //...
  getType(rule) {
    if (rule.type === undefined && (rule.pattern instanceof RegExp)) {
      rule.type = 'pattern';
    }
    if (typeof (rule.validator) !== 'function' &&
      (rule.type && !validators.hasOwnProperty(rule.type))) {
      throw new Error(format('Unknown rule type %s', rule.type));
    }
    return rule.type || 'string';
  },
  getValidationMethod(rule) {
    if (typeof rule.validator === 'function') {
      return rule.validator;
    }
    const keys = Object.keys(rule);
    const messageIndex = keys.indexOf('message');
    if (messageIndex !== -1) {
      keys.splice(messageIndex, 1);
    }
    if (keys.length === 1 && keys[0] === 'required') {
      return validators.required;
    }
    return validators[this.getType(rule)] || false;
  },
};

Schema.register = function register(type, validator) {
  if (typeof validator !== 'function') {
    throw new Error('Cannot register a validator by type, validator is not a function');
  }
  validators[type] = validator;
};
```

通过源码，我们也可以看到该校验器和内置校验器如array、boolean校验器平级。

因此我们也可以实现一个type类型为’remote’远程校验器。

```javascript
function remoteValidator = (rule, value, callback, source, options){
  const { field, action, queryData = {}, format } = rule;
  if ( !action ){
    console.warn('action is required');
    return;
  };

  let data = {
    ...queryData,
    [field]: value
  };
  data = format && typeof format === 'function' ? format(data, source, options) : data;

  fetch(action,queryData).then(res => {
    if ( res.model === true ) callback();
    else callback('error');
  }); 
};
```

如上的’remote’远程校验器用到了validator函数能够访问rule对象的能力。相应的rule对象也需要{ type: ‘remote’, action: url, queryData: {} }之类的数据格式。需要说明的是，format函数也可以通过async-validator模块内置的transform配置函数实现，该transform函数只接受value作为其参数，虽然完整的待校验数据source需要由外部注入transform函数体内。在async-validator模块中，rule对象中可配置的transform属性即提供了预制校验数据转化的能力。

#### 校验组合

通常，单个校验器无法满足实际的开发需求，async-validator模块使用校验规则数组[ rule ]校验source数据的某个属性。因此在校验某个数据的时候，可以设置多个校验器如[{ type: ‘array’, required: true, min: 1 }, { validator: function(rule, value, callback){} }]。

#### 深度校验

对于复杂的对象，async-validator模块提供了深度校验的能力。当rule.type为’object’或’object’时，通过设置rule.defaultField或rule.fields属性(其值为{ key: [rule] }结构)，async-validator模块将会以rule.defaultField与rule.fields合并对象作为校验规则，并校验source内的深层属性。特别的，rule.defaultField可以用来校验数据结构相同的数组项；rule.fields只作用于某个对象属性或某个数组项。该深层属性可以是数组或对象，通过for…in获得数组项或对象属性，并作相应校验。关于这一点，笔者也将在校验流程那一段落加以描述。

### 校验流程

整体校验流程为：

1. 创建Schema实例，let schema = new Schema(descriptor)，其中descriptor即校验规则rules；
2. 调用schema.define(rules)方法，注册校验规则rules；
3. 调用schema.validate(source, options, callback)方法，校验数据source。

笔者将深入讨论第3步schema.validate方法的内部机理，即async-validator模块的校验过程。

在schema.validate方法，主要的工作流程为：

1. 参数转换。若options参数不存在，callback回调取次参。
2. 检查校验规则是否存在。若不存在，执行callback回调。
3. 设置内部回调complete函数，为callback回调传入参数errors = [{ field, message }]和参数fields = { [fieldName] }。
4. 获取校验文案，并写入options.messages。
5. 校验规则转换。将参数rules复合为series数组，其中，series数组的每一项为{ rule, value, source, field }形式。其中，value值由rule.transform(value)方法转换后获得；rule的数据格式为{ type, validator, fullField, field, … }诸如此类，rule.type通过schema.getType(rule)方法获得，rule.validator通过schema.getValidationMethod(rule)获得。
6. 调用asyncMap(series, options, singleValidator, completeCallback)函数校验数据。其中，参数singleValidator接受用于执行单个校验器，并操控下一个校验器的执行；completeCallback为最终回调函数，用于执行内部回调complete函数。singleValidator函数接受参数data和doIt，data即遍历着的series数组项；doIt函数等同koa模块的next函数，用于执行下一个校验器或者最终回调。且，若校验为平行校验，doIt函数负责传递参数错误对象数组，由utils.js文件中的asyncParallelArray函数将所有校验器的错误对象数组构建成单一数组，供completeCallback回调处理。若校验为有序校验，且options.first为真值，doIt函数接收到参数错误对象数组、并交由utils.js文件中的asyncSerialArray函数处理的时候，将直接调用completeCallback回调，中断后续校验器的执行；当options.first为否值，对错误对象的处理与平行校验相同。
  * 通过rule.type、rule.fields、rule.defaultField判断是否深度校验。若是，内部变量deep置为真。
  * 定义addFullfield函数，用于获取深度校验时嵌套对象属性的fullField。
  * 定义单次校验后执行的回调函数cb。cb的实现机制中，包含将错误对象加工为[{ field, message }]数据格式；通过rule.defaultField、rule.fields构建深度校验Schema实例，并在该Schema实例的回调中启动后续校验器。
  * 执行rule.validator(rule, value, cb, source, options)作校验。若返回Promise实例，cb将在该Promise实例的then方法中执行。

制作成简易的流程图为：

![image](async-validator.png)

#### 平行、有序校验逻辑再梳理

在async-validator源码中，平行、有序校验逻辑由util.js文件中asyncMap、asyncSerialArray、asyncParallelArray这三个函数支撑，其实现原理类似于async模块。其中，asyncParallelArray函数用于实现平行校验，在某个异步校验器执行过程中，平行调用下一个校验器；asyncSerialArray用于实现有序校验，在异步校验器执行完成后，再启用下一个校验器。

关于asyncParallelArray(arr, func, callback)函数，其工作流即遍历数组arr，对数组项分别调用func函数加以处理，并通过手动执行的次参收集错误对象或调用callback回调。当遍历完成时，callback回调将被执行。

```javascript
// src/util.js
function asyncParallelArray(arr, func, callback) {
  const results = [];
  let total = 0;
  const arrLength = arr.length;

  function count(errors) {
    results.push.apply(results, errors);
    total++;
    if (total === arrLength) {
      callback(results);
    }
  }

  arr.forEach((a) => {
    func(a, count);
  });
}
```

关于asyncSerialArray(arr, func, callback)函数，其工作流是在内部构建一个next函数，通过该next函数调用func，并将arr数组项及next函数本身作为func的参数，由此有序遍历arr。

```javascript
// src/util.js
function asyncSerialArray(arr, func, callback) {
  let index = 0;
  const arrLength = arr.length;

  function next(errors) {
    if (errors && errors.length) {
      callback(errors);
      return;
    }
    const original = index;
    index = index + 1;
    if (original < arrLength) {
      func(arr[original], next);
    } else {
      callback([]);
    }
  }

  next([]);
}
```

关于asyncMap函数，首先它判断options.first是否为真值，若为真值，调用asyncSerialArray处理series数组，当某一规则校验失败时，即终止校验，执行callback回调。若options.first为否值，构建next函数包装callback，目的是将所有校验器的失败文案合二为一，在传入callback回调中；再根据options.firstFields是否为真值，分别执行asyncSerialArray、asyncParallelArray函数。

```javascript
// src/util.js
export function asyncMap(objArr, option, func, callback) {
  if (option.first) {
    const flattenArr = flattenObjArr(objArr);
    return asyncSerialArray(flattenArr, func, callback);
  }
  let firstFields = option.firstFields || [];
  if (firstFields === true) {
    firstFields = Object.keys(objArr);
  }
  const objArrKeys = Object.keys(objArr);
  const objArrLength = objArrKeys.length;
  let total = 0;
  const results = [];
  const next = (errors) => {
    results.push.apply(results, errors);
    total++;
    if (total === objArrLength) {
      callback(results);
    }
  };
  objArrKeys.forEach((key) => {
    const arr = objArr[key];
    if (firstFields.indexOf(key) !== -1) {
      asyncSerialArray(arr, func, next);
    } else {
      asyncParallelArray(arr, func, next);
    }
  });
}
```

#### 反思校验流程

首先笔者需要介绍的是怎样在自定义校验器中应用promise。代码见下。

```javascript
function covertErrToPromise(err) {
  let promise = new Promise(function(resolve, reject){
    if ( err ) reject(err);
    else resolve();
  }));

  return promise;
};

function ValidatorDecorator(validator){
  return function(rule, value, cb, source, options){
    let result = validator(rule, value, cb, source, options);
    return covertErrToPromise(result);
  };
}

@ValidatorDecorator
function customValidator(rule, value, cb, source, options){
  // ...validate 
};
```

不得不说，async-validator模块有个缺点，即需要调用者自行构建Promise实例，而不能直接使用unjq-ajax、react-resource、fetch之类模块返回的Promise实例。与此同时，返回的Promise实例将阻塞后续的校验器。

因此笔者有个疑问，为什么async-validator模块在组织异步逻辑的时候，没有采用Promise的写法？

如果async-validator模块在实现中率先判断rule.validator的返回值是否为Promise实例（通过instanceof判断是否Promise实例，或者通过thenable之类的函数判断返回值是否包含then方法），对于同步校验校验器，将其转化为Promise实例，再将cb回调挂载到该Promise的then方法上；对于异步校验器，不作任何处理。这样可以简化自定义校验器的书写流程，校验通过返回错误文案，成功返回null或不返回；异步校验可以采用async函数书写，也不需要构建Promise实例。

在性能方面，仍旧参数options的first、firstFields属性判断是否该启用有序校验；默认使用平行校验。

### 错误文案

在async-validator模块中，错误文案通过Schema实例的messages方法添加。在内置或自定义校验器中，通过options参数的messages方法获得，再由util.js文件里的format函数格式化。介于async-validator模块不对外提供format方法，对于使用者注册的type校验器，需要自行构建format函数，手动格式化。

此外，async-validator模块不提供国际化的能力。因此，在切换语言时，需要使用者自行调用Schema实例的messages方法切换加载的语言包。关于async-validator模块的设计，完全可以为Schema实例添加一个setLanguage方法，用以切换语言，随后再调用messages添加当前项目实际需要的错误文案。

```javascript
schema.messages({
  default: '字段 %s 校验失败',
  required: '%s 必填',
  enum: '%s 必须是 %s 中的一个',
  whitespace: '%s 不能为空',
  date: {
    format: '%s 日期对象 %s 无效，当转换 %s时',
    parse: '%s 日期对象不能被解析, %s 无效',
    invalid: '%s 日期对象 %s 无效',
  },
  types: {
    string: '%s 不是一个 %s',
    method: '%s 不是一个 %s (function)',
    array: '%s 不是一个 %s',
    object: '%s 不是一个 %s',
    number: '%s 不是一个 %s',
    date: '%s 不是一个 %s',
    boolean: '%s 不是一个 %s',
    integer: '%s 不是一个 %s',
    float: '%s 不是一个 %s',
    regexp: '%s 不是一个有效的 %s',
    email: '%s 不是一个有效的 %s',
    url: '%s 不是一个有效的 %s',
    hex: '%s 不是一个有效的 %s',
  },
  string: {
    len: '字段 %s 须包含 %s 个字符',
    min: '字段 %s 须大于 %s 个字符',
    max: '字段 %s 须小于 %s 个字符',
    range: '字段 %s 须小于 %s 到 %s 个字符',
  },
  number: {
    len: '字段 %s 须等于 %s',
    min: '字段 %s 须大于 %s',
    max: '字段 %s 须小于 %s',
    range: '字段 %s 须介于 %s 、 %s 之间',
  },
  array: {
    len: '字段 %s 的长度须等于 %s',
    min: '字段 %s 的长度须大于 %s',
    max: '字段 %s 的长度须小于 %s',
    range: '字段 %s 的长度须介于 %s 、 %s 之间',
  },
  pattern: {
    mismatch: '字段 %s 的值 %s 不匹配正则 %s',
  },
})
```

### 应用

#### async-validator在rc-form组件中的应用

rc-form组件是ant-design组件库中表单组件的底层实现。关于该表单组件的实现，笔者将在以后的文章中探讨。在这篇文章中，笔者只截取rc-form组件(2.1.6版本)对async-validator模块的使用。

在业务上，一则当表单元素失去焦点时需要校验该表单项，另一方面整张表单在提交时需要校验所有未隐藏的表单项。以上两点，rc-form组件都基于validateFieldsInternal方法创建新的AsyncValidator实例实现。参数action用于过滤校验条件，适用于第一种情形。参数options.force为真时，在事件过程中已被校验的表单项仍需再度校验；options.firstFields需要执行阻塞式校验的字段；其他属性同async-validator模块。

```javascript
// rc-form/src/createBaseField.js
validateFieldsInternal(fields, {
  fieldNames,
  action,
  options = {},
}, callback) {
  const allRules = {};
  const allValues = {};
  const allFields = {};
  const alreadyErrors = {};
  fields.forEach((field) => {
    const name = field.name;
    // 表单项在事件过程中已被校验，将不予再次校验
    if (options.force !== true && field.dirty === false) {
      if (field.errors) {
        set(alreadyErrors, name, { errors: field.errors });
      }
      return;
    }
    const fieldMeta = this.fieldsStore.getFieldMeta(name);
    const newField = {
      ...field,
    };
    newField.errors = undefined;
    newField.validating = true;
    newField.dirty = true;
    allRules[name] = this.getRules(fieldMeta, action);
    allValues[name] = newField.value;
    allFields[name] = newField;
  });
  this.setFields(allFields);
  // in case normalize
  Object.keys(allValues).forEach((f) => {
    allValues[f] = this.fieldsStore.getFieldValue(f);
  });
  if (callback && isEmptyObject(allFields)) {
    callback(isEmptyObject(alreadyErrors) ? null : alreadyErrors,
      this.fieldsStore.getFieldsValue(fieldNames));
    return;
  }
  const validator = new AsyncValidator(allRules);
  if (validateMessages) {
    validator.messages(validateMessages);
  }
  validator.validate(allValues, options, (errors) => {
    const errorsGroup = {
      ...alreadyErrors,
    };
    if (errors && errors.length) {
      errors.forEach((e) => {
        const fieldName = e.field;
        if (!has(errorsGroup, fieldName)) {
          set(errorsGroup, fieldName, { errors: [] });
        }
        const fieldErrors = get(errorsGroup, fieldName.concat('.errors'));
        fieldErrors.push(e);
      });
    }
    const expired = [];
    const nowAllFields = {};
    Object.keys(allRules).forEach((name) => {
      const fieldErrors = get(errorsGroup, name);
      const nowField = this.fieldsStore.getField(name);
      // avoid concurrency problems
      // 校验过程中数据变更，提醒需要再次校验
      if (nowField.value !== allValues[name]) {
        expired.push({
          name,
        });
      } else {
        nowField.errors = fieldErrors && fieldErrors.errors;
        nowField.value = allValues[name];
        nowField.validating = false;
        nowField.dirty = false;
        nowAllFields[name] = nowField;
      }
    });
    this.setFields(nowAllFields);
    if (callback) {
      if (expired.length) {
        expired.forEach(({ name }) => {
          const fieldErrors = [{
            message: `${name} need to revalidate`,
            field: name,
          }];
          set(errorsGroup, name, {
            expired: true,
            errors: fieldErrors,
          });
        });
      }

      callback(isEmptyObject(errorsGroup) ? null : errorsGroup,
        this.fieldsStore.getFieldsValue(fieldNames));
    }
  });
},

getRules(fieldMeta, action) {
  const actionRules = fieldMeta.validate.filter((item) => {
    return !action || item.trigger.indexOf(action) >= 0;
  }).map((item) => item.rules);
  return flattenArray(actionRules);
},
```

#### 表单项校验

针对第一点，rc-form组件的实现策略是通过getFieldProps方法生成传入表单元素的事件绑定函数props如{ onChange: () => {} }。绑定函数内部将调用validateFieldsInternal方法对当前的表单项进行校验。

```javascript
// rc-form/src/createBaseField.js
getFieldProps(name, usersFieldOption = {}) {
  // ...

  const validateRules = normalizeValidateRules(validate, rules, validateTrigger);
  const validateTriggers = getValidateTriggers(validateRules);
  validateTriggers.forEach((action) => {
    if (inputProps[action]) return;
    inputProps[action] = this.getCacheBind(name, action, this.onCollectValidate);
  });

  // ...

  return inputProps;
},
```

其中，参数validate = [{ rules, trigger }]、rules、validateTrigger均为usersFieldOption的属性。validate可以设置多组校验规则的不同触发方式，如失去焦点(‘onBlur’)或数据改变(‘onChange’)。参数validateTrigger即usersFieldOption.rules校验规则的触发方式，默认为’onChange’。this.getCacheBind方法用于以this.cachedBind = { name: { action: () => {} } }缓存校验函数。

```javascript
// rc-form/src/createBaseField.js
onCollectValidate(name_, action, ...args) {
  // this.onCollectCommon方法用于执行onChange等副作用函数，并获取表单项的值
  const { field, fieldMeta } = this.onCollectCommon(name_, action, args);
  const newField = {
    ...field,
    dirty: true,
  };
  this.validateFieldsInternal([newField], {
    action,
    options: {
      firstFields: !!fieldMeta.validateFirst,
    },
  });
},
```

可以看到，inputProps注入表单项的绑定函数this.onCollectValidate方法将调用this.validateFieldsInternal校验表单项。特别需要指明的是，fieldMeta为{ name, trigger, valuePropName, …usersFieldOption, validate }数据格式。其中，usersFieldOption.validateFirst决定当前校验字段是否执行阻塞式校验；usersFieldOption.getValueFromEvent用于将事件参数event等转化成表单项额值。usersFieldOption.onChange等方法当同名事件发生时将被调用。

#### 表单校验

rc-form组件中，校验整张表单通过显式调用this.validateFields方法实现。在该方法内部，也将调用this.validateFieldsInternal校验所有表单项。

```javascript
// rc-form/src/createBaseField.js
validateFields(ns, opt, cb) {
  // getParams用于参数转化
  const { names, callback, options } = getParams(ns, opt, cb);
  const fieldNames = names ?
    this.fieldsStore.getValidFieldsFullName(names) :
    this.fieldsStore.getValidFieldsName();
  const fields = fieldNames
    .filter(name => {
      const fieldMeta = this.fieldsStore.getFieldMeta(name);
      return hasRules(fieldMeta.validate);
    }).map((name) => {
      const field = this.fieldsStore.getField(name);
      field.value = this.fieldsStore.getFieldValue(name);
      return field;
    });
  if (!fields.length) {
    if (callback) {
      callback(null, this.fieldsStore.getFieldsValue(fieldNames));
    }
    return;
  }
  if (!('firstFields' in options)) {
    options.firstFields = fieldNames.filter((name) => {
      const fieldMeta = this.fieldsStore.getFieldMeta(name);
      return !!fieldMeta.validateFirst;
    });
  }
  this.validateFieldsInternal(fields, {
    fieldNames,
    options,
  }, callback);
},
```

#### 后台校验

笔者将探讨的是async-validator模块在koa框架中的使用。

因为AsyncValidator实例的validate方法采用回调的方式组织异步逻辑，所以需要将其转化为Promise实例，这个过程又可以通过在中间件中提供ctx.validate方法实现。

```javascript
const asyncValidatorMiddleware = async function(ctx, next){
  ctx.validate = (rules, values, opts) => {
    let options = Object.assign(opts);

    let validator = new AsyncValidator(rules);
    let promise = new Promise(resolve => {
      validator.validate(values, options, errs => {
        if ( errs ){
          resolve();
        } else {
          let msgs = errs.map(err=>err.message).join('; ');
          ctx.throw(400, msgs);
        };
      });
    }));

    return promise;
  };
  
  next();
};
```

### 制作校验器图形界面

在本小节，笔者将结合实际的项目经验，概要地设计一个可订制校验规则的图形界面。

首先，我们需要一个面板预设单组校验器的校验模式，比如限制某字段必须介于1到10之间，我们就可以在这块面板中限定该字段的值只能为数值，且开启’range’范围校验；接着，在另一块面板配置实际的校验数据，如min = 1, max = 10，由此生成实际的校验规则。

在第一块面板中，预设的校验模式最终将会吐出{ validatorCode, validatorName, validatorDescription, validatorRules }形式的数据结构。其中validatorCode为校验器code，validatorName为校验器名称，validatorDesription为校验器描述，validatorRules = [ rule ]以数组形式存储各字段的校验规则。rule对象的数据结构譬如{ fieldCode, fieldName, fieldDecription, type, validateStrategy, renderType }。在rule对象中，fieldCode为校验字段code；fieldName为校验字段名称；fieldDecription为校验字段描述；type为类型，可选’number’, ‘string’, ‘date’, ‘enum’, ‘array’, ‘object’, ‘custom’（当type为’array’或’object’时，嵌套设置其下属数组项或属性的校验规则；当type为’custom’时，校验方式由程序约定，通过fieldCode查找）；validateStrategy为校验策略，当type为’string’时，可选的校验策略包含’regexp’（正则匹配）, ‘expired’（严格相等）, ‘url’等，余略；renderType为渲染形式，当type为’string’时，renderType自然为输入框形式，这意味着在第二块面板中需要渲染一个输入框以配置正则或期望值，余略。

关于校验类型type和校验策略validateStrategy、渲染形式renderType的关系，笔者用下图简要说明：

![image](yuanxing1.png)

特别的，枚举类型额外需要输入框，以配置枚举值key-value键值对。

第一块面板用原型图展示，即为：

![image](yuanxing2.png)

至于弹窗形式的单字段校验规则配置，笔者不再赘述。

需要说明的是，针对type类型为’array’或’object’类型的校验规则，因其配置繁琐，完全可以使用type = ‘custom’类型予以校验，通过validatorCode与fieldCode获取后台写死的校验器，这样既对用户透明，也不容易出错。

有了第一块面板，我们再设计第二块面板，即实际校验数据配置面板。这里我们引入校验组合的概念，即单个校验组合下可配置多个校验器，校验组合之间有序排列，逐个校验，其中一个出错，即为校验失败。至于校验器配置弹窗，其渲染方式完全由第一块面板填充的数据决定，如我们配置了validateRules = [{ fieldCode: ‘field1’, fieldName: ‘field1’, type: ‘string’, validateStragery: ‘expired’, renderType: ‘input’ }, { fieldCode: ‘field2’, fieldName: ‘field2’, type: ‘enum’, validateStragery: ‘multiple’, renderType: ‘select’ }]，校验器的配置弹窗即显示一个输入框和一个多选下拉框，前者用于决定第一个字段的期望值，后者用于决定第二个字段须为下拉框选中项中的一个。在这里，笔者只给出校验组合的原型图，至于校验器的配置弹窗，不再赘述。

有了以上两块面板及其输入数据，我们就可以拼装实际的校验规则。笔者依然以前述的validatorRules为例，当用户配置了如下的校验组合，validatorCombinations = [[{ field1: ‘aaa’, field2: [‘1’, ‘2’, ‘3’] }]]。生成的实际校验器即为{ type: ‘expired’, expired: ‘aaa’ }及{ type: ‘enum’, enum: [‘1’, ‘2’, ‘3’] }。针对type值为’expired’的校验规则，预先需要在AsyncValidator构造函数中注册expiredValidator校验器如let expiredValidator = (val, rule) => { if ( val === rule.expired ) cb(); else cb(util.format(‘value %s is not valid, %s expired’, val, rule.expired)) }。

值得反思的是，以上的图形界面，并没有提供关联校验的能力。

### jquery-validate

相比async-validator模块专司于数据校验，jquery-validate模块还承担着视图层的工作。因此数据在流入流出校验器时，都可能需要判断数据所属节点的标签名。在这里，笔者只概要地介绍该插件数据校验方面的流程。

#### 内置校验器

在jquery-validate模块源码中，校验方法注册在$.validator.prototype.methods（后文将用methods代替）属性中。校验方法的函数形式均为function(value, element, param)，参数value为待校验的值，element为待校验的元素，param为相应校验规则的值。校验方法又包含required非空检验，email校验，url校验，date校验，dateISO年-月-日校验，number数值（包含小数）校验，digits正整数校验，creditcard信用卡号校验，minlength最小长度校验，maxlength最大长度校验，rangelength长度范围校验，min最小值校验，max最大值校验，range范围校验，equalTo关联元素等值校验，remote远程校验。

挂载在元素上的校验规则通过let rules = { fieldName:{ required: true, min: 1 } }; $(“#form”).validate({ rules })形式设定。因此各校验方法获得的param参数也在于配置项rules.filedName对象，如methods.required校验方法获得的param为true，methods.min方法获得的param为1。特别的，在methods.required校验方法中，预先调用this.depend方法判断先决条件是否成立（前提是将required校验方法作为预先设置的最基础的校验器，其他校验方法都调用this.optional方法校验元素的值非空后，才进行特定的数据校验。该先决条件的成立与否都不影响校验显示结果），当param为字符串时，需要相应元素在页面中存在，这在联动显示隐藏的业务场景特别有用；当param为函数时，需要函数返回真值，这可用在关联校验等业务场景中；当param为布尔值时，无须判断先决条件。

```javascript
$.validator.prototype.methods = {
  // ...
  
  elementValue: function (element) {
    var val,
      $element = $(element),
      type = element.type;

    if (type === "radio" || type === "checkbox") {
      return this.findByName(element.name).filter(":checked").val();
    } else if (type === "number" && typeof element.validity !== "undefined") {
      return element.validity.badInput ? false : $element.val();
    }

    val = $element.val();
    if (typeof val === "string") {
      return val.replace(/\r/g, "");
    }
    return val;
  },

  checkable: function (element) {
    return (/radio|checkbox/i).test(element.type);
  },

  getLength: function (value, element) {
    switch (element.nodeName.toLowerCase()) {
      case "select":
        return $("option:selected", element).length;
      case "input":
        if (this.checkable(element)) {
          return this.findByName(element.name).filter(":checked").length;
        }
    }
    return value.length;
  },

  depend: function (param, element) {
    return this.dependTypes[typeof param] ? this.dependTypes[typeof param](param, element) : true;
  },

  dependTypes: {
    "boolean": function (param) {
      return param;
    },
    "string": function (param, element) {
      return !!$(param, element.form).length;
    },
    "function": function (param, element) {
      return param(element);
    }
  },

  optional: function (element) {
    var val = this.elementValue(element);
    return !$.validator.methods.required.call(this, val, element) && "dependency-mismatch";
  },

  methods: {
    // {required:true}、{required:function(){return $(anotherEle).val()<12}}依赖条件  
    required: function(value, element, param) { 
      if (!this.depend(param, element)) {
        return "dependency-mismatch";
      }
      if (element.nodeName.toLowerCase() === "select") {
        // could be an array for select-multiple or a string, both are fine this way  
        var val = $(element).val();
        return val && val.length > 0;
      }
      if (this.checkable(element)) {
        return this.getLength(value, element) > 0;
      }
      return value.length > 0;
    },

    email: function(value, element) {
      return this.optional(element) || /^[a-zA-Z0-9.!#$%&'*+\/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$/.test(value);
    },
    url: function(value, element) {
      return this.optional(element) || /^(?:(?:(?:https?|ftp):)?\/\/)(?:\S+(?::\S*)?@)?(?:(?!(?:10|127)(?:\.\d{1,3}){3})(?!(?:169\.254|192\.168)(?:\.\d{1,3}){2})(?!172\.(?:1[6-9]|2\d|3[0-1])(?:\.\d{1,3}){2})(?:[1-9]\d?|1\d\d|2[01]\d|22[0-3])(?:\.(?:1?\d{1,2}|2[0-4]\d|25[0-5])){2}(?:\.(?:[1-9]\d?|1\d\d|2[0-4]\d|25[0-4]))|(?:(?:[a-z\u00a1-\uffff0-9]-*)*[a-z\u00a1-\uffff0-9]+)(?:\.(?:[a-z\u00a1-\uffff0-9]-*)*[a-z\u00a1-\uffff0-9]+)*(?:\.(?:[a-z\u00a1-\uffff]{2,})).?)(?::\d{2,5})?(?:[/?#]\S*)?$/i.test(value);
    },
    date: function(value, element) {
      return this.optional(element) || !/Invalid|NaN/.test(new Date(value).toString());
    },
    dateISO: function(value, element) {
      return this.optional(element) || /^\d{4}[\/\-](0?[1-9]|1[012])[\/\-](0?[1-9]|[12][0-9]|3[01])$/.test(value);
    },
    number: function(value, element) {
      return this.optional(element) || /^(?:-?\d+|-?\d{1,3}(?:,\d{3})+)?(?:\.\d+)?$/.test(value);
    },
    digits: function(value, element) {
      return this.optional(element) || /^\d+$/.test(value);
    },

    // http://jqueryvalidation.org/creditcard-method/  
    // based on http://en.wikipedia.org/wiki/Luhn_algorithm  
    creditcard: function(value, element) {
      if (this.optional(element)) {
        return "dependency-mismatch";
      }

      if (/[^0-9 \-]+/.test(value)) {
        return false;
      }
      var nCheck = 0,
        nDigit = 0,
        bEven = false,
        n, cDigit;

      value = value.replace(/\D/g, "");

      // http://developer.ean.com/general_info/Valid_Credit_Card_Types  
      if (value.length < 13 || value.length > 19) {
        return false;
      }

      for (n = value.length - 1; n >= 0; n--) {
        cDigit = value.charAt(n);
        nDigit = parseInt(cDigit, 10);
        if (bEven) {
          if ((nDigit *= 2) > 9) {
            nDigit -= 9;
          }
        }
        nCheck += nDigit;
        bEven = !bEven;
      }

      return (nCheck % 10) === 0;
    },

    minlength: function(value, element, param) {
      var length = $.isArray(value) ? value.length : this.getLength(value, element);
      return this.optional(element) || length >= param;
    },
    maxlength: function(value, element, param) {
      var length = $.isArray(value) ? value.length : this.getLength(value, element);
      return this.optional(element) || length <= param;
    },
    rangelength: function(value, element, param) {
      var length = $.isArray(value) ? value.length : this.getLength(value, element);
      return this.optional(element) || (length >= param[0] && length <= param[1]);
    },

    min: function(value, element, param) {
      return this.optional(element) || value >= param;
    },
    max: function(value, element, param) {
      return this.optional(element) || value <= param;
    },
    range: function(value, element, param) {
      return this.optional(element) || (value >= param[0] && value <= param[1]);
    },

    equalTo: function(value, element, param) {
      // bind to the blur event of the target in order to revalidate whenever the target field is updated  
      // TODO find a way to bind the event just once, avoiding the unbind-rebind overhead  
      var target = $(param);
      if (this.settings.onfocusout) {
        target.off(".validate-equalTo").on("blur.validate-equalTo", function () {
          $(element).valid();// 执行验证  
        });
      }
      return value === target.val();
    },

    // ...
  }
}
```

#### 远程校验

关于methods.remote远程校验。与async-validadtor模块不同的是，首先async-validadtor模块并没有提供远程校验方法，其次async-validadtor模块有异步流程控制，jquery-validate模块没有，而是通过远程校验起始时自增1、结束时自减1的标识符this.pendingRequest，以及远程校验中元素的集合this.pending来感知远程校验的状态，随后才能提交表单。

在jquery-validate模块中，远程校验的校验规则配置形式为rule = { remote: url }或rule = { url, …params }。参数url为请求地址，其余参数params以深拷贝的形式改变$.ajax方法的参数，可以包含data属性改变请求数据（默认值为表单项name值和表单项数据组成的键值对），success改变成功时的回调函数。默认的成功回调success中，当响应为真值，即为校验成功；为否值，即为校验失败。改写这个成功回调，需要调用者对jquery-validate模块的机制比较理解。

在性能方面，jquery-validate模块会在校验元素的’data-previousValue’属性中缓存上次的校验数据及结果。当元素的当前值和缓存数据相同时，直接使用缓存中的校验结果。为避免同一个校验元素连续发送多次请求，jquery-validate模块通过扩展$.ajax实现，当有新一轮请求时，将前一轮请求abort终结掉。

```javascript
$.validator.prototype = {
  // ...

  previousValue: function (element) {
    return $.data(element, "previousValue") || $.data(element, "previousValue", {
      old: null,
      valid: true,
      message: this.defaultMessage(element, "remote")
    });
  },

  startRequest: function (element) {
    if (!this.pending[element.name]) {
      this.pendingRequest++;
      this.pending[element.name] = true;
    }
  },

  stopRequest: function (element, valid) {
    this.pendingRequest--;
    // sometimes synchronization fails, make sure pendingRequest is never < 0  
    if (this.pendingRequest < 0) {
      this.pendingRequest = 0;
    }
    delete this.pending[element.name];
    if (valid && this.pendingRequest === 0 && this.formSubmitted && this.form()) {
      $(this.currentForm).submit();
      this.formSubmitted = false;
    } else if (!valid && this.pendingRequest === 0 && this.formSubmitted) {
      $(this.currentForm).triggerHandler("invalid-form", [this]);
      this.formSubmitted = false;
    }
  },
}

// http://jqueryvalidation.org/remote-method/
$.validator.prototype.methods.remote = function(value, element, param) {
  if (this.optional(element)) {
    return "dependency-mismatch";
  }

  var previous = this.previousValue(element),// 获取上一次的验证结果、值。提示文本  
    validator, data;

  if (!this.settings.messages[element.name]) {
    this.settings.messages[element.name] = {};
  }
  previous.originalMessage = this.settings.messages[element.name].remote;
  this.settings.messages[element.name].remote = previous.message;

  param = typeof param === "string" && { url: param } || param;

  // 当前验证与上一次情形相同，返回前一次验证结果  
  if (previous.old === value) {
    return previous.valid;
  }

  previous.old = value;
  validator = this;
  this.startRequest(element);
  data = {};
  data[element.name] = value;
  $.ajax($.extend(true, {
    mode: "abort",
    port: "validate" + element.name,// port和元素name相关，发送ajax后pendingRequests[port]标记为有值，阻止同一个元素发送两次ajax  
    dataType: "json",
    data: data, 
    context: validator.currentForm,
    success: function (response) {
      var valid = response === true || response === "true",
        errors, message, submitted;

      validator.settings.messages[element.name].remote = previous.originalMessage;
      if (valid) {
        submitted = validator.formSubmitted;
        validator.prepareElement(element);
        validator.formSubmitted = submitted;
        validator.successList.push(element);
        delete validator.invalid[element.name];
        validator.showErrors();
      } else {
        errors = {};
        message = response || validator.defaultMessage(element, "remote");
        errors[element.name] = previous.message = $.isFunction(message) ? message(value) : message;
        validator.invalid[element.name] = true;
        validator.showErrors(errors);
      }
      previous.valid = valid;
      validator.stopRequest(element, valid);
    }
  }, param));
  return "pending";  
}

var pendingRequests = {},
  ajax;
// Use a prefilter if available (1.5+)  
// 为ajax方法添加abort、port标志，阻止同一验证元素在前一次ajax未完成时，再度发起ajax请求
if ($.ajaxPrefilter) {
  // ajaxPrefilter发送前的预处理，setting请求的所有参数，xhr经过jquery封装的XMLHttpRequest对象  
  $.ajaxPrefilter(function (settings, _, xhr) {
    var port = settings.port;
    if (settings.mode === "abort") {
      if (pendingRequests[port]) {
        pendingRequests[port].abort();
      }
      pendingRequests[port] = xhr;
    }
  });
} else {
  ajax = $.ajax; 
  $.ajax = function (settings) {
    var mode = ("mode" in settings ? settings : $.ajaxSettings).mode,
      port = ("port" in settings ? settings : $.ajaxSettings).port;
    if (mode === "abort") {
      if (pendingRequests[port]) {
        pendingRequests[port].abort();// abort()用来终止ajax请求，还是会执行success回调，返回值为空  
      }
      pendingRequests[port] = ajax.apply(this, arguments);
      return pendingRequests[port];
    }
    return ajax.apply(this, arguments);
  };
}
```

#### 注册校验器

在jquery-validate模块中，注册校验方法通过调用addMethod实现。与async-validadtor模块相比较，jquery-validate模块稍显得不够灵活，只能将自定义校验器注册为methods下属的方法，而不能直接使用外部函数。

```javascript
$.validator = {
  // …

  addMethod: function (name, method, message) {
    $.validator.methods[name] = method;
    $.validator.messages[name] = message !== undefined ? message : $.validator.messages[name];
    if (method.length < 3) {
      $.validator.addClassRules(name, $.validator.normalizeRule(name));
    }
  },
}
```

#### 校验

在jquery-validate模块中，实际的校验过程通过调用this.check方法执行，无论是整张表单校验，还是单个表单项校验。在check方法内部，也即遍历校验规则，调用methods下属的方法完成校验。

```javascript
$.validator.prototype = {
  // ... 

  check: function (element) {
    element = this.validationTargetFor(this.clean(element));// 获取当前验证元素  

    var rules = $(element).rules(),// 获取元素的验证规则  
      rulesCount = $.map(rules, function (n, i) {
        return i;
      }).length,
      dependencyMismatch = false,
      val = this.elementValue(element),
      result, method, rule;

    for (method in rules) {
      rule = { method: method, parameters: rules[method] };
      try {

        result = $.validator.methods[method].call(this, val, element, rule.parameters);

        // 依赖条件不成立，跳过验证  
        if (result === "dependency-mismatch" && rulesCount === 1) {
          dependencyMismatch = true;
          continue;
        }
        dependencyMismatch = false;

        if (result === "pending") {
          this.toHide = this.toHide.not(this.errorsFor(element));
          return;
        }

        if (!result) {
          this.formatAndAdd(element, rule);// 获取错误文案，并向this.errorList塞值
          return false;
        }
      } catch (e) {
        if (this.settings.debug && window.console) {
          console.log("Exception occurred when checking element " + element.id + ", check the '" + rule.method + "' method.", e);
        }
        if (e instanceof TypeError) {
          e.message += ".  Exception occurred when checking element " + element.id + ", check the '" + rule.method + "' method.";
        }

        throw e;
      }
    }
    if (dependencyMismatch) {
      return;
    }
    if (this.objectLength(rules)) {
      this.successList.push(element);
    }
    return true;
  },
}
```

#### 错误文案

在jquery-validate模块中，错误文案的实现与async-validator模块基本相同。因为jquery-validate模块没有灵活的自定义校验器概念，错误文案都通过配置项注入，可以用通用的format方法对其格式化。

```javascript
$.validator = {
  // ...
  
  format: function (source, params) {
    if (arguments.length === 1) {
      return function () {
        var args = $.makeArray(arguments);
        args.unshift(source);
        return $.validator.format.apply(this, args);
      };
    }
    if (arguments.length > 2 && params.constructor !== Array) {
      params = $.makeArray(arguments).slice(1);
    }
    if (params.constructor !== Array) {
      params = [params];
    }
    $.each(params, function (i, n) {
      source = source.replace(new RegExp("\\{" + i + "\\}", "g"), function () {
        return n;
      });
    });
    return source;
  },

  messages: {
    required: "This field is required.",
    remote: "Please fix this field.",
    email: "Please enter a valid email address.",
    url: "Please enter a valid URL.",
    date: "Please enter a valid date.",
    dateISO: "Please enter a valid date ( ISO ).",
    number: "Please enter a valid number.",
    digits: "Please enter only digits.",
    creditcard: "Please enter a valid credit card number.",
    equalTo: "Please enter the same value again.",
    maxlength: $.validator.format("Please enter no more than {0} characters."),
    minlength: $.validator.format("Please enter at least {0} characters."),
    rangelength: $.validator.format("Please enter a value between {0} and {1} characters long."),
    range: $.validator.format("Please enter a value between {0} and {1}."),
    max: $.validator.format("Please enter a value less than or equal to {0}."),
    min: $.validator.format("Please enter a value greater than or equal to {0}.")
  },
}

$.validator.prototype = {
  // ...

  defaultMessage: function (element, method) {
    return this.findDefined(// this.findDefined输出首个真值的参数
      this.customMessage(element.name, method),// html设置的错误文案，字符串
      this.customDataMessage(element, method),// js代码设置的错误文案，字符串或函数
      // title is never undefined, so handle empty string as undefined  
      !this.settings.ignoreTitle && element.title || undefined,
      $.validator.messages[method],
      "<strong>Warning: No message defined for " + element.name + "</strong>"
    );
  },

  formatAndAdd: function (element, rule) {
    var message = this.defaultMessage(element, rule.method),// 获取错误文案
      theregex = /\$?\{(\d+)\}/g;
    if (typeof message === "function") {
      message = message.call(this, rule.parameters, element);
    } else if (theregex.test(message)) {
      message = $.validator.format(message.replace(theregex, "{$1}"), rule.parameters);
    }
    this.errorList.push({
      message: message,
      element: element,
      method: rule.method
    });

    this.errorMap[element.name] = message;
    this.submitted[element.name] = message;
  },

}
```

### jquery-validation

jquery-validation模块的校验流程完全仿照jquery-validate，只是增加了额外的校验方法和语言包。笔者扼要说明一下部分校验方法的功能，代码不再赘述。

* require_from_group方法校验指定相关元素(name值查找)选中项的个数不小于指定值时，予以校验通过
* skip_or_fill_minimum方法校验相关元素(name值查找)未赋值或选中项个数不小于指定值时，予以校验通过。
* pattern方法正则匹配。
* notEqualTo方法校验不等于指定值。
* integer方法校验整数，可以时负整数。
* maxWords、minWords、rangeWords方法校验字符数量，剔除html标签和标点符号后。
* strippedminlength方法校验文本长度不能小于指定值。
* nowhitespace方法校验非空字符串。
* lettersonly方法校验字符串。
* letterswithbasicpunc方法校验带基本标点符号的字符串，通过正则表达式 /^[a-z-.,()’”\s]+$/i 。
* alphanumeric方法校验值是否为字符串、下划线、数值中的一种，通过正则表达式 /^\w+$/i 。
* url2方法校验网址。
* dateFA、dateITA、dateNL校验日期。
* time、time12h校验时分，24小时制或12小时制。
* accept方法校验上传文件的类型。
* extension方法校验文件的扩展名。
* ipv4方法校验IP v4地址。
* ipv6方法校验IP v6地址。
* currency方法校验货币金额，可带有币种标识。
* mobileNL、mobileUK方法校验特定地区或国家的手机号。
* phoneNL、phonesUK、phoneUK、phoneUS方法校验特定地区或国家的移动电话号码。
* postalcodeBR、postalCodeCA、postalcodeNL、postcodeUK、zipcodeUS、ziprange方法校验特定地区或国家的邮政编码。
* bankaccountNL方法校验是否银行账号。
* giroaccountNL方法校验是否转账账号。
* bankorgiroaccountNL方法校验是否银行账号或者转账账号。
* iban方法校验国际银行账号。
* bic方法校验是否商业识别码。
* cifES方法校验是否西班牙纳税识别号。
* cpfBR方法校验是否巴西纳税识别号。
* creditcardtypes方法校验信用卡号是否为特定类型。
* vinUS方法校验汽车编号。
* stateUS方法校验美国的州或地区。