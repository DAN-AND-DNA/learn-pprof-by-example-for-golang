#  golang pprof 教程

本人刚做主程的时候，一个人负责开发腾讯的一个30万日活的小型项目的服务器，那时候用的就是golang/c++。那时候经验不足的我感觉做性能测试、性能优化
和内存泄露检测会比较难。

## 内容

- [原理](#原理)
- [为pprof做准备](#为pprof做准备)
- [通用pprof](#通用pprof)
- [http专属pprof](#http专属pprof)


## 原理


## 为pprof做准备




## 通用pprof

例程为:
```go
package main
  
import (
        "log"
        "os"
        "runtime/pprof"
        "time"
        "runtime"
)

var msgqueue chan uint64
var timerqueue chan struct{}

var testmap map[int]interface{}


func init(){
        msgqueue = make(chan uint64, 10000)
        timerqueue = make(chan struct{}, 10)
        testmap = make (map[int]interface{})
}
func main() {
        c, err := os.Create("cpu_profile.prof")
        if err != nil {
                log.Fatal(err)
        }

        defer c.Close()
        m, err := os.Create("mem_profile.prof")
        if err != nil {
                log.Fatal(err)
        }


        m1, err := os.Create("mem_profile1.prof")
        if err != nil {
                log.Fatal(err)
        }

        defer m.Close()
        defer m1.Close()

        pprof.StartCPUProfile(c)
        defer pprof.StopCPUProfile()

        testmap[1]= runheaptest()

        runtime.GC()
        pprof.Lookup("heap").WriteTo(m, 0)
        //FIXME running in another thread or blocking main thread !!!
        go runcputest()

        go func(){
                time.Sleep(10 * time.Second)
                log.Println("tick")
                timerqueue <- struct{}{}

        }()
        testmap[2]= runheaptest()

        runtime.GC()
        pprof.Lookup("heap").WriteTo(m1, 0)
        //run in main thread
        procmsg()
}

func runheaptest()([]int) {
        mem :=  make([]int, 100000, 120000)
        return mem
}

func runcputest(){
        var i uint64 = 0
        for {
                msgqueue <- i
                i++
        }
}

func procmsg(){
        for {
                select {
                case _ = <-msgqueue:
                case _ = <-timerqueue:
                        log.Println("timeout")
                        return
                }
        }
}
```

install然后运行上述程序将会得到 cpu_profle.prof 和 mem_profile.prof 2个文件，前者代表针对cpu的取样，后者代表针对heap的统计，可以通过pprof来分析上述产生的文件例如:

    $ go tool pprof en cpu_profle.prof 
    File: en
    Type: cpu
    Time: May 18, 2019 at 12:05am (CST)
    Duration: 20.18s, Total samples = 20.27s (100.46%)
    Entering interactive mode (type "help" for commands, "o" for options)
    
可以查询消耗cpu排名前10的函数，例如:

    (pprof) top
    Showing nodes accounting for 18680ms, 92.16% of 20270ms total
    Dropped 20 nodes (cum <= 101.35ms)
    Showing top 10 nodes out of 19
          flat  flat%   sum%        cum   cum%
        6440ms 31.77% 31.77%    14060ms 69.36%  runtime.selectgo
        2420ms 11.94% 43.71%     2440ms 12.04%  runtime.unlock
        2380ms 11.74% 55.45%     2960ms 14.60%  runtime.lock
        1880ms  9.27% 64.73%     1880ms  9.27%  runtime.selectrecv
        1320ms  6.51% 71.24%     1320ms  6.51%  runtime.newselect
        1180ms  5.82% 77.06%     3320ms 16.38%  runtime.selunlock
         970ms  4.79% 81.85%    19000ms 93.73%  main.procmsg
         800ms  3.95% 85.79%     3280ms 16.18%  runtime.sellock
         770ms  3.80% 89.59%      770ms  3.80%  runtime.duffzero
         520ms  2.57% 92.16%      520ms  2.57%  runtime.procyield
         
可以查询具体函数详情，例如

    (pprof) list main.procmsg
    Total: 20.27s
    ROUTINE ======================== main.procmsg in /home/dan/Desktop/src/work/src/en/testprof.go
             970ms        19s (flat, cum) 93.73% of Total
             .          .     60:   }
             .          .     61:}
             .          .     62:
             .          .     63:func procmsg(){
             .          .     64:   for {
         410ms     16.56s     65:           select {
         380ms      1.31s     66:           case _ = <-msgqueue:
         180ms      1.13s     67:           case _ = <-timerqueue:
             .          .     68:                   log.Println("timeout")
             .          .     69:                   return
             .          .     70:           }
             .          .     71:   }
             .          .     72:}
    (pprof)
    
 可以导出callgrind文件进行可视化分析，例如:
 
    (pprof) callgrind
    Generating report in profile001.callgraph.out
    (pprof) 
    
如果Linux里有KDE可以用KCachegrind打开callgrind.out，进行函数优化:
 ![demo](https://github.com/DAN-AND-DNA/learn-pprof-by-example-for-golang/blob/master/img/2019518-161135.jpg)
 
 
 内存泄露分析，前后对比 mem_profile.prof和 mem_profile1.prof就可以看出heap的变化，例如:
 
    $ go tool pprof mem_profile.prof en 
    en: parsing profile: unrecognized profile format
    Fetched 1 source profiles out of 2
    File: en
    Type: inuse_space
    Time: May 18, 2019 at 5:39pm (CST)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top
    Showing nodes accounting for 2.25MB, 100% of 2.25MB total
          flat  flat%   sum%        cum   cum%
        1.16MB 51.36% 51.36%     1.16MB 51.36%  runtime/pprof.StartCPUProfile
        1.10MB 48.64%   100%     2.25MB   100%  main.main
             0     0%   100%     2.25MB   100%  runtime.main
    (pprof) list main.main
    Total: 2.25MB
    ROUTINE ======================== main.main in /home/dan/Desktop/src/work/src/en/testprof.go
          1.10MB     2.25MB (flat, cum)   100% of Total
             .          .     38:   }
             .          .     39:
             .          .     40:   defer m.Close()
             .          .     41:   defer m1.Close()
             .          .     42:
             .     1.16MB     43:   pprof.StartCPUProfile(c)
             .          .     44:   defer pprof.StopCPUProfile()
             .          .     45:
             .          .     46:   testmap[1]= runheaptest()
             .          .     47:
             .          .     48:   runtime.GC()
             .          .     49:   pprof.Lookup("heap").WriteTo(m, 0)
             .          .     50:   //FIXME running in another thread or blocking main thread !!!
             .          .     51:   go runcputest()
             .          .     52:
             .          .     53:   go func(){
             .          .     54:           time.Sleep(10 * time.Second)
             .          .     55:           log.Println("tick")
             .          .     56:           timerqueue <- struct{}{}
             .          .     57:
             .          .     58:   }()
             .          .     59:   testmap[2]= runheaptest()
             .          .     60:
             .          .     61:   runtime.GC()
             .          .     62:   pprof.Lookup("heap").WriteTo(m1, 0)
             .          .     63:   //run in main thread
             .          .     64:   procmsg()
             .          .     65:}
             .          .     66:
             .          .     67:func runheaptest()([]int) {
        1.10MB     1.10MB     68:   mem :=  make([]int, 100000, 120000)
             .          .     69:   return mem
             .          .     70:}
             .          .     71:
             .          .     72:func runcputest(){
             .          .     73:   var i uint64 = 0
    (pprof) quit
    dan@dan-VirtualBox:~/Desktop/src/work/bin$ go tool pprof mem_profile1.prof en 
    en: parsing profile: unrecognized profile format
    Fetched 1 source profiles out of 2
    File: en
    Type: inuse_space
    Time: May 18, 2019 at 5:39pm (CST)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top
    Showing nodes accounting for 3.35MB, 100% of 3.35MB total
          flat  flat%   sum%        cum   cum%
        2.19MB 65.44% 65.44%     3.35MB   100%  main.main
        1.16MB 34.56%   100%     1.16MB 34.56%  runtime/pprof.StartCPUProfile
             0     0%   100%     3.35MB   100%  runtime.main
    (pprof) list main.main
    Total: 3.35MB
    ROUTINE ======================== main.main in /home/dan/Desktop/src/work/src/en/testprof.go
        2.19MB     3.35MB (flat, cum)   100% of Total
             .          .     38:   }
             .          .     39:
             .          .     40:   defer m.Close()
             .          .     41:   defer m1.Close()
             .          .     42:
             .     1.16MB     43:   pprof.StartCPUProfile(c)
             .          .     44:   defer pprof.StopCPUProfile()
             .          .     45:
             .          .     46:   testmap[1]= runheaptest()
             .          .     47:
             .          .     48:   runtime.GC()
             .          .     49:   pprof.Lookup("heap").WriteTo(m, 0)
             .          .     50:   //FIXME running in another thread or blocking main thread !!!
             .          .     51:   go runcputest()
             .          .     52:
             .          .     53:   go func(){
             .          .     54:           time.Sleep(10 * time.Second)
             .          .     55:           log.Println("tick")
             .          .     56:           timerqueue <- struct{}{}
             .          .     57:
             .          .     58:   }()
             .          .     59:   testmap[2]= runheaptest()
             .          .     60:
             .          .     61:   runtime.GC()
             .          .     62:   pprof.Lookup("heap").WriteTo(m1, 0)
             .          .     63:   //run in main thread
             .          .     64:   procmsg()
             .          .     65:}
             .          .     66:
             .          .     67:func runheaptest()([]int) {
        2.19MB     2.19MB     68:   mem :=  make([]int, 100000, 120000)
             .          .     69:   return mem
             .          .     70:}
             .          .     71:
             .          .     72:func runcputest(){
             .          .     73:   var i uint64 = 0
    (pprof) 
 
## http专属pprof

