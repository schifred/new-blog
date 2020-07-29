---
title: Form 组件
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd,validator'
abbrlink: aa2570fd
date: 2018-11-04 06:00:00
updated: 2018-11-04 06:00:00
---

ant design 中的 Form 组件基于 rc-form 实现。本文第一部分将介绍 rc-form 库；第二部分再介绍 ant design 中的 Form 组件。

### rc-form

常规收集表单数据并作校验，只需以 store 实时记录表单数据，校验后重绘表单。这样的思路以业务代码为例，就是，以数据模型 model 集成数据处理操作，再通过 setState 将 model 中的实时数据注入组件中，并驱动组件重绘（除了 setState 方法以外，也可以使用 forceUpdate 方法重绘组件，并在 render 阶段重新访问 model 中的实时数据）。从业务角度对数据及其操作进行建模，必然着眼于实际的业务场景，其类结构也会和数据表有千丝万缕的联系；而表单中的数据更具一般性特征，即能对应多个数据表，对其进行抽象也须从视图层入手。可以推想的是，抽象的表单数据模型必然包含字段名和字段的值构成的映射 valuesMap，字段名和校验结果构成的映射 errorsMap，以及字段名和校验状态构成的映射 validatingMap，这样才能绘制出表单中的字段项。

在 rc-form 中，上述数据模型的具体实现为 FieldsStore 类。如前所述，FieldsStore 实例与视图层的交互逻辑为，在用户行为驱动字段项的数据改变时，即时存储表单数据及校验文案，继而调用表单组件实例的 forceUpdate 方法强制重绘；在绘制过程中，再从 FieldsStore 实例读取实时的表单数据、校验文案及校验状态。建模方面，FieldsStore 实例以 fields 属性存储表单的实时数据，其结构为键值对形式 { [name]: { value, errors, validating, dirty, touched } } 。其中，name 为字段名；value 为字段的值；errors 为校验结果；validating 为校验状态；dirty 为脏值标识（当字段的值已作变更、但未作校验时，那么脏值标识就为 true；已作校验则置为 false）；touched 为更新标识，即用户行为触发时，字段值已作收集标识，收集的值通常是更新后的值 （若 FieldsStore 实例在 onChange 发生时收集数据，touched 标识也意味着数据已作变更。当然，如果此时用户再将数据更新为初始值，touched 标识将依旧为 true，并不能反映表单数据的更新状态。但是在一般情况下，可以根据 touched 标识判断表单数据是否有过更改）。

除了实时更新的数据外，驱动校验需要校验规则 validateRules、触发事件 validateTrigger；字段的初始值 initialValue（当字段的值还没存入 fields 中时，以初始值替代）；从 event 对象获取字段的值，也跟 dom 节点的类型相关，如 input, radio, checkbox 类型，可以借助 getValueFromEvent 函数从 event 对象获取的字段的值；注入字段组件的值需要作特殊的数据转换，其一 radio 原生组件通过 props.checked 接受字段的实时值，其二对于自定义组件，不只接受实时值的 props 属性是特殊的，收集的数据和注入组件中的数据也会存在结果差异，这可以借助 getValueProps 函数作转换操作；在某些场合下，收集的表单数据需要经过特殊转换（使用案例：[全选按钮](https://codepen.io/afc163/pen/JJVXzG?editors=001)），这可以借助 normalize 函数实现。FieldsStore 实例使用 fieldsMeta 属性存储这些元数据，其结构为 { [name]: { validate, initialValue, getValueFromEvent, valuePropName, getValueProps, normalize } }。其中，validate 包含 validateRules, validateTrigger。

与业务实体不同的是，FieldsStore 实例仅止于存储字段的表单数据和元数据，提供一些便捷的访问器操作，却没有包含数据校验等操作，也没有关联上表单及字段组件。对于用户自定义表单组件，需要提供获取、更新及校验表单数据的方法，以便组织与远程接口密切相关的交互逻辑。对于字段组件，校验规则等与指定字段强关联的配置项适合在使用字段组件时通过 props 注入；同时，在字段组件的值发生变更时，需要收集该字段的值及启动对该字段的校验，因此需要将特定的绑定函数添加到 props 中。这些字段元数据写入 FieldsStore 就适合在绘制字段组件的过程中实现，因此就需要特定的方法用于装饰字段组件或其 props 属性。在 rc-form 中，BaseForm 用于实现这部分功能。

想要为用户自定义组件注入工具函数，可以使用 HOC 高阶组件将工具函数配置为自定义子组件的 props 形式实现。BaseForm 就是这样的高阶组件。在 BaseForm 的 render 阶段，将为用户自定义组件注入 props.form 操纵表单的工具函数集。工具函数集中包含 getFieldValue, getFieldsValue, getFieldError, getFieldsError, isFieldsValidating, isFieldValidating, isFieldTouched, isFieldsTouched 方法用于获取字段的值、校验文案、校验状态、是否更新标识等；setFieldsInitialValue 方法用于设置字段的初始值；setFieldsValue 方法用于设置字段的值；setFields 方法用于设置表单的实时数据，包含字段的值及校验文案等；resetFields 方法用于重置表单；validateFields 方法用于校验表单；getFieldProps 方法用于转换传入字段组件的 props 数据（包含特定的绑定函数），并收集字段组件的元数据；getFieldDecorator 方法基于 getFieldProps 方法，不同于 getFieldProps 方法用于装饰字段组件的 props，getFieldDecorator 用于直接装饰字段组件，这样就可以直接获取并封装传入字段组件实例的 props.onChange 等属性或方法；getFieldInstance 方法用于获取字段实例。

关于校验文案和校验状态的绘制，参见本文的第二部分。

以时序图的方式表达 rc-form 的工作流程为：

![image](rc-form1.png)

rc-form 的类图结构为：

![image](rc-form2.png)

#### FieldsStore

如上文所述，FieldsStore 基本用作表单字段元数据和实时数据的存储器。此外，rc-form 支持以嵌套结构定义字段名，即使用 ‘.’, ‘[|]’ 作为分割符，如 ‘a.b’ 意味着 a 对象下的 b 属性；’c[0]’ 意味着 c 数组的首项。这一机制借助于 lodash 类库的 set, get 方法和内置的 flattenFields 函数实现的。并且，FieldsStore 提供 isValidNestedFieldName 方法用于校验表单中的字段名不能作为其他字段名的成员。

flattenFields(maybeNestedFields, isLeafNode, errorMessage) 函数用于将嵌套数据扁平化，如将 { a: { b: 1 } } 转化成 { ‘a.b’: 1 }。参数 maybeNestedFields 即嵌套数据；参数 isLeafNode 用于校验扁平化后的数据成员的合理性；参数 errorMessage 校验不合理时的警告文案。其实现如：

```javascript
function treeTraverse(path = '', tree, isLeafNode, errorMessage, callback) {
  if (isLeafNode(path, tree)) {
    callback(path, tree);
  } else if (tree === undefined || tree === null) {
    // Do nothing
  } else if (Array.isArray(tree)) {
    tree.forEach((subTree, index) => treeTraverse(
      `${path}[${index}]`,
      subTree,
      isLeafNode,
      errorMessage,
      callback
    ));
  } else { // It's object and not a leaf node
    if (typeof tree !== 'object') {
      warning(false, errorMessage);
      return;
    }
    Object.keys(tree).forEach(subTreeKey => {
      const subTree = tree[subTreeKey];
      treeTraverse(
        `${path}${path ? '.' : ''}${subTreeKey}`,
        subTree,
        isLeafNode,
        errorMessage,
        callback
      );
    });
  }
}

function flattenFields(maybeNestedFields, isLeafNode, errorMessage) {
  const fields = {};
  treeTraverse(undefined, maybeNestedFields, isLeafNode, errorMessage, (path, node) => {
    fields[path] = node;
  });
  return fields;
}
```

元数据以 this.fieldsMeta = { [name]: { validate, hidden, getValueFromEvent, initialValue, valuePropName, getValueProps, normalize } } 形式存储（name 为字段名，下同）。以下是字段元数据中各属性的意义。

* validate 校验规则和触发事件，[{ rules, trigger }] 形式。
* hidden 设置为 true 时，getFieldsValue, getFieldsError 等方法将无法获取该字段的数据及校验信息等实时数据。本文假设设置了 hidden 为 true 的字段为虚拟隐藏项。
* getValueFromEvent(event) 用于从 event 对象中获取字段的值。
initialValue 字段的初始值。
* valuePropName 约定字段的值以何种 props 属性注入字段组件中。
* getValueProps(value) 用于转化字段的值，输出 props 以注入字段组件中。
* normalize(newValue, oldValue, values) 用于转换存入 FieldsStore 实例的字段值。

实时数据以 this.fields = { [name]: { value, errors, validating, dirty, touched } } 形式存储。以下是字段实时数据中各属性的意义。

* value 字段的值。
* errors 校验文案，数组形式。
* validating 校验状态。
* dirty 脏值标识。真值时意味着字段数据已作变更，但未作校验。
* touched 更新标识。真值时意味着用户行为已促使字段数据发生了变更。

对于元数据，rc-form 实现的访问器机制极为简单，即通过 setFieldMeta(name, meta) 赋值或更新某个字段的元数据，通过 getFieldMeta(name) 获取某个字段的元数据。辅助方法 getAllFieldsName 用于获取 this.feildsMeta 中所有字段名列表。同时，元数据是在字段组件渲染阶段创建的，其存在与否也意味字段组件是否呈现在视图中。实例方法 flattenRegisteredFields(fields) 即基于此实现，既校验与参数 fields 数据的字段组件是否已渲染，又将 fields 数据扁平化。特别的，setFieldsInitialValue(initialValues) 实例方法首先将参数 initialValues 注入为 flattenRegisteredFields 方法的参数以校验相关的字段组件是否以渲染，再将初始值写入 feildsMeta 属性中。因此，针对元数据的操作包含 setFieldMeta, getFieldMeta, setFieldsInitialValue 以及下文的 clearField 四种。

对于实时数据，rc-form 实现的访问器机制较为复杂。其一，未经用户操作字段组件或开发者显示调用 setFields 赋值表单数据，字段的实时数据将不存在 fields 属性中（下文将这些字段称为未收集字段）；其二，既需要支持全量更新表单的实时数据，又需要支持部分更新表单的实时数据；其三，字段中的实时数据成员需要单独获取。以上第一条，使得 FieldsStore 并不存在 getFields 方法，而是先通过 getNotCollectedFields 方法获取未收集字段的初始值，再构建 getNestedAllFields 方法遍历 fields 中的字段以获取到收集到的实时数据；第二条，FieldsStore 提供 updateFields(fields) 方法用于全量更新实时数据，setFields(fields) 用于部分更新实时数据；第三条，使 FieldsStore 在 getField(name) 获取字段的实时数据以外，还提供 getFieldMember(name, member) 用于获取实时数据的成员，并以此构建了 isFieldValidating, isFieldTouched 方法，用于获取字段的校验状态和更新状态。

当然，表单除了更新数据和单字段实时数据获取以外，还有对实时数据如表单数据、校验信息和校验状态的全量获取。为此，FieldsStore 先行提供了辅助函数。其中，getAllFieldsName 基于 fieldsMeta 元数据，获取表单的全量字段名；getValidFieldsName 方法用于获取剔除虚拟隐藏项后的字段名列表；getValidFieldsFullName(maybePartialName) 方法基于 getValidFieldsName，所有以 maybePartialName 起始或等值的字段名列表，但不包含虚拟隐藏项。在此基础上，getNestedField(name, getter) 方法将获取所有以 name 起始或等值的字段名列表，并使用 reduce 方法加以遍历，通过 getter(name) 函数如 getFieldError 等实例方法，以获取字段数据或校验信息。不同于 getNestedField 先使用 getValidFieldsFullName 方法获取匹配的字段名列表，getNestedFields(names, getter) 则直接使用 reduce 遍历参数 names 或剔除虚拟隐藏项后的字段名列表，以获取字段数据或校验信息。

由于字段值的特殊性，即当实时数据尚未存入 fields 时，须以 fieldsMeta 中的初始值代替。因此，FieldsStore 提供了 getValueFromFields(name, fields) 用于从参数 fields 中的实时值或 fieldsMeta 元数据中的初始值。

有了这些辅助函数，FieldsStore 才得以实现：

* getFieldValue(name) 用于获取匹配字段的值（匹配字段指以 name 起始或与 name 等值的字段，下同）。
* getFieldError(name) 用于获取匹配字段的校验信息。
* isFieldValidating(name) 用于获取匹配字段的校验状态。
* isFieldTouched(name) 用于获取匹配字段的更新标识。
* getFieldsValue(names) 用于获取指定字段或表单的全量值数据，但不包含虚拟隐藏项；
* getFieldsError(names) 用于获取指定字段或表单的全量校验信息，但不包含虚拟隐藏项。
* isFieldsValidating(names) 用于获取指定字段或表单的全量校验状态，但不包含虚拟隐藏项。
* isFieldsTouched(names) 用于获取指定字段或表单的全量更新标识，但不包含虚拟隐藏项。

除此以外，FieldsStore 中与 BaseForm 交互相关的方法还包含：

* getFieldValuePropValue(fieldMeta) 基于参数 fieldMeta 获取注入字段组件的 props 属性。该 props 属性为字段的值内容，可经由 fieldMeta.valuePropName, fieldMeta.getValueProps(value) 处理，通过 BaseForm 实例的 getFieldProps 方法注入字段组件。
* getAllValues 根据 fieldsMeta 属性获取表单的全量数据，包含虚拟隐藏项。
* setFieldsInitialValue 见上文，设置字段的初始值。
* updateFields(fields) 见上文，全量更新表单的实时数据。
* setFields(fields) 见上文，部分更新表单的实时数据。
* resetFields(ns) 方法只输出匹配字段或全部字段的空值（这些字段的实时数据均已收集），本身并不改变 fields 存储的数据。
* clearField(name) 用于清除字段的元数据和实时数据。

#### BaseForm

如上文所说，BaseForm 作为自定义组件的外层容器，它用于为字段组件绑定数据收集和校验的方法，以更新 FieldsStore 实例存储的实时数据，同时将操作表单的工具函数集通过 props 注入到用户自定义表单中。其实现为：

首先，通过 createBaseForm(option, mixins) 创建装饰函数。装饰函数可以为用户自定义表单组件包裹上 HOC 容器，即 BaseForm 组件。参数 option 能为 HOC 组件提供 validateMessages, onFieldsChange, onValuesChange, mapProps, mapPropsToFields, fieldNameProp, fieldMetaProp, fieldDataProp, formPropName, name 配置项，参数 mixins 为混入 HOC 组件的实例方法。

* validateMessages 用于更改 async-validator 库的配置文案。
* onFieldsChange(props, changedFields, oldFields) 当 BaseForm 组件实例的 setFields 方法执行时被调用，包含用户行为促使数据收集或校验时，开发者显式调用 setFields, setFieldsValue, resetFields 时，BaseForm 机制 saveRef 方法执行阶段恢复实时数据时（见下文）。
* onValuesChange(props, changedValues, oldValues) 当用户行为触发表单数据收集或校验时（在 onFieldsChange 方法前执行），或开发者显示调用 setFieldsValue 方法时（在 onFieldsChange 方法后执行），都将调用 onValuesChange 函数。
* mapProps({ [formPropName] }, restProps) 用于修改注入自定义表单组件的 props。{ [formPropName] } 即 BaseForm 组件实例注入用户自定义组件的表单操作函数集；执行上下文为 BaseForm 组件实例。
* mapPropsToFields(props) 将 BaseForm 组件实例获得的 props 转化为 FieldsStore 构造器的参数 fields，通常用于将状态管理器中的 store 数据转换为表单所需的 fields。
* fieldNameProp 作为字段组件接受字段名的 props 属性名，其值默认为字段名或表单名加字段名的形式，因此可以通过访问字段组件实例的 props 获取到字段名。
* fieldMetaProp 作为字段组件接受元数据的 props 属性名，因此可以通过访问字段组件实例的 props 获取到该字段的元数据。
* fieldDataProp 作为字段组件接受实时数据的 props 属性名，因此可以通过访问字段组件实例的 props 获取到该字段的实时数据。
* formPropName 作为自定义表单组件接受表单操作函数集的 props 属性名，默认为 ‘form’。
* name 表单名。

其次，在 BaseForm 组件实例的 getInitialState 阶段，将调用 option.mapPropsToFields 以获得初始 fields，并创建 FieldsStore 实例（备注：在 BaseForm 组件的 componentWillReceiveProps 生命周期中，也将调用 option.mapPropsToFields 获取 fields，以便使用 FieldsStore 实例的 updateFields 全量更新缓存数据）。除此而外，getInitialState 方法还将创建 instances, cachedBind, clearedFieldMetaCache, renderFields, domFields 缓存，并为 BaseForm 组件注入 getFieldsValue, getFieldValue, setFieldsInitialValue, getFieldsError, getFieldError, isFieldValidating, isFieldsValidating, isFieldsTouched, isFieldTouched 实例方法（意义见上文）。

* instances 缓存字段组件实例。
* cachedBind 缓存绑定函数（包含收集和校验字段的实例方法 onCollect, onCollectValidate）及 saveRef 引用函数。
* clearedFieldMetaCache 缓存待移除字段的实时数据和元数据。getFieldProps 方法执行时清除 clearedFieldMetaCache[name] 缓存数据。saveRef 方法执行时将根据字段组件的渲染状态，尝试使用 clearedFieldMetaCache[name] 恢复 FieldsStore 实例中的实时数据和元数据，在清除 clearedFieldMetaCache[name] 缓存；或者将 FieldsStore 实例中的实时数据和元数据存入 clearedFieldMetaCache[name] 缓存。见下文。
* renderFields 缓存 getFieldProps 方法已执行标识。
* domFields 缓存字段组件实例仍在视图中展示的标识。

其次，执行 render 方法，将表单操作函数集通过 props 注入用户自定义组件。因此，在用户自定义组件中，开发者可以获取表单的实时数据，或者更新表单数据，或者校验表单，以完成特定渲染。以下是开发者可调用的方法。

* setFields(maybeNestedFields, callback) 以参数 maybeNestedFields 部分更新 FieldsStore 实例中的实时数据，并执行 option.onFieldsChange 方法，再调用 forceUpdate 重绘表单并执行回调。
* setFieldsValue(changedValues, callback) 基于 setFields 方法，更新表单数据，并执行 option.onValuesChange 方法。若 changedValues 中相关的字段组件未作渲染，予以警告提示。
* resetFields(ns) 基于 setFields 方法，重置匹配字段的表单数据或全量表单数据，并清除相关字段的 clearedFieldMetaCache 缓存。
* validateFields(ns, opt, cb) 基于 validateFieldsInternal，校验匹配字段的表单数据或全量表单数据。参数 opt 作为 validateFieldsInternal 的参数 options，cb 为校验完成后的回调。
* getFieldInstance(name)，获取字段组件实例。

其次，在渲染字段组件的过程中，BaseForm 提供 getFieldProps 实例方法用于装饰注入字段组件的 props，以及 getFieldDecorator 实例方法用于装饰字段组件。

* getFieldProps(name, usersFieldOption) 首先将清理 clearedFieldMetaCache 缓存，其次为 FieldsStore 实例收集字段的元数据，如转换校验规则；最终将输出转化后的 props 以用于字段组件的渲染。相关 props 属性包含：指定字段组件的 ref 引用函数为 BaseForm 内置的 saveRef 实例方法；为字段组件绑定 onCollect, onCollectValidate 实例方法，以在用户行为发生时收集或校验表单数据；将元数据 { [fieldMetaProp]: fieldMeta }，实时数据 { [fieldDataProp]: field }，字段名 { [fieldNameProp]: name } 注入字段组件实例。
* getFieldDecorator(name, fieldOption) 基于 getFieldProps 方法，直接装饰字段组件，以获得开发者设定在字段组件实例上的 ref 引用函数及 props.onChange 等绑定函数等，并以 fieldMeta.ref, fieldMeta.originalProps 形式存入元数据中。这样就能在 BaseForm 实例的 saveRef 执行过程中，可以调用开发者设定在字段组件实例上的 ref 引用函数；在 BaseForm 实例的 onCollect, onCollectValidate 执行过程中，可以调用开发者设定在字段组件实例上的 onChange 绑定函数。getFieldDecorator 函数也将使用初始值绘制字段项。

为字段组件绑定的 onCollect, onCollectValidate 方法均基于 onCollectCommon(name, action, args) 实例方法。其执行逻辑为：在 action 事件触发时，首先调用开发者配置的绑定函数 fieldMeta[action] 或 fieldMeta.originalProps[action]，其次通过 fieldMeta.getValueFromEvent 从 event 对象获取字段的值或者取值，其次执行挂载于表单上的 option.onValuesChange 绑定函数，最终返回 { name, field, fieldMeta }。下面是 onCollect, onCollectValidate 方法的简要实现逻辑。

* onCollect(name_, action, …args) 基于 setFields 方法，在事件触发时，实时更新 FieldsStore 实例存储的实时数据。
* onCollectValidate(name_, action, …args) 基于 validateFieldsInternal, setFields 方法，在事件触发时，既实时更新 FieldsStore 实例存储的实时数据，又实时校验字段。

对于字段组件，BaseForm 有一种缓存刷新机制。当字段组件实例从视图中移除时，须得调用 FieldsStore 实例的 clearField 方法以清除缓存的实时数据和元数据。这一过程通常在 ref 引用函数内执行，通过参数 —— 组件实例的真值情况判断字段组件是否已作销毁。而当字段组件更新时，react 16 的机制会在 getFieldProps 方法执行之后，调用两次 ref 引用函数，第一次 ref 引用函数的参数为否值，表示前一个实例需要被销毁；第二次 ref 引用函数的参数为真值，表示创建新的字段组件实例。当执行第一次 ref 引用函数时，FieldsStore 实例的实时数据和元数据都将被销毁。这样，即便字段组件仍旧在视图中有所表现，元数据如校验规则的丢失也将使促使字段组件在值变更时无法得到正常的校验。鉴于此，BaseForm 提供 clearedFieldMetaCache 属性缓存待移除字段的实时数据和元数据，且在第二次执行 ref 引用函数时尝试恢复 FieldsStore 实例中的相应数据。

* saveRef(name, _, component) 作为 BaseForm 为字段组件提供的引用函数，其在字段组件挂载、卸载、重绘阶段都会被调用。可根据参数 component 判断组件的渲染状态。当 component 为空值，使用 clearedFieldMetaCache[name] = { field, meta } 收集字段的元数据和表单数据，并清除 FieldsStore 实例中的相关数据，清理 instances[name], cachedBind[name] 等缓存；若 component 为字段组件实例，domFields[name] 缓存标记将置为真值，instances[name] 将缓存字段组件实例，并尝试使用 clearedFieldMetaCache 缓存以恢复 FieldsStore 实例中的相关数据，等恢复完成后，再行清理 clearedFieldMetaCache[name] 缓存。
* recoverClearedField(name) 通过 clearedFieldMetaCache[name] 缓存恢复 FieldsStore 实例存储的字段实时数据和元数据，完成后清除 clearedFieldMetaCache[name] 缓存。

字段数据缓存并恢复机制的相关源码为：

```javascript
getFieldProps(name, usersFieldOption = {}) {
  if (!name) {
    throw new Error('Must call `getFieldProps` with valid name string!');
  }
  if (process.env.NODE_ENV !== 'production') {
    warning(
      this.fieldsStore.isValidNestedFieldName(name),
      'One field name cannot be part of another, e.g. `a` and `a.b`.'
    );
    warning(
      !('exclusive' in usersFieldOption),
      '`option.exclusive` of `getFieldProps`|`getFieldDecorator` had been remove.'
    );
  }

  delete this.clearedFieldMetaCache[name];

  const fieldOption = {
    name,
    trigger: DEFAULT_TRIGGER,
    valuePropName: 'value',
    validate: [],
    ...usersFieldOption,
  };

  const {
    rules,
    trigger,
    validateTrigger = trigger,
    validate,
  } = fieldOption;

  const fieldMeta = this.fieldsStore.getFieldMeta(name);
  if ('initialValue' in fieldOption) {
    fieldMeta.initialValue = fieldOption.initialValue;
  }

  const inputProps = {
    ...this.fieldsStore.getFieldValuePropValue(fieldOption),
    ref: this.getCacheBind(name, `${name}__ref`, this.saveRef),
  };
  if (fieldNameProp) {
    inputProps[fieldNameProp] = formName ? `${formName}_${name}` : name;
  }

  const validateRules = normalizeValidateRules(validate, rules, validateTrigger);
  const validateTriggers = getValidateTriggers(validateRules);
  validateTriggers.forEach((action) => {
    if (inputProps[action]) return;
    inputProps[action] = this.getCacheBind(name, action, this.onCollectValidate);
  });

  // make sure that the value will be collect
  if (trigger && validateTriggers.indexOf(trigger) === -1) {
    inputProps[trigger] = this.getCacheBind(name, trigger, this.onCollect);
  }

  const meta = {
    ...fieldMeta,
    ...fieldOption,
    validate: validateRules,
  };
  this.fieldsStore.setFieldMeta(name, meta);
  if (fieldMetaProp) {
    inputProps[fieldMetaProp] = meta;
  }

  if (fieldDataProp) {
    inputProps[fieldDataProp] = this.fieldsStore.getField(name);
  }

  // This field is rende#f81d22, record it
  this.renderFields[name] = true;

  return inputProps;
},

saveRef(name, _, component) {
  if (!component) {
    // after destroy, delete data
    this.clearedFieldMetaCache[name] = {
      field: this.fieldsStore.getField(name),
      meta: this.fieldsStore.getFieldMeta(name),
    };
    this.clearField(name);
    delete this.domFields[name];
    return;
  }
  this.domFields[name] = true;
  this.recoverClearedField(name);
  const fieldMeta = this.fieldsStore.getFieldMeta(name);
  if (fieldMeta) {
    const ref = fieldMeta.ref;
    if (ref) {
      if (typeof ref === 'string') {
        throw new Error(`can not set ref string for ${name}`);
      }
      ref(component);
    }
  }
  this.instances[name] = component;
},

recoverClearedField(name) {
  if (this.clearedFieldMetaCache[name]) {
    this.fieldsStore.setFields({
      [name]: this.clearedFieldMetaCache[name].field,
    });
    this.fieldsStore.setFieldMeta(name, this.clearedFieldMetaCache[name].meta);
    delete this.clearedFieldMetaCache[name];
  }
}
```

其次，在 BaseForm 组件的 componentDidMount, componentDidUpdate 生命周期中，将调用 cleanUpUselessFields 实例方法，以根据 renderFields, domFields 缓存判断字段组件渲染状态，如字段组件未作渲染，调用 clearField 实例方法清理 FieldsStore 实例存储的字段实时数据和元数据、以及 instances[name], cachedBind[name] 缓存。

以上功能的实现基于 getCacheBind, getRules, validateFieldsInternal 实例方法。其中，getRules(fieldMeta, action) 用于从 fieldMeta 元数据中获取指定事件的校验规则。下面是 getCacheBind, validateFieldsInternal 方法的简要实现逻辑及相关源码。

* getCacheBind(name, action, fn) 以 cachedBind[name][action] = { fn, oriFn } 形式缓存 onCollect, onCollectValidate, saveRef 方法，便于使用 name 进行查找。
* validateFieldsInternal(fields, { fieldNames, action, options }, callback) 校验指定字段。可复用之前的校验结果，收集待校验的字段并予以校验，转化校验文案并执行 callback 回调。若异步校验期间字段的值发生改变，将得到校验已过期文案。在校验起始阶段，调用 setFields 记录校验中状态；校验结束阶段，调用 setFields 记录校验结果。

```javascript
getCacheBind(name, action, fn) {
  if (!this.cachedBind[name]) {
    this.cachedBind[name] = {};
  }
  const cache = this.cachedBind[name];
  if (!cache[action] || cache[action].oriFn !== fn) {
    cache[action] = {
      fn: fn.bind(this, name, action),
      oriFn: fn,
    };
  }
  return cache[action].fn;
},

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
        const field = get(errorsGroup, fieldName);
        if (typeof field !== 'object' || Array.isArray(field)) {
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
}
```

#### 其他

createForm(options) 基于 createBaseForm，为 HOC 组件注入默认的 getForm 方法，将 form 表单操作函数工具集通过 props 注入到用户自定义表单组件中。form 中含有的方法，参见上文。

createDOMForm(option) 基于 createBaseForm，在传递给用户自定义表单组件的 props.form 混入 validateFieldsAndScroll 方法，校验失败时滚动到首个错误字段处。该方法引用了 react-dom，只适用于浏览器平台，不适用于手机端。

validateFieldsAndScroll(ns, opt, cb) 基于 dom-scroll-into-view 库实现，首先通过 getBoundingClientRect 获得校验失败字段的 top 值，其次通过 getComputedStyle 或 style 属性获得字段节点首个滚动的父元素，最后调用 dom-scroll-into-view 库提供的 api 滚动页面。
FormScope 以组件形式提供接口。option 配置项通过 props 传入，在 render 阶段调用 createDOMForm，并将 form 传入函数组件 children 中。

createFormField 生成 Field 实例。

### Form

#### Form 组件

Form 组件本身并不承载逻辑，而是通过 props.className, props.prefixCls, props.layout, props.hideRequiredMark, props.onSubmit 设定注入 form 原生节点的样式类及绑定函数，以影响表单内部节点渲染时的样式。同时，Form 组件将为子组件传入 context.vertical 以区分是水平布局，还是垂直布局。

Form 组件拥有 Item 静态属性指向 FormItem 组件；createFormField 静态方法指向 rc-form 提供的同名方法；createForm 静态方法调用 rc-form 提供的 createBaseForm 方法，用于装饰用户自定义表单组件。

#### FormItem 组件

FormItem 组件用于设定表单项的布局，其可配置的 props 属性包含必填标记 hideRequiredMark, 字段名 label, 校验文案 help, 额外内容 extra。

同受控组件和非受控组件，FormItem 组件提供两种使用方式：其一，当未设定校验信息相关的 props 属性时，FormItem 组件将自动根据内部字段组件实例的状况渲染校验文案及校验状态；其二，当设定校验信息相关的 props 属性时，FormItem 组件将根据开发者传入的 props 渲染校验文案及校验状态。在第一种使用方式下，FormItem 组件只可以包含一个字段组件；在第二种使用方式下，FormItem 组件中可以包含多个字段组件，布局也更为灵活。这里说的相关 props 属性包含：校验文案 help, 校验状态 validateStatus（用于绘制反馈图标）, 必填标识 required, 字段名 id（影响点击 label 时聚焦哪个字段元素）。

那么，FormItem 又是怎样自动收集字段组件的校验数据呢？因为在 BaseForm 组件提供的 getFieldProp 方法，字段的字段名、元数据和实时数据都将作为特殊的 props 属性传入到字段组件中，所以作为字段组件容器的 FormItem，就可以通过这些特殊的 props 属性判断子组件实例是不是一个字段组件实例，当其为字段组件实例时，进一步收集实时的校验信息，从校验规则中获取是否必填标识，以完成渲染。

此外，FormItem 可以使用 props.labelCol, props.wrapperCol 属性栅格化布局标签组件和字段组件，其实现借助于 antd 提供的 Row, Col 组件。当点击标签 label 时，FormItem 提供的绑定函数能为字段组件获得焦点。这里不再多加介绍。

获取字段组件实例的相关源码为：

```javascript
getControls(children: React.ReactNode, recursively: boolean) {
  let controls: React.ReactElement<any>[] = [];
  const childrenArray = React.Children.toArray(children);
  for (let i = 0; i < childrenArray.length; i++) {
    if (!recursively && controls.length > 0) {
      break;
    }

    const child = childrenArray[i] as React.ReactElement<any>;
    if (child.type &&
      (child.type as any === FormItem || (child.type as any).displayName === 'FormItem')) {
      continue;
    }
    if (!child.props) {
      continue;
    }
    if (FIELD_META_PROP in child.props) { // And means FIELD_DATA_PROP in child.props, too.
      controls.push(child);
    } else if (child.props.children) {
      controls = controls.concat(this.getControls(child.props.children, recursively));
    }
  }
  return controls;
}
```

以上即简要分析了 ant-design 中 Form 组件的实现，文中难免有谬误或思考上的不足处，仍望读者海涵。