## Introducción David Andres Castellanos Angulo
## Parcial tercer semestre Lenguajes de transduccion y programación
## 19 de noviembre de 2025
## Universidad Sergio Arboleda

En este caso, para el parcial, lo que hacemos primeroes desarrollar un conjunto de herramientas orientadas al
diseño e implementación de gramáticas para lnguajes formales con
capacidades específicas. En primer lugar, lo que se nos pide es que fue modelemos una
función capaz de generar una **gramática de atributos** para un lenguaje
de programación orientado a realizar operaciones tipo **SQL (CRUD)**. Esto en cuanto al primer punto.
Acá lo que hacemos es que definimos reglas sintácticas y atributos
semánticos que permiten validar estructuras como `SELECT`, `INSERT`,
`UPDATE` y `DELETE`, verificando tanto tipos como la consistencia del
esquema utilizado. Pues esto es lo que normalmente usariamso al estar programando
en lenguaje SQL.

Posteriormente, lo que se buscaba era diseñar una **gramática dedicada al cálculo del
producto punto** entre matrices de diferentes dimensiones. En este caso,
vemos que lo que hicimos fue establecer reglas que reconocen literales de
matrices, operaciones vectoriales y expresiones que reqieren comprobar
compatibilidad de dimensiones. Entoncess es poor esta razón que
la gramática incorpora atributos como `rows`, `cols` y `type`, que permiten
detectar errores y asegurar que la operación `dot(A, B)` o el
operador matricial `A @ B` se}
evalúe correctamente.

Finalmente, para este último punto, lo que se nos pedia es que ya usaramos
directamente, igual esto es como una continuación del punto anterior,
por lo que entonces debemos tomar en cuenta lo que hicimos y  entonces ya en
nuestro **ANTLR** lo que nos vamos a dar cuesnta es que es la gramática
correspondiente al punto anterior como lo dije anteriormente. Entonces basicamente
para este caso se puede notar que acá lo que hacemos es generar el
parser automáticamente y luego construir un **Visitor en Python** que
evalúa los atributos, realiza las comproabciones semánticas necesarias y
produce el resultado o los mensajes de error adecuados. Con esto se
completa un flujo que va desde el diseño formal de la gramática hasta su
ejecución práctica como mini-lenguaje especializado por decirlo asi.
