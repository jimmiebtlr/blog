## Go Basics: For loop suprise!

What happens when we run this code?

```
func main() {
	data := []string{"Hello", "World"}
	
	toPrint := []*string{}
	for _, a := range data {
		toPrint = append(toPrint, &a)
	}
	
	for _, p := range toPrint {
		fmt.Println(*p)
	}
}
```
 [Playground](https://play.golang.org/p/F0doQDv8C9l) 

The answer is 
```
World
World
```


Another example of the same issue

```
func PrintIt(a *string) {
	go func() {
		fmt.Println(*a)
	}()
}

func main() {
	data := []string{"Hello", "World"}
	
	for _, a := range data {
		PrintIt(&a)
	}
	time.Sleep(500 * time.Millisecond)
}
```
 [Playground](https://play.golang.org/p/AppkIyPScRH) 


### What?

When you loop over a slice or array, the loop variable has the same address on each iteration.  Using that address rather than evaluating immediately will cause the values to be the last in the array (or a more complex/racey behavior if it's evaluating while loop is running).

### How to fix this

The problem is basically using the address of a loop variable instead of evaluating immediately.

There are 2 easy ways of handling this.

Use the value immediately, useful for simple cases.
```
func main() {
	data := []string{"Hello", "World"}
	
	for _, a := range data {
        fmt.Println(a)
	}
}
```

Use an extra variable, more generally useful.  This works by simply making a new variable and putting the value there, then if we take it's address that memory stays as the correct value whenever we use it in the future.
```
func PrintIt(a *string) {
	go func() {
		fmt.Println(*a)
	}()
}

func main() {
	data := []string{"Hello", "World"}

	for _, a := range data {
		b := a
		PrintIt(&b)
	}
	time.Sleep(time.Second)
}```
 [Playground](https://play.golang.org/p/7deMeSXDrve) 

An important variation on the above, you can also shadow the loop variable by changing 
```
b := a 
```
into
```
a := b
```

This can effectively protect you from accidentally using the loop variable.
[Playground](https://play.golang.org/p/0iEzP1BeraD)

### Conclusion

This is the most common gotcha in golang in my opinion.  

Always be on the look out when working with loops.  Be sure to test your code, including in a way that would catch this bug.  Also of note, `go vet` can catch simple examples of this, use your linters as well!