+++
title =  "Debugging a bad assumption"
date  = 2020-02-27T09:56:04-05:00
draft = false
+++

Sometime in 2019 I wrote one of my first Swift programs. It wasn't anything complicated, and it didn't do anything important. But it did these two things:
- the program ran continuously, performing some calculations.
- from time to time the binary printed output to stdout.

Here's a slightly simplified version of the code:

```swift
while true { print("Hello, world!") }
```

And it worked fine. Running `./main` I saw it was doing the things I expected it to. 
I needed to call my Swift binary from a Go program, which is mostly straightforward to implement too. Here it is:

```Go
func main() {
	cmd := exec.Command("./main")
	stdout, err := cmd.StdoutPipe()
	if err != nil {
		log.Fatal(err)
	}

	if err := cmd.Start(); err != nil {
		log.Fatal(err)
	}

	scanner := bufio.NewScanner(stdout)
	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}

	if err := scanner.Err(); err != nil {
		log.Fatal(err)
	}

	log.Fatal(cmd.Wait())
}
```

When I ran the Go program, nothing happened. I could see that things _were_ happening. The Go binary was running but not producing the expected output. And the Swift process it called was also running. But the output I was expecting wasn't there. No matter what I tried, I could not figure out why there seemed to be an issue. 

Reading the simplified versions of the code you might have already seen the issue. The actual programs I wrote had more lines of code, and debugging was harder. I tried a few things before getting to this area of the code. When I narrowed it down, the program was stalling at `for scanner.Scan() {`. It made no sense to me. I replaced `main.swift` with `main.go` which looked like this:

```Go
for { fmt.Println("Hello World") }
```

And everything worked as expected. My Go binary was calling my other Go binary and output was printed to stdout. So what the hell!

Turns out, in Swift(and as I learned, other C languages) stdout is buffered by default. When invoked in a terminal `print` will print the output immediately, but if someone is reading stdout, the buffer won't be flushed until the program exits. That's actually how I stumbled on the solution, because writing a short lived program which would exit after calling `print` would return expected output on the Go side. But making it long running would hang again. 

On the Go side, [stdout is not buffered](https://grokbase.com/p/gg/golang-nuts/158czyvzjt/go-nuts-os-stdout-is-not-buffered). At this point most of the code I've ever written has been Go, which explains why I was assuming the behavior of `print` in another language. Even though the Swift behavior is consistent with C, and Go is doing the less common thing here. 
As it happens, I'm not the only one who found this behavior surprising. There is a [discussion thread](https://lists.swift.org/pipermail/swift-dev/Week-of-Mon-20160328/001587.html) about `print` buffering on the Swift mailing lists. 

I modified the Swift code to use `FileHandle.standardOutput` directly, instead of `print` and that fixed the issue. 

```Swift
let output = FileHandle.standardOutput
let newLine = "\n".data(using: .utf8)!

while true {
  output.write("Hello, World!".data(using: .utf8)!)
  output.write(newLine)
}
```

So now I know, and won't repeat it. In fact, a few month later I wrote something similar and I knew to avoid this problem entirely. 

![img](https://media.giphy.com/media/8JbMnRHcDVH3O/giphy.gif)

But how do we learn new things in tech? Is making one mistake after another, forever, the best way to become an expert at something? It sure feels that way when learning new languages or tools.

