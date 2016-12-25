+++
date = "2015-08-15T21:02:25-04:00"
title = "Sysadmin productivity with Go"
+++

Over the past 6 months, I've been learning [Go](http://golang.org/) and using it to solve various system administration tasks. One of the first useful things I wrote in Go, was a program that read the report plist, which [AutoPkg](https://github.com/autopkg/autopkg) outputs, and piped the output to Slack.  
With the release of AutoPkg [0.5.0](https://github.com/autopkg/autopkg/releases/tag/v0.5.0), the format of `--report-plist` had changed, which meant that I would have to rewrite my script. By this time I had a better handle on Go, especially it's [concurrency primitives](https://blog.golang.org/share-memory-by-communicating). 

AutoPkg lets you specify a text file with a list of recipes to check. 
```
MSWord2016.munki
OracleJava8.munki
AdobeFlashPlayer.pkg
Atom.munki
VLC.munki
```
Given the text file above, AutoPkg will check each recipe, *sequentially*. With a big enough list, the process can easily take 30 minutes or more. We can illustrate the current process by borrowing this cute picture from the [Concurrency is not parallelism](http://blog.golang.org/concurrency-is-not-parallelism) go talk.  
![](https://talks.golang.org/2012/waza/gophersimple1.jpg)
An this example, the pile of books represents the recipes autopkg must process. 
We can speed this process a bit, by separating the work. Let's assign one gopher(goroutine) to read each recipe from a file, and create work for other gophers.
```Go
recipes := make(chan string)
... 

func readRecipeList(recipes chan string) {
        file, err := os.Open(conf.RecipesFile)
        if err != nil {
                log.Fatal(err)
        }
        defer file.Close()

        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
                recipe := scanner.Text()
                recipes <- recipe
        }
        close(recipes)
}
```
Here we open a text file, read each line, and send each recipe on a channel. When we are done, we close the recipes channel.

      for recipe := range recipes {
              go func(recipe string) {
                      reports <- runAutopkg(recipe)
              }(recipe)
      }
Adding the `go` keyword before a function call, executes that function in a separate goroutine. 
In this case, we call  
```
func runAutopkg(recipe string) *autopkgReport {
// Exec autopkg and return report.plist
}
```
for each recipe, and pass the report that the function returns to a new channel, called *reports*.
We can take the reports channel, and pass it to different interpreter functions, that can do something based on the report(also concurrently).  
We can wrap all of the above into a new function called `process()`, that looks like this:

```
var wg sync.WaitGroup
recipes := make(chan string)
reports := make(chan *autopkgReport)

go readRecipeList(recipes)

go notifySlack(reports)

for recipe := range recipes {
		wg.Add(1)
       go func(recipe string) {
               reports <- runAutopkg(recipe)
               wg.Done()
       }(recipe)
}

wg.Wait()
close(reports)

done <- true
```
The code has two new additions. The first is a WaitGroup variable. The WaitGroup allows us to synchronize all the concurrent processes and wait until they complete. We also added the line 
```
done <- true
```
Somewhere else in our code, we created `done := make(chan bool)`, a boolean channel and by sending `<-true` on the channel, we are signaling that the our process() function has completed. 
The process can be roughly illustrated by the picture below.
![](https://i.imgur.com/FmRvhP4.png)

We can now repeat the whole process at a specified interval by creating a ticker. 
```
func main() {
        done := make(chan bool)
        ticker := time.NewTicker(time.Minute * 5).C
        for {
                go process(done)
                <-done
                <-ticker
        }
}
```
The process will now repeat every 5 minutes, except that we're also blocking the loop by listening on the `<-done` channel, so if the previous process is still executing, the program will wait. 

The full program, including an OS X binary is available on [github](https://github.com/groob/autopkgd).  
We can use a similar process to solve many problems we encounter in the real world. 
If you found the above example intriguing, here's an [entertaining talk](https://www.youtube.com/watch?v=woCg2zaIVzQ) that illustrates the same concepts more concisely.   
*The drawings used in this blog post were created by illustrator [Renee French](http://www.reneefrench.com/)*.
