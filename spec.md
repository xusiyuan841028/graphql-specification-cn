# GraphQL语言规范

制定中的草案 – October 2016

## 简介

该文档是GraphQL的RFC规范草案，GraphQL是Facebook在2012年创造的一门查询语言，用来描述客户-服务器应用中数据模型的能力和需求。该语言标准的制定开始于2015年。GraphQL是一门全新的、正在进化中的语言，尚未完成。在未来，该规范会继续增加重要的增强特性。

版权声明  
Copyright \(c\) 2015‐2016, Facebook, Inc. All rights reserved.

只要满足以下条件，就允许在修改或不修改的情况下以源代码和二进制形式重新分发和使用：

* 源码的再分发必须保留上述版权声明，

Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.  
Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.  
Neither the name Facebook nor the names of its contributors may be used to endorse or promote products derived from this software without specific prior written permission.

## 1 总览 Overview {#1_overview}

asdfasdfasdf

## 6 执行 Execution {#6_execution}

GraphQL通过`执行(execution)`从一个`请求(request)`中生成一个`响应(response)`。

一个用来执行的请求包含了以下信息：

* 使用的`模式（schema）`，通常由GraphQL服务单独提供
* 一个`文档(document)`包含了需要执行的GraphQL`操作(operations)`和`片段(fragments)`
* 可选：被执行的文档中的操作的名称
* 可选：操作中定义的所有变量的值
* 一个初始值，对应被执行的根类型。初始值概念地表达了一个GraphQL服务中所有可用的数据。通常一个GraphQL服务在每个请求中总是使用相同的初始值。

按所给的这些信息，`ExecuteRequest()`产生一个按下述Response章节规范格式化的响应

### 6.1 执行请求 Executing Requests

在执行一个请求之前，执行器\(executor）需要一个已经被解析过的文档（按本规范”Query Language”章节中的定义），如果文档定义了多个操作，还需要一个选定的操作名称，否则文档仅能包含唯一一个操作。请求的结果，由依照以下”Executing Operations”章节规范执行该操作的结果决定。

`ExecuteRequest(schema, document, operationName, variableValues, initialValue)`：

1. `operation`设为`GetOperation(document, operationName)`的结果
2. `coercedVariableValues`设为`CoerceVariableValues(schema, operation, variableValues)`的结果
3. 如果`operation`是一个查询操作返回：
   1. `ExecuteQuery(operation, schema, coercedVariableValues, initialValue)`
4. 否则`operation`是一个修改操作：
   1. 返回`ExecuteMutation(operation, schema, coercedVariableValues, initialValue)`

`GetOperation(document, operationName)`：

1. 如果`operationName`是`null`：
   1. 如果文档仅包含一个操作
      1. 返回该操作
   2. 否则产生一个需要`operationName`的查询错误
2. 否则
   1. `operation`设为文档中命名为`operationName`的操作
   2. 如果`operation`没有找到，产生一个查询错误
   3. 返回`operation`

#### 6.1.1 验证请求

按照“Validation”章节中的解释，只有通过所有验证规则的请求才会被执行。如果验证错误是已知的，应该记录在响应的`errors`列表字段中，而请求必须不执行且失败。

典型地，验证在执行前的请求环境中立即被执行，但如果一个请求和之前验证过的请求完全一样的话，一个GraphQL服务可能跳过立即的验证来执行一个请求。一个GraphQL服务应该仅执行那些之前已经通过验证且没有修改过的请求。

例如：请求可能在开发阶段验证过了，如果以后再也没有修改过，或一个服务可能对一个请求执行过一次验证，记录了验证结果以避免未来重复验证相同的请求。

#### 6.1.2 变量值的强制类型转换

如果操作定义了变量，那么这些变量的值，需要使用变量声明类型的输入转换规则进行转换。如果在转换过程中碰到任何的查询错误，则操作不执行且失败。  
`CoerceVariableValues(schema, operation, variableValues)`：

1. `coercedValues`设为一个空的无序Map
2. `variableDefinitions`设为`operation`定义的变量
3. 对`variableDefinitions`中的每个`variableDefinition`
   1. `variableName`设为`variableDefinition`的名称
   2. `variableType`设为`variableDefinition`预期的类型
   3. `defaultValue`设为`variableDefinition`的默认值
   4. `value`设为`variableValues`中名称`variableName`对应的值
   5. 如果`value`不存在（`variableValues`中没有提供）
      1. 如果`defaultValue`存在（包括`null`）
         1. 添加一条记录到`coercedValues`,名称为`variableName`，值为`defaultValue`
      2. 否则，如果`variableType`是一个Non-Nullable类型，抛出一个查询错误
      3. 否则，继续处理下一个变量定义
   6. 否则，如果`value`不能按照`variableType`的输入转换规则进行类型转换，抛出一个查询错误
   7. `coercedValue`设为`value`按照`variableType`的输入转换规则进行类型转换的结果
   8. 添加一条记录到`coercedValues`,名称为`variableName`，值为`coercedValue`
4. 返回`coercedValues`

> NOTE 该算法和`CoerceArgumentValues()`非常类似

### 6.2 执行操作 Executing Operations

按照本规范的”Type System”章节描述的类型系统，必须提供一个查询（Query）根对象类型。如果支持修改（Mutation），还要提供一个修改根对象类型。

一个初始值可以在执行一个查询时提供：

`ExecuteQuery(query, schema, variableValues, initialValue)`：

1. `queryType`设为`schema`的根查询（Query）类型
2. 断言：`queryType`是一个Object类型
3. `selectionSet`设为`query`中的顶级选择集（Selection Set）
4. `data`设为正常运行（允许并行化）`ExecuteSelectionSet(selectionSet, queryType, initialValue, variableValues)`的结果
5. `errors`设为执行选择集时产生的任何`errors`字段[^1]
6. 返回一个无序Map包含`data`和`errors`

如果操作是一个修改，操作的结果是在修改根对象类型上执行操作的顶级选择集的结果。该选择集应该是串行化执行的。

在一个修改操作中，顶级字段会在底层的数据系统中产生副作用（side-effect）。这些修改的串行化执行避免了产生这些副作用时出现竞态条件（race condition）。  
`ExecuteMutation(mutation, schema, variableValues, initialValue)`：

1. `mutationType`设为`schema`的根Mutation类型
2. 断言：`mutationType`是一个Object类型
3. `selectionSet`设为`mutation`中的顶级选择集（Selection Set）
4. `data`设为正常运行（允许并行化）`ExecuteSelectionSet(selectionSet, mutationType, initialValue, variableValues)`的结果
5. `errors`设为执行选择集时产生的任何`errors`字段[^1]
6. 返回一个无序Map包含`data`和`error`

### 6.3 执行选择集 Executing Selection Sets

无论是并行还是串行执行一个选择，都需要知道被求值的对象的值和类型。

首先，选择集被转成一个分组的字段集，然后分组的字段集中每个被描述的字段都会生成一条记录存储到响应Map。

`ExecuteSelectionSet(selectionSet, objectType, objectValue, variableValues)`:

1. `groupedFieldSet`设为`CollectFields(objectType, selectionSet, variableValues)`的结果
2. 初始化`resultMap`为一个空的无序Map
3. 取`groupedFieldSet`中的每一条记录，名字和值分别为`responseKey`和`fields`：
   1. `fieldName`设为`fields`的第一条记录的名字。注意：如果使用了别名，该值不会受影响。
   2. fieldType设为 `objectType`中`fieldName`的值指定的字段定义返回类型
   3. 如果`fieldType`是**null**：
      1. 继续处理`groupedFieldSet`中的下一条记录
   4. `responseValue`设为执行`ExecuteField(objectType, objectValue, fields, fieldType, variableValues)`
   5. 在`resultMap`中添加一条记录，名字为`responseKey`，值为`responseValue`
4. 返回`resultMap`

> NOTE `responseMap`是按字段出现在第一次出现在查询中的次序排序。在“6.3.2 字段集合”章节会有更详细的解释。

#### 6.3.1 标准执行和串行执行 Normal and Serial Execution

通常执行器能够按其选择的任何次序（通常是并行）执行一个分组字段集内所有字段查询。因为字段的解析和顶层的修改字段不同，必须总是不受副作用的影响且幂等，执行次序不应该影响执行结果，而且因此服务器要有选择其认为最优的次序来执行字段查询的自由。

例如，指定以下分组字段集正常地执行：

```
{
  birthday {
    month
  }
  address {
    street
  }
} 
```
For example, given the following grouped field set to be executed normally:

{  
  birthday {  
    month  
  }  
  address {  
    street  
  }  
}  
A valid GraphQL executor can resolve the four fields in whatever order it chose \(however of course birthday must be resolved before month, and address before street\).

When executing a mutation, the selections in the top most selection set will be executed in serial order.

When executing a grouped field set serially, the executor must consider each entry from the grouped field set in the order provided in the grouped field set. It must determine the corresponding entry in the result map for each item to completion before it continues on to the next item in the grouped field set:

For example, given the following selection set to be executed serially:

{  
  changeBirthday\(birthday: $newBirthday\) {  
    month  
  }  
  changeAddress\(address: $newAddress\) {  
    street  
  }  
}  
The executor must, in serial:

Run ExecuteField\(\) for changeBirthday, which during CompleteValue\(\) will execute the { month } sub‐selection set normally.  
Run ExecuteField\(\) for changeAddress, which during CompleteValue\(\) will execute the { street } sub‐selection set normally.  
As an illustrative example, let’s assume we have a mutation field changeTheNumber that returns an object containing one field, theNumber. If we execute the following selection set serially:

{  
  first: changeTheNumber\(newNumber: 1\) {  
    theNumber  
  }  
  second: changeTheNumber\(newNumber: 3\) {  
    theNumber  
  }  
  third: changeTheNumber\(newNumber: 2\) {  
    theNumber  
  }  
}  
The executor will execute the following serially:

Resolve the changeTheNumber\(newNumber: 1\) field  
Execute the { theNumber } sub‐selection set of first normally  
Resolve the changeTheNumber\(newNumber: 3\) field  
Execute the { theNumber } sub‐selection set of second normally  
Resolve the changeTheNumber\(newNumber: 2\) field  
Execute the { theNumber } sub‐selection set of third normally  
A correct executor must generate the following result for that selection set:

{  
  "first": {  
    "theNumber": 1  
  },  
  "second": {  
    "theNumber": 3  
  },  
  "third": {  
    "theNumber": 2  
  }  
}  
6.3.2Field Collection

Before execution, the selection set is converted to a grouped field set by calling CollectFields\(\). Each entry in the grouped field set is a list of fields that share a response key. This ensures all fields with the same response key \(alias or field name\) included via referenced fragments are executed at the same time.

As an example, collecting the fields of this selection set would collect two instances of the field a and one of field b:

{  
  a {  
    subfield1  
  }  
  ...ExampleFragment  
}

fragment ExampleFragment on Query {  
  a {  
    subfield2  
  }  
  b  
}  
The depth‐first‐search order of the field groups produced by CollectFields\(\) is maintained through execution, ensuring that fields appear in the executed response in a stable and predictable order.

CollectFields\(objectType, selectionSet, variableValues, visitedFragments\)  
If visitedFragments if not provided, initialize it to the empty set.  
Initialize groupedFields to an empty ordered map of lists.  
For each selection in selectionSet:  
If selection provides the directive @skip, let skipDirective be that directive.  
If skipDirective‘s if argument is true or is a variable in variableValues with the value true, continue with the next selection in selectionSet.  
If selection provides the directive @include, let includeDirective be that directive.  
If includeDirective‘s if argument is not true and is not a variable in variableValues with the value true, continue with the next selection in selectionSet.  
If selection is a Field:  
Let responseKey be the response key of selection.  
Let groupForResponseKey be the list in groupedFields for responseKey; if no such list exists, create it as an empty list.  
Append selection to the groupForResponseKey.  
If selection is a FragmentSpread:  
Let fragmentSpreadName be the name of selection.  
If fragmentSpreadName is in visitedFragments, continue with the next selection in selectionSet.  
Add fragmentSpreadName to visitedFragments.  
Let fragment be the Fragment in the current Document whose name is fragmentSpreadName.  
If no such fragment exists, continue with the next selection in selectionSet.  
Let fragmentType be the type condition on fragment.  
If DoesFragmentTypeApply\(objectType, fragmentType\) is false, continue with the next selection in selectionSet.  
Let fragmentSelectionSet be the top‐level selection set of fragment.  
Let fragmentGroupedFieldSet be the result of calling CollectFields\(objectType, fragmentSelectionSet, visitedFragments\).  
For each fragmentGroup in fragmentGroupedFieldSet:  
Let responseKey be the response key shared by all fields in fragmentGroup  
Let groupForResponseKey be the list in groupedFields for responseKey; if no such list exists, create it as an empty list.  
Append all items in fragmentGroup to groupForResponseKey.  
If selection is an InlineFragment:  
Let fragmentType be the type condition on selection.  
If fragmentType is not null and DoesFragmentTypeApply\(objectType, fragmentType\) is false, continue with the next selection in selectionSet.  
Let fragmentSelectionSet be the top‐level selection set of selection.  
Let fragmentGroupedFieldSet be the result of calling CollectFields\(objectType, fragmentSelectionSet, variableValues, visitedFragments\).  
For each fragmentGroup in fragmentGroupedFieldSet:  
Let responseKey be the response key shared by all fields in fragmentGroup  
Let groupForResponseKey be the list in groupedFields for responseKey; if no such list exists, create it as an empty list.  
Append all items in fragmentGroup to groupForResponseKey.  
Return groupedFields.  
DoesFragmentTypeApply\(objectType, fragmentType\)  
If fragmentType is an Object Type:  
if objectType and fragmentType are the same type, return true, otherwise return false.  
If fragmentType is an Interface Type:  
if objectType is an implementation of fragmentType, return true otherwise return false.  
If fragmentType is a Union:  
if objectType is a possible type of fragmentType, return true otherwise return false.  
6.4Executing Fields

Each field requested in the grouped field set that is defined on the selected objectType will result in an entry in the response map. Field execution first coerces any provided argument values, then resolves a value for the field, and finally completes that value either by recursively executing another selection set or coercing a scalar value.

ExecuteField\(objectType, objectValue, fieldType, fields, variableValues\)  
Let field be the first entry in fields.  
Let argumentValues be the result of CoerceArgumentValues\(objectType, field, variableValues\)  
Let resolvedValue be ResolveFieldValue\(objectType, objectValue, fieldName, argumentValues\).  
Return the result of CompleteValue\(fieldType, fields, resolvedValue, variableValues\).  
6.4.1Coercing Field Arguments

Fields may include arguments which are provided to the underlying runtime in order to correctly produce a value. These arguments are defined by the field in the type system to have a specific input type: Scalars, Enum, Input Object, or List or Non‐Null wrapped variations of these three.

At each argument position in a query may be a literal value or a variable to be provided at runtime.

CoerceArgumentValues\(objectType, field, variableValues\)  
Let coercedValues be an empty unordered Map.  
Let argumentValues be the argument values provided in field.  
Let fieldName be the name of field.  
Let argumentDefinitions be the arguments defined by objectType for the field named fieldName.  
For each argumentDefinition in argumentDefinitions:  
Let argumentName be the name of argumentDefinition.  
Let argumentType be the expected type of argumentDefinition.  
Let defaultValue be the default value for argumentDefinition.  
Let value be the value provided in argumentValues for the name argumentName.  
If value is a Variable:  
Let variableName be the name of Variable value.  
Let variableValue be the value provided in variableValues for the name variableName.  
If variableValue exists \(including null\):  
Add an entry to coercedValues named argName with the value variableValue.  
Otherwise, if defaultValue exists \(including null\):  
Add an entry to coercedValues named argName with the value defaultValue.  
Otherwise, if argumentType is a Non‐Nullable type, throw a field error.  
Otherwise, continue to the next argument definition.  
Otherwise, if value does not exist \(was not provided in argumentValues:  
If defaultValue exists \(including null\):  
Add an entry to coercedValues named argName with the value defaultValue.  
Otherwise, if argumentType is a Non‐Nullable type, throw a field error.  
Otherwise, continue to the next argument definition.  
Otherwise, if value cannot be coerced according to the input coercion rules of argType, throw a field error.  
Let coercedValue be the result of coercing value according to the input coercion rules of argType.  
Add an entry to coercedValues named argName with the value coercedValue.  
Return coercedValues.  
Variable values are not coerced because they are expected to be coerced before executing the operation in CoerceVariableValues\(\), and valid queries must only allow usage of variables of appropriate types.  
6.4.2Value Resolution

While nearly all of GraphQL execution can be described generically, ultimately the internal system exposing the GraphQL interface must provide values. This is exposed via ResolveFieldValue, which produces a value for a given field on a type for a real value.

As an example, this might accept the objectType Person, the field "soulMate", and the objectValue representing John Lennon. It would be expected to yield the value representing Yoko Ono.

ResolveFieldValue\(objectType, objectValue, fieldName, argumentValues\)  
Let resolver be the internal function provided by objectType for determining the resolved value of a field named fieldName.  
Return the result of calling resolver, providing objectValue and argumentValues.  
It is common for resolver to be asynchronous due to relying on reading an underlying database or networked service to produce a value. This necessitates the rest of a GraphQL executor to handle an asynchronous execution flow.  
6.4.3Value Completion

After resolving the value for a field, it is completed by ensuring it adheres to the expected return type. If the return type is another Object type, then the field execution process continues recursively.

CompleteValue\(fieldType, fields, result, variableValues\)  
If the fieldType is a Non‐Null type:  
Let innerType be the inner type of fieldType.  
Let completedResult be the result of calling CompleteValue\(innerType, fields, result, variableValues\).  
If completedResult is null, throw a field error.  
Return completedResult.  
If result is null \(or another internal value similar to null such as undefined or NaN\), return null.  
If fieldType is a List type:  
If result is not a collection of values, throw a field error.  
Let innerType be the inner type of fieldType.  
Return a list where each list item is the result of calling CompleteValue\(innerType, fields, resultItem, variableValues\), where resultItem is each item in result.  
If fieldType is a Scalar or Enum type:  
Return the result of “coercing” result, ensuring it is a legal value of fieldType, otherwise null.  
If fieldType is an Object, Interface, or Union type:  
If fieldType is an Object type.  
Let objectType be fieldType.  
Otherwise if fieldType is an Interface or Union type.  
Let objectType be ResolveAbstractType\(fieldType, result\).  
Let subSelectionSet be the result of calling MergeSelectionSets\(fields\).  
Return the result of evaluating ExecuteSelectionSet\(subSelectionSet, objectType, result, variableValues\) normally \(allowing for parallelization\).  
Resolving Abstract Types

When completing a field with an abstract return type, that is an Interface or Union return type, first the abstract type must be resolved to a relevant Object type. This determination is made by the internal system using whatever means appropriate.

A common method of determining the Object type for an objectValue in object‐oriented environments, such as Java or C\#, is to use the class name of the objectValue.  
ResolveAbstractType\(abstractType, objectValue\)  
Return the result of calling the internal method provided by the type system for determining the Object type of abstractType given the value objectValue.  
Merging Selection Sets

When more than one fields of the same name are executed in parallel, their selection sets are merged together when completing the value in order to continue execution of the sub‐selection sets.

An example query illustrating parallel fields with the same name with sub‐selections.

{  
  me {  
    firstName  
  }  
  me {  
    lastName  
  }  
}  
After resolving the value for me, the selection sets are merged together so firstName and lastName can be resolved for one value.

MergeSelectionSets\(fields\)  
Let selectionSet be an empty list.  
For each field in fields:  
Let fieldSelectionSet be the selection set of field.  
If fieldSelectionSet is null or empty, continue to the next field.  
Append all selections in fieldSelectionSet to selectionSet.  
Return selectionSet.  
6.4.4Errors and Non-Nullability

If an error is thrown while resolving a field, it should be treated as though the field returned null, and an error must be added to the "errors" list in the response.

If the result of resolving a field is null \(either because the function to resolve the field returned null or because an error occurred\), and that field is of a Non-Null type, then a field error is thrown. The error must be added to the "errors" list in the response.

If the field returns null because of an error which has already been added to the "errors" list in the response, the "errors" list must not be further affected. That is, only one error should be added to the errors list per field.

Since Non-Null type fields cannot be null, field errors are propagated to be handled by the parent field. If the parent field may be null then it resolves to null, otherwise if it is a Non-Null type, the field error is further propagated to it’s parent field.

If all fields from the root of the request to the source of the error return Non-Null types, then the "data" entry in the response should be null.

# graphql



