# go-fuzz: randomized testing for Go

Go-fuzz is a coverage-guided [fuzzing solution](http://en.wikipedia.org/wiki/Fuzz_testing) for testing of Go packages.
Fuzzing is mainly applicable to packages that parse complex inputs (both text
and binary), and is especially useful for hardening of systems that parse inputs
from potentially malicious users (e.g. anything accepted over a network).sn1

## Usage

First, you need to write a test function of the form:
```go
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
it will slow down fuzzing and nobody will see the output anyway. The exception
is printing info about a bug just before panicking.

Here is an example of a simple Fuzz function for image/png package:
```go
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
```go
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

Put the initial corpus into the workdir/corpus directory (in our case
```examples/png/corpus```). Go-fuzz will add own inputs to the corpus directory.
Consider committing the generated inputs to your source control system, this
will allow you to restart go-fuzz without losing previous work.

Examples directory contains a bunch of examples of test functions and initial
input corpuses for various packages.

The next step is to get go-fuzz:
```
$ go get github.com/dvyukov/go-fuzz/go-fuzz
$ go get github.com/dvyukov/go-fuzz/go-fuzz-build
```

Then, build the test program with necessary instrumentation:
```
$ go-fuzz-build github.com/dvyukov/go-fuzz/examples/png
```
This will produce png-fuzz.zip archive.

Now we are ready to go:
```
$ go-fuzz -bin=./png-fuzz.zip -workdir=examples/png
```

Go-fuzz will generate and test various inputs in an infinite loop. Workdir is
used to store persistent data like current corpus and crashers, it allows fuzzer
to continue after restart. Discovered bad inputs are stored in workdir/crashers
dir; where file without a suffix contains binary input, file with .quoted suffix
contains quoted input that can be directly copied into a reproducer program or a
test, file with .output suffix contains output of the test on this input. Every
few seconds go-fuzz prints logs of the form:
```
2015/04/25 12:39:53 slaves: 500, corpus: 186 (42s ago), crashers: 3,
     restarts: 1/8027, execs: 12009519 (121224/sec), cover: 0.31%, uptime: 1m39s
```
Where ```slaves``` means number of tests running in parallel (set with -procs
flag). ```corpus``` is current number of interesting inputs the fuzzer has
discovered, time in brackets says when the last interesting input was
discovered. ```crashers``` is number of discovered bugs (check out
workdir/crashers dir). ```restarts``` is the rate with which the fuzzer restarts
test processes. The rate should be close to 1/10000 (which is the planned
restart rate); if it is considerably higher than 1/10000, consider fixing already
discovered bugs which lead to frequent restarts. ```execs``` is total number of
test executions, and the number in brackets is the average speed of test
executions. ```cover``` is density of hashed coverage bitmap, ideally this value
should be smaller than 5%, otherwise fuzzer can miss new interesting inputs.
And finally ```uptime``` is uptime of the process.

### Random Notes

go-fuzz-build builds the program with gofuzz build tag, this allows to put the
Fuzz function implementation directly into the tested package, but exclude it
from normal builds with ```// +build gofuzz``` directive.

If your inputs contain a checksum, it can make sense to append/update the checksum
in the ```Fuzz``` function. The chances that go-fuzz will generate the correct
checksum are very low, so most work will be in vain otherwise.

Go-fuzz can utilize several machines. To do this, start master process separately:
```
$ go-fuzz -workdir=examples/png -master=127.0.0.1:8745
```
It will manage persistent corpus and crashers and coordinate work of slave processes.
Then run one or more slave processes as:
```
$ go-fuzz -bin=./png-fuzz.zip -slave=127.0.0.1:8745 -procs=10
```

## Credits and technical details

Go-fuzz fuzzing logic is heavily based on [american fuzzy lop](http://lcamtuf.coredump.cx/afl/),
so refer to [AFL readme](http://lcamtuf.coredump.cx/afl/README.txt) if you are
interested in technical details. AFL is written and maintained by
[Michal Zalewski](http://lcamtuf.coredump.cx/). Some of the mutations employed
by go-fuzz are inspired by work done by Mateusz Jurczyk, Gynvael Coldwind and
[Felix Gröbert](https://twitter.com/fel1x).

## Trophies

- [spec: non-integral constant can be converted to int](https://github.com/golang/go/issues/11350) **fixed**
- [cmd/compile: out of fixed registers](https://github.com/golang/go/issues/11352)
- [cmd/compile: truncates constants](https://github.com/golang/go/issues/11326)
- [cmd/compile: overflow in int -> string](https://github.com/golang/go/issues/11330)
- [cmd/compile: bad HMUL](https://github.com/golang/go/issues/11358) **fixed**
- [cmd/compile: treecopy Name](https://github.com/golang/go/issues/11361)
- [cmd/compile: accepts invalid identifiers](https://github.com/golang/go/issues/11359)
- [cmd/compile: hangs compiling hex fp constant](https://github.com/golang/go/issues/11364)
- [cmd/compile: mishandles int->complex conversion](https://github.com/golang/go/issues/11365)
- [cmd/compile: allows to define blank methods on builtin types](https://github.com/golang/go/issues/11366)
- [cmd/compile: mis-calculates a constant](https://github.com/golang/go/issues/11369) **fixed**
- [cmd/compile: interface conversion panic](https://github.com/golang/go/issues/11540)
- [cmd/compile: nil pointer dereference](https://github.com/golang/go/issues/11588)
- [fmt: Printf loops on invalid verb spec](https://github.com/golang/go/issues/10674) **fixed**
- [fmt: incorrect overflow detection](https://github.com/golang/go/issues/10695) **fixed**
- [fmt: index out of range](https://github.com/golang/go/issues/10675) **fixed**
- [fmt: index out of range (2)](https://github.com/golang/go/issues/10745) **fixed**
- [fmt: index out of range (3)](https://github.com/golang/go/issues/10770) **fixed**
- [fmt: index out of range (4)](https://github.com/golang/go/issues/10771) **fixed**
- [fmt: index out of range (5)](https://github.com/golang/go/issues/10945) **fixed**
- [fmt: index out of range (6)](https://github.com/golang/go/issues/11376) **fixed**
- [regexp: slice bounds out of range](https://github.com/golang/go/issues/11176)
- [regexp: slice bounds out of range (2)](https://github.com/golang/go/issues/11178)
- [regexp: LiteralPrefix lies about completeness](https://github.com/golang/go/issues/11172)
- [regexp: LiteralPrefix lies about completeness (2)](https://github.com/golang/go/issues/11175)
- [regexp: POSIX regexp takes 4 seconds to execute](https://github.com/golang/go/issues/11181)
- [regexp: confusing behavior on invalid utf-8 sequences](https://github.com/golang/go/issues/11185)
- [time: allows signs for year/tz in format string](https://github.com/golang/go/issues/11128)
- [math/big: incorrect string->Float conversion](https://github.com/golang/go/issues/11341)
- [net/http: can't send star request](https://github.com/golang/go/issues/11202) **fixed**
- [net/http: allows empty header names](https://github.com/golang/go/issues/11205) **fixed**
- [net/http: allows invalid characters in header values](https://github.com/golang/go/issues/11207)
- [net/http: allows %-encoding after \[\]](https://github.com/golang/go/issues/11208) **fixed**
- [net/mail: ParseAddress/String corrupt address](https://github.com/golang/go/issues/11292)
- [net/mail: parses invalid address](https://github.com/golang/go/issues/11293)
- [net/mail: fails to escape address](https://github.com/golang/go/issues/11294)
- [net/textproto: fails to trim header value](https://github.com/golang/go/issues/11204)
- [archive/zip: cap out of range](https://github.com/golang/go/issues/10956) **fixed**
- [archive/zip: bad file size](https://github.com/golang/go/issues/10957) **fixed**
- [archive/zip: unexpected EOF](https://github.com/golang/go/issues/11144) **fixed**
- [archive/zip: file with wrong checksum is successfully decompressed](https://github.com/golang/go/issues/11146) **fixed**
- [archive/tar: slice bounds out of range](https://github.com/golang/go/issues/10959) **fixed**
- [archive/tar: slice bounds out of range (2)](https://github.com/golang/go/issues/10960) **fixed**
- [archive/tar: slice bounds out of range (3)](https://github.com/golang/go/issues/10966) **fixed**
- [archive/tar: slice bounds out of range (4)](https://github.com/golang/go/issues/10967) **fixed**
- [archive/tar: slice bounds out of range (5)](https://github.com/golang/go/issues/11167) **fixed**
- [archive/tar: deadly hang](https://github.com/golang/go/issues/10968) **fixed**
- [archive/tar: invalid memory address or nil pointer dereference](https://github.com/golang/go/issues/11168)
- [archive/tar: Reader.Next returns nil header](https://github.com/golang/go/issues/11169) **fixed**
- [archive/tar: eats file data](https://github.com/golang/go/issues/11170)
- [archive/tar: Writer incorrectly encodes header data](https://github.com/golang/go/issues/11171)
- [encoding/gob: panic: drop](https://github.com/golang/go/issues/10272) **fixed**
- [encoding/gob: makeslice: len out of range](https://github.com/golang/go/issues/10273) [3 bugs] **fixed**
- [encoding/gob: stack overflow](https://github.com/golang/go/issues/10415) **fixed**
- [encoding/gob: excessive memory consumption](https://github.com/golang/go/issues/10490) **fixed**
- [encoding/gob: decoding hangs](https://github.com/golang/go/issues/10491) **fixed**
- [encoding/gob: pointers to zero values are not initialized in Decode](https://github.com/golang/go/issues/11119)
- [encoding/xml: allows invalid comments](https://github.com/golang/go/issues/11112)
- [encoding/json: detect circular data structures when encoding](https://github.com/golang/go/issues/10769)
- [encoding/asn1: index out of range](https://github.com/golang/go/issues/11129) **fixed**
- [encoding/asn1: incorrectly handles incorrect utf8 strings](https://github.com/golang/go/issues/11126) **fixed**
- [encoding/asn1: slice is lost during marshal/unmarshal](https://github.com/golang/go/issues/11130)
- [encoding/asn1: call of reflect.Value.Type on zero Value](https://github.com/golang/go/issues/11127)
- [encoding/asn1: Unmarshal accepts negative dates](https://github.com/golang/go/issues/11134) **fixed**
- [encoding/pem: can't decode encoded message](https://github.com/golang/go/issues/10980) **fixed**
- [crypto:x509: input not full blocks](https://github.com/golang/go/issues/11215) **fixed**
- [crypto/x509: division by zero](https://github.com/golang/go/issues/11233) **fixed**
- [image/jpeg: unreadByteStuffedByte call cannot be fulfilled](https://github.com/golang/go/issues/10387) **fixed**
- [image/jpeg: index out of range](https://github.com/golang/go/issues/10388) **fixed**
- [image/jpeg: invalid memory address or nil pointer dereference](https://github.com/golang/go/issues/10389) **fixed**
- [image/jpeg: Decode hangs](https://github.com/golang/go/issues/10413) **fixed**
- [image/jpeg: excessive memory usage](https://github.com/golang/go/issues/10532) **fixed**
- [image/png: slice bounds out of range](https://github.com/golang/go/issues/10414) **fixed**
- [image/png: interface conversion: color.Color is color.NRGBA, not color.RGBA](https://github.com/golang/go/issues/10423) **fixed**
- [image/png: nil deref](https://github.com/golang/go/issues/10493) **fixed**
- [image/gif: image block is out of bounds](https://github.com/golang/go/issues/10676) **fixed**
- [image/gif: Decode returns an image with empty palette](https://github.com/golang/go/issues/11150) **fixed**
- [image/gif: LoopCount changes on round trip](https://github.com/golang/go/issues/11287) **fixed**
- [image/gif: Disposal is corrupted after round trip](https://github.com/golang/go/issues/11288)
- [image/gif: EOF instead of UnexpectedEOF](https://github.com/golang/go/issues/11390)
- [compress/flate: hang](https://github.com/golang/go/issues/10426) **fixed**
- [compress/lzw: compress/decompress corrupts data](https://github.com/golang/go/issues/11142) **fixed**
- [text/template: leaks goroutines on errors](https://github.com/golang/go/issues/10574#ref-issue-71873016)
- [text/template: Call using string as type int](https://github.com/golang/go/issues/10800) **fixed**
- [text/template: Call using complex128 as type string](https://github.com/golang/go/issues/10946) **fixed**
- [html/template: unidentified node type in allIdents](https://github.com/golang/go/issues/10610) **fixed**
- [html/template: unidentified node type in allIdents (2)](https://github.com/golang/go/issues/10801) **fixed**
- [html/template: unidentified node type in allIdents (3)](https://github.com/golang/go/issues/11118) **fixed**
- [html/template: unidentified node type in allIdents (4)](https://github.com/golang/go/issues/11356) **fixed**
- [html/template: escaping {{else}} is unimplemented](https://github.com/golang/go/issues/10611) **fixed**
- [html/template: runtime error: slice bounds out of range](https://github.com/golang/go/issues/10612) **fixed**
- [html/template: runtime error: slice bounds out of range (2)](https://github.com/golang/go/issues/10613) **fixed**
- [html/template: invalid memory address or nil pointer dereference](https://github.com/golang/go/issues/10615) **fixed**
- [html/template: panic: Call using zero Value argument](https://github.com/golang/go/issues/10634) **fixed**
- [html/template: nil pointer dereference](https://github.com/golang/go/issues/10673) **fixed**
- [html/template: slice bounds out of range](https://github.com/golang/go/issues/10799) **fixed**
- [mime: ParseMediaType parses invalid media types](https://github.com/golang/go/issues/11289)
- [mime: Parse/Format corrupt parameters](https://github.com/golang/go/issues/11290)
- [mime: Parse/Format corrupt parameters (2)](https://github.com/golang/go/issues/11291)
- [go/parser: eats \r in comments](https://github.com/golang/go/issues/11151)
- [go/format: turns correct program into incorrect one](https://github.com/golang/go/issues/11274)
- [go/format: non-idempotent format](https://github.com/golang/go/issues/11275)
- [go/format: adds }](https://github.com/golang/go/issues/11276) **fixed**
- [debug/elf: index out of range](https://github.com/golang/go/issues/10996)
- [debug/elf: makeslice: len out of range](https://github.com/golang/go/issues/10997)
- [debug/elf: slice bounds out of range](https://github.com/golang/go/issues/10999)
- [x/image/webp: index out of range](https://github.com/golang/go/issues/10383) **fixed**
- [x/image/webp: invalid memory address or nil pointer dereference](https://github.com/golang/go/issues/10384) **fixed**
- [x/image/webp: excessive memory consumption](https://github.com/golang/go/issues/10790)
- [x/image/webp: excessive memory consumption (2)](https://github.com/golang/go/issues/11395)
- [x/image/tiff: integer divide by zero](https://github.com/golang/go/issues/10393) **fixed**
- [x/image/tiff: index out of range](https://github.com/golang/go/issues/10394) **fixed**
- [x/image/tiff: slice bounds out of range](https://github.com/golang/go/issues/10395) **fixed**
- [x/image/tiff: index out of range](https://github.com/golang/go/issues/10597) **fixed**
- [x/image/tiff: slice bounds out of range](https://github.com/golang/go/issues/10596) **fixed**
- [x/image/tiff: integer divide by zero](https://github.com/golang/go/issues/10711) **fixed**
- [x/image/tiff: index out of range](https://github.com/golang/go/issues/10712) **fixed**
- [x/image/tiff: index out of range](https://github.com/golang/go/issues/11386)
- [x/image/tiff: excessive memory consumption](https://github.com/golang/go/issues/11389)
- [x/image/{tiff,bmp}: EOF instead of UnexpectedEOF](https://github.com/golang/go/issues/11391)
- [x/image/bmp: hang on degenerate image](https://github.com/golang/go/issues/10746) **fixed**
- [x/image/bmp: makeslice: len out of range](https://github.com/golang/go/issues/10396) **fixed**
- [x/image/bmp: out of memory](https://github.com/golang/go/issues/10399) **fixed**
- [x/net/icmp: runtime error: slice bounds out of range](https://github.com/golang/go/issues/10951)
- [x/net/html: void element <link> has child nodes](https://github.com/golang/go/issues/10535)
- [x/net/spdy: unexpected EOF](https://github.com/golang/go/issues/10539) **fixed**
- [x/net/spdy: EOF](https://github.com/golang/go/issues/10540) **fixed**
- [x/net/spdy: fatal error: runtime: out of memory](https://github.com/golang/go/issues/10542) **fixed**
- [x/net/spdy: stream id zero is disallowed](https://github.com/golang/go/issues/10543) **fixed**
- [x/net/spdy: processing of 35 bytes takes 7 seconds](https://github.com/golang/go/issues/10544) **fixed**
- [x/net/spdy: makemap: size out of range](https://github.com/golang/go/issues/10545) **fixed**
- [x/net/spdy: makeslice: len out of range](https://github.com/golang/go/issues/10547) **fixed**
- [x/crypto/ssh: Server panic on invalid input](https://github.com/golang/go/issues/11348) **fixed**
- [x/crypto/openpgp: ReadMessage(): Panic on invalid input in packet.nextSubpacket](https://github.com/golang/go/issues/11503)
- [x/crypto/openpgp: ReadMessage(): Panic on invalid input in packet.PublicKeyV3.setFingerPrintAndKeyId](https://github.com/golang/go/issues/11504)
- [x/crypto/openpgp: ReadMessage(): Panic on invalid input in math/big.nat.div](https://github.com/golang/go/issues/11505)
- [x/tools/go/types: panics on invalid constant](https://github.com/golang/go/issues/11325) **fixed**
- [x/tools/go/types: compiling hangs](https://github.com/golang/go/issues/11327)
- [x/tools/go/types: stupid shift](https://github.com/golang/go/issues/11328)
- [x/tools/go/types: line number out of range](https://github.com/golang/go/issues/11329)
- [x/tools/go/types: assertion failed](https://github.com/golang/go/issues/11347)
- [x/tools/go/types: converts fp constant to string](https://github.com/golang/go/issues/11353) **fixed**
- [x/tools/go/types: converts complex constant to string](https://github.com/golang/go/issues/11357) **fixed**
- [x/tools/go/types: misses '-' in error message](https://github.com/golang/go/issues/11367) **fixed**
- [x/tools/go/types: compiles invalid program with overflow](https://github.com/golang/go/issues/11368)
- [x/tools/go/types: allows duplicate switch cases](https://github.com/golang/go/issues/11578)
- [gccgo: bogus index out of bounds](https://github.com/golang/go/issues/11522)
- [gccgo: does not see stupidness of shift count](https://github.com/golang/go/issues/11524)
- [gccgo: bogus integer constant overflow](https://github.com/golang/go/issues/11525)
- [gccgo: segmentation fault](https://github.com/golang/go/issues/11526)
- [gccgo: segmentation fault (2)](https://github.com/golang/go/issues/11536)
- [gccgo: segmentation fault (3)](https://github.com/golang/go/issues/11558)
- [gccgo: segmentation fault (4)](https://github.com/golang/go/issues/11559)
- [gccgo: internal compiler error in set_type](https://github.com/golang/go/issues/11537)
- [gccgo: internal compiler error in global_variable_set_init](https://github.com/golang/go/issues/11541)
- [gccgo: internal compiler error: in wide_int_to_tree](https://github.com/golang/go/issues/11542)
- [gccgo: internal compiler error in record_var_depends_on](https://github.com/golang/go/issues/11543)
- [gccgo: internal compiler error in Builtin_call_expression](https://github.com/golang/go/issues/11544)
- [gccgo: internal compiler error in check_bounds](https://github.com/golang/go/issues/11545)
- [gccgo: internal compiler error in do_determine_type](https://github.com/golang/go/issues/11546)
- [gccgo: internal compiler error in backend_numeric_constant_expression](https://github.com/golang/go/issues/11548)
- [gccgo: internal compiler error in type_size](https://github.com/golang/go/issues/11554)
- [gccgo: internal compiler error in type_size (2)](https://github.com/golang/go/issues/11555)
- [gccgo: internal compiler error in type_size (3)](https://github.com/golang/go/issues/11556)
- [gccgo: internal compiler error in do_get_backend](https://github.com/golang/go/issues/11560)
- [gccgo: internal compiler error in create_tmp_var](https://github.com/golang/go/issues/11568)
- [gccgo: internal compiler error in methods](https://github.com/golang/go/issues/11579)
- [gccgo: accepts invalid UTF-8](https://github.com/golang/go/issues/11527)
- [gccgo: spurious expected newline error](https://github.com/golang/go/issues/11528)
- [gccgo: can apply ^ to true](https://github.com/golang/go/issues/11529)
- [gccgo: hangs](https://github.com/golang/go/issues/11530)
- [gccgo: hangs (2)](https://github.com/golang/go/issues/11531)
- [gccgo: hangs (3)](https://github.com/golang/go/issues/11539)
- [gccgo: rejects valid imaginary literal](https://github.com/golang/go/issues/11532)
- [gccgo: rejects valid fp literal](https://github.com/golang/go/issues/11533)
- [gccgo: accepts program with invalid identifier](https://github.com/golang/go/issues/11535)
- [gccgo: accepts program with invalid identifier (2)](https://github.com/golang/go/issues/11547)
- [gccgo: compiles weird construct](https://github.com/golang/go/issues/11561)
- [gccgo: can do bitwise or on fp constants](https://github.com/golang/go/issues/11566)
- [gccgo: treats nil as type](https://github.com/golang/go/issues/11567)
- [gccgo: does not understand greek capiltal letter yot](https://github.com/golang/go/issues/11569)
- [gccgo: allows to refer to builtin function not in call expression](https://github.com/golang/go/issues/11570)
- [gccgo: bogus incompatible types in binary expression error](https://github.com/golang/go/issues/11572)
- [gccgo: allows multiple definitions of a function](https://github.com/golang/go/issues/11573)
- [gccgo: can shift by complex number](https://github.com/golang/go/issues/11574)
- [gccgo: knowns unknown escape sequence](https://github.com/golang/go/issues/11575)
- [gccgo: internal compiler error in start_function](https://github.com/golang/go/issues/11576)
- [gccgo: heap-buffer-overflow in Lex::skip_cpp_comment](https://github.com/golang/go/issues/11577)
- [gccgo: does not convert untyped complex 0i to int in binary operation involving an int](https://github.com/golang/go/issues/11563)
- [github.com/golang/protobuf: call of reflect.Value.SetMapIndex on zero Value](https://github.com/golang/protobuf/issues/27) **fixed**
- [github.com/golang/protobuf: call of reflect.Value.Interface on zero Value in MarshalText](https://github.com/golang/protobuf/issues/33) **fixed**
- [github.com/golang/protobuf: Invalid map is successfully decoded](https://github.com/golang/protobuf/issues/34)
- [github.com/golang/protobuf: MarshalText incorrectly handles unknown bytes](https://github.com/golang/protobuf/issues/35)
- [github.com/golang/protobuf: MarshalText fails and prints to stderr](https://github.com/golang/protobuf/issues/36)
- [code.google.com/p/freetype-go: 42 crashers](https://code.google.com/p/freetype-go/issues/detail?id=17) [42 bugs]
- [github.com/cryptix/wav: 2 panics in header decoding](https://github.com/cryptix/wav/commit/2f49a0df0d213ee323f694e7bdee8b8a097dc698#diff-f86b763600291cbceee077a33133434a) **fixed**
- [github.com/spf13/hugo: 7 crashers](https://github.com/spf13/hugo/search?q=go-fuzz&type=Issues) **7 fixed**
- [github.com/Sereal/Sereal: 8 crashers](https://github.com/Sereal/Sereal/commit/c254cc3f2c48caffee6cd04ea8100a0150357a44) **fixed**
- [github.com/bradfitz/http2: Server.handleConn hangs](https://github.com/bradfitz/http2/issues/53)
- [github.com/bradfitz/http2: nil pointer dereference in hpack.HuffmanDecode](https://github.com/bradfitz/http2/issues/56)
- [github.com/bradfitz/http2: serverConn.readFrames goroutine leak](https://github.com/bradfitz/http2/issues/58)
- [github.com/golang/snappy: index out of range panic](https://github.com/golang/snappy/issues/11)
- [github.com/bkaradzic/go-lz4: slice bounds out of range](https://github.com/bkaradzic/go-lz4/commit/b8d4dc7b31511bf5f39dfdb02d2ea7662eb8407c) **fixed**
- [github.com/gocql/gocql: slice bounds out of range](https://github.com/gocql/gocql/commit/332853ab7b3c719dd67c657394139491c1f6deb7) **fixed**
- [github.com/gocql/gocql: slice bounds out of range](https://github.com/gocql/gocql/commit/58d90fab97daa2d9edd6e7a1b2a22bee8ce12c72) **fixed**
- [github.com/tdewolff/minify: 8 crashers](https://github.com/tdewolff/minify/wiki) **fixed**
- [github.com/russross/blackfriday: index out of range panic in scanLinkRef](https://github.com/russross/blackfriday/issues/172) **fixed**
- [github.com/russross/blackfriday: index out of range panic in isReference](https://github.com/russross/blackfriday/issues/173) **fixed**
- [github.com/youtube/vitess/go/vt/sqlparser: index out of range](https://github.com/youtube/vitess/issues/767) **fixed**
- [github.com/youtube/vitess/go/vt/sqlparser: statement serialized incorrectly](https://github.com/youtube/vitess/issues/797)
- [github.com/youtube/vitess/go/vt/sqlparser: statement serialized incorrectly (2)](https://github.com/youtube/vitess/issues/798)
- [gopkg.in/mgo.v2/bson: slice bounds out of range](https://github.com/go-mgo/mgo/issues/116) **fixed**
- [gopkg.in/mgo.v2/bson: Document is corrupted](https://github.com/go-mgo/mgo/issues/117) **fixed**
- [gopkg.in/mgo.v2/bson: Attempted to marshal empty Raw document](https://github.com/go-mgo/mgo/issues/120) **fixed**

**If you find some bugs with go-fuzz and are comfortable with sharing them, I would like to add them to this list.** Please either send a pull request for README.md (preferable) or file an issue. If the source code is closed, you can say just "found N bugs in project X". Thank you.
