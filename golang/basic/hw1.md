# Homework1
---
## right_triangle
```bash
package main

import "fmt"

var h int
func main() {
        fmt.Print("high = ")
        fmt.Scanln(&h)
        for i := 1; i <= h; i++{
                for n := 1; n <= 1; n++{
                        fmt.Print("*")
                }
                fmt.Println("")
        }
}
```
## pyramid
```bash
package main

import "fmt"

var h, o int

func main() {
        fmt.Print("high = ")
        fmt.Scanln(&h)
        o = h - 1
        for i := 1; i <= h; i++ {
                for n := 1; n <= o; n++{
                        fmt.Print(" ")
                }
                for m := 1; m <= 2*i-1; m++{
                        fmt.Print("*")
                }
                fmt.Println("")
                o = o - 1
}
```
## 99
```bash
package main

import "fmt"

for i := 1; i <= 9; i++{
        for f := 1; f <= 9; f++{
        
```
## reverse
```bash
```
