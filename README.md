# gotesting
Quick'n'dirty testing functions for go

## License

MIT. Check LICENSE for more info.

## Usage

```go
// Fails the test if a condition is not met. Returns the same value so that you
// can chain dependencies.
if assert(t, goIsFun, "go *is* fun %s", user) {
    assert(t, isGoCoder(user), "envious?")
}

// Fails the test if two values are no equal (deep). Returns true if they are
// equal.
if equals(t, 42, theAnswerToLifeTheUniverseAndEverything) {
    equalsf(t, planetaryComputer, deepThought.CreateComputer(), "still 42")
}

// Fails the test if two values are not equal (deep). Returns true if they are
// not equal.
if notEquals(t, Quick, testingInGo) {
    notEqualsf(t, Slow, testingWithThisLibrary, "I must have got it wrong")
}

// Fails the test if the error is not nil. Returns true if the error is not nil.
conn, err := net.Dial("tcp", "skynet.org:80")
if ok(t, err) {
    err = ddos(conn);
    okf(err, "Call Sarah Conner")
}

// Fails and terminates the test if error in not nil.
conn, err := net.Dial("tcp", "skynet.org:80")
okNow(t, err)

// Records calls and asserts that they happened.
type singRecorder struct{
    callRecorder
}

func (w *singRecorder) Sing(p string) error {
    w.record(p)
    return nil
}

a := &singRecorder{}
a.Write("Never gonna give you up")
a.Write("Never gonna let you down")
a.Write("Never gonna run around and desert you")

// c checks that the next call matches by name and arguments. Returns true
// if the call matches.
// ec checks that the next call doesn't exist (i.e. not too many calls were 
// made). Returns true if no more calls exist.
c, ec := a.createAsserter(t)
c("Sing", "Never gonna give you up")
c("Sing", "Never gonna let you down")
c("Sing", "Never gonna run around and desert you")
c("Sing", "Never gonna make you cry")
c("Sing", "Never gonna say goodbye")
if c("Sing", "Never gonna tell a lie and hurt you") {
    // Great success!
}
ec()

```

## Installation: Download

Download the file into project.

```sh
wget https://github.com/jcdickinson/gotesting/blob/master/package_test.go
```

## Installation: ✂️ 📋 😍

Copy the following code into a file.

```go
package *_test

import (
	"reflect"
	"runtime"
	"strings"
	"testing"
)

// assert checks the provided condition and fails the test if it is false.
// assert formats its arguments using default formatting, analogous to Println,
// and records the text in the error log if the condition is false. The return
// value is equal to condition.
func assert(tb testing.TB, condition bool, format string, a ...interface{}) bool {
	if !condition {
		tb.Helper()
		tb.Errorf(format, a...)
		return false
	}
	return true
}

// ok checks the provided error, fails the test if it is not nil and records
// the error in the log. The return value is true when err is nil.
func ok(tb testing.TB, err error) bool {
	if err != nil {
		tb.Helper()
		tb.Errorf("unexpected error: %v", err)
		return false
	}
	return true
}

// okf checks the provided error and fails the test if it is not nil. okf
// formats its arguments using default formatting, analogous to Println, and
// records the text in the error log if err is not nil. The return value is true
// when err is nil.
func okf(tb testing.TB, err error, format string, a ...interface{}) bool {
	if err != nil {
		tb.Helper()
		v := append(a, err)
		tb.Errorf(format, v...)
		return false
	}
	return true
}

// okNow checks the provided error, fails the test if it is not nil and records
// the error in the log. The test is aborted if err is not nil.
func okNow(tb testing.TB, err error) {
	if err != nil {
		tb.Helper()
		tb.Errorf("unexpected error: %v", err)
		tb.FailNow()
	}
}

// oknow checks the provided error and fails the test if it is not nil. okNowf
// formats its arguments using default formatting, analogous to Println, and
// records the text in the error log if err is not nil. The test is aborted if
// err is not nil.
func okNowf(tb testing.TB, err error, format string, a ...interface{}) {
	if err != nil {
		tb.Helper()
		v := append(a, err)
		tb.Errorf(format, v...)
		tb.FailNow()
	}
}

// equals checks the provided values for deep equality, fails the test if
// they are not equal and records a message in the log. The return value is true
// if the values are equal.
func equals(tb testing.TB, exp, act interface{}) bool {
	if !reflect.DeepEqual(exp, act) {
		tb.Helper()
		tb.Errorf("expected %#v, got %#v", exp, act)
		return false
	}
	return true
}

// notEquals checks the provided values for deep equality and fails the test if
// they are equal and records a message in the log. The return value is true
// if the values are not equal.
func notEquals(tb testing.TB, unexp, act interface{}) bool {
	tb.Helper()
	if reflect.DeepEqual(unexp, act) {
		tb.Errorf("did not expected %#v", unexp)
		return false
	}
	return true
}

// call contains information about a called function.
type call struct {
	// name contains the name of the function.
	name string
	// args contains the arguments that were passed to the function.
	args []interface{}
}

// callRecorder contains information about functions that were called.
type callRecorder struct {
	// calls contains all the functions called.
	calls []call
}

// record adds a function to a callRecorder. It accepts the parameters passed
// to the method.
func (c *callRecorder) record(args ...interface{}) {
	pc, _, _, ok := runtime.Caller(1)
	if ok {
		name := runtime.FuncForPC(pc - 1).Name()
		i := strings.LastIndex(name, ".")
		if i >= 0 {
			name = name[i+1:]
		}
		c.calls = append(c.calls, call{name, args})
	}
}

// callAsserter checks that the next function has the correct name and
// parameters, fails the test if they are not equal and records a message in
// the log. The return value is true if the name and parameters are equal.
type callAsserter func(name string, args ...interface{}) bool

// endCallAsserter checks that there are no more functions, fails the test if
// there are and records a message in the log. The return value is true if there
// are no more functions.
type endCallAsserter func() bool

// createAsserter creates a callAsserter than can be used to ensure that
// a sequence of calls was satisfied.
func (c callRecorder) createAsserter(tb testing.TB) (callAsserter, endCallAsserter) {
	i := 0
	failed := false
	ca := func(name string, args ...interface{}) bool {
		if i >= len(c.calls) {
			if !failed {
				failed = true
				tb.Helper()
				tb.Errorf("expected more than %d calls", i)
			}
			return false
		}

		call := c.calls[i]
		if call.name != name {
			tb.Errorf("%d: expected call %s, got call %s", i, name, call.name)
			i++
			return false
		} else if !reflect.DeepEqual(args, call.args) {
			tb.Errorf("%d: expected args %v, got args %v", i, args, call.args)
			i++
			return false
		}
		i++
		return true
	}

	ec := func() bool {
		if i < len(c.calls) {
			tb.Errorf("expected exactly %d calls, got %d", i, len(c.calls))
			return false
		}
		return true
	}

	return ca, ec
}
```