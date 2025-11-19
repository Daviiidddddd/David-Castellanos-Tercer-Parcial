# Repositorio: David Castellanos
Punto 1



---

## Punto De SQL (leer introduccion)
```markdown
# sql-attribute-grammar

Proyecto de ejemplo: Gramática de atributos para un mini-lenguaje SQL (CRUD) y utilidades para generar una representación programable de la gramática a partir de un *schema*.

## Contenido
- `src/generate_attribute_grammar.py` — funciónn que genera una representación (JSON) de la gramática de atributos a partir de un schema.
- `grammar/attribute_grammar.json` — ejemplo de salida 
- `examples/schema_example.json` — schema de ejemplo usado para generar la gramática.
- `examples/usage.py` — script de ejemplo que genera y guarda la gramática.




```bash
python3 examples/usage.py
```

Se hace el  `grammar/attribute_grammar.json` con la representación programable de la gramática basada en el `examples/schema_example.json`.


```

---

## src/generate_attribute_grammar.py
```python
"""generate_attribute_grammar.py
Genera una representación programable (JSON) de una gramática de atributos para un mini-SQL CRUD.
"""
import json
from typing import Dict, Any


def generate_attribute_grammar(schema: Dict[str, Dict[str, str]]) -> Dict[str, Any]:
    """
    Genera una estructura de gramática de atributos para un mini-SQL CRUD.

    schema: {
       "table_name": {"col1":"type1", "col2":"type2", ...},
       ...
    }

    Devuelve un diccionario listo para serializar a JSON.
    """
    G: Dict[str, Any] = {}
    G["schema"] = schema

    G["productions"] = {
        "Program -> StmtList": {
            "attrs": {"env(H)": "schema", "errors(S)": "list", "code(S)": "string"},
            "rules": [
                "Program.env = schema",
                "Program.errors = StmtList.errors",
                "Program.code = StmtList.code",
            ],
        },
        "StmtList -> Stmt StmtList": {
            "attrs": {"env(H)": "env", "errors(S)": "list", "code(S)": "string"},
            "rules": [
                "Stmt.env = StmtList.env",
                "StmtList1.env = StmtList.env",
                "StmtList.errors = concat(Stmt.errors, StmtList1.errors)",
                "StmtList.code = concat(Stmt.code, '\\n', StmtList1.code)",
            ],
        },
        "SelectStmt -> SELECT SelectList FROM TableRef WhereOpt": {
            "attrs": {"env(H)": "env", "errors(S)": "list", "code(S)": "string"},
            "rules": [
                "SelectList.env = env",
                "TableRef.env = env",
                "WhereOpt.env = env",
                "if any table in TableRef not in env: add error",
                "resolve columns in SelectList using TableRef.tables",
                "if WhereOpt present and WhereOpt.type != 'bool' -> add error",
                "SelectStmt.code = 'SELECT ' + SelectList.code + ' FROM ' + TableRef.code + (optional WHERE)",
            ],
        },
        "InsertStmt -> INSERT INTO Table OptColList VALUES ValueList": {
            "attrs": {"env(H)": "env", "errors(S)": "list", "code(S)": "string"},
            "rules": [
                "if Table not in env -> add error",
                "if OptColList present -> targetCols = OptColList.cols else targetCols = all columns of Table",
                "if len(targetCols) != len(ValueList.exprs) -> add error",
                "for each i: check compatible(ValueList.exprs[i].type, columnType)",
                "InsertStmt.code = canonical INSERT SQL string",
            ],
        },
    }

    G["utilities"] = {
        "lookupTable": "lookup table in schema dict",
        "lookupColumn": "lookup column type in schema",
        "compatible": "function that decides type compatibility",
        "numeric_promote": "promote numeric types",
    }

    G["expr_rules"] = {
        "Expr -> Expr + Expr": [
            "Expr.type = numeric_promote(Expr1.type, Expr2.type) or 'error'",
            "Expr.code = '(' + Expr1.code + ' + ' + Expr2.code + ')'",
        ],
        "Column -> id": [
            "search id in tables_in_scope from TableRef; if none -> error; if multiple -> ambiguity error; else set Column.type and Column.code"
        ],
    }

    G["notes"] = [
        "Los atributos env deben propagarse desde Program a cada sentencia.",
        "Errors se concatenan bottom-up; code se sintetiza.",
        "Para soportar alias y join, TableRef debe resolver alias -> table mapping.",
    ]

    return G


if __name__ == '__main__':
    # Ejemplo de schema
    schema_example = {
        "users": {"id": "int", "name": "string", "age": "int"},
        "orders": {"id": "int", "user_id": "int", "total": "float"},
    }

    grammar = generate_attribute_grammar(schema_example)

    # Guardar a archivo JSON
    import os
    os.makedirs("grammar", exist_ok=True)
    with open("grammar/attribute_grammar.json", "w", encoding="utf-8") as f:
        json.dump(grammar, f, indent=2, ensure_ascii=False)

    print("Se generó grammar/attribute_grammar.json")
```

---

## gramatica
```json
{
  "schema": {
    "users": {
      "id": "int",
      "name": "string",
      "age": "int"
    },
    "orders": {
      "id": "int",
      "user_id": "int",
      "total": "float"
    }
  },
  "productions": {
    "Program -> StmtList": {
      "attrs": {
        "env(H)": "schema",
        "errors(S)": "list",
        "code(S)": "string"
      },
      "rules": [
        "Program.env = schema",
        "Program.errors = StmtList.errors",
        "Program.code = StmtList.code"
      ]
    },
    "StmtList -> Stmt StmtList": {
      "attrs": {
        "env(H)": "env",
        "errors(S)": "list",
        "code(S)": "string"
      },
      "rules": [
        "Stmt.env = StmtList.env",
        "StmtList1.env = StmtList.env",
        "StmtList.errors = concat(Stmt.errors, StmtList1.errors)",
        "StmtList.code = concat(Stmt.code, '\n', StmtList1.code)"
      ]
    },
    "SelectStmt -> SELECT SelectList FROM TableRef WhereOpt": {
      "attrs": {
        "env(H)": "env",
        "errors(S)": "list",
        "code(S)": "string"
      },
      "rules": [
        "SelectList.env = env",
        "TableRef.env = env",
        "WhereOpt.env = env",
        "if any table in TableRef not in env: add error",
        "resolve columns in SelectList using TableRef.tables",
        "if WhereOpt present and WhereOpt.type != 'bool' -> add error",
        "SelectStmt.code = 'SELECT ' + SelectList.code + ' FROM ' + TableRef.code + (optional WHERE)"
      ]
    },
    "InsertStmt -> INSERT INTO Table OptColList VALUES ValueList": {
      "attrs": {
        "env(H)": "env",
        "errors(S)": "list",
        "code(S)": "string"
      },
      "rules": [
        "if Table not in env -> add error",
        "if OptColList present -> targetCols = OptColList.cols else targetCols = all columns of Table",
        "if len(targetCols) != len(ValueList.exprs) -> add error",
        "for each i: check compatible(ValueList.exprs[i].type, columnType)",
        "InsertStmt.code = canonical INSERT SQL string"
      ]
    }
  }
}
```

---

## examples/schema_example.json
```json
{
  "users": {"id":"int","name":"string","age":"int"},
  "orders": {"id":"int","user_id":"int","total":"float"}
}
```

---

## el ejemplo
```python
"""Ejemplo de uso: genera el JSON de la gramática usando el schema de examples/schema_example.json"""
import json
from pathlib import Path
from src.generate_attribute_grammar import generate_attribute_grammar

p = Path(__file__).parent
schema_path = p / "schema_example.json"
with open(schema_path, "r", encoding="utf-8") as f:
    schema = json.load(f)

G = generate_attribute_grammar(schema)

out_dir = Path("grammar")
out_dir.mkdir(parents=True, exist_ok=True)
with open(out_dir / "attribute_grammar.json", "w", encoding="utf-8") as f:
    json.dump(G, f, indent=2, ensure_ascii=False)

print("Grammar generated: grammar/attribute_grammar.json")
```

---

## .gitignore
```text
__pycache__/
*.pyc
grammar/attribute_grammar.json
```

---



### Referencias / PDFs de trabajo
- `/mnt/data/09.pdf`
- `/mnt/data/08 (1).pdf`

---

**Fin del paquete de archivos.**

