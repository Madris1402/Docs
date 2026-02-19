---
tags:
  - API
---
> [!abstract] Índice
> ```table-of-contents
> 
> ```

---
Es un framework moderno para construir [[APIs]] con Python basado en ***Type Hints*** estándar de Python. Haciéndolo veloz, compatible con los estándares ***OpenAPI*** (Swagger) y JSON Schema.

Utiliza tipado estático para validar los datos automáticamente reduciendo los errores en tiempo de ejecución.

### Parámetros y Tipos de Datos
En FastAPI contamos con ***Path Parameters*** y ***Query Parameters***
#### Path Parameters

```python
from fastapi import FastAPI

app = FastAPI()

# Definimos una ruta que espera un parámetro 'item_id'
@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id, "tipo_de_dato": str(type(item_id))}
```

Contamos con un decorador `@app.get` en el que se declaran variables para la URL con llaves `{}`.
Mientras que en la función `read_item` se usan ***Type Hints*** `: int` para definir el tipo de dato que esperamos.

FastAPI tiene la capacidad de convertir una petición GET a la URL `/items/10` de cadena a entero automáticamente. Si llegase a recibir una cadena `/items/diez` cuando espera un entero, este regresaría un error 422 informando que los datos ingresados no son válidos en lugar de un error 500 o 404.

El código 422 (Unprocessable Entity) significa "Entendí la petición, pero el formato de los datos es incorrecto", y la respuesta es un JSON:

```JSON
{
    "detail": [
        {
            "loc": ["path", "item_id"],
            "msg": "value is not a valid integer",
            "type": "type_error.integer"
        }
    ]
}
```

#### Query Parameters
Suelen usarse para filtrar, ordenar o paginar los resultados. Son valores que se encuentran al final de la URL después de un signo de interrogación `?`.

En FastAPI <mark style="background:rgba(136, 49, 204, 0.2)">si declaras un parámetro que No esté incluido en la ruta del decorador, se asume que es un Query Parameter</mark>

```python
fake_items_db = [{"item_name": "Foo"}, {"item_name": "Bar"}, {"item_name": "Baz"}]

@app.get("/items/")
def read_item(skip: int = 0, limit: int = 10):
    return fake_items_db[skip : skip + limit]
```

En este ejemplo `skip` y `limit` son parámetros de query y como se les asigno un valor, son opcionales. Para llamarlos en la URL se vería así: `/items/?skip=x&limit=y` donde `x`, `y` son enteros que el usuario defina.

### Request Body
Este se utiliza para enviar datos complejos, en este caso un objeto JSON para crear un registro.

Para ello se necesita la librería ***Pydantic*** y esta es la forma en la que se estructura:

```python
from typing import Optional
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: float = 10.5
```

### Documentación Automática

FastAPI genera automáticamente una página interactiva en la que se puede probar la API. Esta ocupa el estándar ***OpenAPI***. Esta página se almacena en la rura `.../docs` y:

1. Lista todas las rutas (`GET`, `PUT`, etc.).
2. Muestra los parámetros que requiere cada una e indica los que son obligatorios.
3. Permite ejecutar las peticiones.

Para que esto funcione necesitamos un servidor ASGI.

#### Uvicorn
Este es el servidor estándar para FastAPI, es un motor que se queda escuchando en un puerto (8000 por defecto) y pasa las peticiones web al código para que las procese.

El comando típico es `uvicorn {nombre del archivo py}:{nombre de la variable que instancia FastAPI} --reload` suponiendo que el archivo se llama `main.py` y la variable es `app = FastAPI()` la llamada a uvicorn será:

```bash
uvicorn main:app --reload
```

`--reload`: hace que el servidor se reinicie solo cada vez que se guardan cambios en el código.

### Inyección de Dependencias
Las dependencias son una función que tiene la tarea de preparar algo (datos, conexiones o validaciones) antes de que se ejecute la ruta principal.

Supongamos que tenemos rutas diferentes que utilizan los mismos parámetros:

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# 1. Definimos la dependencia (es solo una función)
def paginacion_comun(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

# 2. Inyectamos la dependencia en la ruta
@app.get("/items/")
def read_items(params: dict = Depends(paginacion_comun)):
    return params

@app.get("/users/")
def read_users(params: dict = Depends(paginacion_comun)):
    return params
```

En el ejemplo:

1. **`Depends(paginacion_comun)`**: Cuando llamas a `/items/?skip=20`, FastAPI ve `Depends`.
   
2. **Ejecución**: FastAPI pone "en pausa" tu función `read_items`, va y ejecuta la función `paginacion_comun`.
   
3. **Extracción**: La función `paginacion_comun` extrae los query params (`skip`, `limit`) de la URL.
   
4. **Entrega**: El resultado (el diccionario) se entrega en la variable `params` de tu ruta principal.

#### Dependencias con `yield`
Utilizamos los generadores `yield` que nos permiten ejecutar código antes y después de una petición en la misma función.

```python
# Simulación de conexión a BD
def get_db():
    db = "Conexión a la BD creada" # 1. Setup (antes de la ruta)
    print("Abriendo base de datos...")
    try:
        yield db  # 2. Inyectamos la conexión a la ruta
    finally:
        # 3. Teardown (después de que la ruta respondió)
        print("Cerrando base de datos...")
        db = "Conexión cerrada"

@app.get("/items/")
def read_items(db = Depends(get_db)):
    # Aquí usamos la BD
    return {"mensaje": "Usando la BD", "estado_db": db}
```

**La secuencia es:**

1. Llega la petición.
   
2. Se ejecuta `get_db` hasta el `yield`.
   
3. Se abre la conexión.
   
4. Se ejecuta tu ruta `read_items`.
   
5. Tu ruta responde al cliente (envía el JSON).
   
6. **Después de responder**, FastAPI vuelve a `get_db` y ejecuta lo que está en el `finally` (cerrar la conexión).

Esto garantiza que siempre se cierren los recursos incluso si ocurren errores en el código.