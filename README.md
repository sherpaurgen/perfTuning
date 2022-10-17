# perfTuning

As a developer/sysadmin you can create [flamegraphs](https://github.com/brendangregg/FlameGraph) to to create visualizations of system performance data recorded with the perf tool. This perf output shows a stack trace followed by a count, for a total of #N number of samples

git clone the flamegraph scripts:
```
cd /home/ubuntu

git clone https://github.com/brendangregg/FlameGraph
```

Sampling a go programme which downloads an linuxmint iso

```
package main
import (
    "fmt"
    "net/http"
    "io"
    "os"
)

func check(e error) {
    if e != nil {
        panic(e)
    }
}

func main() {
    d1 := []byte("helo world\n")
    for i := 0; i < 10000; i++ {
        resp, err := http.Get("https://mirrors.layeronline.com/linuxmint/stable/21/linuxmint-21-cinnamon-64bit.iso")
        //body, err := io.ReadAll(resp.Body)
        fmt.Println(resp.StatusCode)
        check(err)
        defer resp.Body.Close()
        file, err := os.Create("/tmp/hello.iso")
        size, err := io.Copy(file, resp.Body)
        defer file.Close()
        fmt.Printf("downloaded %s with size %d", file, size)
        err = os.WriteFile("/tmp/check.txt", d1, 0644)
        check(err)
    }
}
```
compile/build the main.go and run it `./main`
Get the process id of main e.g `ps aux | grep main`

`perf record -a -F 99 -g -p 1464 -- sleep 20`
Running above command creates a perf.data file

`perf script > perf.script ` //it will by default read perf.data from current working directory and redirects stdout to a file `perf.script` (ascii file)
This command reads the input file and displays the trace recorded.

Creating flame graph:-
`./FlameGraph/stackcollapse-perf.pl perf.script  | ./FlameGraph/flamegraph.pl > flame1006.svg`

Download and view flame1006.svg in browser

![flame1006.svg](https://github.com/sherpaurgen/perfTuning/blob/main/flame1006.svg?raw=true"flame1006.svg")

Here we can observe that io.copybuffer function is using most cpu time [io.copy source](https://cs.opensource.google/go/go/+/refs/tags/go1.19.2:src/io/io.go;l=386)
looking further net.(*netFD).Read is the function call using most cpu time. This net.(*netFD).Read implements func (*IPConn) Read
[Conn](https://pkg.go.dev/net#Conn) is a generic stream-oriented network connection.
this function reads data from the connection. This func https://go.dev/src/net/http/transfer.go ¶
we also see ksys_read() is called. This function is responsible for retrieving the struct fd that corresponds with the file descriptor passed in by the user. The struct fd structure contains the struct file_operations structure within it.

![fg2.png](https://github.com/sherpaurgen/perfTuning/blob/main/fg2.png "fg2.png")
(image is png instead of SVG as we cannot view svg in github but i think the point is comprehensible)

`sock_read_iter` is fired when receiving a message on a socket
By looking at the graph we can conclude that the cpu usage by main programme is spent mostly on reading data from the connection.

https://dev.to/sherpaurgen/flamegraphs-part-1-2ncl
