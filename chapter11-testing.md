# Chapter 11 - Testing

Testing (automated testing) is the practice of writing small programs that check the code under test (production code) behaves as expected for certain inputs, which are usually carefully chose to exercise certain features or randomly chosen to ensure broad coverage.

Go's approach to testing 
- rather low-tech
- relies on one command `go test`
- set of conventions for writing tests that `go test` can run
- effective for testing
- extends to benchmarks and systematic examples for documentation

Writing test code not much different from original program
- short functions that focus on part of the task
- careful of boundary conditions
- think about data structures
- reason about what results a computation should produce from suitable inputs


## The `go test` Tool

- `go test` subcommand is a test driver for Go packages that are organized according to certain conventions
- in a package directory, files ending with `_test.go` are not ordinarily part of the package when built with `go build` but are part of the package when built with `go test`
- in a `*_test.go` file, three kinds of functions are treated in a special way: tests, benchmarks and examples
- `go test` tool scans the `*_test.go` files for these special functions, creates a temporary `main` package, that calls them all in a proper way, builds and runs it, reports the results and cleans up.

_test function_ : whose name begins with `Test`
- exercises some program logic for correct behavior
- `go test` calls the test function and reports the result -- `PASS` or `FAIL`

_benchmark function_: name begins with `Benchmark`
- measures the performance of some operation
- `go test` reports the mean execution time of the operation

_example function_: name begins with `Example`
- provides machine-checked documentation

## `Test` functions

- each test file must import the `Testing` package
- test function signature:
```
func TestName(t *testing.T) {
    //...
}
```
- test function name begins with `Test` and optional suffix _`Name`_ begins with capital letter
- `t` parameter provides methods for reporting test failures and logging additional info

`go test -v`: prints name and execution time of each test in the package
`go test -run=<regex>`: runs only those tests whose function names match the regex pattern


### Example:

Writing tests during development, testing for bug reports, fixing bugs

`word.go`:
simple function to check if a string is a palindrome

```
// Package word provides utilities for word games.
package word

// IsPalindrome reports whether s reads the same forward and backward.
func IsPalindrome(s string) bool {
    for i := range s {
        if s[i] != s[len(s)-1-i] {
            return false
        }
    return true
}
```

`word_test.go`:
checks that `isPalindrome` function gives the right answer for a single input and returns errors via `t.Error`

```
package word

import "testing"

func TestPalindrome(t *testing.T) {
    if !IsPalindrome("detartrated") {
        t.Error(`IsPalindrome("detartrated") = false`)
    }
    if !IsPalindrome("kayak") {
        t.Error(`IsPalindrome("kayak") = false`)
    }
}

func TestNonPalindrome(t *testing.T) {
    if IsPalindrome("palindrome") {
        t.Error(`IsPalindrome("palindrome") = true`)
    }
}
```

`go test` or `go build` with no arguments operates on the current directory

**Bug reports:**

`été`, `A man, a plan, a canal: Panama.`

To reproduce these bugs, write more tests - it's easier and faster than having to run the code for these inputs and trying to step through the code to find the error.

```
func TestFrenchPalindrome(t *testing.T) {
    if !IsPalindrome("été") {
        t.Error(`IsPalindrome("été") = false`)
    }
}

func TestCanalPalindrome(t *testing.T) {
    input := "A man, a plan, a canal: Panama"
    if !IsPalindrome(input) {
        t.Errorf(`IsPalindrome(%q) = false`, input)
    }
}
```
`Errorf()` functions the same way as `Printf()` and provides formatting

With these new tests, `go test` command fails with informative errors that help us reproduce the bugs.

Investigation yields the cause: 
- `isPalindrome` uses byte sequences, not rune sequences, hence non-ASCII characters in `été` confuse it
- not ignoring spaces, punctuation and letter case

**New and improved function:**

```
// Package word provides utilities for word games.
package word

import "unicode"

// IsPalindrome reports whether s reads the same forward and backward.
// Letter case is ignored, as are non-letters.
func IsPalindrome(s string) bool {
    var letters []rune
    for _, r := range s {
        if unicode.IsLetter(r) {
            letters = append(letters, unicode.ToLower(r))
        }
    }
    for i := range letters {
        if letters[i] != letters[len(letters)-1-i] {
            return false
        }
    }
    return true
}
```

**New and improved test cases:**

Combine all test cases into a table.

```
func TestIsPalindrome(t *testing.T) {
    var tests = []struct {
        input string
        want  bool
    }{
        {"", true},
        {"a", true},
        {"aa", true},
        {"ab", false},
        {"kayak", true},
        {"detartrated", true},
        {"A man, a plan, a canal: Panama", true},
        {"Evil I did dwell; lewd did I live.", true},
        {"Able was I ere I saw Elba", true},
        {"été", true},
        {"Et se resservir, ivresse reste.", true},
        {"palindrome", false}, // non-palindrome
        {"desserts", false},   // semi-palindrome
    }
    for _, test := range tests {
        if got := IsPalindrome(test.input); got != test.want {
            t.Errorf("IsPalindrome(%q) = %v", test.input, got)
        }
    }
}
```

### _Table-driven_ testing style

- useful for checking that a function works on inputs selected to exercise interesting cases in the logic 
- very commonly used in Go
- easy to add or remove table entries as needed
- assertion logic is not duplicated
- can produce a good error message


### Gotchas

- output of a failing test does not include the full stack trace at the moment of the call to `t.Errorf`
- `t.Errorf` does not panic or stop the execution of the other tests (unlike other assertion failures in other languages)
- tests are independent of each other
- if an early table entry in a test fails, later table entries will still be checked (hence it's easier to catch multiple failures in a single run)
- when we really want to stop a test function, we can use `t.Fatal` or `t.Fatalf` -- must  be called from the same go routine as the `Test` function, not from another one created during the test.

### Writing failure messages

- usually of the form "`f(x) = y, want z`"
  - `f(x)` explains the attempted operation and input
  - `y` is the actual output
  - `z` is the expected output
- whenever applicable, actual Go syntax is used for `f(x)`
- displaying `x` is important for table-driven testing because a given assertion is executed many times with many input values
- avoid boilerplate and redundant information
  - for a boolean function, avoid `want z` part since it adds no information
- if `x`, `y` or `z` is lengthy, print a concise summary instead
- strive to help a programmer who is debugging a test failure

## Randomized Testing

- explores a broader range of inputs by constructing inputs at random
- how do we know what output to expect?
  - write an alternative implementation of the function (using a less efficient but simpler and clearer algorithm) and compare the results of the two implementations
  - create input values according to a pattern so we know what output to expect

### Example:

create input values according to a pattern - create a random palindrome and check if our function returns true with that as input

```
“import "math/rand"

// randomPalindrome returns a palindrome whose length and contents
// are derived from the pseudo-random number generator rng.
func randomPalindrome(rng *rand.Rand) string {
    n := rng.Intn(25) // random length up to 24
    runes := make([]rune, n)
    for i := 0; i < (n+1)/2; i++ {
        r := rune(rng.Intn(0x1000)) // random rune up to '\u0999'
        runes[i] = r
        runes[n-1-i] = r
    }
    return string(runes)
  }

func TestRandomPalindromes(t *testing.T) {
    // Initialize a pseudo-random number generator.
    seed := time.Now().UTC().UnixNano()
    t.Logf("Random seed: %d", seed)
    rng := rand.New(rand.NewSource(seed))

    for i := 0; i < 1000; i++ {
        p := randomPalindrome(rng)
        if !IsPalindrome(p) {
            t.Errorf("IsPalindrome(%q) = false", p)
        }
    }
}
```

- randomized tests are nondeterministic
- failure logs should report sufficient information to reproduce the failure
- sometimes simpler to print the seed than to dump the entire input data structure
- using current time as source of randomness results in new inputs at every execution of the test - especially useful for automated system running tests periodically

## Testing a Command

- `go test` tool usually used for testing library packages
  - with little effort, can be used for testing commands as well
- `main` package usually produces an executable program
  - but can be imported as a library to

- test code is in the same package as production code
- package name is `main` and there is a `main` function defined in the package
- during testing this package acts as a library and `main` function is ignored
- make sure that the code being tested does not call `log.Fatal` or `os.Exit` since these will stop the processes (calling these functions is the sole right of `main` function)
- expected errors such as bad user input, missing files, improper configuration should be reported by non-nil `error` value

### Example

`echo.go`
```
// Echo prints its command-line arguments.
package main

import (
    "flag"
    "fmt"
    "io"
    "os"
    "strings"
)

var (
    n = flag.Bool("n", false, "omit trailing newline")
    s = flag.String("s", " ", "separator")
)

var out io.Writer = os.Stdout // modified during testing

func main() {
    flag.Parse()
    if err := echo(!*n, *s, flag.Args()); err != nil {
        fmt.Fprintf(os.Stderr, "echo: %v\n", err)
        os.Exit(1)
    }
}

func echo(newline bool, sep string, args []string) error {
    fmt.Fprint(out, strings.Join(args, sep))
    if newline {
        fmt.Fprintln(out)
    }
    return nil
}
```

In the test, various different arguments and flags will be passed in to `echo()` to check the output in each case. To reduce dependence on global variables, we pass in `args` to `echo()`

Since `echo()` writes to a global `io.Writer` (`out`) instead of `os.Stdout`, we can substitute this with another `Writer` implementation in the tests and use that to inspect the output later

`echo_test.go`
```
package main

import (
    "bytes"
    "fmt"
    "testing"
)

func TestEcho(t *testing.T) {
    var tests = []struct {
        newline bool
        sep     string
        args    []string
        want    string
    }{
        {true, "", []string{}, "\n"},
        {false, "", []string{}, ""},
        {true, "\t", []string{"one", "two", "three"}, "one\ttwo\tthree\n"},
        {true, ",", []string{"a", "b", "c"}, "a,b,c\n"},
        {false, ":", []string{"1", "2", "3"}, "1:2:3"},
    }

    for _, test := range tests {
        descr := fmt.Sprintf("echo(%v, %q, %q)",
            test.newline, test.sep, test.args)

        out = new(bytes.Buffer) // captured output
        if err := echo(test.newline, test.sep, test.args); err != nil {
            t.Errorf("%s failed: %v", descr, err)
            continue
        }
        got := out.(*bytes.Buffer).String()
        if got != test.want {
            t.Errorf("%s = %q, want %q", descr, got, test.want)
        }
    }
}
```


## White-Box Testing

Catgegorizing tests by level of knowledge of the internal workings of the package under test:
- Black box testing
  - assumes nothing about the package except what's exposed by its API and specified by its documentation - the package's internals are opaque
  - more robust
  - need fewer updates as the software evolves
  - reveal flaws in API design
  - example: `TestIsPalindrome()`
- White box (clear box) testing
  - priviledged access to internal functions and data structure of the package
  - can make observations and changes that an ordinary client cannot
  - provide more detailed coverage of trickier parts of the implementation
  - example: `TestEcho()`
  - replace parts of production code with easy to test "fake" implementations (similar to writing to a package level `out` in `TestEcho()`)
    - fake implementations are 
      - simpler to configure
      - easier to observe
      - more reliable and predictable
      - avoid undesirable effect of updating a production database

### Example:

"Fake" implementation to avoid sending out "real" notification emails.
Quota-checking logic in a web service that provides networked storage to users. When users exceed 90% of their quota, the system sends them a warning email.

`storage1.go`:

```
package storage

import (
    "fmt"
    "log"
    "net/smtp"
)

var usage = make(map[string]int64)
func bytesInUse(username string) int64 { return usage[username] } 

// Email sender configuration.
// NOTE: never put passwords in source code!
const sender = "notifications@example.com"
const password = "correcthorsebatterystaple"
const hostname = "smtp.example.com"

const template = `Warning: you are using %d bytes of storage,
%d%% of your quota.`

func CheckQuota(username string) {
    used := bytesInUse(username)
    const quota = 1000000000 // 1GB
    percent := 100 * used / quota
    if percent < 90 {
        return // OK
    }
    msg := fmt.Sprintf(template, used, percent)
    auth := smtp.PlainAuth("", sender, password, hostname)
    err := smtp.SendMail(hostname+":587", auth, sender,
        []string{username}, []byte(msg))
    if err != nil {
        log.Printf("smtp.SendMail(%s) failed: %s", username, err)
    }
}
```

To test this without sending out a real email, we store the logic to send email in a new function and store that in an unexported package level variable:

`storage2.go`:


```
var notifyUser = func(username, msg string) {
    auth := smtp.PlainAuth("", sender, password, hostname)
    err := smtp.SendMail(hostname+":587", auth, sender,
        []string{username}, []byte(msg))
    if err != nil {
        log.Printf("smtp.SendEmail(%s) failed: %s", username, err)
    }
}

func CheckQuota(username string) {
    used := bytesInUse(username)
    const quota = 1000000000 // 1GB
    percent := 100 * used / quota
    if percent < 90 {
        return // OK
    }
    msg := fmt.Sprintf(template, used, percent)
    notifyUser(username, msg)
}
```

Now, we can substitute a fake notification mechanism for the real one and check if the notified user and contents of the email are as expected

```
package storage

import (
    "strings"
    "testing"
)

func TestCheckQuotaNotifiesUser(t *testing.T) {
    var notifiedUser, notifiedMsg string
    notifyUser = func(user, msg string) {
        notifiedUser, notifiedMsg = user, msg
    }

    const user = "joe@example.org"
    usage[user]=  980000000 // simulate a 980MB-used condition

    CheckQuota(user)
    if notifiedUser == "" && notifiedMsg == "" {
        t.Fatalf("notifyUser not called")
    }
    if notifiedUser != user {
        t.Errorf("wrong user (%s) notified, want %s",
            notifiedUser, user)
    }
    const wantSubstring = "98% of your quota"
    if !strings.Contains(notifiedMsg, wantSubstring) {
        t.Errorf("unexpected notification message <<%s>>, "+
            "want substring %q", notifiedMsg, wantSubstring)
    }
}
```

Pitfall/gotcha:
- we modified the implementation of `notifyUser` so `CheckUserQuota()` will not work as expected any more
- at the end of the test, we need to set the `notifyUser` to its original value using `defer()`

Modified test:

```
func TestCheckQuotaNotifiesUser(t *testing.T) {
    // Save and restore original notifyUser.
    saved := notifyUser
    defer func() { notifyUser = saved }()

    // Install the test's fake notifyUser.
    var notifiedUser, notifiedMsg string
    notifyUser = func(user, msg string) {
        notifiedUser, notifiedMsg = user, msg
    }
    // ...rest of test...
}
```

Uses of this pattern:
- temporarily save and restore global variables (like command line flags, debugging options, perfomance parameters)
- install and remove hooks that cause production code to call test code when something interesting happens
- coax the production code in rare events like timeouts, errors, etc.
This pattern is safe because `go test` does not run multiple tests concurrently.

## External Test Packages

External test packages avoid import cycles and allow tests, especially integration tests (which test the interaction of several components), to import other packages freely, exactly as an application would.

### Problem:
Test of a lower-level package imports higher-level package. This will cause a cycle in the import graph and won't be allowed in Go.

### Solution:
Create an external test package, i.e. a file in the same package whose package name is suffixed with `_test`. This suffix is an indication to `go test` that it should build an additional package with just these files and run its tests. In terms of design layers, an external test package is located higher than both the packages it depends on.

### Example:

`net/url`: contains a URL parser
`net/http`: contains a web server and an HTTP client library
- higher level `net/http` depends on `net/url`
- `net/url` contains a test that demonstrates the interaction between URLs and HTTP client library
- test function in `net/url` will create an import cycle
- create an externa test package by putting the tests under `net/url` in a file with package name `url_test`

### `go list`

Summarize which Go files in a package directory are production code, in-package tests and external tests
Example:
```
# production code - included during `go build`

$ go list -f={{.GoFiles}} fmt
[doc.go format.go print.go scan.go]


# ---
# in-package test files - only included during `go test`

$ go list -f={{.TestGoFiles}} fmt
[export_test.go]


# ---
# external test package files - must include fmt package to use it
# only included during `go test`

$ go list -f={{.XTestGoFiles}} fmt
[fmt_test.go scan_test.go stringer_test.go]
```

#### The curious case of `export_test.go`

Sometimes an external test package may need priviledged access to the internals of the package under test - if a white box test has an import cycle.
The trick to use in such a case is:
- add the declaration to an in-package `_test.go` file to expose the necessary internals
- gives the external test a back door into the internals of the package
- if the source file exists solely for this purpose and contains no tests then we name it `export_test.go`

## Writing Effective Tests

A good test 
- has a user interface
- does not explode on failure
- prints a clear and succint description of the failure and any other necessary context
- should not give up after a single run, but try to report several failures in a single run
- maintainer should not need to read the source code to decipher the failure

### Example of a bad test

```
import (
    "fmt"
    "strings"
    "testing"
)

// A poor assertion function.
func assertEqual(x, y int) {
    if x != y {
        panic(fmt.Sprintf("%d != %d", x, y))
    }
}

func TestSplit(t *testing.T) {
    words := strings.Split("a:b:c", ":")
    assertEqual(len(words), 3)
    // ...
}
```

### Example of a good test

```
func TestSplit(t *testing.T) {
    s, sep := "a:b:c", ":"
    words := strings.Split(s, sep)
    if got, want := len(words), 3; got != want {
        t.Errorf("Split(%q, %q) returned %d words, want %d",
            s, sep, got, want)
    }
    // ...
}
```
next step is to replace `if` with a table-driven test loop

## Avoiding Brittle Tests

Tests that fail when a sound change was made to the program are called _brittle_.
Most brittle tests are written as _change detectors_ or _status quo_ tests - their cost can quickly outweigh the benefits due to their breakage on any change to production code

To avoid brittle tests:
- check only properties you care about
- test the program's simpler and more stable interfaces instead of internal functions
- use assertions selectively
- don't check for exact string matches but substrings which are less likely to change
- write a function to break down a complex output to its essentials so more reliable assertions can be made

## Coverage

The degree to which a test suite exercises that package under test is called the test's coverage.
- cannot be quantified precisely
- heuristics that can help direct testing efforts to where they are most beneficial

### Statement coverage

Fraction of source statements that are executed at least once during the test

**`cover` tool:**
- integrated into `go test`
- used to measure statement coverage

to see usage:
```
$ go tool cover
Usage of 'go tool cover':
Given a coverage profile produced by 'go test':
    go test -coverprofile=c.out

Open a web browser displaying annotated source code:
    go tool cover -html=c.out
...
```
(`go tool` command runs one of the executables from the Go toolchain. These programs live in the directory `$GOROOT/pkg/tool/${GOOS}_${GOARCH}`)


Run a test with `-coverprofile` flag to see it's statement coverage:
```
$ go test -run=Coverage -coverprofile=c.out gopl.io/ch7/eval
ok      gopl.io/ch7/eval    0.032s  coverage: 68.5% of statements
```

`coverprofile` flag enables the collection of coverage data by instrumenting the production code
- it modifies a copy of the source code
- before each block of statements is executed, a boolean variable is set, with one variable per block
- before the modified program exits, it writes the value of each variable to the specified log file c.out 
- prints a summary of the fraction of statements that were executed
- if all you need is the summary, use `go test -cover`

`go test` with the `-covermode=count` flag
- instrumentation for each block increments a counter instead of setting a boolean
- resulting log of execution counts of each block enables quantitative comparisons between “hotter” blocks, which are more frequently executed, and “colder” ones.

After gathering the data, run the cover tool
- processes the log
- generates an HTML report
- opens it in a new browser window

```
$ go tool cover -html=c.out
```
Each statement is covered green if it was covered and red if it wasn't covered.


## `Benchmark` Functions

Benchmarking - measuring the performance of a function on a fixed workload

`Benchmark` function looks like a test  function but the name starts with `Benchmark` and takes `*testing.B` as the argument
- ``*testing.B` provides most of the same functions as `*testing.T` plus some performance measurement related methods and exposes an int field N which controls the number of iterations

### Example:

```
“import "testing"

func BenchmarkIsPalindrome(b *testing.B) {
    for i := 0; i < b.N; i++ {
        IsPalindrome("A man, a plan, a canal: Panama")
    }
}”
```
To run it:
```
$ go test -bench=.
PASS
BenchmarkIsPalindrome-8 1000000              1035 ns/op
ok      gopl.io/ch11/word2      2.179s
```

- benchmark name’s numeric suffix, 8 here, indicates the value of GOMAXPROCS, which is important for concurrent benchmarks
- each call to `IsPalindrome` took about 1.035 microseconds, averaged over 1,000,000 runs. 
- benchmark runner initially has no idea how long the operation takes, it makes some initial measurements using small values of N and then extrapolates to a value large enough for a stable timing measurement to be made

The reason the loop is implemented by the benchmark function, and not by the calling code in the test driver, is so that the benchmark function has the opportunity to execute any necessary one-time setup code outside the loop without this adding to the measured time of each iteration. If this setup code is still perturbing the results, the `testing.B` parameter provides methods to stop, resume, and reset the timer, but these are rarely needed.

### Memory allocation benchmarks

`-benchmem` command-line flag will include memory allocation statistics in its report

```
“ go test -bench=. -benchmem
PASS
BenchmarkIsPalindrome    1000000  1026 ns/op   304 B/op  4 allocs/op
```

### Relative timings of operations

Comparative benchmarks take the form of a single parameterized function, called from several Benchmark functions with different values

```
func benchmark(b *testing.B, size int) { /* ... */ }
func Benchmark10(b *testing.B)   { benchmark(b, 10) }
func Benchmark100(b *testing.B)  { benchmark(b, 100) }
func Benchmark1000(b *testing.B) { benchmark(b, 1000) }
```

Do not use `b.N` as input size. Unless you interpret it as an iteration count for a fixed-size input, the results of your benchmark will be meaningless.


## Profiling

- automated approach to performance measurement based on sampling a number of profile events during execution
- then extrapolating from them during a post-processing step
- the resulting statistical summary is called a profile

Go supports many kinds of profiling
- each concerned with a different aspect of performance
- all of them involve recording a sequence of events of interest
- each has an accompanying stack trace(the stack of function calls active at the moment of the event)
- `go test` tool has built-in support for several kinds of profiling

CPU profile
- identifies the functions whose execution requires the most CPU time
- currently running thread on each CPU is interrupted periodically by the operating system every few milliseconds
- each interruption records one profile event before normal execution resumes

Heap profile
- identifies the statements responsible for allocating the most memory
- profiling library samples calls to the internal memory allocation routines so that on average, one profile event is recorded per 512KB of allocated memory.

Blocking profile
- identifies the operations responsible for blocking goroutines the longest (such as system calls, channel sends and receives, and acquisitions of locks)
- profiling library records an event every time a goroutine is blocked by one of these operations.

Gathering a profile for code under test can be performed by using these flags. Using more than one flag at a time may skew the results:
```
$ go test -cpuprofile=cpu.out
$ go test -blockprofile=block.out
$ go test -memprofile=mem.out”
```

To profile non-test programs, we can use the `runtime` API (details not mentioned in the chapter)

### Analyzing a profile using `pprof`

- accessed using `go tool pprof`
- `pprof` has many features and arguments
- for basic use, it only needs executable that produced the profile and the profile log as arguments
- for efficiency of profiling, the log contains function addresses instead of names, hence the executable is needed by `pprof`
- `go test` usually discards the executable but if profiling is on, it saves it under `foo.test`

#### Example:
Gather and display a simple cpu profile for one of the benchmarks from the `net/http` package. It is usually better to profile specific benchmarks that have been constructed to be representative of workloads one cares about. Benchmarking test cases is almost never representative, which is why we disabled them by using the filter `-run=NONE`.

```
$ go test -run=NONE -bench=ClientServerParallelTLS64 \
    -cpuprofile=cpu.log net/http
PASS
BenchmarkClientServerParallelTLS64-8  1000
   3141325 ns/op  143010 B/op  1747 allocs/op
ok      net/http       3.395s

$ go tool pprof -text -nodecount=10 ./http.test cpu.log
2570ms of 3590ms total (71.59%)
Dropped 129 nodes (cum <= 17.95ms)
Showing top 10 nodes out of 166 (cum >= 60ms)
    flat  flat%   sum%     cum   cum%
  1730ms 48.19% 48.19%  1750ms 48.75%  crypto/elliptic.p256ReduceDegree
   230ms  6.41% 54.60%   250ms  6.96%  crypto/elliptic.p256Diff
   120ms  3.34% 57.94%   120ms  3.34%  math/big.addMulVVW
   110ms  3.06% 61.00%   110ms  3.06%  syscall.Syscall
    90ms  2.51% 63.51%  1130ms 31.48%  crypto/elliptic.p256Square
    70ms  1.95% 65.46%   120ms  3.34%  runtime.scanobject
    60ms  1.67% 67.13%   830ms 23.12%  crypto/elliptic.p256Mul
    60ms  1.67% 68.80%   190ms  5.29%  math/big.nat.montgomery
    50ms  1.39% 70.19%    50ms  1.39%  crypto/elliptic.p256ReduceCarry
    50ms  1.39% 71.59%    60ms  1.67%  crypto/elliptic.p256Sum
```

`-text` flag specifies the output format, in this case, a textual table with one row per function, sorted so the “hottest” functions—those that consume the most CPU cycles—appear first.

`-nodecount=10 flag` limits the result to 10 rows.

#### `pprof`'s Graphical Displays

These require GraphViz, which can be downloaded from www.graphviz.org. The -web flag then renders a directed graph of the functions of the program, annotated by their CPU profile numbers and colored to indicate the hottest functions

### Recommended Reading

"Profiling Go Programs" article on Go Blog

## `Example` Functions

- treated specially by `go test`
- name starts with `Example`
- has neither parameters nor results

Example for `isPalindrome`:

```
func ExampleIsPalindrome() {
    fmt.Println(IsPalindrome("A man, a plan, a canal: Panama"))
    fmt.Println(IsPalindrome("palindrome"))
    // Output:
    // true
    // false
}
```

`Example` functions serve three purposes. 

1. The primary one is documentation:
  - a good example can be a more succinct or intuitive way to convey the behavior of a library function
  - an example can also demonstrate the interaction between several types and functions belonging to one API
  - example functions are real Go code, subject to compile-time checking, so they don’t become stale as the code evolves

2. Examples are executable tests run by go test
  - if the example function contains a final `// Output`: comment the test driver will execute the function and check that what it printed to its standard output matches the text within the comment

3. Hands-on experimentation
  - `godoc` server at `golang.org` uses the Go Playground to let the user edit and run each example function from within a web browser
  - often the fastest way to get a feel for a particular function or language feature

### Suffix of the `Example` function decides its location in documentation

web-based documentation server `godoc` associates `example` functions with the function or package they exemplify
- `ExampleIsPalindrome` would be shown with the documentation for the `IsPalindrome` function
- `example` function called just `Example` would be associated with the `word` package as a whole.
