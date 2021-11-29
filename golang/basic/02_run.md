# 基本運作形式

## hello

```bash
檔名：hello.go

package main

import "fmt"

func main(){
    fmt.Println("hello go")
}
```

# run
```bash
linux:~ # go run hello.go
```
# compile to binary
```bash
linux:~ # go build hello.go
linux:~ # ./hello
