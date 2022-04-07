+++
title = "Go Dirty Tricks."
outputs = ["Reveal"]
[logo]
src = "kazdream_logo.svg"
width = "5%"
[reveal_hugo]
custom_theme = "reveal-hugo/themes/robot-lung.css"
margin = 0
+++

## Go Dirty Tricks.

- Зачем нужны грязные приёмы.
- Как их применять и чем они "грязны".

---

Грязные приёмы это:

- Строки.
- Слайсы.
- Финализаторы.
- Двоичные представления.

---

Что там со строками?

---

String-based API:

```go
func HashString(h hash.Hash64, s string) uint64 {
	fmt.Fprint(h, s)
	return h.Sum64()
}
```

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

💡  Не делай аллокаций.

`string(...)`, как правило, делает аллокацию.
Исключения:

```go
// Условный оператор:
if s == string(...) {
	...
}

// Переключатель
switch string(...) {
	...
}
```

---

# 🤗

That's it.

Happy Hugo-ing!
