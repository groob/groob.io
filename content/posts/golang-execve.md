+++
date = "2017-02-25T20:36:36-05:00"
title = "Exploring the exec system call with Go"

+++

If you've created a Docker container, you've likely seen a `docker-entrypoint.sh` script which ends with `exec $@`. The [`ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#/entrypoint) directive is a common way to add some sort of initialization to a docker container. It can be used to update some configuration based on environment variables passed to the container, generate some random data or do anything else necessary for the process to start. Today I want to focus on the `exec $@` line that such an entrypoint script often ends with. 

<!--more-->

Until recently, my understanding of `exec $@` was limited to "runs the arguments passed to the script at the end". It was enough for me to use it, but not to be able to actually explain it. Then, one day I wanted to write a Docker entrypoint in Go instead of bash. My usual way of shelling out to another process with `os/exec` wouldn't work here. A process started with `os/exec` is still managed by the original process. I wanted my entrypoint to do its thing, but then let the next process take over completely. That's what `exec $@` does. So, I had to figure out how to express that in Go. Google didn't really help, but I asked in the [Gophers Slack](https://invite.slack.golangbridge.org/) and [Kevin Burke](https://twitter.com/derivativeburke) pointed me to `syscall.Exec` and showed me a quick example of his own. Soon enough, I had an example of my own: 

```
package entrypoint

func Exec() {
	flag.Parse()
	if len(os.Args) == 1 {
		return
	}
	cmd, err := exec.LookPath(os.Args[1])
	if err != nil {
		log.Fatal(err)
	}
	if err := syscall.Exec(cmd, flag.Args(), os.Environ()); err != nil {
		log.Fatal(err)
	}
}
```

I could now add `entrypoint.Exec()` at the end of a Go script and have it behave exactly like the shell one liner.

# syscall.Exec and its properties

```
// Exec invokes the execve(2) system call.
func Exec(argv0 string, argv []string, envv []string) (err error) {
```

The godoc for `syscall.Exec` points us at [`man 2 execve`](http://www.unix.com/man-page/bsd/2/EXECVE/).

> execve() transforms the calling process into a new process.  The new process is constructed from an ordinary file, whose name is pointed to by path, called the new process file.  This file is either an executable object file, or a file of data for an interpreter.

The man page is helpful, but it reads like it was written for C programmers, so writing a few working examples is actually more productive.
I started with something simple. Trying to call `ls -al`.

```
func main() {
	err := syscall.Exec("/bin/ls", []string{"-al"}, os.Environ())
	log.Println(err)
}
```

Unfortunately, it did not work. After some trial an error, I realized I must have the binary name as the first argument in my `args` slice. The first argument to `exec` is the path, and the second is an array of arguments, starting with the command name. 

```
func main() {
	err := syscall.Exec("/bin/ls", []string{"ls","-al"}, os.Environ())
	log.Println(err)
}
```

The example above works, but still doesn't tell us much about what is actually happening. Let's add a few debug statements to our program.  
I wrote a small binary which prints its process ID, calls `exec` with its own path and arguments and prints a `goodbye` string before exiting.
```
func main() {
	fmt.Printf("%s: pid is %d\n", os.Args[0], os.Getpid())
	if err := syscall.Exec(os.Args[0], os.Args, os.Environ()); err != nil {
		log.Fatal(err)
	}
	fmt.Println("goodbye")
}
```

Here is the output of running my `./exec-self` binary:
```
./exec-self: pid is 55415
./exec-self: pid is 55415
./exec-self: pid is 55415
./exec-self: pid is 55415
./exec-self: pid is 55415
./exec-self: pid is 55415
./exec-self: pid is 55415
```

Two interesting details immediately pop out: 
1. The pid stays the same.
2. `goodbye` is never printed.

With that observation we can conclude that a process created by `syscall.Exec` will inherit its process ID and completely replace the original.
Looking at the manpage shows that a range of other properties of the original process are inherited, such as working directory and open file descriptors.
Until now I was just using `exec $@` in my Docker entrypoint scripts, but now having a better understanding of `exec`, I see lots of ways I can take advantage of it in my work.  

# Deploying a Go binary to Heroku

I did get to use the exec syscall recently, while working on deploying an application to Heroku. At [work](https://kolide.co/) we release our product as a Go binary which users can run on their own. One of the deployment methods we wanted to support right away was a Click to Deploy button for Heroku. The Kolide app is written with the guidelines of a [12 factor app](https://12factor.net/) in mind, so deploying to Heroku should be simple. However, Heroku does not provide a immediately obvious way of deploying an arbitrary binary to its platform. You must go through their buildpack process. There were a few things I could've done, such as [committing the binary to a git repo](https://github.com/freeformz/go-bin). I chose to use the Go buildpack to write a script which:

1. Downloads the latest.zip folder from our download site.
2. Extracts the linux binary from the zip and writes it as an executable to disk.
3. Sets all the required environment vars necessary for our binary to run. 
4. Uses `syscall.Exec` to start the real process once setup is complete.

The above strategy proved to work out quite well and I had a working script in no time. With the exception of importing the mysql library to parse a DSN string, I was able to accomplish everything with the Go standard library. You can see the entire script as part of the 1-Click Deploy [pull request](https://github.com/kolide/kolide-quickstart/pull/28).


