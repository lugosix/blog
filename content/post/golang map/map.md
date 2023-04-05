---
title: "golang map 分析与思考"
date: 2022-07-21T16:00:39+08:00
draft: true
---
# 1 使用
```go
package main

import "fmt"

func main() {
    m := make(map[int]int)
    m[1] = 2
    fmt.Println("map value", m)
}
```
# 2 实现
执行 ```go  tool compile -S main.go``` 获取 plan9 汇编结果
```bash
[root@10 go1.18.3]# ./bin/go  tool compile -S main.go 
"".main STEXT size=171 args=0x0 locals=0x58 funcid=0x0 align=0x0
        0x0000 00000 (main.go:5)        TEXT    "".main(SB), ABIInternal, $88-0
        0x0000 00000 (main.go:5)        CMPQ    SP, 16(R14)
        0x0004 00004 (main.go:5)        PCDATA  $0, $-2
        0x0004 00004 (main.go:5)        JLS     161
        0x000a 00010 (main.go:5)        PCDATA  $0, $-1
        0x000a 00010 (main.go:5)        SUBQ    $88, SP
        0x000e 00014 (main.go:5)        MOVQ    BP, 80(SP)
        0x0013 00019 (main.go:5)        LEAQ    80(SP), BP
        0x0018 00024 (main.go:5)        FUNCDATA        $0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
        0x0018 00024 (main.go:5)        FUNCDATA        $1, gclocals·d527b79a98f329c2ba624a68e7df03d6(SB)
        0x0018 00024 (main.go:5)        FUNCDATA        $2, "".main.stkobj(SB)
        0x0018 00024 (main.go:6)        PCDATA  $1, $0
        0x0018 00024 (main.go:6)        CALL    runtime.makemap_small(SB)
        0x001d 00029 (main.go:6)        MOVQ    AX, "".m+40(SP)
        0x0022 00034 (main.go:7)        MOVQ    AX, BX
        0x0025 00037 (main.go:7)        MOVL    $1, CX
        0x002a 00042 (main.go:7)        LEAQ    type.map[int]int(SB), AX
        0x0031 00049 (main.go:7)        PCDATA  $1, $1
        0x0031 00049 (main.go:7)        CALL    runtime.mapassign_fast64(SB)
        0x0036 00054 (main.go:7)        MOVQ    $2, (AX)
        0x003d 00061 (main.go:8)        MOVUPS  X15, ""..autotmp_11+48(SP)
        0x0043 00067 (main.go:8)        MOVUPS  X15, ""..autotmp_11+64(SP)
        0x0049 00073 (main.go:8)        LEAQ    type.string(SB), AX
        0x0050 00080 (main.go:8)        MOVQ    AX, ""..autotmp_11+48(SP)
        0x0055 00085 (main.go:8)        LEAQ    ""..stmp_0(SB), AX
        0x005c 00092 (main.go:8)        MOVQ    AX, ""..autotmp_11+56(SP)
        0x0061 00097 (main.go:8)        LEAQ    type.map[int]int(SB), AX
        0x0068 00104 (main.go:8)        MOVQ    AX, ""..autotmp_11+64(SP)
        0x006d 00109 (main.go:8)        MOVQ    "".m+40(SP), AX
        0x0072 00114 (main.go:8)        MOVQ    AX, ""..autotmp_11+72(SP)
        0x0077 00119 (<unknown line number>)    NOP
        0x0077 00119 ($GOROOT/src/fmt/print.go:274)     MOVQ    os.Stdout(SB), BX
        0x007e 00126 ($GOROOT/src/fmt/print.go:274)     LEAQ    go.itab.*os.File,io.Writer(SB), AX
        0x0085 00133 ($GOROOT/src/fmt/print.go:274)     LEAQ    ""..autotmp_11+48(SP), CX
        0x008a 00138 ($GOROOT/src/fmt/print.go:274)     MOVL    $2, DI
        0x008f 00143 ($GOROOT/src/fmt/print.go:274)     MOVQ    DI, SI
        0x0092 00146 ($GOROOT/src/fmt/print.go:274)     PCDATA  $1, $0
        0x0092 00146 ($GOROOT/src/fmt/print.go:274)     CALL    fmt.Fprintln(SB)
        0x0097 00151 (main.go:9)        MOVQ    80(SP), BP
        0x009c 00156 (main.go:9)        ADDQ    $88, SP
        0x00a0 00160 (main.go:9)        RET
        0x00a1 00161 (main.go:9)        NOP
        0x00a1 00161 (main.go:5)        PCDATA  $1, $-1
        0x00a1 00161 (main.go:5)        PCDATA  $0, $-2
        0x00a1 00161 (main.go:5)        CALL    runtime.morestack_noctxt(SB)
        0x00a6 00166 (main.go:5)        PCDATA  $0, $-1
        0x00a6 00166 (main.go:5)        JMP     0
```
第15行调用 makemap_small 源码如下
```go
// makemap_small implements Go map creation for make(map[k]v) and
// make(map[k]v, hint) when hint is known to be at most bucketCnt
// at compile time and the map needs to be allocated on the heap.
func makemap_small() *hmap {
    h := new(hmap)
    h.hash0 = fastrand()
    return h
}
```
可以看到返回值是一个 hmap 实例的指针（即 m 内保存的这个 hmap 的地址）
## 2.1 主要数据结构
```go
// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    count     int    // 有效的元素个数，必须在第一个位置 (len() 函数会返回此值)
    flags     uint8  // 标识读写状态，用于并发检测
    B         uint8  // log2(bucket 数量) (最多可容纳 loadFactor * 2^B 个)
    noverflow uint16 // 近似的溢出 bucket 数量
    hash0     uint32 // 哈希种子，赋予 hash 函数随机性


    buckets    unsafe.Pointer // 2^B 个 bucket 的数组，元素为0时这个值可能会为 nil.
    oldbuckets unsafe.Pointer // 扩容时用于保存之前 buckets 的字段，它的大小是当前 buckets 的一半。只有在扩容时不为 nil
    nevacuate  uintptr        // 扩容进度计数 (小于此地址的 buckets 迁移完成)

    extra *mapextra // 用于优化 gc
}


// A bucket for a Go map.
type bmap struct {
    // tophash generally contains the top byte of the hash value
    // for each key in this bucket. If tophash[0] < minTopHash,
    // tophash[0] is a bucket evacuation state instead.
    tophash [bucketCnt]uint8
    // Followed by bucketCnt keys and then bucketCnt elems.
    // NOTE: packing all the keys together and then all the elems together makes the
    // code a bit more complicated than alternating key/elem/key/elem/... but it allows
    // us to eliminate padding which would be needed for, e.g., map[int64]int8.
    // Followed by an overflow pointer.
	
	/*
	type bmap struct {
    	topbits  [8]uint8
    	keys     [8]keytype
    	values   [8]valuetype
    	pad      uintptr
    	overflow uintptr
	}
	*/
}


// mapextra holds fields that are not present on all maps.
type mapextra struct {
    // 如果 key 和 elem 都不包含指针并且是内联的，那么我们就把bucket类型标记为不包含指针。这样就可以避免扫描这样的地图。
    // 然而，bmap.overflow是一个指针。为了保持 overflow buckets 的存活，我们在 hmap.extra.overflow 和 hmap.extra.oldoverflow 中存储所有 overflow buckets 的指针。
    // overflow 和 oldoverflow 只在 key 和 elem 不包含指针时使用。
    // overflow 包含 hmap.buckets 的 overflow buckets。oldoverflow 包含 hmap.oldbuckets 的 overflow buckets。
    // The indirection allows to store a pointer to the slice in hiter.
    overflow    *[]*bmap
    oldoverflow *[]*bmap


    // nextOverflow holds a pointer to a free overflow bucket.
    nextOverflow *bmap
}
```
|  hmap|bmap|访问 map  |
| ------ |----| ------ |
| ![](/golang_map/1.png) |![](/golang_map/2.png)  | ![](/golang_map/3.png)|
## 2.2 写
因没有泛型且需要有良好的性能，针对不同类型有多种实现
```go
func mapassign(mapType *byte, hmap map[any]any, key *any) (val *any)
func mapassign_fast32(mapType *byte, hmap map[any]any, key uint32) (val *any)
func mapassign_fast32ptr(mapType *byte, hmap map[any]any, key unsafe.Pointer) (val *any)
func mapassign_fast64(mapType *byte, hmap map[any]any, key uint64) (val *any)
func mapassign_fast64ptr(mapType *byte, hmap map[any]any, key unsafe.Pointer) (val *any)
func mapassign_faststr(mapType *byte, hmap map[any]any, key string) (val *any)
```
### 2.2.1 mapassign_fast64
<details> <summary>源码摘录（点击展开）</summary>

```go
func mapassign_fast64(t *maptype, h *hmap, key uint64) unsafe.Pointer {
	/*
    t runtime.maptype {
        typ: runtime._type {size: 8, ptrdata: 8, hash: 592976720, tflag: tflagExtraStar (2), align: 8, fieldAlign: 8, kind: 53, equal: nil, gcdata: *1, str: 7924, ptrToThis: 0},
        key: *runtime._type {size: 8, ptrdata: 0, hash: 4149441018, tflag: tflagUncommon|tflagExtraStar|tflagNamed|tflagRegularMemory (15), align: 8, fieldAlign: 8, kind: 2, equal: runtime.memequal64, gcdata: *0, str: 470, ptrToThis: 16960},
        elem: *runtime._type {size: 8, ptrdata: 0, hash: 4149441018, tflag: tflagUncommon|tflagExtraStar|tflagNamed|tflagRegularMemory (15), align: 8, fieldAlign: 8, kind: 2, equal: runtime.memequal64, gcdata: *0, str: 470, ptrToThis: 16960},
        bucket: *runtime._type {size: 144, ptrdata: 0, hash: 3152881843, tflag: tflagExtraStar (2), align: 8, fieldAlign: 8, kind: 25, equal: nil, gcdata: *0, str: 12601, ptrToThis: 0},
        hasher: runtime.memhash64,
        keysize: 8,
        elemsize: 8,
        bucketsize: 144,
        flags: 4,}
	h runtime.hmap {
        count: 0,
        flags: 0,
        B: 0,
        noverflow: 0,
        hash0: 2451457200,
        buckets: unsafe.Pointer(0x0),
        oldbuckets: unsafe.Pointer(0x0),
        nevacuate: 0,
        extra: *runtime.mapextra nil,}
	key int 1
    */
	
    if h == nil {
        panic(plainError("assignment to entry in nil map"))
    }
    if raceenabled {
        callerpc := getcallerpc()
        racewritepc(unsafe.Pointer(h), callerpc, abi.FuncPCABIInternal(mapassign_fast64))
    }
    if h.flags&hashWriting != 0 {
        throw("concurrent map writes")
    }
    hash := t.hasher(noescape(unsafe.Pointer(&key)), uintptr(h.hash0))


    // Set hashWriting after calling t.hasher for consistency with mapassign.
    h.flags ^= hashWriting

    if h.buckets == nil {
        h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
    }

again:
    bucket := hash & bucketMask(h.B)
    if h.growing() {
        growWork_fast64(t, h, bucket)
    }
    b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))

    var insertb *bmap
    var inserti uintptr
    var insertk unsafe.Pointer

bucketloop:
    for {
        for i := uintptr(0); i < bucketCnt; i++ {
            if isEmpty(b.tophash[i]) {
                if insertb == nil {
                    insertb = b
                    inserti = i
                }
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            k := *((*uint64)(add(unsafe.Pointer(b), dataOffset+i*8)))
            if k != key {
                continue
            }
            insertb = b
            inserti = i
            goto done
        }
        ovf := b.overflow(t)
        if ovf == nil {
            break
        }
        b = ovf
    }

    // Did not find mapping for key. Allocate new cell & add entry.

    // If we hit the max load factor or we have too many overflow buckets,
    // and we're not already in the middle of growing, start growing.
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again // Growing the table invalidates everything, so try again
    }

    if insertb == nil {
        // The current bucket and all the overflow buckets connected to it are full, allocate a new one.
        insertb = h.newoverflow(t, b)
        inserti = 0 // not necessary, but avoids needlessly spilling inserti
    }
    insertb.tophash[inserti&(bucketCnt-1)] = tophash(hash) // mask inserti to avoid bounds checks

    insertk = add(unsafe.Pointer(insertb), dataOffset+inserti*8)
    // store new key at insert position
    *(*uint64)(insertk) = key

    h.count++

done:
    elem := add(unsafe.Pointer(insertb), dataOffset+bucketCnt*8+inserti*uintptr(t.elemsize))
    if h.flags&hashWriting == 0 {
        throw("concurrent map writes")
    }
    h.flags &^= hashWriting
    return elem
}
```

</details>

## 2.3 扩容
负载因子（loadFactor）= 元素数量/ (2^B)

### 2.3.1 两种扩容方式
加倍扩容：负载/装载因子大于6.5，bucket 都快装满了

等量扩容：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。
||等量扩容|增量扩容|
|---|---|---|
|扩容前|![](/golang_map/4.png)||
|扩容后|![](/golang_map/5.png)|![](/golang_map/6.png)|
### 2.3.2 搬迁
![](/golang_map/7.png)  
![](/golang_map/8.png)
# 3 感悟&思考
1. 怎么变成线程安全
2. 与分布式 kv 的关系
3. 与业务层自己做分片的关系
4. 扩容的启发

注：图片部分非原创