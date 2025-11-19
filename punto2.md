# Punto 2 — Gramática de atributos para producto punto / multiplicación de matrices

En este archivo está explicado **todo** lo que se modeló para resolver el producto punto (multiplicación matricial) entre dos matrices de diferentes dimensiones. En este caso lo que se hizo fue diseñar una gramática de un mini-lenguaje que permite declarar matrices, verificar compatibilidad de dimensiones y generar código intermedio (pseudocódigo) para realizar la multiplicación.

---

## 1. Objetivo y enfoque
En este caso lo que se hizo fue concentrarnos en tres objetivos principales:

- Verificar semánticamente que dos matrices sean compatibles para multiplicación (es decir, que el número de columnas de la primera coincida con el número de filas de la segunda).  
- Sintetizar la forma (filas y columnas) y el tipo del resultado.  
- Generar código intermedio que realice la multiplicación (bucles triple for) o, si procede, una llamada optimizada a una librería (por ejemplo BLAS).

Acá lo que pasa es que la gramática no solo describe la sintaxis, sino que también captura atributos (filas, columnas, tipo de elemento, si las dimensiones son constantes en compilación, errores y código generado) que se pasan y combinan siguiendo reglas semánticas.

---

## 2. No terminales principales y una visión general
Entonces podemos ver la estructura sintáctica que diseñamos: declaraciones de matrices, asignaciones y expresiones para multiplicación.

```
Program   → DeclList StmtList
DeclList  → Decl DeclList | ε
StmtList  → Stmt StmtList | ε

Decl      → id ':' MatrixType ';'
MatrixType→ 'matrix' '<' BaseType ',' Dim ',' Dim '>'
Dim       → NUMBER | id
BaseType  → 'int' | 'float' | 'double'

Stmt     → Assign ';'
Assign   → id '=' Expr

Expr     → MatMulExpr
MatMulExpr → PrimaryExpr '*' PrimaryExpr
PrimaryExpr → id | '(' Expr ')' | MatLiteral
MatLiteral → '[' RowList ']'
RowList  → Row ',' RowList | Row
Row      → '[' ExprList ']'
ExprList → Expr ',' ExprList | Expr
```

En este caso lo que se hizo fue permitir dimensiones ya sea como literales numéricas o como identificadores (para soportar dimensiones dinámicas o dependientes de variables).

---

## 3. Atributos principales (qué guardan y cómo se usan)

En la gramática se definieron atributos heredados y sintetizados. Acá lo que pasa es que estos son los más importantes:

- `env (H)` — tabla de símbolos heredada: mapea nombre → {rows, cols, elemType, dimsKnown}.  
- `rows (S)` — número de filas de la matriz expresada (puede ser entero o símbolo/`unknown`).  
- `cols (S)` — número de columnas de la matriz expresada.  
- `elemType (S)` — tipo del elemento (`int`, `float`, `double`, `unknown`).  
- `dimsKnown (S)` — booleano que indica si filas/cols son constantes conocidas en tiempo de compilación.  
- `code (S)` — string con el código intermedio generado (p. ej. pseudocódigo en C).  
- `errors (S)` — lista de errores semánticos (mensajes con información de posición/causa).

Entonces podemos ver que `env` se pasa desde el programa hacia las declaraciones y sentencias; los demás atributos se sintetizan desde abajo hacia arriba para cada expresión.

---

## 4. Reglas semánticas clave (explicadas con palabras)

A continuación se detallan, con el enfoque "En este caso lo que se hizo fue..." y "Acá lo que pasa es que...", las reglas más relevantes.

### Declaración de matriz
**Producción:** `Decl → id : MatrixType ;`

- En este caso lo que se hizo fue insertar en la tabla de símbolos (`env`) una entrada para `id` con sus atributos: `rows`, `cols`, `elemType` y `dimsKnown`.  
- Acá lo que pasa es que si `Dim` es un `NUMBER` se guarda el valor numérico; si es un `id` se guarda el nombre simbólico y `dimsKnown` se marca como `false`.

### Referencia a identificador (PrimaryExpr → id)

- En este caso lo que se hizo fue resolver el identificador en `env`.  
- Acá lo que pasa es que si `id` existe se sintetizan `rows`, `cols`, `elemType`, `dimsKnown` y `code` = `id`.  
- Entonces podemos ver que si `id` no está declarado se añade un error y los atributos se marcan como desconocidos.

### Multiplicación matricial (MatMulExpr → E1 '*' E2)

- En este caso lo que se hizo fue aplicar la verificación de compatibilidad: se comparan `E1.cols` con `E2.rows`.  
- Acá lo que pasa es que si ambos valores son constantes y no coinciden, se añade un error de incompatibilidad. Si alguno es simbólico o variable, se marca `dimsKnown = false` y se genera un chequeo en tiempo de ejecución.  
- Entonces podemos ver que el resultado sintetiza `rows = E1.rows` y `cols = E2.cols` (estas son las dimensiones de la matriz resultante), y `elemType = numeric_promote(E1.elemType, E2.elemType)`.

### Generación de código (regla de código sintetizado)

- En este caso lo que se hizo fue dos alternativas: si `dimsKnown == true` se genera código con límites constantes (bucles `for` con literales); si `dimsKnown == false` se genera código que incluye una comprobación en tiempo de ejecución `if (E1.cols != E2.rows) runtime_error(...)` y luego bucles usando variables como límites.

- Acá lo que pasa es que el pseudocódigo que sintetizamos es el clásico triple-loop:

```c
// ejemplo cuando dimsKnown == true
T C[rows][cols];
for (int i = 0; i < rows; ++i) {
  for (int j = 0; j < cols; ++j) {
    C[i][j] = 0;
    for (int k = 0; k < common; ++k) {
      C[i][j] += A[i][k] * B[k][j];
    }
  }
}
```

- Entonces podemos ver que si conviene optimizar se podría detectar `dimsKnown` y matrices grandes para generar llamadas a BLAS (`cblas_sgemm`) en lugar de bucles.

---

## 5. Reglas semánticas formales (ejemplos compactos)

A modo de guía, acá lo que se hizo fue resumir las reglas en forma casi formal:

```
-- Para PrimaryExpr → id
PrimaryExpr.env = env
if id ∈ env:
    PrimaryExpr.rows  = env[id].rows
    PrimaryExpr.cols  = env[id].cols
    PrimaryExpr.elemType = env[id].elemType
    PrimaryExpr.dimsKnown = env[id].dimsKnown
    PrimaryExpr.code = id
else:
    PrimaryExpr.errors = ["variable 'id' no declarada"]
    PrimaryExpr.rows = PrimaryExpr.cols = 'unknown'
```

```
-- Para MatMulExpr → E1 '*' E2
MatMulExpr.env = env
E1.env = env
E2.env = env

-- Chequeo de compatibilidad:
if E1.cols y E2.rows son constantes:
    if E1.cols != E2.rows: add error
    MatMulExpr.rows = E1.rows
    MatMulExpr.cols = E2.cols
    MatMulExpr.dimsKnown = E1.dimsKnown AND E2.dimsKnown
else:
    MatMulExpr.rows = E1.rows
    MatMulExpr.cols = E2.cols
    MatMulExpr.dimsKnown = false

MatMulExpr.elemType = numeric_promote(E1.elemType, E2.elemType)
MatMulExpr.errors = concat(E1.errors, E2.errors, posible_incompatibilidad)
```

En este caso lo que se hizo fue priorizar la detección temprana de errores semánticos y mantener la información necesaria para generar código correcto.

---

## 6. Ejemplo paso a paso (caso concreto)

**Declaraciones:**

```
A : matrix<int, 2, 3>;
B : matrix<int, 3, 4>;
C : matrix<int, 2, 4>;
```

En este caso lo que se hizo fue llenar la tabla `env` con tres entradas que contienen dimensiones constantes y tipo `int`. Acá lo que pasa es que al evaluar `C = A * B;`:

1. `A` tiene `rows=2, cols=3`.
2. `B` tiene `rows=3, cols=4`.
3. Se chequea `A.cols == B.rows` → `3 == 3` OK.
4. Resultado: `rows=2, cols=4, elemType=int` y `dimsKnown=true`.
5. Código sintetizado (ejemplo generado):

```c
int C[2][4];
for (int i = 0; i < 2; ++i) {
  for (int j = 0; j < 4; ++j) {
    C[i][j] = 0;
    for (int k = 0; k < 3; ++k) {
      C[i][j] += A[i][k] * B[k][j];
    }
  }
}
```

Entonces podemos ver que la gramática garantiza la compatibilidad y produce código listo para compilar o transformar.

---

## 7. Caso con dimensiones simbólicas (dinámicas)

**Declaraciones:**

```
A : matrix<int, n, m>;
B : matrix<int, m, p>;
```

Acá lo que pasa es que `n`, `m` y `p` son símbolos (no valores constantes). En este caso lo que se hizo fue marcar `dimsKnown = false` para esas matrices y generar código que incluya una verificación en ejecución:

```c
if (A.cols != B.rows) runtime_error("Incompatibilidad de dimensiones");
int C[A.rows][B.cols];
for (int i = 0; i < A.rows; ++i) {
  for (int j = 0; j < B.cols; ++j) {
    C[i][j] = 0;
    for (int k = 0; k < A.cols; ++k) {
      C[i][j] += A[i][k] * B[k][j];
    }
  }
}
```

Entonces podemos ver que esto permite soportar matrices con tamaños decididos en tiempo de ejecución y, a la vez, conservar seguridad mediante cheques en runtime.

---

## 8. Utilidades y funciones auxiliares necesarias

En este caso lo que se hizo fue mencionar las funciones que la implementación práctica necesitaría:

- `numeric_promote(t1, t2)` — decide tipo resultado (`float` si uno es `float`, etc.).  
- `is_numeric(t)` — verifica si un tipo soporta multiplicación/aritmética.  
- `mkError(msg, pos)` — crear y almacenar errores semánticos.  
- `const_value(d)` — si `d` es literal devuelve su valor; si es identificador devuelve valor simbólico.

Acá lo que pasa es que sin esas utilidades sería difícil implementar las reglas semánticas de forma robusta.

---

## 9. Extensiones y recomendaciones (qué más se puede hacer)

- Optimización: en este caso lo que se hizo fue contemplar la posibilidad de detectar `dimsKnown` y matrices grandes para generar llamadas a BLAS en vez de bucles, lo que mejora rendimiento.  
- Soporte mixto de tipos: Acá lo que pasa es que si multiplicamos `int` por `float` debemos promover a `float` y añadir casts si es necesario.  
- Vectores y broadcasting: Entonces podemos ver que es factible agregar reglas para vectores (1-D) y algunas formas de broadcasting para compatibilizar dimensiones.  
- Mejores mensajes de error: incluir posición/línea en `mkError` para ayudar al usuario.

---

## 10. (Opcional) Implementación sugerida

En este caso lo que se hizo fue describir una posible implementación práctica: un validador que reciba un AST simple donde `Assign` y `MatMulExpr` ya estén parseados, y aplique las reglas anteriores para devolver `{ errors, code, inferredType }`.

Acá lo que pasa es que puedo generar ese script si lo necesitás; basta con decirme y lo creo como archivo Python que use la tabla `env` y produzca el código pseudoc en C que mostramos.

---

### Referencia a materiales usados
Utilicé como apoyo las notas y ejemplos que tenés subidos en el repositorio de trabajo: `/mnt/data/09.pdf`.


**Fin del documento — punto2.md**

