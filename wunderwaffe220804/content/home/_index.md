+++
title = "Go Dirty Tricks."
outputs = ["Reveal"]
[logo]
src = "kazdream_logo.svg"
width = "5%"
[reveal_hugo]
custom_theme = "reveal-hugo/themes/robot-lung.css"
# none/fade/slide/convex/concave/zoom
transition = "fade"
margin = 0
+++

## Go Dirty Tricks.

- Зачем нужны грязные приёмы.
- Как их применять и чем они "грязны".

---

Грязные приёмы это:

- Строки.
- Финализаторы.
- Двоичные представления.

---

### Строки.

```go
func HashString(h hash.Hash64, s string) uint64 {
	fmt.Fprint(h, s)
	return h.Sum64()
}
```

Функция из другого проекта. Редактировать нельзя.

---

В чем здесь проблема?

```go
func () {
	h := fnv1a.New64()
	ch := make(chan []byte, 16)

	...
	for b := range ch {
		hashValue := HashString(h, string(b))
		// work with hashValue
	}
}
```

---

💡 Не делай аллокаций.

`string(...)`, как правило, делает аллокацию.
Исключения:

```go
// Условный оператор:
if "what" == string(...) {
	...
}

// Переключатель
switch string(...) {
case "what":`
	...
}
```

---

Исправляем:

```go
func HashBytes(h hash.Hash64, b []byte) uint64 {
	var s string

	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	sh.Data = uintptr(unsafe.Pointer(&b[0]))
	sh.Len = len(b)

	return HashString(h, s)
}
```

---

Опасность!

```go
func HashBytes(h hash.Hash64, b []byte) uint64 {
	var s string

	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	sh.Data = uintptr(unsafe.Pointer(&b[0]))
	sh.Len = len(b)

	// b нету, значит можно его освободить

	return HashString(h, s)
}
```

---

Исправляем, дубль 2:

```go
func HashBytes(h hash.Hash64, b []byte) uint64 {
	var s string

	sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
	sh.Data = uintptr(unsafe.Pointer(&b[0]))
	sh.Len = len(b)

	hashValue := HashString(h, s)
	runtime.KeepAlive(b)

	return hashValue
}
```

---

### Финализаторы.

Есть библиотека на C.
```c
struct awesome;

struct awesome *create_awesome();
void free_awesome(struct awesome *arg);
```

И есть обёртка на Go:
```go
type Awesome C.struct_awesome;
```

---

Не очень удобно:

```go
func CreateAwesome() *Awesome {
	return (*Awesome)(C.create_awesome())
}

func FreeAwesome(arg *Awesome) {
	C.free_awesome((*C.struct_awesome)(arg))
}
```

---

Можно сделать с финализатором:
```go
type Awesome struct {
	ptr *C.struct_awesome
}

func CreateAwesome() *Awesome {
	awe := &Awesome{
		ptr: C.create_awesome(),
	}
	
	runtime.SetFinalizer(awe, func(awe *Awesome) {
		C.free_awesome(awe.ptr)
	})
	
	return awe
}
```
Цена: indirection.

---

### Двоичное представление

```go
type SomeData struct {
	X uint64
	Y uint16
	Z uint32
}

func HashSomeData(h hash.Hash64, s *SomeData) uint64 {
	h.Reset()
	binary.Write(h, binary.BigEndian, s)
	return h.Sum64()
}
```

---

Давайте ускоримся:

```go
type SomeData struct {
	X uint64
	Y uint16
	Z uint32
}

func HashSomeDataFaster(h hash.Hash64, s *SomeData) uint64 {
	var buf [16]byte
	b := buf[:]
	binary.BigEndian.PutUint64(b[0:8], s.X)
	binary.BigEndian.PutUint16(b[8:10], s.Y)
	binary.BigEndian.PutUint32(b[10:14], s.Z)
	h.Reset()
	h.Write(buf[:14])
	return h.Sum64()
}
```

---

Давайте ещё ускоримся:

```go
type SomeData struct {
	X uint64
	Y uint16
	Z uint32
}

func HashSomeDataEvenFaster(h hash.Hash64, s *SomeData) uint64 {
	b := (*[unsafe.Sizeof(*s)]byte)(unsafe.Pointer(s))
	h.Write(b[:])
	return h.Sum64()
}
```

---

### Минусы языка

- Внезапные оптимизации :)
- GC начинает мешать при >1M аллокациях
  * “У вас ещё есть pointer? Тогда мы идём к вам!”
- Нельзя объявить alignment
  * “Cache line alignment? Зачем тебе?”
- Cgo
  * не любит stack-alloc;
  * не делает inline;
  * нельзя передавать go pointer.


---

🤗  Happy Go-ing!
![Alt
Text](https://miro.medium.com/max/1400/1*jcS_Wu-nlRcg8vVu8Su6Gg.jpeg)
