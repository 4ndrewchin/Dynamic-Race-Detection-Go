# Understanding and Evaluating Dynamic Race Detection with Go

Andrew Chin
```
TODO:
- add introduction material (merge ppt)
- finish optimization discussion
- finish RTL discussion
- change bullet points into text
```

## A Deep Dive Into the Go Compiler Codebase

#### Prerequisites

In order to follow along, the reader should have the following source code:
 1. LLVM source code (we use 9.0.0) but any recent version should be ok (https://releases.llvm.org/download.html)
 2. Compiler-RT source code (a subproject in the LLVM family, also 9.0.0). This download should be placed into `llvm/projects/`, where llvm is the source code downloaded in step 1.
 3. Go source code (cloned from master on Jan 3, 2020, but any recent version should be ok) (https://github.com/golang/go)

 In the rest of this document, we will refer to relative file paths within `llvm` and `go`. These refer to the directories you have just downloaded.

---

#### Go Compiler

The Go Compiler's job is to take a group of source code files written in the Go programming language and transform them into executable binaries. It has four high-level phases.

1. Parsing

2. Type-Checking and AST Transformations

3. Generic SSA

4. Lower SSA and Machine Code

Compiler Instrumentation occurs during phases 3 and 4 (During SSA construction and as a separate SSA pass).

The heart of the compiler is located at `go/src/cmd/compile/internal/gc/main.go` (EVERYTHING BRANCHES FROM THIS FILE -- to fully understand the compiler read through this file, although only the latter half is relevant for this research)

---

#### TSAN (ThreadSanitizer)

OUT OF SCOPE FOR THESIS?
- Should understand current approaches to race detection using static and dynamic analysis
- Concepts: shadow memory, happens-before, vector clocks

Refs:

- Main source of information: https://github.com/google/sanitizers/wiki
- Original Paper (2009): https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35604.pdf
- TSAN V2 Paper (2011): https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/37278.pdf 
- Source code (TSAN is a tool under the LLVM Compiler Run Time project): https://compiler-rt.llvm.org/ , download at https://releases.llvm.org/download.html 

Some Direct Wiki Links:
- TSAN Overview: https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual
- Races: https://github.com/google/sanitizers/wiki/ThreadSanitizerAboutRaces
- Core Algorithm: https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm
- Go Manual: https://github.com/google/sanitizers/wiki/ThreadSanitizerGoManual

---

#### TSAN in Go

How is the race detection library implemented in Go?

- It's not! The runtime state machine and core interface is compiled from the original LLVM project and linked to the Go runtime as an external library. 

What is implemented in the Go runtime then?

- The Go compiler still needs to instrument emitted code for use by the LLVM race detection runtime. This includes standard instrumentation of reads and writes and inserting race functions into new Go-specific synchronization primitives such as channels.

---

#### Emitting Instrumented ASM for Race Detection

- Clang: `clang -S -masm=intel -fsanitize=thread <source>.go` outputs `<source>.s`

- Go Compiler: `go tool compile -S -race <source>.go > <source>.s` outputs `<source>.s`

   - to view SSA, prepend above with `GOSSAFUNC=<func_name>`. This will generate `ssa.html`, which provides an interactive look at each phase in compilation from source code to AST to SSA after each optimization pass.

---

#### Source Code: LLVM

- The source code for race detection in LLVM is well compartmentalized.

- LLVM instrumentation: `llvm/lib/Transforms/Instrumentation`

   - Instrumentation for TSAN is all in `ThreadSanitizer.cpp`

- Runtime Library: `llvm/projects/compiler-rt/lib/tsan/`

- Some instrumentation optimizations:
```
   1. Redundant accesses currently handled:
      - read-before-write (within same BB, no calls between)
      - not captured variables
   2. Patterns that should not survive after classic compiler optimizations are not handled:
      - two reads from the same temp should be eliminated by CSE (common subexpression elimination)
      - two writes should be eliminated by DSE (dead store elimination)
      - etc. (out-of-scope for this project)

see llvm/lib/Transforms/Instrumentation/ThreadSanitizer.cpp:chooseInstructionsToInstrument()
```

---

#### Source Code: Go

- Go package `race` for manually instrumenting code for the race detector: `go/src/internal/race/race.go` (also `norace.go` for when race detection is not enabled)
	```go
	func Acquire(addr unsafe.Pointer) {
		runtime.RaceAcquire(addr)
	}

	func Release(addr unsafe.Pointer) {
		runtime.RaceRelease(addr)
	}

	func ReleaseMerge(addr unsafe.Pointer) {
		runtime.RaceReleaseMerge(addr)
	}

	func Disable() {
		runtime.RaceDisable()
	}

	func Enable() {
		runtime.RaceEnable()
	}

	func Read(addr unsafe.Pointer) {
		runtime.RaceRead(addr)
	}

	func Write(addr unsafe.Pointer) {
		runtime.RaceWrite(addr)
	}

	func ReadRange(addr unsafe.Pointer, len int) {
		runtime.RaceReadRange(addr, len)
	}

	func WriteRange(addr unsafe.Pointer, len int) {
		runtime.RaceWriteRange(addr, len)
	}
	```

- race instrumentation API used by the go compiler: `go/src/runtime/race.go` (also `race0.go` when race detection is not enabled)
	<details>
		<summary>see code</summary>

		```go
		// Public race detection API, present iff build with -race.

		func RaceRead(addr unsafe.Pointer)
		func RaceWrite(addr unsafe.Pointer)
		func RaceReadRange(addr unsafe.Pointer, len int)
		func RaceWriteRange(addr unsafe.Pointer, len int)

		func RaceErrors() int {
			var n uint64
			racecall(&__tsan_report_count, uintptr(unsafe.Pointer(&n)), 0, 0, 0)
			return int(n)
		}

		//go:nosplit

		// RaceAcquire/RaceRelease/RaceReleaseMerge establish happens-before relations
		// between goroutines. These inform the race detector about actual synchronization
		// that it can't see for some reason (e.g. synchronization within RaceDisable/RaceEnable
		// sections of code).
		// RaceAcquire establishes a happens-before relation with the preceding
		// RaceReleaseMerge on addr up to and including the last RaceRelease on addr.
		// In terms of the C memory model (C11 §5.1.2.4, §7.17.3),
		// RaceAcquire is equivalent to atomic_load(memory_order_acquire).
		func RaceAcquire(addr unsafe.Pointer) {
			raceacquire(addr)
		}

		//go:nosplit

		// RaceRelease performs a release operation on addr that
		// can synchronize with a later RaceAcquire on addr.
		//
		// In terms of the C memory model, RaceRelease is equivalent to
		// atomic_store(memory_order_release).
		func RaceRelease(addr unsafe.Pointer) {
			racerelease(addr)
		}

		//go:nosplit

		// RaceReleaseMerge is like RaceRelease, but also establishes a happens-before
		// relation with the preceding RaceRelease or RaceReleaseMerge on addr.
		//
		// In terms of the C memory model, RaceReleaseMerge is equivalent to
		// atomic_exchange(memory_order_release).
		func RaceReleaseMerge(addr unsafe.Pointer) {
			racereleasemerge(addr)
		}

		//go:nosplit

		// RaceDisable disables handling of race synchronization events in the current goroutine.
		// Handling is re-enabled with RaceEnable. RaceDisable/RaceEnable can be nested.
		// Non-synchronization events (memory accesses, function entry/exit) still affect
		// the race detector.
		func RaceDisable() {
			_g_ := getg()
			if _g_.raceignore == 0 {
				racecall(&__tsan_go_ignore_sync_begin, _g_.racectx, 0, 0, 0)
			}
			_g_.raceignore++
		}

		//go:nosplit

		// RaceEnable re-enables handling of race events in the current goroutine.
		func RaceEnable() {
			_g_ := getg()
			_g_.raceignore--
			if _g_.raceignore == 0 {
				racecall(&__tsan_go_ignore_sync_end, _g_.racectx, 0, 0, 0)
			}
		}

		// Private interface for the runtime.

		const raceenabled = true

		// For all functions accepting callerpc and pc,
		// callerpc is a return PC of the function that calls this function,
		// pc is start PC of the function that calls this function.
		func raceReadObjectPC(t *_type, addr unsafe.Pointer, callerpc, pc uintptr) {
			kind := t.kind & kindMask
			if kind == kindArray || kind == kindStruct {
				// for composite objects we have to read every address
				// because a write might happen to any subobject.
				racereadrangepc(addr, t.size, callerpc, pc)
			} else {
				// for non-composite objects we can read just the start
				// address, as any write must write the first byte.
				racereadpc(addr, callerpc, pc)
			}
		}

		func raceWriteObjectPC(t *_type, addr unsafe.Pointer, callerpc, pc uintptr) {
			kind := t.kind & kindMask
			if kind == kindArray || kind == kindStruct {
				// for composite objects we have to write every address
				// because a write might happen to any subobject.
				racewriterangepc(addr, t.size, callerpc, pc)
			} else {
				// for non-composite objects we can write just the start
				// address, as any write must write the first byte.
				racewritepc(addr, callerpc, pc)
			}
		}

		//go:noescape
		func racereadpc(addr unsafe.Pointer, callpc, pc uintptr)

		//go:noescape
		func racewritepc(addr unsafe.Pointer, callpc, pc uintptr)

		type symbolizeCodeContext struct {
			pc   uintptr
			fn   *byte
			file *byte
			line uintptr
			off  uintptr
			res  uintptr
		}

		var qq = [...]byte{'?', '?', 0}
		var dash = [...]byte{'-', 0}

		const (
			raceGetProcCmd = iota
			raceSymbolizeCodeCmd
			raceSymbolizeDataCmd
		)

		// Callback from C into Go, runs on g0.
		func racecallback(cmd uintptr, ctx unsafe.Pointer) {
			switch cmd {
			case raceGetProcCmd:
				throw("should have been handled by racecallbackthunk")
			case raceSymbolizeCodeCmd:
				raceSymbolizeCode((*symbolizeCodeContext)(ctx))
			case raceSymbolizeDataCmd:
				raceSymbolizeData((*symbolizeDataContext)(ctx))
			default:
				throw("unknown command")
			}
		}

		// raceSymbolizeCode reads ctx.pc and populates the rest of *ctx with
		// information about the code at that pc.
		//
		// The race detector has already subtracted 1 from pcs, so they point to the last
		// byte of call instructions (including calls to runtime.racewrite and friends).
		//
		// If the incoming pc is part of an inlined function, *ctx is populated
		// with information about the inlined function, and on return ctx.pc is set
		// to a pc in the logically containing function. (The race detector should call this
		// function again with that pc.)
		//
		// If the incoming pc is not part of an inlined function, the return pc is unchanged.
		func raceSymbolizeCode(ctx *symbolizeCodeContext) {
			pc := ctx.pc
			fi := findfunc(pc)
			f := fi._Func()
			if f != nil {
				file, line := f.FileLine(pc)
				if line != 0 {
					if inldata := funcdata(fi, _FUNCDATA_InlTree); inldata != nil {
						inltree := (*[1 << 20]inlinedCall)(inldata)
						for {
							ix := pcdatavalue(fi, _PCDATA_InlTreeIndex, pc, nil)
							if ix >= 0 {
								if inltree[ix].funcID == funcID_wrapper {
									// ignore wrappers
									// Back up to an instruction in the "caller".
									pc = f.Entry() + uintptr(inltree[ix].parentPc)
									continue
								}
								ctx.pc = f.Entry() + uintptr(inltree[ix].parentPc) // "caller" pc
								ctx.fn = cfuncnameFromNameoff(fi, inltree[ix].func_)
								ctx.line = uintptr(line)
								ctx.file = &bytes(file)[0] // assume NUL-terminated
								ctx.off = pc - f.Entry()
								ctx.res = 1
								return
							}
							break
						}
					}
					ctx.fn = cfuncname(fi)
					ctx.line = uintptr(line)
					ctx.file = &bytes(file)[0] // assume NUL-terminated
					ctx.off = pc - f.Entry()
					ctx.res = 1
					return
				}
			}
			ctx.fn = &qq[0]
			ctx.file = &dash[0]
			ctx.line = 0
			ctx.off = ctx.pc
			ctx.res = 1
		}

		type symbolizeDataContext struct {
			addr  uintptr
			heap  uintptr
			start uintptr
			size  uintptr
			name  *byte
			file  *byte
			line  uintptr
			res   uintptr
		}

		func raceSymbolizeData(ctx *symbolizeDataContext) {
			if base, span, _ := findObject(ctx.addr, 0, 0); base != 0 {
				ctx.heap = 1
				ctx.start = base
				ctx.size = span.elemsize
				ctx.res = 1
			}
		}

		// Race runtime functions called via runtime·racecall.
		//go:linkname __tsan_init __tsan_init
		var __tsan_init byte

		//go:linkname __tsan_fini __tsan_fini
		var __tsan_fini byte

		//go:linkname __tsan_proc_create __tsan_proc_create
		var __tsan_proc_create byte

		//go:linkname __tsan_proc_destroy __tsan_proc_destroy
		var __tsan_proc_destroy byte

		//go:linkname __tsan_map_shadow __tsan_map_shadow
		var __tsan_map_shadow byte

		//go:linkname __tsan_finalizer_goroutine __tsan_finalizer_goroutine
		var __tsan_finalizer_goroutine byte

		//go:linkname __tsan_go_start __tsan_go_start
		var __tsan_go_start byte

		//go:linkname __tsan_go_end __tsan_go_end
		var __tsan_go_end byte

		//go:linkname __tsan_malloc __tsan_malloc
		var __tsan_malloc byte

		//go:linkname __tsan_free __tsan_free
		var __tsan_free byte

		//go:linkname __tsan_acquire __tsan_acquire
		var __tsan_acquire byte

		//go:linkname __tsan_release __tsan_release
		var __tsan_release byte

		//go:linkname __tsan_release_merge __tsan_release_merge
		var __tsan_release_merge byte

		//go:linkname __tsan_go_ignore_sync_begin __tsan_go_ignore_sync_begin
		var __tsan_go_ignore_sync_begin byte

		//go:linkname __tsan_go_ignore_sync_end __tsan_go_ignore_sync_end
		var __tsan_go_ignore_sync_end byte

		//go:linkname __tsan_report_count __tsan_report_count
		var __tsan_report_count byte

		// Mimic what cmd/cgo would do.
		//go:cgo_import_static __tsan_init
		//go:cgo_import_static __tsan_fini
		//go:cgo_import_static __tsan_proc_create
		//go:cgo_import_static __tsan_proc_destroy
		//go:cgo_import_static __tsan_map_shadow
		//go:cgo_import_static __tsan_finalizer_goroutine
		//go:cgo_import_static __tsan_go_start
		//go:cgo_import_static __tsan_go_end
		//go:cgo_import_static __tsan_malloc
		//go:cgo_import_static __tsan_free
		//go:cgo_import_static __tsan_acquire
		//go:cgo_import_static __tsan_release
		//go:cgo_import_static __tsan_release_merge
		//go:cgo_import_static __tsan_go_ignore_sync_begin
		//go:cgo_import_static __tsan_go_ignore_sync_end
		//go:cgo_import_static __tsan_report_count

		// These are called from race_amd64.s.
		//go:cgo_import_static __tsan_read
		//go:cgo_import_static __tsan_read_pc
		//go:cgo_import_static __tsan_read_range
		//go:cgo_import_static __tsan_write
		//go:cgo_import_static __tsan_write_pc
		//go:cgo_import_static __tsan_write_range
		//go:cgo_import_static __tsan_func_enter
		//go:cgo_import_static __tsan_func_exit

		//go:cgo_import_static __tsan_go_atomic32_load
		//go:cgo_import_static __tsan_go_atomic64_load
		//go:cgo_import_static __tsan_go_atomic32_store
		//go:cgo_import_static __tsan_go_atomic64_store
		//go:cgo_import_static __tsan_go_atomic32_exchange
		//go:cgo_import_static __tsan_go_atomic64_exchange
		//go:cgo_import_static __tsan_go_atomic32_fetch_add
		//go:cgo_import_static __tsan_go_atomic64_fetch_add
		//go:cgo_import_static __tsan_go_atomic32_compare_exchange
		//go:cgo_import_static __tsan_go_atomic64_compare_exchange

		// start/end of global data (data+bss).
		var racedatastart uintptr
		var racedataend uintptr

		// start/end of heap for race_amd64.s
		var racearenastart uintptr
		var racearenaend uintptr

		func racefuncenter(callpc uintptr)
		func racefuncenterfp(fp uintptr)
		func racefuncexit()
		func raceread(addr uintptr)
		func racewrite(addr uintptr)
		func racereadrange(addr, size uintptr)
		func racewriterange(addr, size uintptr)
		func racereadrangepc1(addr, size, pc uintptr)
		func racewriterangepc1(addr, size, pc uintptr)
		func racecallbackthunk(uintptr)

		// racecall allows calling an arbitrary function f from C race runtime
		// with up to 4 uintptr arguments.
		func racecall(fn *byte, arg0, arg1, arg2, arg3 uintptr)

		// checks if the address has shadow (i.e. heap or data/bss)
		//go:nosplit
		func isvalidaddr(addr unsafe.Pointer) bool {
			return racearenastart <= uintptr(addr) && uintptr(addr) < racearenaend ||
				racedatastart <= uintptr(addr) && uintptr(addr) < racedataend
		}

		//go:nosplit
		func raceinit() (gctx, pctx uintptr) {
			// cgo is required to initialize libc, which is used by race runtime
			if !iscgo {
				throw("raceinit: race build must use cgo")
			}

			racecall(&__tsan_init, uintptr(unsafe.Pointer(&gctx)), uintptr(unsafe.Pointer(&pctx)), funcPC(racecallbackthunk), 0)

			// Round data segment to page boundaries, because it's used in mmap().
			start := ^uintptr(0)
			end := uintptr(0)
			if start > firstmoduledata.noptrdata {
				start = firstmoduledata.noptrdata
			}
			if start > firstmoduledata.data {
				start = firstmoduledata.data
			}
			if start > firstmoduledata.noptrbss {
				start = firstmoduledata.noptrbss
			}
			if start > firstmoduledata.bss {
				start = firstmoduledata.bss
			}
			if end < firstmoduledata.enoptrdata {
				end = firstmoduledata.enoptrdata
			}
			if end < firstmoduledata.edata {
				end = firstmoduledata.edata
			}
			if end < firstmoduledata.enoptrbss {
				end = firstmoduledata.enoptrbss
			}
			if end < firstmoduledata.ebss {
				end = firstmoduledata.ebss
			}
			size := alignUp(end-start, _PageSize)
			racecall(&__tsan_map_shadow, start, size, 0, 0)
			racedatastart = start
			racedataend = start + size

			return
		}

		var raceFiniLock mutex

		//go:nosplit
		func racefini() {
			// racefini() can only be called once to avoid races.
			// This eventually (via __tsan_fini) calls C.exit which has
			// undefined behavior if called more than once. If the lock is
			// already held it's assumed that the first caller exits the program
			// so other calls can hang forever without an issue.
			lock(&raceFiniLock)
			racecall(&__tsan_fini, 0, 0, 0, 0)
		}

		//go:nosplit
		func raceproccreate() uintptr {
			var ctx uintptr
			racecall(&__tsan_proc_create, uintptr(unsafe.Pointer(&ctx)), 0, 0, 0)
			return ctx
		}

		//go:nosplit
		func raceprocdestroy(ctx uintptr) {
			racecall(&__tsan_proc_destroy, ctx, 0, 0, 0)
		}

		//go:nosplit
		func racemapshadow(addr unsafe.Pointer, size uintptr) {
			if racearenastart == 0 {
				racearenastart = uintptr(addr)
			}
			if racearenaend < uintptr(addr)+size {
				racearenaend = uintptr(addr) + size
			}
			racecall(&__tsan_map_shadow, uintptr(addr), size, 0, 0)
		}

		//go:nosplit
		func racemalloc(p unsafe.Pointer, sz uintptr) {
			racecall(&__tsan_malloc, 0, 0, uintptr(p), sz)
		}

		//go:nosplit
		func racefree(p unsafe.Pointer, sz uintptr) {
			racecall(&__tsan_free, uintptr(p), sz, 0, 0)
		}

		//go:nosplit
		func racegostart(pc uintptr) uintptr {
			_g_ := getg()
			var spawng *g
			if _g_.m.curg != nil {
				spawng = _g_.m.curg
			} else {
				spawng = _g_
			}

			var racectx uintptr
			racecall(&__tsan_go_start, spawng.racectx, uintptr(unsafe.Pointer(&racectx)), pc, 0)
			return racectx
		}

		//go:nosplit
		func racegoend() {
			racecall(&__tsan_go_end, getg().racectx, 0, 0, 0)
		}

		//go:nosplit
		func racectxend(racectx uintptr) {
			racecall(&__tsan_go_end, racectx, 0, 0, 0)
		}

		//go:nosplit
		func racewriterangepc(addr unsafe.Pointer, sz, callpc, pc uintptr) {
			_g_ := getg()
			if _g_ != _g_.m.curg {
				// The call is coming from manual instrumentation of Go code running on g0/gsignal.
				// Not interesting.
				return
			}
			if callpc != 0 {
				racefuncenter(callpc)
			}
			racewriterangepc1(uintptr(addr), sz, pc)
			if callpc != 0 {
				racefuncexit()
			}
		}

		//go:nosplit
		func racereadrangepc(addr unsafe.Pointer, sz, callpc, pc uintptr) {
			_g_ := getg()
			if _g_ != _g_.m.curg {
				// The call is coming from manual instrumentation of Go code running on g0/gsignal.
				// Not interesting.
				return
			}
			if callpc != 0 {
				racefuncenter(callpc)
			}
			racereadrangepc1(uintptr(addr), sz, pc)
			if callpc != 0 {
				racefuncexit()
			}
		}

		//go:nosplit
		func raceacquire(addr unsafe.Pointer) {
			raceacquireg(getg(), addr)
		}

		//go:nosplit
		func raceacquireg(gp *g, addr unsafe.Pointer) {
			if getg().raceignore != 0 || !isvalidaddr(addr) {
				return
			}
			racecall(&__tsan_acquire, gp.racectx, uintptr(addr), 0, 0)
		}

		//go:nosplit
		func raceacquirectx(racectx uintptr, addr unsafe.Pointer) {
			if !isvalidaddr(addr) {
				return
			}
			racecall(&__tsan_acquire, racectx, uintptr(addr), 0, 0)
		}

		//go:nosplit
		func racerelease(addr unsafe.Pointer) {
			racereleaseg(getg(), addr)
		}

		//go:nosplit
		func racereleaseg(gp *g, addr unsafe.Pointer) {
			if getg().raceignore != 0 || !isvalidaddr(addr) {
				return
			}
			racecall(&__tsan_release, gp.racectx, uintptr(addr), 0, 0)
		}

		//go:nosplit
		func racereleasemerge(addr unsafe.Pointer) {
			racereleasemergeg(getg(), addr)
		}

		//go:nosplit
		func racereleasemergeg(gp *g, addr unsafe.Pointer) {
			if getg().raceignore != 0 || !isvalidaddr(addr) {
				return
			}
			racecall(&__tsan_release_merge, gp.racectx, uintptr(addr), 0, 0)
		}

		//go:nosplit
		func racefingo() {
			racecall(&__tsan_finalizer_goroutine, getg().racectx, 0, 0, 0)
		}

		// The declarations below generate ABI wrappers for functions
		// implemented in assembly in this package but declared in another
		// package.

		//go:linkname abigen_sync_atomic_LoadInt32 sync/atomic.LoadInt32
		func abigen_sync_atomic_LoadInt32(addr *int32) (val int32)

		//go:linkname abigen_sync_atomic_LoadInt64 sync/atomic.LoadInt64
		func abigen_sync_atomic_LoadInt64(addr *int64) (val int64)

		//go:linkname abigen_sync_atomic_LoadUint32 sync/atomic.LoadUint32
		func abigen_sync_atomic_LoadUint32(addr *uint32) (val uint32)

		//go:linkname abigen_sync_atomic_LoadUint64 sync/atomic.LoadUint64
		func abigen_sync_atomic_LoadUint64(addr *uint64) (val uint64)

		//go:linkname abigen_sync_atomic_LoadUintptr sync/atomic.LoadUintptr
		func abigen_sync_atomic_LoadUintptr(addr *uintptr) (val uintptr)

		//go:linkname abigen_sync_atomic_LoadPointer sync/atomic.LoadPointer
		func abigen_sync_atomic_LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)

		//go:linkname abigen_sync_atomic_StoreInt32 sync/atomic.StoreInt32
		func abigen_sync_atomic_StoreInt32(addr *int32, val int32)

		//go:linkname abigen_sync_atomic_StoreInt64 sync/atomic.StoreInt64
		func abigen_sync_atomic_StoreInt64(addr *int64, val int64)

		//go:linkname abigen_sync_atomic_StoreUint32 sync/atomic.StoreUint32
		func abigen_sync_atomic_StoreUint32(addr *uint32, val uint32)

		//go:linkname abigen_sync_atomic_StoreUint64 sync/atomic.StoreUint64
		func abigen_sync_atomic_StoreUint64(addr *uint64, val uint64)

		//go:linkname abigen_sync_atomic_SwapInt32 sync/atomic.SwapInt32
		func abigen_sync_atomic_SwapInt32(addr *int32, new int32) (old int32)

		//go:linkname abigen_sync_atomic_SwapInt64 sync/atomic.SwapInt64
		func abigen_sync_atomic_SwapInt64(addr *int64, new int64) (old int64)

		//go:linkname abigen_sync_atomic_SwapUint32 sync/atomic.SwapUint32
		func abigen_sync_atomic_SwapUint32(addr *uint32, new uint32) (old uint32)

		//go:linkname abigen_sync_atomic_SwapUint64 sync/atomic.SwapUint64
		func abigen_sync_atomic_SwapUint64(addr *uint64, new uint64) (old uint64)

		//go:linkname abigen_sync_atomic_AddInt32 sync/atomic.AddInt32
		func abigen_sync_atomic_AddInt32(addr *int32, delta int32) (new int32)

		//go:linkname abigen_sync_atomic_AddUint32 sync/atomic.AddUint32
		func abigen_sync_atomic_AddUint32(addr *uint32, delta uint32) (new uint32)

		//go:linkname abigen_sync_atomic_AddInt64 sync/atomic.AddInt64
		func abigen_sync_atomic_AddInt64(addr *int64, delta int64) (new int64)

		//go:linkname abigen_sync_atomic_AddUint64 sync/atomic.AddUint64
		func abigen_sync_atomic_AddUint64(addr *uint64, delta uint64) (new uint64)

		//go:linkname abigen_sync_atomic_AddUintptr sync/atomic.AddUintptr
		func abigen_sync_atomic_AddUintptr(addr *uintptr, delta uintptr) (new uintptr)

		//go:linkname abigen_sync_atomic_CompareAndSwapInt32 sync/atomic.CompareAndSwapInt32
		func abigen_sync_atomic_CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)

		//go:linkname abigen_sync_atomic_CompareAndSwapInt64 sync/atomic.CompareAndSwapInt64
		func abigen_sync_atomic_CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)

		//go:linkname abigen_sync_atomic_CompareAndSwapUint32 sync/atomic.CompareAndSwapUint32
		func abigen_sync_atomic_CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)

		//go:linkname abigen_sync_atomic_CompareAndSwapUint64 sync/atomic.CompareAndSwapUint64
		func abigen_sync_atomic_CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
		```
	</details>
	

- Actual instrumentation is spread out, integrated into the SSA construction process (by actual, we mean like prepending a load instruction with the function `racewrite`).

	- `go/cmd/compile/internal/gc/racewalk.go:instrument()`: marks a function for instrumentation during SSA and appends/prepends function block with racefuncenter/racefuncexit (also contains overview of racewalk "pass" in comments!)
		```go
		func instrument(fn *Node) {
			if fn.Func.Pragma&Norace != 0 {
				return
			}

			if !flag_race || !ispkgin(norace_inst_pkgs) {
				fn.Func.SetInstrumentBody(true)
			}

			if flag_race {
				lno := lineno
				lineno = src.NoXPos

				if thearch.LinkArch.Arch.Family != sys.AMD64 {
					fn.Func.Enter.Prepend(mkcall("racefuncenterfp", nil, nil))
					fn.Func.Exit.Append(mkcall("racefuncexit", nil, nil))
				} else {

					// nodpc is the PC of the caller as extracted by
					// getcallerpc. We use -widthptr(FP) for x86.
					// This only works for amd64. This will not
					// work on arm or others that might support
					// race in the future.
					nodpc := nodfp.copy()
					nodpc.Type = types.Types[TUINTPTR]
					nodpc.Xoffset = int64(-Widthptr)
					fn.Func.Dcl = append(fn.Func.Dcl, nodpc)
					fn.Func.Enter.Prepend(mkcall("racefuncenter", nil, nil, nodpc))
					fn.Func.Exit.Append(mkcall("racefuncexit", nil, nil))
				}
				lineno = lno
			}
		}
		```

	- `go/cmd/compile/internal/gc/ssa.go:instrument()`: Instrument memory loads and stores during SSA building
		```go
		func (s *state) instrument(t *types.Type, addr *ssa.Value, wr bool) {
			if !s.curfn.Func.InstrumentBody() {
				return
			}

			w := t.Size()
			if w == 0 {
				return // can't race on zero-sized things
			}

			if ssa.IsSanitizerSafeAddr(addr) {
				return
			}

			var fn *obj.LSym
			needWidth := false

			if flag_msan {
				fn = msanread
				if wr {
					fn = msanwrite
				}
				needWidth = true
			} else if flag_race && t.NumComponents(types.CountBlankFields) > 1 {
				// for composite objects we have to write every address
				// because a write might happen to any subobject.
				// composites with only one element don't have subobjects, though.
				fn = racereadrange
				if wr {
					fn = racewriterange
				}
				needWidth = true
			} else if flag_race {
				// for non-composite objects we can write just the start
				// address, as any write must write the first byte.
				fn = raceread
				if wr {
					fn = racewrite
				}
			} else {
				panic("unreachable")
			}

			args := []*ssa.Value{addr}
			if needWidth {
				args = append(args, s.constInt(types.Types[TUINTPTR], w))
			}
			s.rtcall(fn, true, nil, args...)
		}
		```
		This function is called immediately before each load, zero, move, and storeType instruction.
	
		```go
			func (s *state) load(t *types.Type, src *ssa.Value) *ssa.Value {
				s.instrument(t, src, false)
				return s.rawLoad(t, src)
			}

			func (s *state) rawLoad(t *types.Type, src *ssa.Value) *ssa.Value {
				return s.newValue2(ssa.OpLoad, t, src, s.mem())
			}

			// NO instrument here
			func (s *state) store(t *types.Type, dst, val *ssa.Value) {
				s.vars[&memVar] = s.newValue3A(ssa.OpStore, types.TypeMem, t, dst, val, s.mem())
			}

			func (s *state) zero(t *types.Type, dst *ssa.Value) {
				s.instrument(t, dst, true)
				store := s.newValue2I(ssa.OpZero, types.TypeMem, t.Size(), dst, s.mem())
				store.Aux = t
				s.vars[&memVar] = store
			}

			func (s *state) move(t *types.Type, dst, src *ssa.Value) {
				s.instrument(t, src, false)
				s.instrument(t, dst, true)
				store := s.newValue3I(ssa.OpMove, types.TypeMem, t.Size(), dst, src, s.mem())
				store.Aux = t
				s.vars[&memVar] = store
			}

			// do *left = right for type t.
			func (s *state) storeType(t *types.Type, left, right *ssa.Value, skip skipMask, leftIsStmt bool) {
				s.instrument(t, left, true)

				if skip == 0 && (!types.Haspointers(t) || ssa.IsStackAddr(left)) {
					// Known to not have write barrier. Store the whole type.
					s.vars[&memVar] = s.newValue3Apos(ssa.OpStore, types.TypeMem, t, left, right, s.mem(), leftIsStmt)
					return
				}

				// store scalar fields first, so write barrier stores for
				// pointer fields can be grouped together, and scalar values
				// don't need to be live across the write barrier call.
				// TODO: if the writebarrier pass knows how to reorder stores,
				// we can do a single store here as long as skip==0.
				s.storeTypeScalars(t, left, right, skip)
				if skip&skipPtr == 0 && types.Haspointers(t) {
					s.storeTypePtrs(t, left, right)
				}
			}
		```

	- `go/cmd/compile/internal/ssa/rewrite.go:needRaceCleanup()`: this is a special SSA optimization pass for removing unnecessary calls to `racefuncentry`/`racefuncexit` (can safely remove from leaf functions with no instrumented memory accesses)

   		- see https://github.com/golang/go/issues/24662

	- `go/cmd/compile/internal/ssa/writebarrier.go:IsSanitizerSafeAddr()`: provides some rules on optimizing instrumentation
		```go
		// IsSanitizerSafeAddr reports whether v is known to be an address
		// that doesn't need instrumentation.
		func IsSanitizerSafeAddr(v *Value) bool {
			for v.Op == OpOffPtr || v.Op == OpAddPtr || v.Op == OpPtrIndex || v.Op == OpCopy {
				v = v.Args[0]
			}
			switch v.Op {
			case OpSP, OpLocalAddr:
				// Stack addresses are always safe.
				return true
			case OpITab, OpStringPtr, OpGetClosurePtr:
				// Itabs, string data, and closure fields are
				// read-only once initialized.
				return true
			case OpAddr:
				sym := v.Aux.(*obj.LSym)
				// TODO(mdempsky): Find a cleaner way to
				// detect this. It would be nice if we could
				// test sym.Type==objabi.SRODATA, but we don't
				// initialize sym.Type until after function
				// compilation.
				if strings.HasPrefix(sym.Name, `""..stmp_`) {
					return true
				}
			}
			return false
		}
		```

---

#### Atomic Instructions

- Package `runtime/internal/atomic` provides library of atomic primitives in Go.

- Function headers are defined in `go/src/sync/atomic/doc.go`, and assembly thunks are defined in `go/src/sync/atomic/asm.s`, which call functions defined in `go/src/runtime/internal/atomic/asm_<arch>.s`.

- When a program is compiled with `-race`, the function headers are linked through `go/src/sync/atomic/races` which calls the assembly defined in `go/src/runtime/race_<arch>.s`.

For example, for CompareAndSwapInt32:
```as
// CompareAndSwap
TEXT	sync∕atomic·CompareAndSwapInt32(SB), NOSPLIT, $0-0
	MOVQ	$__tsan_go_atomic32_compare_exchange(SB), AX
	CALL	racecallatomic<>(SB)
	RET

...

// Generic atomic operation implementation.
// AX already contains target function.
TEXT	racecallatomic<>(SB), NOSPLIT, $0-0
	// Trigger SIGSEGV early.
	MOVQ	16(SP), R12
	MOVL	(R12), R13
	// Check that addr is within [arenastart, arenaend) or within [racedatastart, racedataend).
	CMPQ	R12, runtime·racearenastart(SB)
	JB	racecallatomic_data
	CMPQ	R12, runtime·racearenaend(SB)
	JB	racecallatomic_ok
racecallatomic_data:
	CMPQ	R12, runtime·racedatastart(SB)
	JB	racecallatomic_ignore
	CMPQ	R12, runtime·racedataend(SB)
	JAE	racecallatomic_ignore
racecallatomic_ok:
	// Addr is within the good range, call the atomic function.
	get_tls(R12)
	MOVQ	g(R12), R14
	MOVQ	g_racectx(R14), RARG0	// goroutine context
	MOVQ	8(SP), RARG1	// caller pc
	MOVQ	(SP), RARG2	// pc
	LEAQ	16(SP), RARG3	// arguments
	JMP	racecall<>(SB)	// does not return
racecallatomic_ignore:
	// Addr is outside the good range.
	// Call __tsan_go_ignore_sync_begin to ignore synchronization during the atomic op.
	// An attempt to synchronize on the address would cause crash.
	MOVQ	AX, R15	// remember the original function
	MOVQ	$__tsan_go_ignore_sync_begin(SB), AX
	get_tls(R12)
	MOVQ	g(R12), R14
	MOVQ	g_racectx(R14), RARG0	// goroutine context
	CALL	racecall<>(SB)
	MOVQ	R15, AX	// restore the original function
	// Call the atomic function.
	MOVQ	g_racectx(R14), RARG0	// goroutine context
	MOVQ	8(SP), RARG1	// caller pc
	MOVQ	(SP), RARG2	// pc
	LEAQ	16(SP), RARG3	// arguments
	CALL	racecall<>(SB)
	// Call __tsan_go_ignore_sync_end.
	MOVQ	$__tsan_go_ignore_sync_end(SB), AX
	MOVQ	g_racectx(R14), RARG0	// goroutine context
	JMP	racecall<>(SB)
```
The asm function`racecallatomic` calls runtime library functions from the linked TSAN library (`llvm/projects/compiler-rt/lib/tsan/rtl/tsan_interface_atomic.cc`).

---

#### Instrumentation in LLVM Clang vs. in Go



---

#### Optimizations

TODO - what cases are covered by classic opts, what are covered by explicit handling, which are not handled?
```
- number lines

- early phielim
	* phi elimination - a phi is redundant if its arguments are all equal. For the purposes of counting, ignore  the phi itself

- early copye

- early deadcode// remove generated dead code to avoid doing pointless work during opt

- short circuit

- decompose args

- decompose user

- pre-opt deadcode

- opt // NB: some generic rules know the name of the opt pass. TODO: split required rules and optimizing rules

- zero arg cse // required to merge OpSB values

- opt deadcode // remove any blocks orphaned during opt

- generic cse

- phiopt

- gcse deadcode // clean out after cse and phiopt

- nilcheckelim

- prove

- fuse plain

- decompose builtin

- softfloat

- late opt// TODO: split required rules and optimizing rules

- dead auto elim

- generic deadcode // remove dead stores, which otherwise mess up store chain

- check bce

- branchelim

- fuse

- dse //dead-store elimination (a dead store is unconditionally followed by another store to the same location, with no intervening load)

- writebarrier // expand write barrier ops

- insert resched checks // insert resched checks in loops.

- lower

- lowered deadcode for cse // deadcode immediately before CSE avoids CSE making dead values live again

- lowered cse

- elim unread autos

- lowered deadcode

- checkLower

- late phielim

- late copyelim

- tighten // move values closer to their uses

- late deadcode

- critical // remove critical edges

- phi tighten // place rematerializable phi args near uses to reduce value lifetimes

- likelyadjust

- layout // schedule blocks

- schedule // schedule values

- late nilcheck

- flagalloc // allocate flags register

- regalloc // allocate int & float registers + stack slots

- loop rotate

- stackframe

- trim // remove empty blocks
```

---

#### Do not instrument addresses that are well-defined to not need instrumentation

- stack addresses

- Itabs, string data, and closure fields are read-only once initialized

- read-only static data

---

#### RTL

- `racefuncentry`/`racefuncexit`: inserted at the beginning and end (respectively) of functions containing instrumented memory accesses
   - used to restore stack traces

- TODO

---

## Experiments

Created and compiled small toy programs with race instrumentation to further understand Go race instrumentation.

#### Stack Variables

#### Mutexes

#### Atomics

#### Escaped Stack Variables

#### Read-before-write

Read-before-write within same basic block, with no calls in-between.

#### Not Captured Variables

## Kubernetes

TODO

## Evaluating Minikube

TODO