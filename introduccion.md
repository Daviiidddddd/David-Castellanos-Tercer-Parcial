## Introducción

En este proyecto se desarrolla un conjunto de herramientas orientadas al
diseño e implementación de gramáticas para lenguajes formales con
capacidades específicas. En primer lugar, lo que se hizo fue modelar una
función capaz de generar una **gramática de atributos** para un lenguaje
de programación orientado a realizar operaciones tipo **SQL (CRUD)**.
Acá lo que hacemos es que definimos reglas sintácticas y atributos
semánticos que permiten validar estructuras como `SELECT`, `INSERT`,
`UPDATE` y `DELETE`, verificando tanto tipos como la consistencia del
esquema utilizado.

Posteriormente, se diseñó una **gramática dedicada al cálculo del
producto punto** entre matrices de diferentes dimensiones. En este caso,
lo que se hizo fue establecer reglas que reconocen literales de
matrices, operaciones vectoriales y expresiones que requieren comprobar
compatibilidad de dimensiones. Por esta razón, la gramática incorpora
atributos como `rows`, `cols` y `type`, que permiten detectar errores y
asegurar que la operación `dot(A, B)` o el operador matricial `A @ B` se
evalúe correctamente.

Finalmente, se implementó en **ANTLR (objetivo Python)** la gramática
correspondiente al punto anterior. Acá lo que hacemos es generar el
parser automáticamente y luego construir un **Visitor en Python** que
evalúa los atributos, realiza las comprobaciones semánticas necesarias y
produce el resultado o los mensajes de error adecuados. Con esto se
completa un flujo que va desde el diseño formal de la gramática hasta su
ejecución práctica como mini-lenguaje especializado.
