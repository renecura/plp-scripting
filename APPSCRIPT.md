# Apunte: Google Apps Script y conexión con una API (PokeAPI)

*Material de apoyo para la clase de Scripting*

## 1. ¿Qué es Google Apps Script?

**Google Apps Script** es un lenguaje de scripting basado en JavaScript que se ejecuta en los servidores de Google y sirve para **automatizar y extender los productos de Google Workspace**: Hojas de cálculo, Documentos, Gmail, Calendar, Drive y otros. Es un ejemplo del segundo tipo de uso que vimos en el apunte anterior: un script que vive *dentro* de otra aplicación y accede a sus objetos.

Además, desde Apps Script se puede hacer peticiones HTTP a servicios externos. Eso lo convierte también en un lenguaje "pegamento" (*glue*): conecta la planilla con el resto de internet.

Tres características útiles para empezar:

- **No requiere instalar nada.** Se programa desde el navegador, dentro de la propia planilla.
- **Está integrado con la planilla.** Un script "atado" a una Hoja de cálculo accede a ella mediante el servicio `SpreadsheetApp`.
- **Corre en la nube.** El código se ejecuta en servidores de Google, con cuotas y límites de tiempo que conviene conocer (sección 8).

## 2. Objetivo del ejercicio

Vamos a conectar una planilla con **PokeAPI**, una API pública y gratuita con datos de Pokémon. El flujo será:

```
  Planilla  ──número──▶  Apps Script  ──HTTP GET──▶  PokeAPI
  Planilla  ◀──datos───  Apps Script  ◀──JSON──────  PokeAPI
```

Construiremos dos tipos de funciones, que son las dos formas básicas de usar Apps Script en una planilla:

|  | Función personalizada | Función con menú |
| --- | --- | --- |
| Cómo se usa | Se escribe en una celda: `=POKEMON_NOMBRE(25)` | Se ejecuta desde un menú o el editor |
| Qué hace | **Devuelve** un valor (o una fila) a la celda donde está | **Escribe** en cualquier celda de la planilla |
| Limitación | No puede modificar otras celdas por su cuenta | Hay que ejecutarla a mano |

El resultado final: una planilla donde se escribe un número de Pokédex y se obtienen su nombre, altura, peso, tipos y estadísticas.

## 3. Preparación

1. Crear una **Hoja de cálculo de Google** nueva.
2. En la fila 1, escribir los encabezados: `Número`, `Nombre`, `Altura (m)`, `Peso (kg)`, `Tipos`, `HP`, `Ataque`, `Defensa`, `Velocidad`.
3. En la columna A, desde la fila 2, escribir algunos números: `1`, `4`, `7`, `25`, `150`.
4. Abrir el editor: menú **Extensiones → Apps Script**. Se abre un proyecto con un archivo `Código.gs` (o `Code.gs`).
5. Borrar el contenido de ejemplo (`function myFunction() {}`) y ponerle un nombre al proyecto.

> **Autorización.** La primera vez que un script hace una petición externa o toca la planilla, Google pide autorizar el acceso. Hay que ejecutar una función desde el editor (botón **Ejecutar**) y aceptar los permisos. Si una fórmula en la celda muestra un error de permisos, es lo primero que se prueba.

## 4. La API: PokeAPI

PokeAPI no requiere cuenta ni clave. Para pedir un Pokémon se hace una petición `GET` a:

```
https://pokeapi.co/api/v2/pokemon/{número o nombre}
```

Por ejemplo, `https://pokeapi.co/api/v2/pokemon/25` devuelve a Pikachu. Se puede probar pegando esa dirección en el navegador. La respuesta es un texto en formato **JSON**, del que resumimos solo lo que vamos a usar:

```json
{
  "id": 25,
  "name": "pikachu",
  "height": 4,
  "weight": 60,
  "types": [
    { "slot": 1, "type": { "name": "electric" } }
  ],
  "stats": [
    { "base_stat": 35, "stat": { "name": "hp" } },
    { "base_stat": 55, "stat": { "name": "attack" } },
    { "base_stat": 40, "stat": { "name": "defense" } },
    { "base_stat": 90, "stat": { "name": "speed" } }
  ]
}
```

La respuesta real es mucho más larga: incluye cientos de movimientos, imágenes y otros datos que no necesitamos.

Lo que hay que observar para poder leerla:

- Hay valores simples (`id`, `name`, `height`, `weight`).
- Hay **listas** (`types`, `stats`), donde cada elemento es a su vez un objeto.
- Hay **objetos anidados**: el nombre del tipo está en `types[0].type.name`, no directamente en `types[0]`.
- Las unidades no son las habituales: `height` está en **decímetros** y `weight` en **hectogramos**. Para llegar a metros y kilogramos hay que dividir por 10 (Pikachu: 4 → 0,4 m y 60 → 6 kg).

| Dato que queremos | Dónde está en el JSON | Columna |
| --- | --- | --- |
| Nombre | `name` | B |
| Altura (m) | `height / 10` | C |
| Peso (kg) | `weight / 10` | D |
| Tipos | `types[].type.name` | E |
| HP | `stats[]` con `stat.name == "hp"` | F |
| Ataque | `stats[]` con `stat.name == "attack"` | G |
| Defensa | `stats[]` con `stat.name == "defense"` | H |
| Velocidad | `stats[]` con `stat.name == "speed"` | I |

> **Uso responsable.** PokeAPI es un servicio gratuito sostenido por la comunidad, y su política de uso pide no abusar: conviene guardar localmente las respuestas ya obtenidas en lugar de pedirlas una y otra vez. Más adelante lo hacemos con una caché.

## 5. Paso 1: una primera función personalizada

Empezamos por lo más simple: una función que reciba un número y devuelva el nombre del Pokémon. Pegar en el editor:

```javascript
/**
 * Devuelve el nombre de un Pokémon según su número en la Pokédex.
 * @param {number} numero Número del Pokémon (1, 25, 150...).
 * @return {string} Nombre del Pokémon.
 * @customfunction
 */
function POKEMON_NOMBRE(numero) {
  const url = 'https://pokeapi.co/api/v2/pokemon/' + numero;
  const respuesta = UrlFetchApp.fetch(url);
  const texto = respuesta.getContentText();
  const pokemon = JSON.parse(texto);
  return pokemon.name;
}
```

Guardar el proyecto (Ctrl+S) y volver a la planilla. En la celda `K2` escribir:

```
=POKEMON_NOMBRE(25)
```

Después de un instante, la celda muestra `pikachu`. También funciona con una referencia: `=POKEMON_NOMBRE(A2)`.

Qué hace cada parte:

1. El comentario `/** ... */` es un bloque de documentación (JSDoc). La etiqueta `@customfunction` hace que la función aparezca con ayuda y autocompletado al escribirla en una celda.
2. El nombre en **MAYÚSCULAS** es una convención: así se distingue de las funciones propias de la planilla (`SUM`, `VLOOKUP`).
3. `UrlFetchApp.fetch(url)` hace la petición HTTP `GET` y devuelve un objeto de respuesta.
4. `getContentText()` extrae el cuerpo de la respuesta como **texto**.
5. `JSON.parse(texto)` convierte ese texto en un **objeto JavaScript**, y a partir de ahí se accede a los campos con la notación de punto: `pokemon.name`.
6. El valor de `return` es lo que aparece en la celda.

Para depurar sin pasar por la planilla, se puede agregar una función de prueba y ejecutarla desde el editor:

```javascript
function probar() {
  Logger.log(POKEMON_NOMBRE(25));
}
```

El resultado aparece en el panel **Registro de ejecución** al pie del editor.

## 6. Paso 2: parsear más datos y devolver una fila

Pedir cada dato por separado (`POKEMON_NOMBRE`, `POKEMON_PESO`...) haría una petición HTTP por función. Es más eficiente **pedir el Pokémon una sola vez** y quedarse con los campos que necesitamos. Separamos entonces el trabajo en dos funciones:

- `obtenerPokemon_(id)`: llama a la API, parsea el JSON y devuelve un objeto **simplificado**.
- `POKEMON_DATOS(numero)`: la función de la planilla, que usa la anterior y devuelve una fila.

```javascript
const API_URL = 'https://pokeapi.co/api/v2/pokemon/';

// Función auxiliar. El guion bajo final la oculta de la planilla.
function obtenerPokemon_(id) {
  const respuesta = UrlFetchApp.fetch(API_URL + id, {
    muteHttpExceptions: true
  });

  if (respuesta.getResponseCode() !== 200) {
    throw new Error('No se encontró el Pokémon: ' + id);
  }

  const p = JSON.parse(respuesta.getContentText());

  const pokemon = {
    id: p.id,
    nombre: p.name,
    altura: p.height / 10,
    peso: p.weight / 10,
    tipos: p.types.map(t => t.type.name),
    stats: {}
  };
  p.stats.forEach(s => {
    pokemon.stats[s.stat.name] = s.base_stat;
  });

  return pokemon;
}

/**
 * Devuelve los datos principales de un Pokémon en una fila.
 * @param {number} numero Número del Pokémon.
 * @return {Array} Nombre, altura, peso, tipos, HP, ataque, defensa, velocidad.
 * @customfunction
 */
function POKEMON_DATOS(numero) {
  const p = obtenerPokemon_(numero);
  return [[
    p.nombre,
    p.altura,
    p.peso,
    p.tipos.join(', '),
    p.stats.hp,
    p.stats.attack,
    p.stats.defense,
    p.stats.speed
  ]];
}
```

En la celda `B2` escribir `=POKEMON_DATOS(A2)`. Una sola fórmula completa **ocho celdas hacia la derecha**.

Puntos para comentar:

- **Arreglo de arreglos.** La planilla es una grilla de filas y columnas, por eso la función devuelve un arreglo *bidimensional*: `[[a, b, c]]` es **una fila** con tres columnas, mientras que `[[a], [b], [c]]` serían tres filas con una columna.
- **Las celdas de destino deben estar vacías.** Si alguna a la derecha tiene contenido, la fórmula da el error `#REF!`.
- **`.map()`** transforma cada elemento de una lista: de la lista de objetos `types` obtenemos la lista de nombres. Con `.join(', ')` se la une en un solo texto (`grass, poison`).
- **`.forEach()`** recorre la lista `stats` y arma un objeto donde se accede por nombre: `pokemon.stats.hp`. Es más legible que recordar en qué posición de la lista está cada estadística.
- **`muteHttpExceptions: true`** evita que `fetch` lance una excepción ante un error HTTP. Así podemos revisar el código de respuesta y dar un mensaje propio: PokeAPI responde `404` si el número no existe.
- **`throw new Error(...)`** hace que la celda muestre el mensaje de error.
- Esta función **transforma** el JSON de la API en el formato que necesita la planilla. Es el mismo papel de "adaptador" que cumplían `griton` y `timido` en el apunte de pipes.

## 7. Paso 3: escribir en la planilla desde un menú

Una función personalizada solo puede **devolver** valores a su propia celda. Para escribir en otras partes de la planilla se usa una función común, que podemos lanzar desde un **menú propio**. Esta función completa de una vez todas las filas que tengan un número en la columna A:

```javascript
function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('Pokémon')
    .addItem('Completar datos', 'completarPlanilla')
    .addToUi();
}

function completarPlanilla() {
  const hoja = SpreadsheetApp.getActiveSheet();
  const ultimaFila = hoja.getLastRow();
  if (ultimaFila < 2) return;

  // Lee los números de la columna A (desde la fila 2)
  const numeros = hoja.getRange(2, 1, ultimaFila - 1, 1).getValues();

  const filas = numeros.map(([numero]) => {
    if (!numero) return ['', '', '', '', '', '', '', ''];
    const p = obtenerPokemon_(numero);
    return [
      p.nombre, p.altura, p.peso, p.tipos.join(', '),
      p.stats.hp, p.stats.attack, p.stats.defense, p.stats.speed
    ];
  });

  // Escribe todo de una sola vez en B2:I{última fila}
  hoja.getRange(2, 2, filas.length, 8).setValues(filas);
}
```

Para probarlo: borrar las celdas `B2:I6` con la fórmula anterior, recargar la planilla (así se ejecuta `onOpen`) y usar el menú **Pokémon → Completar datos**.

Lo nuevo en este paso:

- **`onOpen()`** es un *trigger simple*: un nombre reservado que Apps Script ejecuta automáticamente cada vez que se abre la planilla. Aquí lo usamos para crear el menú.
- **`SpreadsheetApp`** es el servicio que representa la planilla. La cadena de llamadas `getActiveSheet()` → `getRange()` → `getValues()` baja de la planilla a la hoja y de la hoja al rango.
- **`getRange(fila, columna, cantidadFilas, cantidadColumnas)`** selecciona un rango. Las posiciones **empiezan en 1**, no en 0.
- **`getValues()`** devuelve un arreglo bidimensional; por eso el `map` recibe cada fila como `[numero]`. Ese `([numero])` es *desestructuración*: toma el primer elemento de la fila.
- **`setValues(filas)`** escribe todo el bloque en una única operación. Hacerlo celda por celda (`setValue` dentro de un ciclo) es mucho más lento, porque cada llamada es un viaje al servidor de la planilla.

Con este paso la planilla pasa a depender del botón, pero ya no de las fórmulas. Se puede elegir entre las dos opciones según el caso: la función personalizada se recalcula sola, y el menú permite un control explícito.

## 8. Paso 4: mejorar con una caché

Una función personalizada **se vuelve a calcular** cada vez que la planilla lo considera necesario: al abrirla, al cambiar un valor del que depende, etc. Sin ninguna protección, cada recálculo repite la petición a PokeAPI. Eso es lento y desaprovecha la API.

Apps Script ofrece `CacheService`, un almacén temporal de pares clave-valor. Modificamos `obtenerPokemon_` para que consulte primero la caché:

```javascript
function obtenerPokemon_(id) {
  const cache = CacheService.getScriptCache();
  const clave = 'pokemon_' + id;

  // 1. ¿Ya lo tenemos guardado?
  const guardado = cache.get(clave);
  if (guardado) return JSON.parse(guardado);

  // 2. Si no, lo pedimos a la API
  const respuesta = UrlFetchApp.fetch(API_URL + id, {
    muteHttpExceptions: true
  });
  if (respuesta.getResponseCode() !== 200) {
    throw new Error('No se encontró el Pokémon: ' + id);
  }
  const p = JSON.parse(respuesta.getContentText());

  const pokemon = {
    id: p.id,
    nombre: p.name,
    altura: p.height / 10,
    peso: p.weight / 10,
    tipos: p.types.map(t => t.type.name),
    stats: {}
  };
  p.stats.forEach(s => {
    pokemon.stats[s.stat.name] = s.base_stat;
  });

  // 3. Guardamos el resultado simplificado por 6 horas
  cache.put(clave, JSON.stringify(pokemon), 21600);
  return pokemon;
}
```

Detalles importantes:

- Lo que se guarda en la caché es el objeto **simplificado**, no la respuesta completa. La caché admite valores de tamaño limitado (alrededor de 100 KB por clave) y la respuesta completa de la API es mucho más grande.
- La caché solo guarda **texto**, por eso se usa `JSON.stringify` al guardar y `JSON.parse` al leer.
- El tercer parámetro de `put` es el tiempo de vida en segundos: `21600` equivale a 6 horas.
- Las funciones `POKEMON_DATOS` y `completarPlanilla` no cambian: la mejora queda **encapsulada** dentro de la función auxiliar.

## 9. Límites y errores frecuentes

| Problema | Causa | Qué hacer |
| --- | --- | --- |
| `#REF!` al usar `POKEMON_DATOS` | Hay celdas ocupadas a la derecha | Despejar las celdas de destino |
| Error de permisos en la celda | El script no fue autorizado | Ejecutar una función desde el editor y aceptar los permisos |
| `No se encontró el Pokémon` | Número inexistente o celda con texto | Revisar el valor de la columna A |
| La fórmula tarda o falla con muchas filas | Una petición por fila y límites de ejecución | Usar la caché o la función del menú |
| `La fórmula no aparece en el autocompletado` | Falta `@customfunction` o el nombre termina en `_` | Revisar el comentario JSDoc y el nombre |
| Cambios que no se ven | El proyecto no se guardó | Guardar (Ctrl+S) y recargar la planilla |

Límites de la plataforma que conviene tener presentes:

- Las funciones personalizadas tienen un **tiempo máximo de ejecución** corto (del orden de 30 segundos).
- `UrlFetchApp` tiene una **cuota diaria** de peticiones, que depende del tipo de cuenta (del orden de miles por día en cuentas personales). Las consultas a la caché no cuentan.
- Una función personalizada **no puede modificar otras celdas** ni usar servicios que exijan permisos especiales de usuario.

## 10. Ejercicios propuestos

1. **Más estadísticas.** Agregar las columnas `Ataque especial` y `Defensa especial` (en la API se llaman `special-attack` y `special-defense`).
2. **Función de un solo dato.** Crear `POKEMON_TIPOS(numero)` y `POKEMON_PESO(numero)` reutilizando `obtenerPokemon_`. Observar que, gracias a la caché, no se repiten las peticiones.
3. **Imagen.** Guardar `p.sprites.front_default` en el objeto simplificado, devolver la URL con una función `POKEMON_SPRITE(numero)` y mostrarla en la celda con `=IMAGE(POKEMON_SPRITE(A2))`.
4. **Búsqueda por nombre.** La API también acepta el nombre (`/pokemon/pikachu`). Hacer que las funciones funcionen con un número o con un nombre. Atención: conviene pasar el nombre a minúsculas con `String(id).toLowerCase()`.
5. **Errores amigables.** Hacer que la función del menú escriba `no encontrado` en la fila cuando falle la petición, en lugar de detenerse. Pista: `try { ... } catch (e) { ... }`.
6. **Comparador.** Crear `POKEMON_MAS_RAPIDO(a, b)` que reciba dos números y devuelva el nombre del Pokémon con mayor velocidad.
7. **Equipo.** En una segunda hoja, armar un equipo de seis Pokémon y calcular con fórmulas de la planilla el promedio de sus estadísticas.

## 11. Resumen

- **Google Apps Script** es un lenguaje de scripting basado en JavaScript que automatiza y extiende las aplicaciones de Google Workspace.
- Una **función personalizada** se usa como una fórmula y devuelve un valor o una fila a la celda donde está; una **función con menú** puede escribir en cualquier parte de la planilla.
- `UrlFetchApp.fetch()` realiza la petición HTTP y `JSON.parse()` convierte la respuesta en un objeto, del que se leen los campos con la notación de punto, `.map()` y `.forEach()`.
- Conviene **pedir los datos una sola vez**, extraer solo lo necesario y guardarlo en `CacheService`.
- Es la misma idea del apunte anterior: un script que actúa como **pegamento**, aquí entre una planilla y un servicio web, transformando los datos de un formato a otro.
