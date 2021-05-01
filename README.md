# Changes I would like to see in the Go Language

## Embedded Struct Instantiation Literals

Currently if you have an embedded struct, such as the following you have to recognize that fact when you instantiate it with literal syntax. 

For example:

```
type Motor struct {
    Type string
}
type Auto struct {
    Category string
    Motor
}
func main() {
    auto := Auto{
        Category: "truck",
        Motor: Motor{
            Type: "V8",
        },
    }
    fmt.Printf("%#v",auto)
}
```

However, that means that refactoring to move prior properties to be properties of embedded structs break any code that instantiates these structs which IMO flies in the face of why I would want to use embedded structs to begin with; to isolate client code from needing to change when the internal implementation of that package code changes.  

Instead I would like to be able to have this work for those already declared types:

```
func main() {
    auto := Auto{
        Category: "truck",
        Type: "V8",
    }
    fmt.Printf("%#v",auto)
}
```
Alternately it would be interesting if we could make this work instead, where the name required in the literal is a concatonation of the type name and the property name:
```
func main() {
    auto := Auto{
        Category: "truck",
        MotorType: "V8",
    }
    fmt.Printf("%#v",auto)
}
```
I know there may be issues with this proposal, but let's consider it a strawman proposal for now and mention that there is in fact an issue when instantiations need a better solution that what we have today.

Along those lines maybe the solution needed is a way to create a third type that combines two or more types?  Maybe a `union` type that would behave just like a struct but that would 

```
type Motor struct {
    Type string
    Fuel string
}
type Auto struct {
    Category string
}
type AutoOpts union {
    Auto
    Motor prefixed Engine
}
func main() {
    auto := AutoOpts{
        Category: "truck",
        EngineType: "V8",
        EngineFuel: "diesel",
    }
    fmt.Printf("%#v",auto)
}
```

