# Punto 3 — Implementación en ANTLR 

En este caso lo que se hizo fue implementar la gramatica del Punto 2 en ANTLR y proveer ejemplos de cómo generar el parser y un *visitor* en Python que haga las comprobaciones semánticas proincipales (compatibilidad de dimensiones) y genere código intermedio   



---

## 1) Archivo de la gramática: `Matrix.g4`

```antlr
grammar Matrix;

program     : declList stmtList EOF ;

declList    : (decl)* ;

decl        : ID ':' matrixType ';' ;

matrixType  : MATRIX '<' baseType ',' dim ',' dim '>' ;

dim         : INT_LITERAL | ID ;

baseType    : INTTYPE | FLOATTYPE | DOUBLETYPE ;

stmtList    : (stmt)* ;

stmt        : assign ';' ;

assign      : ID '=' expr ;

expr        : matMulExpr
            | primaryExpr
            ;

matMulExpr  : primaryExpr '*' primaryExpr ;

primaryExpr : ID
            | '(' expr ')'
            | matLiteral
            ;

matLiteral  : '[' rowList ']' ;

rowList     : row (',' row)* ;

row         : '[' exprList ']' ;

exprList    : expr (',' expr)* ;

// ------------------
// Lexer tokens
MATRIX      : 'matrix' ;
INTTYPE     : 'int' ;
FLOATTYPE   : 'float' ;
DOUBLETYPE  : 'double' ;

INT_LITERAL : [0-9]+ ;
ID          : [a-zA-Z_][a-zA-Z_0-9]* ;

WS          : [ \t\r\n]+ -> skip ;
COMMENT     : '//' ~[\r\n]* -> skip ;

// Symbols (these are single chars, no token names needed but listed for clarity)
// '<' '>' ',' ':' '=' ';' '[' ']' '(' ')' '*'
```

En este caso lo que se hizo fue definir reglas para declaraciones, expresiones y literales de matrices. Acá lo que pasa es que las palabras reservadas (`matrix`, `int`, `float`, `double`) están definidas como tokens específicos para evitar confusiones con identificadores. Que es algo que me ha pasado antes

---

## 2) Còmo generar el parser para Python?

Entonces podemos ver los pasos, siempre y cuando tengamos el antl `antlr4` en el path:

1. Instalamos el runtime de pyhotn

```bash
pip install antlr4-python3-runtime
```

2. Generar el parser y lexer para Python3:

```bash
antlr4 -Dlanguage=Python3 Matrix.g4 -visitor -no-listener
```

Esto generará archivos como `MatrixParser.py`, `MatrixLexer.py`, `MatrixVisitor.py`, etc.

---

## 3) Visitor en Python — validaciones y generación de código

Acá lo que pasa es que te dejo un ejemplo simple de `MatrixVisitor` que implementa:

- Tabla de símbolos (`env`) con filas/cols/tipo y si las dims son conocidas.
- Comprobación en `A * B` de que `A.cols == B.rows` si ambas son constantes; si no lo son, se genera un chequeo runtime en el código.



```python
# matrix_checker.py
from MatrixParser import MatrixParser
from MatrixVisitor import MatrixVisitor

class SymbolInfo:
    def __init__(self, rows, cols, elem_type, dims_known):
        self.rows = rows
        self.cols = cols
        self.elem_type = elem_type
        self.dims_known = dims_known

class CheckerVisitor(MatrixVisitor):
    def __init__(self):
        self.env = {}  # name -> SymbolInfo
        self.errors = []

    # helper
    def is_int_literal(self, text):
        try:
            int(text)
            return True
        except:
            return False

    # decl: ID ':' matrixType ';'
    def visitDecl(self, ctx:MatrixParser.DeclContext):
        name = ctx.ID().getText()
        mt = ctx.matrixType()
        base = mt.baseType().getText()
        d1 = mt.dim(0).getText()
        d2 = mt.dim(1).getText()
        if self.is_int_literal(d1):
            rows = int(d1)
            d1_known = True
        else:
            rows = d1
            d1_known = False
        if self.is_int_literal(d2):
            cols = int(d2)
            d2_known = True
        else:
            cols = d2
            d2_known = False
        self.env[name] = SymbolInfo(rows, cols, base, d1_known and d2_known)
        return None

    # assign: ID '=' expr
    def visitAssign(self, ctx:MatrixParser.AssignContext):
        target = ctx.ID().getText()
        expr_node = ctx.expr()
        res = self.visit(expr_node)
        # res expected: dict with rows, cols, elem_type, dims_known, code, errors
        if target not in self.env:
            self.errors.append(f"Variable '{target}' no declarada")
        else:
            # check compatibility with declared target if dims known
            sym = self.env[target]
            if res is None:
                return None
            if res.get('errors'):
                self.errors.extend(res['errors'])
            if sym.dims_known and res['dims_known']:
                if sym.rows != res['rows'] or sym.cols != res['cols']:
                    self.errors.append(f"Dimensiones del LHS '{target}' no coinciden con resultado: esperado {sym.rows}x{sym.cols} got {res['rows']}x{res['cols']}")
            # Here we could store generated code or reference
            print('\n// Codigo generado para asignacion a', target)
            print(res.get('code',''))
        return None

    # primaryExpr: ID | '(' expr ')' | matLiteral
    def visitPrimaryExpr(self, ctx:MatrixParser.PrimaryExprContext):
        if ctx.ID():
            name = ctx.ID().getText()
            if name not in self.env:
                return {'errors':[f"Variable '{name}' no declarada"]}
            sym = self.env[name]
            return {'rows': sym.rows, 'cols': sym.cols, 'elem_type': sym.elem_type, 'dims_known': sym.dims_known, 'code': name, 'errors': []}
        elif ctx.matLiteral():
            # For simplicity, not fully implemented. You could parse literal sizes
            lit = self.visit(ctx.matLiteral())
            return lit
        else:
            return self.visit(ctx.expr())

    # matMulExpr: primaryExpr '*' primaryExpr
    def visitMatMulExpr(self, ctx:MatrixParser.MatMulExprContext):
        left = self.visit(ctx.primaryExpr(0))
        right = self.visit(ctx.primaryExpr(1))
        errors = []
        if left is None or right is None:
            return {'errors':['subexpr null']}
        # collect lower-level errors
        if left.get('errors'): errors.extend(left['errors'])
        if right.get('errors'): errors.extend(right['errors'])

        # try to get dims
        lcols = left.get('cols')
        rrows = right.get('rows')
        rows = left.get('rows')
        cols = right.get('cols')
        dims_known = False
        runtime_check = ''
        if isinstance(lcols, int) and isinstance(rrows, int):
            if lcols != rrows:
                errors.append(f"Incompatibilidad de dimensiones: {lcols} != {rrows}")
            dims_known = left.get('dims_known') and right.get('dims_known')
        else:
            # generamos chequeo en runtime
            runtime_check = f"if ({lcols} != {rrows}) runtime_error('Incompatibilidad de dimensiones');\n"
            dims_known = False

        # tipo elem
        etype = left.get('elem_type')
        if left.get('elem_type') == 'float' or right.get('elem_type') == 'float':
            etype = 'float'

        # generar codigo
        code = ''
        code += runtime_check
        code += f"// alloc result C [{rows}][{cols}]\n"
        if dims_known:
            code += f"for (int i=0;i<{rows};++i) {{\n"
            code += f"  for (int j=0;j<{cols};++j) {{\n"
            code += f"    C[i][j] = 0;\n"
            code += f"    for (int k=0;k<{lcols};++k) {{\n"
            code += f"      C[i][j] += {left['code']}[i][k] * {right['code']}[k][j];\n"
            code += f"    }}\n  }}\n}}\n"
        else:
            code += f"for (int i=0;i<{rows};++i) {{\n  for (int j=0;j<{cols};++j) {{\n    C[i][j]=0;\n    for (int k=0;k<{lcols};++k) {{\n      C[i][j] += {left['code']}[i][k] * {right['code']}[k][j];\n    }}\n  }}\n}}\n"

        return {'rows': rows, 'cols': cols, 'elem_type': etype, 'dims_known': dims_known, 'code': code, 'errors': errors}

    # matLiteral and others could be implemented similarly

```


---

## 4) Script de ejemplo para ejecutar (main)

Archivo: `run_checker.py`

```python
from antlr4 import FileStream, CommonTokenStream
from MatrixLexer import MatrixLexer
from MatrixParser import MatrixParser
from matrix_checker import CheckerVisitor

input_file = 'ejemplo.txt'  # contiene el programa a analizar

stream = FileStream(input_file, encoding='utf-8')
lexer = MatrixLexer(stream)
tokens = CommonTokenStream(lexer)
parser = MatrixParser(tokens)
tree = parser.program()

visitor = CheckerVisitor()
visitor.visit(tree)

if visitor.errors:
    print('Errores semanticos:')
    for e in visitor.errors:
        print(' -', e)
else:
    print('Analisis completado sin errores')
```

Y un `ejemplo.txt` mínimo:

```
A : matrix<int, 2, 3>;
B : matrix<int, 3, 4>;
C : matrix<int, 2, 4>;
C = A * B;
```

Entonces podemos ver que al ejecutar `run_checker.py` se imprimirá el código generado para la asignación y no habrá errores.

---

## 5) Notas y limitaciones (hablando claro)

- En este caso lo que se hizo fue una versión minimalista. No implementé parsing real de literales de matriz, ni evaluación de expresiones internas (solo se asume IDs y multiplicacion directa).  
- Acá lo que pasa es que los tipos y las dimensiones simbólicas se tratan como strings; para una implementación robusta convendría normalizarlos y manejar expresiones aritméticas en dims.  
- Entonces podemos ver que el visitor debe extenderse para manejar matrices literales, casts de tipo, y mejores mensajes de error con linea/columna.

---

## 6) Resumen final

En este caso lo que se hizo fue:

- Definir la gramática `Matrix.g4` en ANTLR para las construcciones necesarias (decl, asign, matmul, literales).
- Dar instrucciones para generar el parser en Python y el runtime a usar.
- Proveer un `CheckerVisitor` en Python que aplica las reglas semánticas del Punto 2 (compatibilidad de dimensiones, generación de pseudocodigo) y un `run_checker.py` para ejecutar.

Acá lo que pasa es que con esto tenes una base para extender y transformar a una herramienta real; la idea era que sea clara y facil de seguir, con lenguaje simple y ejemplos.

---

