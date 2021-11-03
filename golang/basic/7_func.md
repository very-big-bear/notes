# func

## string format

```go
fmt.Printf("bool: %t\n", true)                      // true 
fmt.Printf("int: %d\n", 123)                        // 123
fmt.Printf("bin: %b\n", 14)                         // 1110
fmt.Printf("char: %c\n", 33)                        // !
fmt.Printf("octet: %o\n", 10)                       // 12
fmt.Printf("hex: %x\n", 456)                        // 1c8
fmt.Printf("float1: %f\n", 78.9)                    // 78.900000
fmt.Printf("float2: %e\n", 123400000.0)             // 1.234000e+08
fmt.Printf("float3: %E\n", 123400000.0)             // 1.234000E+08
fmt.Printf("str1: %s\n", "\"string\"")              // "string"
fmt.Printf("str2: %q\n", "\"string\"")              // "\"string\""
fmt.Printf("str3: %x\n", "hex this")                // 6865782074686973

fmt.Printf("width1: |%6d|%6d|\n", 12, 345)          // |    12|   345|
fmt.Printf("width2: |%6.2f|%6.2f|\n", 1.2, 3.45)    // |  1.20|  3.45|
fmt.Printf("width3: |%-6.2f|%-6.2f|\n", 1.2, 3.45)  // |1.20  |3.45  |
fmt.Printf("width4: |%6s|%6s|\n", "foo", "b")       // |   foo|     b|
fmt.Printf("width5: |%-6s|%-6s|\n", "foo", "b")     // |foo   |b     |

s := fmt.Sprintf("sprintf: a %s", "string")
fmt.Println(s)
fmt.Fprintf(os.Stderr, "io: an %s\n", "error")

type point struct {
	x, y int
}
p := point{1, 2}
fmt.Printf("struct1: %v\n", p)      // {1 2}                   //(相應值的默認格式)
fmt.Printf("struct2: %+v\n", p)     // {x:1 y:2}               //(打印結構體時，會添加字段名)
fmt.Printf("struct3: %#v\n", p)     // main.point{x:1, y:2}    //(相應值的Go語法表示)
fmt.Printf("type: %T\n", p)         // main.point              //(相應值的類型的Go語法表示)
fmt.Printf("pointer: %p\n", &p)     // 0xc000130010            //memory 位置，(十六進制表示，前綴0x)
```


---

## function

```go
func plus(a int, b int) int {
    return a + b
}

func plusPlus(a, b, c int) int {
    return a + b + c
}

func main() {
    res := plus(1, 2)
    fmt.Println("1+2 =", res)       // 3

res = plusPlus(1, 2, 3)
    fmt.Println("1+2+3 =", res)     // 6
}
```

---

## multiple return value

```go
func vals() (int, int) {
    return 3, 7
}

func main() {
    a, b := vals()
    fmt.Println(a)      // 3
    fmt.Println(b)      // 7

    _, c := vals()
    fmt.Println(c)      // 7
}
```

---

## variadic function

```go
func sum(nums ...int) {               //不確定有多少整數以...表示
    fmt.Print(nums, " ")
    total := 0
    for _, num := range nums {
        total += num
    }
    fmt.Println(total)
}

func main() {
    sum(1, 2)
    sum(1, 2, 3)

    nums := []int{1, 2, 3, 4}
    sum(nums...)
}
```





