参考连接: https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode

### mapstructure 映射与结构体 [![Godoc](https://godoc.org/github.com/mitchellh/mapstructure?status.svg)](https://godoc.org/github.com/mitchellh/mapstructure)

mapstructure is a Go library for decoding generic map values to structures and vice versa反之亦然, while providing helpful error handling.

This library is most useful when decoding values from some data stream (JSON, Gob, etc.) where you don't *quite* know the structure of the underlying data until you read a part of it. 当你从一些数据流(如:JSON, Gob等)解码, 你想从数据中获取一部分数据而不想了解具体的底层结构时, 非常有用,  You can therefore read a `map[string]interface{}` and use this library to decode it into the proper underlying native Go structure 你可以使用该库将`map[string]interface{}`类型解码为原生Go结构体.

#### Installation

Standard `go get`:

```
$ go get github.com/mitchellh/mapstructure
```

#### Usage & Example

For usage and examples see the [Godoc](http://godoc.org/github.com/mitchellh/mapstructure).

The `Decode` function has examples associated with it there.

#### But Why?!

Go offers fantastic standard libraries for decoding formats such as JSON. The standard method is to have a struct pre-created, and populate that struct from the bytes of the encoded format. This is great, but the problem is if you have configuration or an encoding that changes slightly depending on specific fields 但是如果你遇到的数据会根据一些字段而略有变化. For example, consider this JSON:

```
{
  "type": "person",
  "name": "Mitchell"
}
```

Perhaps we can't populate a specific structure without first reading the "type" field from the JSON 根据type类型决定后面字段数据. We could always do two passes over the decoding of the JSON (reading the "type" first, and the rest later). However, it is much simpler to just decode this into a `map[string]interface{}` structure, read the "type" key, then use something like this library to decode it into the proper structure.

## ![img](https://pkg.go.dev/static/img/pkg-icon-doc_20x12.svg)Documentation

### [Overview概览](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-overview)

Package mapstructure exposes functionality to convert one arbitrary Go type into another, typically to convert a map[string]interface{} into a native Go structure. (`mapstructure`包暴露了有用的方法, 将任意的go类型转化为另一种类型, 典型应用是将`map[string]interface{}`类型转化为原生Go结构体)

The Go structure can be arbitrarily complex , containing slices, other structs, etc. and the decoder will properly decode nested maps and so on into the proper structures in the native Go struct. See the examples to see what the decoder is capable of. Go结构体可以是复合多变, 由切片组成,或其他结构嵌套组成等, 解码器会正确的解码嵌套map为结构体.

The simplest function to start with is Decode.

#### Field Tags [字段标签](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Field_Tags)

When decoding to a struct, mapstructure will use the field name by default to perform the mapping 默认使用字段名来映射解析(不区分大小写). For example, if a struct has a field "Username" then mapstructure will look for a key in the source value of "username" (case insensitive) 比如map结构中小写`username`与结构体中的Username字段映射.

```
type User struct {
    Username string
}
```

You can change the behavior of mapstructure by using struct tags. The default struct tag that mapstructure looks for is "mapstructure" but you can customize it using DecoderConfig. 你可以在结构体中添加标签来改变解析行为, 默认使用标签`mapstructure`.

#### Renaming Fields [字段重命名](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Renaming_Fields)

To rename the key that mapstructure looks for, use the "mapstructure" tag and set a value directly. For example, to change the "username" example above to "user":

```
type User struct {
    Username string `mapstructure:"user"`
}
```

#### Embedded Structs and Squashing [嵌套结构和压缩简化](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Embedded_Structs_and_Squashing)

Embedded structs are treated as if they're another field with that name. By default, the two structs below are equivalent when decoding with mapstructure:

```
type Person struct {
    Name string
}
//等效1
type Friend struct {
    Person //匿名嵌套
}

//等效2
type Friend struct {
    Person Person
}
```

This would require an input that looks like below:

```
map[string]interface{}{
    "person": map[string]interface{}{"name": "alice"},
}
```

If your "person" value is NOT nested, then you can append ",squash" to your tag value and mapstructure will treat it as if the embedded struct were part of the struct directly 使用`squash`标签将Person直接作为结构的一部分. Example:

```
type Friend struct {
    Person `mapstructure:",squash"`
}
```

Now the following input would be accepted:

```
map[string]interface{}{
    "name": "alice",
}
```

DecoderConfig has a field that changes the behavior of mapstructure to always squash embedded structs.

#### Remainder Values [剩下的值](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Remainder_Values)

If there are any unmapped keys in the source value, mapstructure by default will silently ignore them 默认忽略剩余的非map值. You can error by setting ErrorUnused in DecoderConfig. If you're using Metadata you can also maintain a slice of the unused keys.

You can also use the ",remain" suffix on your tag to collect all unused values in a map. The field with this tag MUST be a map type and should probably be a "map[string]interface{}" or "map[interface{}]interface{}". See example below:

```
type Friend struct {
    Name  string
    Other map[string]interface{} `mapstructure:",remain"`  //使用remain标签将剩余的键搜集到Other字段中
}
```

Given the input below, Other would be populated with the other values that weren't used (everything but "name"):

```
map[string]interface{}{
    "name":    "bob",
    "address": "123 Maple St.",
}
```

#### Omit Empty Values [忽略空值](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Omit_Empty_Values)

When decoding from a struct to any other value 将结构解码为其他类型, you may use the ",omitempty" suffix on your tag to omit that value if it equates to the zero value. The zero value of all types is specified in the Go specification.

For example, the zero type of a numeric type is zero ("0"). If the struct field value is zero and a numeric type, the field is empty, and it won't be encoded into the destination type.

```
type Source {
    Age int `mapstructure:",omitempty"`
}
```

#### Unexported fields [不暴露的私有字段(小写)](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Unexported_fields)

Since unexported (private) struct fields cannot be set outside the package where they are defined, the decoder will simply skip them.

For this output type definition:

```
type Exported struct {
    private string // this unexported field will be skipped
    Public string
}
```

Using this map as input:

```
map[string]interface{}{
    "private": "I will be ignored",
    "Public":  "I made it through!",
}
```

The following struct will be decoded:

```
type Exported struct {
    private: "" // field is left with an empty string (zero value)
    Public: "I made it through!"
}
```

#### Other Configuration [其他配置](https://pkg.go.dev/github.com/mitchellh/mapstructure#hdr-Other_Configuration)

mapstructure is highly configurable. See the DecoderConfig struct for other features and options that are supported. 灵活配置

### Index [索引](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-index)

- [func Decode(input interface{}, output interface{}) error](https://pkg.go.dev/github.com/mitchellh/mapstructure#Decode)
- [func DecodeHookExec(raw DecodeHookFunc, from reflect.Value, to reflect.Value) (interface{}, error)](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookExec)
- [func DecodeMetadata(input interface{}, output interface{}, metadata *Metadata) error](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeMetadata)
- [func WeakDecode(input, output interface{}) error](https://pkg.go.dev/github.com/mitchellh/mapstructure#WeakDecode)
- [func WeakDecodeMetadata(input interface{}, output interface{}, metadata *Metadata) error](https://pkg.go.dev/github.com/mitchellh/mapstructure#WeakDecodeMetadata)
- [func WeaklyTypedHook(f reflect.Kind, t reflect.Kind, data interface{}) (interface{}, error)](https://pkg.go.dev/github.com/mitchellh/mapstructure#WeaklyTypedHook)
- [type DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFunc)
- - [func ComposeDecodeHookFunc(fs ...DecodeHookFunc) DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#ComposeDecodeHookFunc)
  - [func RecursiveStructToMapHookFunc() DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#RecursiveStructToMapHookFunc)
  - [func StringToIPHookFunc() DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToIPHookFunc)
  - [func StringToIPNetHookFunc() DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToIPNetHookFunc)
  - [func StringToSliceHookFunc(sep string) DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToSliceHookFunc)
  - [func StringToTimeDurationHookFunc() DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToTimeDurationHookFunc)
  - [func StringToTimeHookFunc(layout string) DecodeHookFunc](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToTimeHookFunc)
- [type DecodeHookFuncKind](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFuncKind)
- [type DecodeHookFuncType](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFuncType)
- [type DecodeHookFuncValue](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFuncValue)
- [type Decoder](https://pkg.go.dev/github.com/mitchellh/mapstructure#Decoder)
- - [func NewDecoder(config *DecoderConfig) (*Decoder, error)](https://pkg.go.dev/github.com/mitchellh/mapstructure#NewDecoder)
- - [func (d *Decoder) Decode(input interface{}) error](https://pkg.go.dev/github.com/mitchellh/mapstructure#Decoder.Decode)
- [type DecoderConfig](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecoderConfig)
- [type Error](https://pkg.go.dev/github.com/mitchellh/mapstructure#Error)
- - [func (e *Error) Error() string](https://pkg.go.dev/github.com/mitchellh/mapstructure#Error.Error)
  - [func (e *Error) WrappedErrors() [\]error](https://pkg.go.dev/github.com/mitchellh/mapstructure#Error.WrappedErrors)
- [type Metadata](https://pkg.go.dev/github.com/mitchellh/mapstructure#Metadata)

#### Examples [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-examples)

- [Decode](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode)
- [Decode (EmbeddedStruct)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-EmbeddedStruct)
- [Decode (Errors)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Errors)
- [Decode (Metadata)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Metadata)
- [Decode (Omitempty)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Omitempty)
- [Decode (RemainingData)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-RemainingData)
- [Decode (Tags)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Tags)
- [Decode (WeaklyTypedInput)](https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-WeaklyTypedInput)

### Constants [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-constants)

This section is empty.

### Variables [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-variables)

This section is empty.

### Functions [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-functions)

#### func [Decode](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L271) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Decode)

```
func Decode(input interface{}, output interface{}) error
```

Decode takes an input structure and uses reflection to translate it to the output structure. output must be a pointer to a map or struct.

<details tabindex="-1" id="example-Decode" class="Documentation-exampleDetails js-exampleContainer" open="" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary><div class="Documentation-exampleDetailsBody" style="box-sizing: border-box;"><p style="box-sizing: border-box; font-size: 1rem; line-height: 1.5rem;">Code:</p><pre class="Documentation-exampleCode" style="box-sizing: border-box; font-size: 0.875rem; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, monospace; background-color: var(--gray-10); border-radius: 0.3em 0.3em 0px 0px; border: 0.0625rem solid rgb(204, 204, 204); margin: 0px; overflow-x: auto; padding: 0.625rem; tab-size: 4; line-height: 1.25rem; white-space: pre-wrap; word-break: break-all; overflow-wrap: break-word;">type Person struct {
	Name   string
	Age    int
	Emails []string
	Extra  map[string]string
}

<span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// This input can come from anywhere, but typically comes from</span>
<span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// something like decoding JSON where we're not quite sure of the</span>
<span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// struct initially.</span>
input := map[string]interface{}{
	"name":   "Mitchell",
	"age":    91,
	"emails": []string{"one", "two", "three"},
	"extra": map[string]string{
		"twitter": "mitchellh",
	},
}

var result Person
err := Decode(input, &amp;result)
if err != nil {
	panic(err)
}

fmt.Printf("%#v", result)
</pre><pre class="Documentation-exampleOutput" style="box-sizing: border-box; font-size: 0.875rem; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, monospace; background-color: var(--gray-10); border-radius: 0px 0px 0.3em 0.3em; border: 0.0625rem solid rgb(204, 204, 204); margin: 0px 0px 0.5rem; overflow-x: auto; padding: 0.625rem; tab-size: 4; line-height: 1.25rem; white-space: pre-wrap; word-break: break-all; overflow-wrap: break-word;">mapstructure.Person{Name:"Mitchell", Age:91, Emails:[]string{"one", "two", "three"}, Extra:map[string]string{"twitter":"mitchellh"}}
</pre></div></details>

<details tabindex="-1" id="example-Decode-EmbeddedStruct" class="Documentation-exampleDetails js-exampleContainer" open="" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (EmbeddedStruct)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-EmbeddedStruct" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary><div class="Documentation-exampleDetailsBody" style="box-sizing: border-box;"><p style="box-sizing: border-box; font-size: 1rem; line-height: 1.5rem;">Code:</p><pre class="Documentation-exampleCode" style="box-sizing: border-box; font-size: 0.875rem; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, monospace; background-color: var(--gray-10); border-radius: 0.3em 0.3em 0px 0px; border: 0.0625rem solid rgb(204, 204, 204); margin: 0px; overflow-x: auto; padding: 0.625rem; tab-size: 4; line-height: 1.25rem; white-space: pre-wrap; word-break: break-all; overflow-wrap: break-word;"><span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// Squashing multiple embedded structs is allowed using the squash tag.</span>
<span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// This is demonstrated by creating a composite struct of multiple types</span>
<span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// and decoding into it. In this case, a person can carry with it both</span>
<span class="comment" style="box-sizing: border-box; color: rgb(0, 102, 0);">// a Family and a Location, as well as their own FirstName.</span>
type Family struct {
	LastName string
}
type Location struct {
	City string
}
type Person struct {
	Family    `mapstructure:",squash"`
	Location  `mapstructure:",squash"`
	FirstName string
}

input := map[string]interface{}{
	"FirstName": "Mitchell",
	"LastName":  "Hashimoto",
	"City":      "San Francisco",
}

var result Person
err := Decode(input, &amp;result)
if err != nil {
	panic(err)
}

fmt.Printf("%s %s, %s", result.FirstName, result.LastName, result.City)
</pre><pre class="Documentation-exampleOutput" style="box-sizing: border-box; font-size: 0.875rem; font-family: SFMono-Regular, Consolas, &quot;Liberation Mono&quot;, Menlo, monospace; background-color: var(--gray-10); border-radius: 0px 0px 0.3em 0.3em; border: 0.0625rem solid rgb(204, 204, 204); margin: 0px 0px 0.5rem; overflow-x: auto; padding: 0.625rem; tab-size: 4; line-height: 1.25rem; white-space: pre-wrap; word-break: break-all; overflow-wrap: break-word;">Mitchell Hashimoto, San Francisco
</pre></div></details>

<details tabindex="-1" id="example-Decode-Errors" class="Documentation-exampleDetails js-exampleContainer" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (Errors)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Errors" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary></details>

<details tabindex="-1" id="example-Decode-Metadata" class="Documentation-exampleDetails js-exampleContainer" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (Metadata)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Metadata" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary></details>

<details tabindex="-1" id="example-Decode-Omitempty" class="Documentation-exampleDetails js-exampleContainer" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (Omitempty)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Omitempty" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary></details>

<details tabindex="-1" id="example-Decode-RemainingData" class="Documentation-exampleDetails js-exampleContainer" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (RemainingData)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-RemainingData" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary></details>

<details tabindex="-1" id="example-Decode-Tags" class="Documentation-exampleDetails js-exampleContainer" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (Tags)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-Tags" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary></details>

<details tabindex="-1" id="example-Decode-WeaklyTypedInput" class="Documentation-exampleDetails js-exampleContainer" style="box-sizing: border-box; margin-top: 1rem;"><summary class="Documentation-exampleDetailsHeader" style="box-sizing: border-box; color: var(--turq-dark); cursor: pointer; margin-bottom: 2rem; outline: none; text-decoration: none;">Example (WeaklyTypedInput)<span>&nbsp;</span><a href="https://pkg.go.dev/github.com/mitchellh/mapstructure#example-Decode-WeaklyTypedInput" style="box-sizing: border-box; color: var(--turq-dark); text-decoration: none; opacity: 0;">¶</a></summary></details>

#### func [DecodeHookExec](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L40) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookExec)

```
func DecodeHookExec(
	raw DecodeHookFunc,
	from reflect.Value, to reflect.Value) (interface{}, error)
```

DecodeHookExec executes the given decode hook. This should be used since it'll naturally degrade to the older backwards compatible DecodeHookFunc that took reflect.Kind instead of reflect.Type.

#### func [DecodeMetadata](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L304) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeMetadata)

```
func DecodeMetadata(input interface{}, output interface{}, metadata *Metadata) error
```

DecodeMetadata is the same as Decode, but is shorthand to enable metadata collection. See DecoderConfig for more info.

#### func [WeakDecode](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L287) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#WeakDecode)

```
func WeakDecode(input, output interface{}) error
```

WeakDecode is the same as Decode but is shorthand to enable WeaklyTypedInput. See DecoderConfig for more info.

#### func [WeakDecodeMetadata](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L321) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#WeakDecodeMetadata)

```
func WeakDecodeMetadata(input interface{}, output interface{}, metadata *Metadata) error
```

WeakDecodeMetadata is the same as Decode, but is shorthand to enable both WeaklyTypedInput and metadata collection. See DecoderConfig for more info.

#### func [WeaklyTypedHook](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L185) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#WeaklyTypedHook)

```
func WeaklyTypedHook(
	f reflect.Kind,
	t reflect.Kind,
	data interface{}) (interface{}, error)
```

WeaklyTypedHook is a DecodeHookFunc which adds support for weak typing to the decoder.

Note that this is significantly different from the WeaklyTypedInput option of the DecoderConfig.

### Types [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#pkg-types)

#### type [DecodeHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L173) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFunc)

```
type DecodeHookFunc interface{}
```

DecodeHookFunc is the callback function that can be used for data transformations. See "DecodeHook" in the DecoderConfig struct.

The type should be DecodeHookFuncType or DecodeHookFuncKind. Either is accepted. Types are a superset of Kinds (Types can return Kinds) and are generally a richer thing to use, but Kinds are simpler if you only need those.

The reason DecodeHookFunc is multi-typed is for backwards compatibility: we started with Kinds and then realized Types were the better solution, but have a promise to not break backwards compat so we now support both.

#### func [ComposeDecodeHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L61) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#ComposeDecodeHookFunc)

```
func ComposeDecodeHookFunc(fs ...DecodeHookFunc) DecodeHookFunc
```

ComposeDecodeHookFunc creates a single DecodeHookFunc that automatically composes multiple DecodeHookFuncs.

The composed funcs are called in order, with the result of the previous transformation.

#### func [RecursiveStructToMapHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L216) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#RecursiveStructToMapHookFunc)

```
func RecursiveStructToMapHookFunc() DecodeHookFunc
```

#### func [StringToIPHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L119) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToIPHookFunc)

```
func StringToIPHookFunc() DecodeHookFunc
```

StringToIPHookFunc returns a DecodeHookFunc that converts strings to net.IP

#### func [StringToIPNetHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L143) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToIPNetHookFunc)

```
func StringToIPNetHookFunc() DecodeHookFunc
```

StringToIPNetHookFunc returns a DecodeHookFunc that converts strings to net.IPNet

#### func [StringToSliceHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L80) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToSliceHookFunc)

```
func StringToSliceHookFunc(sep string) DecodeHookFunc
```

StringToSliceHookFunc returns a DecodeHookFunc that converts string to []string by splitting on the given sep.

#### func [StringToTimeDurationHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L100) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToTimeDurationHookFunc)

```
func StringToTimeDurationHookFunc() DecodeHookFunc
```

StringToTimeDurationHookFunc returns a DecodeHookFunc that converts strings to time.Duration.

#### func [StringToTimeHookFunc](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go#L163) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#StringToTimeHookFunc)

```
func StringToTimeHookFunc(layout string) DecodeHookFunc
```

StringToTimeHookFunc returns a DecodeHookFunc that converts strings to time.Time.

#### type [DecodeHookFuncKind](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L181) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFuncKind)

```
type DecodeHookFuncKind func(reflect.Kind, reflect.Kind, interface{}) (interface{}, error)
```

DecodeHookFuncKind is a DecodeHookFunc which knows only the Kinds of the source and target types.

#### type [DecodeHookFuncType](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L177) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFuncType)

```
type DecodeHookFuncType func(reflect.Type, reflect.Type, interface{}) (interface{}, error)
```

DecodeHookFuncType is a DecodeHookFunc which has complete information about the source and target types.

#### type [DecodeHookFuncValue](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L185) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecodeHookFuncValue)

```
type DecodeHookFuncValue func(from reflect.Value, to reflect.Value) (interface{}, error)
```

DecodeHookFuncRaw is a DecodeHookFunc which has complete access to both the source and target values.

#### type [Decoder](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L254) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Decoder)

```
type Decoder struct {
	// contains filtered or unexported fields

}
```

A Decoder takes a raw interface value and turns it into structured data, keeping track of rich error information along the way in case anything goes wrong. Unlike the basic top-level Decode method, you can more finely control how the Decoder behaves using the DecoderConfig structure. The top-level Decode method is just a convenience that sets up the most basic Decoder.

#### func [NewDecoder](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L339) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#NewDecoder)

```
func NewDecoder(config *DecoderConfig) (*Decoder, error)
```

NewDecoder returns a new decoder for the given configuration. Once a decoder has been returned, the same configuration must not be used again.

#### func (*Decoder) [Decode](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L373) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Decoder.Decode)

```
func (d *Decoder) Decode(input interface{}) error
```

Decode decodes the given raw interface to the target pointer specified by the configuration.

#### type [DecoderConfig](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L189) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#DecoderConfig)

```
type DecoderConfig struct {
	// DecodeHook, if set, will be called before any decoding and any
	// type conversion (if WeaklyTypedInput is on). This lets you modify
	// the values before they're set down onto the resulting struct.
	//
	// If an error is returned, the entire decode will fail with that
	// error.
	DecodeHook DecodeHookFunc

	// If ErrorUnused is true, then it is an error for there to exist
	// keys in the original map that were unused in the decoding process
	// (extra keys).
	ErrorUnused bool

	// ZeroFields, if set to true, will zero fields before writing them.
	// For example, a map will be emptied before decoded values are put in
	// it. If this is false, a map will be merged.
	ZeroFields bool

	// If WeaklyTypedInput is true, the decoder will make the following
	// "weak" conversions:
	//
	//   - bools to string (true = "1", false = "0")
	//   - numbers to string (base 10)
	//   - bools to int/uint (true = 1, false = 0)
	//   - strings to int/uint (base implied by prefix)
	//   - int to bool (true if value != 0)
	//   - string to bool (accepts: 1, t, T, TRUE, true, True, 0, f, F,
	//     FALSE, false, False. Anything else is an error)
	//   - empty array = empty map and vice versa
	//   - negative numbers to overflowed uint values (base 10)
	//   - slice of maps to a merged map
	//   - single values are converted to slices if required. Each
	//     element is weakly decoded. For example: "4" can become []int{4}
	//     if the target type is an int slice.
	//
	WeaklyTypedInput bool

	// Squash will squash embedded structs.  A squash tag may also be
	// added to an individual struct field using a tag.  For example:
	//
	//  type Parent struct {
	//      Child `mapstructure:",squash"`
	//  }
	Squash bool

	// Metadata is the struct that will contain extra metadata about
	// the decoding. If this is nil, then no metadata will be tracked.
	Metadata *Metadata

	// Result is a pointer to the struct that will contain the decoded
	// value.
	Result interface{}

	// The tag name that mapstructure reads for field names. This
	// defaults to "mapstructure"
	TagName string
}
```

DecoderConfig is the configuration that is used to create a new decoder and allows customization of various aspects of decoding.

#### type [Error](https://github.com/mitchellh/mapstructure/blob/v1.4.0/error.go#L12) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Error)

```
type Error struct {
	Errors []string
}
```

Error implements the error interface and can represents multiple errors that occur in the course of a single decode.

#### func (*Error) [Error](https://github.com/mitchellh/mapstructure/blob/v1.4.0/error.go#L16) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Error.Error)

```
func (e *Error) Error() string
```

#### func (*Error) [WrappedErrors](https://github.com/mitchellh/mapstructure/blob/v1.4.0/error.go#L30) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Error.WrappedErrors)

```
func (e *Error) WrappedErrors() []error
```

WrappedErrors implements the errwrap.Wrapper interface to make this return value more useful with the errwrap and go-multierror libraries.

#### type [Metadata](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go#L260) [¶](https://pkg.go.dev/github.com/mitchellh/mapstructure#Metadata)

```
type Metadata struct {
	// Keys are the keys of the structure which were successfully decoded
	Keys []string

	// Unused is a slice of keys that were found in the raw value but
	// weren't decoded since there was no matching field in the result interface
	Unused []string
}
```

Metadata contains information about decoding a structure that is tedious or difficult to get otherwise.

## ![img](https://pkg.go.dev/static/img/pkg-icon-file_16x12.svg)Source Files

[View all](https://github.com/mitchellh/mapstructure/tree/v1.4.0)

- [decode_hooks.go](https://github.com/mitchellh/mapstructure/blob/v1.4.0/decode_hooks.go)
- [error.go](https://github.com/mitchellh/mapstructure/blob/v1.4.0/error.go)
- [mapstructure.go](https://github.com/mitchellh/mapstructure/blob/v1.4.0/mapstructure.go)