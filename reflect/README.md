# Go 语言中的反射

## 什么是反射

反射是指一种自描述和自控制的能力，通过采用某种机制来实现对自己行为的描述和监测，并能根据自身行为的状态和结果，调整或修改所描述行为的状态和相关语义。

Go语言实现了反射，即在运行时动态的调用对象的方法，属性和类型，Go语言的 reflect 包即包含了反射相关的工具。

## 为什么需要反射

- 如果需要动态的调用对象的方法，此时可以使用反射来实现
- 需要根据传入的参数的特定类型执行不同的动作

## reflect 包

### 变量类型

在Go语言中一个变量包括(type, value)两个部分，type又可以区分为static type和concrete type。

- static type: 编码时变量的类型，例如int，string等
- concrete type: 运行时系统看见的类型

### reflect.Type

可以通过 reflect.TypeOf 来获取变量的类型信息:

```
v := 1
t := reflect.TypeOf(v)
```

### reflect.Value

可以通过 reflect.ValueOf 来获取变量的值信息:

```
v := 1
value := reflect.ValueOf(v)
```

当获取到变量的值信息之后，可以通过对应的 Interface 方法获取具体的值，然后通过 type assert 获取具体的值。

```
v0, ok := value.Interface().(int)
```

### Type 和 Kind

考虑以下代码，[例子1](https://play.golang.org/p/brWQ97Ez1OT):

```
package main

import (
	"fmt"
	"reflect"
)

func demo01() {
	type example struct {
		field1 string
		field2 int
	}
	
	r := reflect.TypeOf(example{})
	fmt.Println(r.String())
	fmt.Println(r.Kind())
	
	type VersionType string
	var Version VersionType
	r = reflect.TypeOf(Version)
	fmt.Println(r.String())
	fmt.Println(r.Kind())
}

func main() {
	demo01()
}
```

从以上输出可以看出，Type 可以理解为程序中定义的类型元数据，Kind是编译器和Go运行时定义的元数据，用于为变量分配内存布局:

- 复合类型，例如Pointer，Array等具有不同的Type和Kind
- 简单类型，例如int，float等具有相同的Type和Kind

### 通过反射调用函数的实例

最后，考虑以下的通过反射来调用函数的实例，[例子2](https://play.golang.org/p/KlDB54DmP1U):

```
package main

import (
	"fmt"
	"reflect"
	"strconv"
)

func call(function interface{}, args ...string) {
	value := reflect.ValueOf(function)
	if value.Kind() != reflect.Func {
		fmt.Println("function is not function")
		return
	}

	parameters := make([]reflect.Type, 0, value.Type().NumIn())
	for i := 0; i < value.Type().NumIn(); i++ {
		arg := value.Type().In(i)
		fmt.Printf("argument %d is %s[%s] type \n", i, arg.Kind(), arg.Name())
		parameters = append(parameters, arg)
	}

	if value.Type().NumIn() != len(args) {
		fmt.Printf("argument %d length doesn't equal to provide length %d \n", value.Type().NumIn(), len(args))
		return
	}

	outs := make([]reflect.Type, 0, value.Type().NumOut())
	for i := 0; i < value.Type().NumOut(); i++ {
		arg := value.Type().Out(i)
		fmt.Printf("out %d is %s[%s] type \n", i, arg.Kind(), arg.Name())
		outs = append(outs, arg)
	}

	if value.Type().NumOut() < 1 {
		fmt.Println("outs length must greater than 0")
		return
	}

	if !outs[len(outs)-1].AssignableTo(reflect.TypeOf((*error)(nil)).Elem()) {
		fmt.Println("last output must be error")
		return
	}
	if !outs[len(outs)-1].Implements(reflect.TypeOf((*error)(nil)).Elem()) {
		fmt.Println("last output must be error")
		return
	}

	argValues := make([]reflect.Value, 0, len(parameters))
	for i := 0; i < len(args); i++ {
		switch parameters[i] {
		case reflect.TypeOf(int(0)):
			v, err := strconv.ParseInt(args[i], 10, 64)
			if err != nil {
				fmt.Printf("argument %d %s convert int failed, %v \n", i, args[i], err)
				return
			}
			argValues = append(argValues, reflect.ValueOf(int(v)))
		case reflect.TypeOf(string("")):
			argValues = append(argValues, reflect.ValueOf(args[i]))
		default:
			fmt.Printf("unsupport type %s[%s] \n", parameters[i].Kind(), parameters[i].Name())
			return
		}
	}

	resultValues := value.Call(argValues)
	for i := 0; i < len(resultValues); i++ {
		switch resultValues[i].Type() {
		case reflect.TypeOf(int(0)):
			fmt.Println("result: ", i, ", ", resultValues[i].Interface().(int))
		case reflect.TypeOf(string("")):
			fmt.Println("result: ", i, ", ", resultValues[i].Interface().(string))
		default:
			fmt.Printf("type: %s[%s], value: %v \n", resultValues[i].Type().Kind(), resultValues[i].Type().Name(), resultValues[i].Interface())
		}
	}
}

func f1(arg1 string, arg2 int) (string, error) {
	fmt.Println("arg1: ", arg1, "  arg2: ", arg2)
	return "f1", nil
}

func main() {
	call(f1, "", "100")
}
```

1. 通过反射获取函数的值

可以通过 reflect.ValueOf 来获取对象的值，然后通过 Kind 来判断对象是否是一个函数.

2. 判断参数个数

通过 reflect.Type 的 NumIn 函数获取对象的参数个数，对比与传入参数是否相等来判断参数个数是否符合。注意此处没有考虑变长参数的情况。

3. 校验最后的返回值是否为 error

通过 **AssignableTo(reflect.TypeOf((*error)(nil)).Elem())** 来判断函数的最后一个返回值是否可以转换为 error 类型，或者通过 Implements 来判断是否实现了 error 接口。此处是强制要求函数必须返回一个 error。

4. 转换传入的参数列表

将传入的参数列表转换为对应的形参类型，在示例中只支持 int 和 string 类型的参数。

5. 调用函数

简单的通过 Call 函数即可实现函数的动态调用。