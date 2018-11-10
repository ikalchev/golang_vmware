# Module Four Lab: GoRoutines and Concurrency

## Go Routines

1. First try at a GoRoutine:

```go
package main

import (  
    "fmt"
)

func troll2() {  
    fmt.Println("They ate her!! And now they're gonna eat me!!") //Why didn't this print?
}
func main() {  
    go troll2()
    fmt.Println("NOOOOOOOOOOOO!!!")
}
```

2. Let's fix number one to give the original goRoutine time to run:

```go
package main

import (  
    "fmt"
    "time"
)

func troll2() {  
    fmt.Println("They ate her!! And now they're gonna eat me!!")
}
func main() {  
    go troll2()
    time.Sleep(1 * time.Second)
    fmt.Println("NOOOOOOOOOOOO!!!")
}
```

3. Now let's run multiple goRoutines concurrently and see how they output:

```go
package main

import (  
    "fmt"
    "time"
)

func numbers(stoptime int) {  
    for i := 1; i <= 5; i++ {
        time.Sleep(stoptime * time.Millisecond)
        fmt.Printf("%d ", i)
    }
}
func alphabets(stoptime int) {  
    for i := 'a'; i <= 'e'; i++ {
        time.Sleep(stoptime * time.Millisecond)
        fmt.Printf("%c ", i)
    }
}
func main() {  
    go numbers(250)
    go alphabets(400)
    time.Sleep(3000 * time.Millisecond)
    fmt.Println("main terminated")
}

```

4. Let's use **channels** here to hold and communicate values from different functions. Here we're going to split an array up between two **goroutines** (you have to imagine that this array was a lot bigger than it actually is):

```go
package main

import "fmt"

func sum(s []int, c chan int) {
    sum := 0
    for _, v := range s {
        sum += v
    }
    c <- sum // send sum to c
}

func main() {
    s := []int{7, 2, 8, -9, 4, 0}
    c := make(chan int)
    go sum(s[:len(s)/2], c) //goroutine passes value to c
    go sum(s[len(s)/2:], c) //other goroutine passes value
    x, y := <-c, <-c // receive from c
```

5. Now let's close queues:

```go
// In a [previous](range) example we saw how `for` and
// `range` provide iteration over basic data structures.
// We can also use this syntax to iterate over
// values received from a channel.

package main

import "fmt"

func main() {

    // We'll iterate over 2 values in the `queue` channel.
    queue := make(chan string, 2)
    queue <- "one"
    queue <- "two"
    close(queue)

    // This `range` iterates over each element as it's
    // received from `queue`. Because we `close`d the
    // channel above, the iteration terminates after
    // receiving the 2 elements.
    for elem := range queue {
        fmt.Println(elem)
    }
}

```

6. Remember how we said that channels are **blocking** functions? Let's demonstrate this. We're going to re-run our previous goroutine example without the __sleep__ portion to show how the reading from channels blocks (therefore allowing the __goroutine__ to finish running before moving on to close out **main**):

```go
package main

import (  
    "fmt"
)

func troll2(blocker chan bool) {  
    fmt.Println("They ate her!! And now they're gonna eat me!!")
    blocker <- true //writing something to the channel
}
func main() {  
    blocker := make(chan bool)
    go troll2(blocker)
    //time.Sleep(1 * time.Second) Don't need that anymore!
    <-blocker //We don't ACTUALLY need to put this write to a variable
    fmt.Println("NOOOOOOOOOOOO!!!")
}
```

7. More practice with channels (simple):

```go
package main

import "fmt"
import "time"

// This is the function we'll run in a goroutine. The
// `done` channel will be used to notify another
// goroutine that this function's work is done.
func worker(done chan bool) {
    fmt.Print("working...")
    time.Sleep(time.Second)
    fmt.Println("done")

    // Send a value to notify that we're done.
    done <- true
}

func main() {

    // Start a worker goroutine, giving it the channel to
    // notify on.
    done := make(chan bool, 1)
    go worker(done)

    // Block until we receive a notification from the
    // worker on the channel.
    <-done
}
```

8. More practice with channels: 

```go
package main

import (
    "fmt"
)

func calcSquares(number int, squareop chan int) {
    sum := 0
    for number != 0 { //shorthand for a loop
        digit := number % 10
        sum += digit * digit
        number /= 10
    }
    squareop <- sum //write the sum to the square operator
}

func calcCubes(number int, cubeop chan int) {
    sum := 0
    for number != 0 {
        digit := number % 10
        sum += digit * digit * digit
        number /= 10
    }
    cubeop <- sum
}

func main() {
    number := 589
    sqrch := make(chan int)
    cubech := make(chan int)
    go calcSquares(number, sqrch)
    go calcCubes(number, cubech)
    squares, cubes := <-sqrch, <-cubech
    fmt.Println("Final output", squares+cubes)
}

```

9. Directional channels:

```go
// When using channels as function parameters, you can
// specify if a channel is meant to only send or receive
// values. This specificity increases the type-safety of
// the program.

package main

import "fmt"

// This `ping` function only accepts a channel for sending
// values. It would be a compile-time error to try to
// receive on this channel.
func ping(pings chan<- string, msg string) {
    pings <- msg
}

// The `pong` function accepts one channel for receives
// (`pings`) and a second for sends (`pongs`).
func pong(pings <-chan string, pongs chan<- string) {
    msg := <-pings
    pongs <- msg
}

func main() {
    pings := make(chan string, 1)
    pongs := make(chan string, 1)
    ping(pings, "passed message")
    pong(pings, pongs)
    fmt.Println(<-pongs)
}
```

10. Select statement example:

```go
// Go's _select_ lets you wait on multiple channel
// operations. Combining goroutines and channels with
// select is a powerful feature of Go.

package main

import "time"
import "fmt"

func main() {

    // For our example we'll select across two channels.
    c1 := make(chan string)
    c2 := make(chan string)

    // Each channel will receive a value after some amount
    // of time, to simulate e.g. blocking RPC operations
    // executing in concurrent goroutines.
    go func() {
        time.Sleep(1 * time.Second)
        c1 <- "one"
    }()
    go func() {
        time.Sleep(2 * time.Second)
        c2 <- "two"
    }()

    // We'll use `select` to await both of these values
    // simultaneously, printing each one as it arrives.
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-c1:
            fmt.Println("received", msg1)
        case msg2 := <-c2:
            fmt.Println("received", msg2)
        }
    }
}
```

11. Select part 2: Which one will be printed out? WHY did it run like this? 

```go
package main

import (  
    "fmt"
    "time"
)

func server1(ch chan string) {  
    time.Sleep(6 * time.Second)
    ch <- "from server1"
}
func server2(ch chan string) {  
    time.Sleep(3 * time.Second)
    ch <- "from server2"

}
func main() {  
    output1 := make(chan string)
    output2 := make(chan string)
    go server1(output1)
    go server2(output2)
    select {
    case s1 := <-output1:
        fmt.Println(s1)
    case s2 := <-output2:
        fmt.Println(s2)
    }
}
```

12. SELECT with default:

```go
package main

import (  
    "fmt"
    "time"
)

func process(ch chan string) {  
    time.Sleep(10500 * time.Millisecond)
    ch <- "HERE'S JOHNNY!!!!"
}

func main() {  
    ch := make(chan string)
    go process(ch)
    for {
        time.Sleep(1000 * time.Millisecond)
        select {
        case v := <-ch:
            fmt.Println("BANG!! CRASH!!.... ", v)
            return
        default:
            fmt.Println("All work and no play makes Jack a Dull Boy")
        }
    }

}
```

13. Buffered channels- simple example that works:

```go
package main

import (  
    "fmt"
)


func main() {  
    ch := make(chan string, 2)
    ch <- "Thanos"
    ch <- "Iron Man"
    fmt.Println(<- ch)
    fmt.Println(<- ch)
}

```

14. Let's take a close look at this one:

```go
package main

import (  
    "fmt"
    "time"
)

func write(ch chan int) {  
    for i := 0; i < 5; i++ {
        ch <- i
        fmt.Println("successfully wrote", i, "to ch")
    }
    close(ch)
}
func main() {  
    ch := make(chan int, 2)
    go write(ch)
    time.Sleep(2 * time.Second)
    for v := range ch {
        fmt.Println("read value", v,"from ch")
        time.Sleep(2 * time.Second)

    }
}
```

15. Capacity and lengths of channels:

```go
package main

import (
	"fmt"
)

func main() {
    mcu := make(chan string, 3)
    mcu <- "Spiderman"
    mcu <- "Thanos"
    mcu <- "Iron Man"
    fmt.Println("marvel cinematic universe can hold this many characters: ", cap(mcu))
    fmt.Println("marvel cinematic universe has this many characters: ", len(mcu))
    fmt.Println("Thanos kills this guy (reads from the mcu channel)", <-mcu)
    fmt.Println("new marvel cinematic universe is", len(mcu))
}

```

16. WaitGroups struct types:

```go
package main

import (  
    "fmt"
    "sync"
    "time"
)

func process(i int, wg *sync.WaitGroup) {  
    fmt.Println("started Goroutine ", i)
    time.Sleep(2 * time.Second)
    fmt.Printf("Goroutine %d ended\n", i)
    wg.Done()
}

func main() {  
    no := 3
    var wg sync.WaitGroup
    //So we've spawned THREE goRoutines here. All three should be going.
    for i := 0; i < no; i++ {
        wg.Add(1)
        go process(i, &wg) //We need to pass the address here or we will be
        //making multiple copies of wg.
    }
    wg.Wait() //By calling this we are going to wait until ALL goroutines are done.
    fmt.Println("All go routines finished executing")
}
```

17. Worker Pools- step by step. FIRST- create two structs:

```go
type Job struct {  
    id       int
    randomno int
}
type Result struct {  
    job         Job
    sumofdigits int
}
//Notice that we're using NESTED structs here. 
//THIS is how you do nested structs. 
//THIS is how you DON'T do nested structs:
/*
type Configuration struct {
    Val   string
    Proxy struct {
        Address string
        Port    string
    }
}
Just make a new struct!
*/
```

18. Now we're going to create our channels that will write to these structs. Remember how we stated that channels could be of __any type__? Well...

```go
var jobs = make(chan Job, 10)  
var results = make(chan Result, 10)  
```

19. Now we're going to create the function that actually **does** a thing. If you want to take the time to understand it- by all means! But really- this could be __literally anything__- from an abstraction point just consider that this is a function that does-a-thing:

```go
func digits(number int) int {  
    sum := 0
    no := number
    for no != 0 {
        digit := no % 10
        sum += digit
        no /= 10
    }
    time.Sleep(2 * time.Second)
    return sum
}
```

20. NOW- back to this- let's write a function that creates a goRoutine. So what's going on here? 

```go
func worker(wg *sync.WaitGroup) {  
    for job := range jobs {
        output := Result{job, digits(job.randomno)}
        results <- output
    }
    wg.Done()
}
```

We're creating a WaitGroup....and saying: okay- the worker is going to take the waitgroup as an input, then range through all of the **jobs** in the **jobs** channel (defined above to be length 10). The output will be written to the **results** channel as type RESULT (the name of the struct). You see- **digits** is the task that the worker is __actually__ performing.

21. The next thing we want to do is create a **worker pool** (that group of eager, waiting workers waiting for tasks):

```go
func createWorkerPool(noOfWorkers int) {  
    var wg sync.WaitGroup
    for i := 0; i < noOfWorkers; i++ {
        wg.Add(1)
        go worker(&wg)
    }
    wg.Wait()
    close(results)
}
```

So what we're doing here is taking the number of workers that we want to create as a parameter. It will call on the workgroup to ADD 1 **before** creating the goRoutine to add to the waitgroup counter. Finally it calls **wg.Wait()** to allow all of the workers to be created. Once this is done it **closes** the results channel as nothing else will need to be written there.

22. Okay! So we've craeted our worker pool. Now we need to allocate jobs to each of our workers:

```go
func allocate(noOfJobs int) {  
    for i := 0; i < noOfJobs; i++ {
        randomno := rand.Intn(999)
        job := Job{i, randomno}
        jobs <- job
    }
    close(jobs)
}
```

SO- the **allocate** function takes the number of jobs we want to create as the input parameter and assigns a random number (up to 998). It then takes a job struct with the job number and random number and assigns it to the **jobs** channel...it then closes the jobs channel after assigning all jobs.

23. Finally we want to create a **result** function that basically just prints the output from the **results** channel. To use this we're going to create a function that takes another channel with a BOOLEAN type in and writes to that:

```go
func result(done chan bool) {  
    for result := range results { //results is the channel that holds our results
        fmt.Printf("Job id %d, input random no %d , sum of digits %d\n", result.job.id, result.job.randomno, result.sumofdigits)
    }
    done <- true
}
```

24. Finally we are going to create our **main** function measuring time:

```go
func main() {  
    startTime := time.Now()
    noOfJobs := 100 //Obvious
    go allocate(noOfJobs)
    done := make(chan bool)
    go result(done)
    noOfWorkers := 10
    createWorkerPool(noOfWorkers)
    <-done
    endTime := time.Now()
    diff := endTime.Sub(startTime)
    fmt.Println("total time taken ", diff.Seconds(), "seconds")
}
```

Okay- so we start the execution time and then calculate total time taken...then display. We set the number of jobs to 100 and allocate them. We then assign 10 workers to teh process, finish the process, and print out the time. Put all of these into your main package then run it. Check out the time!!

25. NOW- switch from 10 to 20 workers. Run again and take a look at the time. What happened? 