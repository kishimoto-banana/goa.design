+++
date="2018-09-06T11:21:49-07:00"
description="github.com/goadesign/goa/examples/error/design"
+++


# design
`import "github.com/goadesign/goa/examples/error/design"`

* [Overview](#pkg-overview)
* [Index](#pkg-index)

## <a name="pkg-overview">Overview</a>



## <a name="pkg-index">Index</a>
* [Variables](#pkg-variables)


#### <a name="pkg-files">Package files</a>
[design.go](/src/github.com/goadesign/goa/examples/error/design/design.go) 



## <a name="pkg-variables">Variables</a>
``` go
var FloatOperands = Type("FloatOperands", func() {
    Attribute("a", Float64, "Left operand")
    Attribute("b", Float64, "Right operand")
    Required("a", "b")
})
```
``` go
var IntOperands = Type("IntOperands", func() {
    Attribute("a", Int, "Left operand")
    Attribute("b", Int, "Right operand")
    Required("a", "b")
})
```







- - -
Generated by [godoc2md](https://godoc.org/github.com/davecheney/godoc2md)
