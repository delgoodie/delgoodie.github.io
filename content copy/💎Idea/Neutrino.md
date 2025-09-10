Default syntax is const, mark methods as non-const

```
+x sort = (
    +x sorted = false,
    (!sorted) ; (
        sorted = true,
        +x i = 0,
        (i < len> RHS - 1) ; (
            (RHS[i] > RHS[i + 1]) ? (
                +x temp = RHS[i]
                RHS[i] = RHS[i + 1]
                RHS[i + 1] = temp
                sorted = false
            )
        )
    )
    return RHS
)
sort> (23, 12, 4, 14, 8) // => (4, 8, 12, 14, 23)
```