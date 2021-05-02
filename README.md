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

## Relaxing variable declaration rules
Go has numerous features that make programming a lot easier than it otherwise could be, and normally those features aare great. However, there are times when those features step all over each other and just frustrate the developer who wanted to uses these niceties but because of the rules of Go, they simple could not.

### Implicit Variable Declaration

One of the great little things about programming in Go is not having to declare the type of variables in an assignment. This simple pleasure means a developer can focus more on the logic and less on the technical mechanics of programming.

The colon-equals operator (`:=`) implicitly declares a variable to be of the type found on the right-hand side. For example, the variable `name` is implicitly declared to be string here:

```
name := "Mike"
```
The above is functionally identical to an explicit type declaration and use of the simple equals operator (`=`):
```
var name string
name = "Mike"
```

### Multiple Return Values
Another of the great little things about programming in Go is being able to return multiple values from a function and also capture multiple values from a function, like so:
```
func getValues() (int,string){
   return 10,"hello"
}
func main() {
   foo,bar := getValues()
   println(foo)
   println(bar)
}
```
Why this is great is you often want to return not only the value from a function but also an indicator as to whether or not an error occurred when executing the code in the function. In languages such as PHP which do not allow for multiple returns you have to resort to hacks, such as returning a special value for an error, always returning a special object that can contain both a value and an error indicator, or "returning" the error via a by-reference parameter.  And none of those are good options.

Indeed, returning indications of errors is such as fundamental pattern that it is one of Go's core idioms, and many functions return their last return value as either `nil`, meaning no error, or an object that implements the `error` interface. For example, using the built-in `ReadFile()` method of the `ioutil` package and a hypothetical `GenerateHTML()` method of a hypothetical `markdown` package:

```
file:= "/path/to/README.md"
content,err := ioutil.ReadFile(file)
if err ~= nil {
   panic(fmt.Sprintf("Could not read file %s; %s",file,err.Error())
}
print( markdown.GenerateHTML(content) )    
```

### When Implicit Variable Declaration and Multiple Return Values Collide
However, all is not as great as it seems. Those two features can step on each other and nullify the benefit. Which is annoying. 

And yes making refactoring a lot more effort than would otherwise be required is a first world problem. But it can also force the developer to make changes that could accidentally break the code's logic, simply because Go forces editing when lines are moved around.

#### Rearranging Lines with Variable Declarations
First, in Go you can only use the implicit declaration operator (`:=`) once in the same scope. So the following is not valid:
```
x := 1
println(x)
x := 2       // Go will complain you have already declared 'x' above.
println(x)
```
Instead you have to use the simple equals operator for the second and subsequent assignments:
```
x := 1
println(x)
x = 2
println(x)
```
Now imagine you needed to swap those pairs of lines. If you are not fastidious you might try to compile this, which will of course not compile:
```
x = 2      // Go will complain you have not declared 'x' here.
println(x)
x := 1
println(x)
```
So you go back and edit those two assignment lines and now it compiles:
```
x := 2
println(x)
x = 1
println(x)
```
#### Capturing Multple Return Values into both Declared and Undeclared Variables
While the above is not a frequent annoyance, wanting to implicitly declare an undeclared variable when capturing mutliple return values when (one of) the other variable(s) is already declared is a real PITA.  For example:

```
func DoHTTPGet(url string) (string,error) {
   client, err := GetHTTPClient()
   if err != nil {
      return "",err
   }
   var content string               // I have to declare `content` because 
   content, err = client.DoGet(url) // err is already declared above. Grrrr.
   if err != nil {
      return "",err
   }
   return content,err
}
```
This is so intensely annoying that I have started using numbered error variables instead, like so:

```
func DoHTTPGet(url string) (string,error) {
   client, err := GetHTTPClient()
   if err != nil {
      return "",err
   }
   content, err1 = client.DoGet(url) 
   if err1 != nil {
      return "",err1
   }
   return content,err
}
```

#### How to Fix These Annoyances?

For the first one annoyance, I would ask that it simply be possible to use `:=` multiple times in the same scope. Certainly the compiler can figure it out, right?

The second annoyance is a bit more complicated, but ideally there should be a solution.  So I'll propose several [strawmen](https://en.wikipedia.org/wiki/Straw_man) to address it:

1. Always allow `:=` to be used for a return value that implements `error` even if `err` already defined in the same scope. Basically this is the same solution as for the first annoyance, but specifically for returning `err` which is by var the most common place where a solution is needed.

2. Allow an assignment operator for each variable, e.g.:

```
func DoHTTPGet(url string) (string,error) {
   client, err := GetHTTPClient()
   if err != nil {
      return "",err
   }
   content := , err = client.DoGet(url) // Notice using := for `content`, and = for `err`
   if err != nil {
      return "",err
   }
   return content,err
}
```
3. Some other approach?

### Declaring Named Return Variables
A third of the great little things about programming in Go is being able to declare named return variables in function signatures.

To be completed...

## Drop Shadowing of Variables

To be completed...

## A `nameof()` Built-in 

To be completed...
