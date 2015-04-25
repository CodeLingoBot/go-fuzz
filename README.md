# go-fuzz: randomized testing for Go

Go-fuzz is a coverage-guided fuzzing solution for testing of Go packages.
Fuzzing is mainly applicable to packages that parse complex inputs (both text
and binary), and is especially useful for hardening of systems that parse inputs
from potentially malicious users (e.g. anything accepted over a network).

## Usage

First, you need to write a test function of the form:
```
func Fuzz(data []byte) int
```
Data is a random input generated by go-fuzz, note that in most cases it is
invalid. The return value is interestingness of the input. The suggested
encoding scheme: 0 - invalid input, 1 - valid input (parsed successfully),
2 - valid and interesting in some way input. Negative values are reserved for
future use. In its basic form the Fuzz function just parses the input, and
go-fuzz ensures that it does not panic, crash the program, allocate insane
amount of memory nor hang. Fuzz function can also do application-level checks,
which will make testing more efficient (discover more bugs). For example,
Fuzz function can serialize all inputs that were successfully deserialized,
thus ensuring that serialization can handle everything deserialization can
produce. Or, Fuzz function can deserialize-serialize-deserialize-serialize
and check that results of first and second serialization are equal. Or, Fuzz
function can feed the input into two different implementations (e.g. dumb and
optimized) and check that the output is equal. To communicate application-level
bugs Fuzz function should panic (os.Exit(1) will work too, but panic message
contains more info). Note that Fuzz function should not output to stdout/stderr,
it will slow down fuzzing and nobody will see the output anyway. The exection
is printing info about a bug just before panicing.

Here is an example of a simple Fuzz function for image/png package:
```
package png

import (
	"bytes"
	"image/png"
)

func Fuzz(data []byte) int {
	png.Decode(bytes.NewReader(data))
	return 0
}
```

A more useful Fuzz function would look like:
```
func Fuzz(data []byte) int {
	img, err := png.Decode(bytes.NewReader(data))
	if err != nil {
		if img != nil {
			panic("img != nil on error")
		}
		return 0
	}
	var w bytes.Buffer
	err = png.Encode(&w, img)
	if err != nil {
		panic(err)
	}
	return 1
}
```

The second step is collection of initial input corpus. Ideally, files in the
corpus are as small as possible and as diverse as possible. You can use inputs
used by unit tests and/or generate them. For example, for an image decoding
package you can encode several small bitmaps (black, random noise, white with
few non-white pixels) with different levels of compressions and use that as the
initial corpus. Go-fuzz will deduplicate and minimize the inputs. So throwing in
a thousand of inputs is fine, diversity is more important.

Examples directory contains a bunch of examples of test functions and initial
input corpuses for various packages.

The next step is to get go-fuzz:
```
$ go get github.com/dvyukov/go-fuzz/...
```

Then, build the test program with necessary instrumentation:
```
$ go-fuzz-build github.com/dvyukov/go-fuzz/examples/png
```
This will produce png-fuzz binary.

Now we are ready to go:
```
$ go-fuzz -bin=./png-fuzz -corpus=examples/png/corpus -workdir=~/png-fuzz
```
Go-fuzz will generate and test various inputs in an infinite loop. Every few
seconds it prints logs of the form:
```
2015/04/22 21:39:11 slaves: 1/32, corpus: 123 (45.019565484s ago), crashers: 1, restarts: 1/9999, execs: 123123123 (200000/sec), cover: 0.38%, uptime: 10m45.019565765s
```


## Trophies

- encoding/gob: panic: drop (https://github.com/golang/go/issues/10272)
- encoding/gob: makeslice: len out of range [3 bugs] (https://github.com/golang/go/issues/10273)
- encoding/gob: stack overflow (https://github.com/golang/go/issues/10415)
- encoding/gob: excessive memory consumption (https://github.com/golang/go/issues/10490)
- encoding/gob: decoding hangs (https://github.com/golang/go/issues/10491)
- image/jpeg: unreadByteStuffedByte call cannot be fulfilled (https://github.com/golang/go/issues/10387)
- image/jpeg: index out of range (https://github.com/golang/go/issues/10388)
- image/jpeg: invalid memory address or nil pointer dereference (https://github.com/golang/go/issues/10389)
- image/jpeg: Decode hangs (https://github.com/golang/go/issues/10413)
- image/jpeg: excessive memory usage (https://github.com/golang/go/issues/10532)
- image/png: slice bounds out of range (https://github.com/golang/go/issues/10414)
- image/png: interface conversion: color.Color is color.NRGBA, not color.RGBA (https://github.com/golang/go/issues/10423)
- image/png: nil deref (https://github.com/golang/go/issues/10493)
- compress/flate: hang (https://github.com/golang/go/issues/10426)
- x/image/webp: index out of range (https://github.com/golang/go/issues/10383)
- x/image/webp: invalid memory address or nil pointer dereference (https://github.com/golang/go/issues/10384)
- x/image/tiff: integer divide by zero (https://github.com/golang/go/issues/10393)
- x/image/tiff: index out of range (https://github.com/golang/go/issues/10394)
- x/image/tiff: slice bounds out of range (https://github.com/golang/go/issues/10395)
- x/image/bmp: makeslice: len out of range (https://github.com/golang/go/issues/10396)
- x/image/bmp: out of memory (https://github.com/golang/go/issues/10399)
- x/net/html: void element <link> has child nodes (https://github.com/golang/go/issues/10535)
- x/net/spdy: unexpected EOF (https://github.com/golang/go/issues/10539)
- x/net/spdy: EOF (https://github.com/golang/go/issues/10540)
- x/net/spdy: fatal error: runtime: out of memory (https://github.com/golang/go/issues/10542)
- x/net/spdy: stream id zero is disallowed (https://github.com/golang/go/issues/10543)
- x/net/spdy: processing of 35 bytes takes 7 seconds (https://github.com/golang/go/issues/10544)
- x/net/spdy: makemap: size out of range (https://github.com/golang/go/issues/10545)
- x/net/spdy: makeslice: len out of range (https://github.com/golang/go/issues/10547)