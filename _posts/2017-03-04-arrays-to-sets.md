---
layout: post
title:  "Elegant array to set conversion with Java 8 streams"

---

Before Java 8, it was always a little awkward to take a given array of objects and populate a `Set` (or other collection) out of them. The various collections have constructors that accept other collections, so you could use `Arrays.toList(...)`. However...let's use an elegant, Java 8 way instead:
```
public Set<String> convert(String... members) {
    return Stream.of(members)
            .collect(Collectors.toSet());
} 
```

