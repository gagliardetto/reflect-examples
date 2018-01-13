# reflect-cheat-sheet

### Table Of Content
- [Read struct tags](#read-struct-tags)
- [Get and set struct fields](#get-and-set-struct-fields)
- [Function calls](#function-calls)
  - [Call to a method without prameters, and without return value](#call-method-without-prameters-and-without-return-value)
  - [Call to a function with list of arguments, and validate return values](#call-function-with-list-of-arguments-and-validate-return-values)
  - [Call to a function with variadic parameter](#call-function-with-variadic-parameter)
  - [Create function at runtime](#create-function-dynamically)
- [Fill slice with values](#fill-slice-with-strings-without-knowing-its-type-use-case-decoder)
- [Set a value of a number](#set-a-value-of-a-number-use-case-decoder)

### Read struct tags

```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email  string `mcl:"email"`
	Name   string `mcl:"name"`
	Age    int    `mcl:"age"`
	Github string `mcl:"github" default:"a8m"`
}

func main() {
	var u User
	t := reflect.TypeOf(u)
	if t.Kind() != reflect.Struct {
		return
	}
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Println(f.Tag.Get("mcl"), f.Tag.Get("default"))
	}
}
```

### Get and set struct fields
```go
package main

import (
	"fmt"
	"reflect"
)

type User struct {
	Email  string `mcl:"email"`
	Name   string `mcl:"name"`
	Age    int    `mcl:"age"`
	Github string `mcl:"github" default:"a8m"`
}

func main() {
	u := &User{Name: "Ariel Mashraki"}
	v := reflect.ValueOf(u).Elem()
	f := v.FieldByName("Github")
	if !f.IsValid() || !f.CanSet() {
		return
	}
	if f.Kind() != reflect.String || f.String() != "" {
		return
	}
	f.SetString("a8m")
	fmt.Printf("Github username was changed to: %q\n", u.Github)
}
```

### Function calls

#### Call method without prameters, and without return value
```go
package main

import (
	"fmt"
	"reflect"
)

type A struct{}

func (A) Hello() { fmt.Println("World") }

func main() {
	v := reflect.ValueOf(A{})
	m := v.MethodByName("Hello")
	if m.Kind() != reflect.Func {
		return
	}
	m.Call(nil)
}
```

#### Call function with list of arguments, and validate return values
```go
package main

import (
	"fmt"
	"reflect"
)

func Add(a, b int) int { return a + b }

func main() {
	v := reflect.ValueOf(Add)
	if v.Kind() != reflect.Func {
		return
	}
	t := v.Type()
	argv := make([]reflect.Value, t.NumIn())
	for i := range argv {
		if t.In(i).Kind() != reflect.Int {
			return
		}
		argv[i] = reflect.ValueOf(i)
	}
	// note that, len(result) == t.NumOut()
	result := v.Call(argv)
	if len(result) != 1 || result[0].Kind() != reflect.Int {
		return
	}
	fmt.Println(result[0].Int())
}
```

#### Call function with variadic parameter
```go
package main

import (
	"fmt"
	"math/rand"
	"reflect"
)

func Sum(x1, x2 int, xs ...int) int {
	sum := x1 + x2
	for _, xi := range xs {
		sum += xi
	}
	return sum
}

func main() {
	v := reflect.ValueOf(Sum)
	if v.Kind() != reflect.Func {
		return
	}
	t := v.Type()
	argc := t.NumIn()
	if t.IsVariadic() {
		argc += rand.Intn(10)
	}
	argv := make([]reflect.Value, argc)
	for i := range argv {
		argv[i] = reflect.ValueOf(i)
	}
	result := v.Call(argv)
	fmt.Println(result[0].Int()) // assume that t.NumOut() > 0 tested above.
}
```

### Create function at runtime
```go
package main

import (
	"fmt"
	"reflect"
)

type Add func(int64, int64) int64

func main() {
	t := reflect.TypeOf(Add(nil))
	mul := reflect.MakeFunc(t, func(args []reflect.Value) []reflect.Value {
		a := args[0].Int()
		b := args[1].Int()
		return []reflect.Value{reflect.ValueOf(a+b)}
	})
	fn, ok := mul.Interface().(Add)
	if !ok {
		return
	}
	fmt.Println(fn(2,3))
}
```

### Fill slice with strings, without knowing its type. Use case: decoder.
```go
package main

import (
	"fmt"
	"io"
	"reflect"
)

func main() {
	var (
		a []string
		b []interface{}
		c []io.Writer
	)
	fmt.Println(fill(&a), a) // pass
	fmt.Println(fill(&b), b) // pass
	fmt.Println(fill(&c), c) // fail
}

func fill(i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	if v.Kind() != reflect.Slice {
		return fmt.Errorf("can't fill non-slice value")
	}
	v.Set(reflect.MakeSlice(v.Type(), 3, 3))
	// validate the type of the slice.
	if !canAssign(v.Index(0)) {
		return fmt.Errorf("can't assign string to slice elements")
	}
	for i, w := range []string{"foo", "bar", "baz"} {
		v.Index(i).Set(reflect.ValueOf(w))
	}
	return nil
}

func canAssign(v reflect.Value) bool {
	return v.Kind() == reflect.String || (v.Kind() == reflect.Interface && v.NumMethod() == 0)
}
```

### Set a value of a number. Use case: decoder. 
```go
package main

import (
	"fmt"
	"reflect"
)

const n = 255

func main() {
	var (
		a int8
		b int16
		c uint
		d float32
		e string
	)
	fmt.Println(fill(&a), a)
	fmt.Println(fill(&b), b)
	fmt.Println(fill(&c), c)
	fmt.Println(fill(&d), c)
	fmt.Println(fill(&e), e)
}

func fill(i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr {
		return fmt.Errorf("non-pointer %v", v.Type())
	}
	// get the value that the pointer v points to.
	v = v.Elem()
	switch v.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		if v.OverflowInt(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetInt(n)
	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		if v.OverflowUint(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetUint(n)
	case reflect.Float32, reflect.Float64:
		if v.OverflowFloat(n) {
			return fmt.Errorf("can't assign value due to %s-overflow", v.Kind())
		}
		v.SetFloat(n)
	default:
		return fmt.Errorf("can't assign value to a non-number type")
	}
	return nil
}
```
