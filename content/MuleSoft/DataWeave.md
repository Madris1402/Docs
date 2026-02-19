---
tags:
  - MuleSoft
---
> [!abstract] Índice
> ```table-of-contents
> ```

---
### Su Función
DataWeave es un traductor de datos entre sistemas, toma los datos de entrada (***Input***), los reordena, les cambia el formato y entrega exactamente lo que el otro sistema necesita (***Output***).

Supongamos que una API manda un JSON, pero nuestro sistema de Bases de Datos solo entiende XML, Data Weave traducirá la estructura de JSON a XML.

### Scripts
En Anypoint Studio, existe un componente llamado **Transform Message**. Al abrirlo se encuentra un script de DataWeave tiene dos partes divididas por tres guiones `---`.

```DataWeave
%dw 2.0                 // 1. Versión del lenguaje
output application/json // 2. En qué formato quieres la SALIDA
---                     // 3. LA FRONTERA (Separa configuración de lógica)
{
  mensaje: "Hola Mundo" // 4. El cuerpo del script (La lógica)
}
```

#### Ejemplo:

Supongamos que necesitamos recuperar datos de Pokémones, la **PokéAPI** nos devuelve un JSON gigante y complejo (con objetos anidados, listas dentro de listas, URLs que no necesitamos).

Supongamos que tu jefe te dice: _"No quiero todo ese ruido. Solo quiero un JSON simple con el nombre en mayúsculas, su primer habilidad y su peso en kilos para nuestro sistema de inventario."_

```JSON
{
  "name": "pikachu",
  "weight": 60,
  "abilities": [
      {"ability": {"name": "static"}},
      {"ability": {"name": "lightning-rod"}}
  ]
}
```

Para transformarlo usaremos este código:


``` DataWeave
%dw 2.0
output application/json
---
{
  id_inventario: 12345,
  nombre_pokemon: upper(payload.name), 
  peso_kg: payload.weight / 10,
  habilidad_principal: payload.abilities[0].ability.name
}
```

Y su resultado:


``` JSON
{
  "id_inventario": 12345,
  "nombre_pokemon": "PIKACHU",
  "peso_kg": 6.0,
  "habilidad_principal": "static"
}
```

### Operadores básicos
En el ejemplo Anterior vimos estos 3 operadores básicos:

1. **Input (Lo que recibimos - `payload`):**
   **`payload`**: En MuleSoft, `payload` es la variable mágica que contiene "los datos que llegaron hasta aquí". Cuando escribimos `payload.name`, estamos navegando dentro del JSON original.
   
2. **Funciones**: Usamos `upper(...)` para convertir texto a mayúsculas. DataWeave tiene cientos de funciones listas (para fechas, matemáticas, textos, etc.).
   
3. **Navegación**: Vimos `payload.abilities[0].ability.name`. Esto es como seguir un mapa:
    
    - Entra a `abilities`.
    - Toma el primer elemento `[0]`.
    - Entra a `ability`.
    - Dame el `name`.

#### Operador Map
Lo que vimos arriba es fácil porque es 1 a 1. Pero, ¿qué pasa si recibes una **lista** de 50 Pokemones y quieres transformarlos _todos_ a la vez?

Aquí entra `map`. Es el comando más usado y suele confundir al principio.

Supongamos que `payload` es una lista de precios: `[10, 20, 30]`. Queremos sumarle $5 a cada uno.

```DataWeave
%dw 2.0
output application/json
---
payload map (precio, indice) -> precio + 5
```

**Resultado:** `[15, 25, 35]`

`map` recorre la lista elemento por elemento.

Volvamos al ejemplo de los Pokémones. Se tiene esta lista de nombres: **Input:** `["charmander", "squirtle", "bulbasaur"]`

Y usamaos este script de DataWeave:

``` DataWeave
%dw 2.0
output application/json
---
payload map (item, index) -> {
    id: index + 1,
    nombre: upper(item)
}
```

El Resultado será:

```JSON
[
  {"id": 1, "nombre": "CHARMANDER"},
  {"id": 2, "nombre": "SQUIRTLE"},
  {"id": 3, "nombre": "BULBASAUR"}
]
```

### Operador Filter
A veces no queremos _toda_ la lista. A veces hay "basura" o datos que no necesitamos.

Supongamos que en esa lista de pokemones (`["charmander", "squirtle", "bulbasaur"]`) solo queremos los que tengan nombres largos (más de 8 letras).

En DataWeave se ve así:

Fragmento de código

``` DataWeave
%dw 2.0
output application/json
---
payload filter (item, index) -> sizeOf(item) > 8
```

- **Charmander** (10 letras)
- **Squirtle** (8 letras)
- **Bulbasaur** (9 letras)

**Resultado:** `["charmander", "bulbasaur"]`