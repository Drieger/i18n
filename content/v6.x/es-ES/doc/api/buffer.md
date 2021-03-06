# Buffer

<!--introduced_in=v0.10.0-->

> Estabilidad: 2 - Estable

Antes de la introducción de [`TypedArray`] en [`ECMAScript 2015`] (ES6), el lenguaje JavaScript no tenía ningún mecanismo para leer o manipular streams de datos binarios. La clase `Buffer` fue introducida como parte del API de Node.js para que sea posible interactuar con streams octet en el contexto de cosas como streams TCP y operaciones del sistema de archivos.

Ahora que [`TypedArray`] ha sido añadido en ES6, la clase `Buffer` implementa la API [`Uint8Array`] de una manera más optimizada y adecuada para los casos de uso de Node.js.

Las instancias de la clase `Buffer` son similares a los arrays de enteros pero corresponden a asignaciones de memoria de tamaño fijo fuera del montículo de V8. El tamaño del `Buffer` es establecido cuando se crea y no puede ser cambiado.

La clase `Buffer` es global dentro de Node.js, por lo que es poco probable que alguna vez uno necesite `require('buffer').Buffer`.

Ejemplos:

```js
// Crea un Buffer lleno de ceros de longitud 10.
const buf1 = Buffer.alloc(10);

// Crea un Buffer de longitud 10, lleno con 0x1.
const buf2 = Buffer.alloc(10, 1);

// Crea un buffer sin inicializar de longitud 10.
// Esto es más rápido que llamar a Buffer.alloc() pero la instancia de Buffer
// devuelta puede contener datos viejos que necesitan ser
// sobrescritos utilizando ya sea fill() o write().
const buf3 = Buffer.allocUnsafe(10);

// Crea un Buffer que contiene [0x1, 0x2, 0x3].
const buf4 = Buffer.from([1, 2, 3]);

// Crea un Buffer que contiene UTF-8 bytes [0x74, 0xc3, 0xa9, 0x73, 0x74].
const buf5 = Buffer.from('tést');

// Crea un Buffer que contiene Latin-1 bytes [0x74, 0xe9, 0x73, 0x74].
const buf6 = Buffer.from('tést', 'latin1');
```

## `Buffer.from()`, `Buffer.alloc()`, y `Buffer.allocUnsafe()`

En versiones de Node.js anteriores a v6, las instancias `Buffer` fueron creadas usando la función constructora `Buffer`, la cual asigna el ` Buffer ` devuelto de forma diferente en función de los argumentos proporcionados:

* Pasar un número como el primer argumento a `Buffer()` (p. e.j `new Buffer(10)`), asigna un nuevo objeto `Buffer` del tamaño especificado. La memoria asignada para tales instancias `Buffer` *no* es inicializada y *puede contener datos sensibles*. Tales instancias de `Buffer` *deben* ser inicializadas *manualmente* usando ya sea [`buf.fill(0)`][`buf.fill()`] o escribiéndole al `Buffer` completamente. Mientras que este comportamiento es *intencional* para mejorar el rendimiento, la experiencia de desarrollo ha demostrado que una distinción más explícita es requerida entre crear un `Buffer` rápido pero no inicializado versus crear un `Buffer` más lento pero seguro.
* Pasar un string, array o `Buffer` como el primer argumento copia los datos del objeto pasado en el `Buffer`.
* Pasar un [`ArrayBuffer`] o un [`SharedArrayBuffer`] devuelve un `Buffer` que comparte la memoria asignada con el buffer del array dado.

Debido a que este comportamiento de `new Buffer()` cambia significativamente basado en el tipo de valor pasado como el primer argumento, aplicaciones que no validan propiamente los argumentos de entrada pasados a `new Buffer()`, o que fallan en inicializar apropiadamente nuevo contenido `Buffer` asignado, pueden producir inadvertidamente problemas de seguridad y confiabilidad en su código.

Para hacer la creación de instancias `Buffer` más confiable y menos propensa a errores, las diversas formas del constructor `new Buffer()` han sido **desaprobadas** y reemplazadas por métodos separados `Buffer.from()`, [`Buffer.alloc()`], y [`Buffer.allocUnsafe()`].

*Los desarrolladores deben migrar todos los usos existentes de los constructores `new Buffer()` a una de estas APIs nuevas.*

* [`Buffer.from(array)`] devuelve un nuevo `Buffer` que contiene una *copia* de los octetos proporcionados.
* [`Buffer.from(arrayBuffer[, byteOffset [, length]])`][`Buffer.from(arrayBuffer)`] devuelve un nuevo `Buffer` que *comparte* la misma memoria asignada como el [`ArrayBuffer`] dado.
* [`Buffer.from(buffer)`] devuelve un nuevo `Buffer` que contiene una *copia* de los contenidos del `Buffer` dado.
* [`Buffer.from(string[, encoding])`][`Buffer.from(string)`] devuelve un nuevo `Buffer` que contiene una *copia* de la string proporcionada.
* [`Buffer.alloc(size[, fill[, encoding]])`][`Buffer.alloc()`] devuelve una instancia `Buffer` "llena" del tamaño especificado. Este método puede ser significativamente más lento que [`Buffer.allocUnsafe(size)`][`Buffer.allocUnsafe()`] pero asegura que las nuevas instancias `Buffer` creadas nunca contengan datos viejos y potencialmente sensibles.
* [`Buffer.allocUnsafe(size)`][`Buffer.allocUnsafe()`] y [`Buffer.allocUnsafeSlow(size)`][`Buffer.allocUnsafeSlow()`] cada uno devuelve un nuevo `Buffer` del `size` especificado, cuyo contenido *debe* ser inicializado usando ya sea [`buf.fill(0)`][`buf.fill()`] o escrito por completo.

Las instancias de `Buffer` devueltas por [`Buffer.allocUnsafe()`] *pueden* ser asignadas fuera de una agrupación de memoria interna compartida si `size` es menor o igual que la mitad [`Buffer.poolSize`]. Las instancias devueltas por [`Buffer.allocUnsafeSlow()`] *nunca* usan la agrupación de memoria interna compartida.

### La opción de línea de comando `--zero-fill-buffers`

<!-- YAML
added: v5.10.0
-->

Node.js puede ser iniciado usando la opción de línea de comando `--zero-fill-buffers` para forzar todas las nuevas instancias `Buffer` asignadas creadas usando ya sea `new Buffer(size)`, [`Buffer.allocUnsafe()`], [`Buffer.allocUnsafeSlow()`] o `new SlowBuffer(size)` a ser *automáticamente zero-filled* sobre la creación. El uso de esta bandera *cambia el comportamiento predeterminado* de estos métodos y *puede tener un impacto significativo* en el rendimiento. Se recomienda el uso de la opción `--zero-fill-buffers` solo cuando sea necesario para imponer que las nuevas instancias `Buffer` no puedan contener datos potencialmente sensibles.

Ejemplo:

```txt
$ node --zero-fill-buffers
> Buffer.allocUnsafe(5);
<Buffer 00 00 00 00 00>
```

### ¿Qué hace `Buffer.allocUnsafe()` y `Buffer.allocUnsafeSlow()`"inseguros"?

Cuando se llama a [`Buffer.allocUnsafe()`] y [`Buffer.allocUnsafeSlow()`], el segmento de la memoria asignada está *sin inicializar*, (no está zeroed-out). Mientras que este diseño hace que la asignación de memoria sea bastante rápida, el segmento asignado de la memoria puede contener datos viejos que son potencialmente sensibles. Usar un `Buffer` creado por [`Buffer.allocUnsafe()`] sin sobrescribir *completamente* la memoria puede permitir que estos datos viejos se filtren cuando la memoria del `Buffer` es leída.

Si bien hay claras ventajas de rendimiento al usar [`Buffer.allocUnsafe()`], se *debe* tener mucho cuidado para así evitar introducir vulnerabilidades de seguridad a una aplicación.

## Buffers y Codificaciones de Caracteres

Las instancias de `Buffer` son comúnmente usadas para representar secuencias de caracteres codificados tales como UTF-8, UCS2, Base64 o incluso datos codificados de forma hexadecimal. Es posible convertir una y otra vez entre instancias de `Buffer` y strings ordinarias de JavaScript utilizando una codificación de caracteres explícita.

Ejemplo:

```js
const buf = Buffer.from('hello world', 'ascii');

// Imprime: 68656c6c6f20776f726c64
console.log(buf.toString('hex'));

// Imprime: aGVsbG8gd29ybGQ=
console.log(buf.toString('base64'));
```

Las codificaciones de caracteres soportados actualmente por Node.js incluyen:

* `'ascii'` - Solo para datos ASCII de 7-bit. Esta codificación es rápida y eliminará el bit alto si se establece.

* `'utf8'` - Multibyte codificado en caracteres Unicode. Muchas páginas web y otros formatos de documento utilizan UTF-8.

* `'utf16le'` - 2 o 4 bytes, caracteres Unicode codificados como little-endian. Los pares sustitutos (U+10000 to U+10FFFF) son soportados.

* `'ucs2'` - Alias de `'utf16le'`.

* `'base64'` - Codificación Base64. Al crear un `Buffer` desde una string, esta codificación también aceptará correctamente "URL y nombre de archivo Safe Alphabet" como es especificado en [RFC4648, Section 5](https://tools.ietf.org/html/rfc4648#section-5).

* `'latin1'` - Una forma de codificar el `Buffer` en una cadena codificada de un byte (como es definido por la IANA en [RFC1345](https://tools.ietf.org/html/rfc1345), página 63, para ser el bloque de suplemento Latin-1 y los códigos de control C0 / C1).

* `'binary'` - Alias para `'latin1'`.

* `'hex'` - Codifica cada byte como dos caracteres hexadecimales.

*Note*: Los navegadores actuales siguen la [WHATWG spec](https://encoding.spec.whatwg.org/) la cual asigna alias a ambos 'latin1' y ISO-8859-1 para win-1252. Esto significa que mientras hacer algo como `http.get()`, si el conjunto de caracteres devuelto es uno de esos listados en la WHATWG spec es posible que el servidor en realidad haya devuelto datos codificados en win-1252, y el uso de la codificación ` 'latin1' ` puede decodificar incorrectamente los caracteres.

## Buffers y TypedArray

Instancias de `Buffer` también son instancias de [`Uint8Array`]. Sin embargo, hay incompatibilidades sutiles con la especificación TypedArray en [`ECMAScript 2015`]. Por ejemplo, mientras que [`ArrayBuffer#slice()`] crea una copia de la porción, la implementación de [`Buffer#slice()`][`buf.slice()`] crea una vista sobre los `Buffer` existentes sin copiar, hacer [`Buffer#slice()`][`buf.slice()`] mucho más eficiente.

También es posible crear nuevas instancias de [`TypedArray`] desde un `Buffer` con las siguientes advertencias:

1. La memoria del objeto `Buffer` se copia al [`TypedArray`], no es compartida.

2. La memoria del objeto `Buffer` es interpretada como un array de distintos elementos, y no como un array, no como un array de bytes del tipo de destino. Es decir, `new Uint32Array(Buffer.from([1, 2, 3, 4]))` crea un elemento 4[`Uint32Array`] con elementos `[1, 2, 3, 4]`, no un [`Uint32Array`] con un solo elemento `[0x1020304]` o `[0x4030201]`.

Es posible crear un nuevo `Buffer` que comparta la misma memoria asignada como un instancia [`TypedArray`] utilizando la propiedad `.buffer` del objeto TypeArray.

Ejemplo:

```js
const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// Copia los contenidos de `arr`
const buf1 = Buffer.from(arr);

// Comparte la memoria con `arr`
const buf2 = Buffer.from(arr.buffer);

// Imprime: <Buffer 88 a0>
console.log(buf1);

// Imprime: <Buffer 88 13 a0 0f>
console.log(buf2);

arr[1] = 6000;

// Imprime: <Buffer 88 a0>
console.log(buf1);

// Imprime: <Buffer 88 13 70 17>
console.log(buf2);
```

Note que al crear un `Buffer` usando un [`TypedArray`]'s `.buffer`, es posible usar solo una porción del [`ArrayBuffer`] subyacente pasando parámetros en `byteOffset` y `length`.

Ejemplo:

```js
const arr = new Uint16Array(20);
const buf = Buffer.from(arr.buffer, 0, 16);

// Imprime: 16
console.log(buf.length);
```

El `Buffer.from()` y el [`TypedArray.from()`] tienen diferentes firmas e implementaciones. Específicamente, las variantes de [`TypedArray`] aceptan un segundo argumento que es una función de mapeo que es invocada en cada elemento del array indicado:

* `TypedArray.from(source[, mapFn[, thisArg]])`

El método `Buffer.from()`, sin embargo, no permite el uso de una función de mapeo:

* [`Buffer.from(array)`]
* [`Buffer.from(buffer)`]
* [`Buffer.from(arrayBuffer[, byteOffset [, length]])`][`Buffer.from(arrayBuffer)`]
* [`Buffer.from(string[, encoding])`][`Buffer.from(string)`]

## Buffers e Iteración ES6

Instancias de `Buffer` pueden ser iteradas sobre la sintaxis [`ECMAScript 2015`] (ES6) `for..of`.

Ejemplo:

```js
const buf = Buffer.from([1, 2, 3]);

// Imprime:
//   1
//   2
//   3
for (const b of buf) {
  console.log(b);
}
```

Adicionalmente, los métodos [`buf.values()`], [`buf.keys()`], y [`buf.entries()`] pueden ser usados para crear iteradores.

## Clase: Buffer

La clase `Buffer` es un tipo global para tratar con datos binarios directamente. Puede ser construida de varias maneras.

### new Buffer(array)

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobada: Use [`Buffer.from(array)`] en su lugar.

* `array` {integer[]} Un array de bytes del cual copiarse.

Asigna un nuevo `Buffer` usando un `array` de octetos.

Ejemplo:

```js
// Crea un nuevo Buffer que contiene los bytes UTF-8 de la string 'buffer'
const buf = new Buffer([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
```

### new Buffer(buffer)

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobado: Use [`Buffer.from(buffer)`] en su lugar.

* `buffer` {Buffer} Un `Buffer` existente del cual copiar datos.

Copia los datos del `buffer` pasado en una nueva instancia de `Buffer`.

Ejemplo:

```js
const buf1 = new Buffer('buffer');
const buf2 = new Buffer(buf1);

buf1[0] = 0x61;

// Imprime: auffer
console.log(buf1.toString());

// Imprime: buffer
console.log(buf2.toString());
```

### new Buffer(arrayBuffer[, byteOffset [, length]])

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobado: Use [`Buffer.from(arrayBuffer[, byteOffset [, length]])`][`Buffer.from(arrayBuffer)`] en su lugar.

* `arrayBuffer` {ArrayBuffer|SharedArrayBuffer} Un [`ArrayBuffer`], [`SharedArrayBuffer`] o la propiedad `.buffer` de un [`TypedArray`].
* `byteOffset` {integer} Índice del primer byte a exponer. **Predeterminado** `0`
* `length` {integer} Número de bytes a exponer. **Predeterminado:** `arrayBuffer.length - byteOffset`

Esto crea una vista del [`ArrayBuffer`] o [`SharedArrayBuffer`] sin copiar la memoria subyacente. Por ejemplo, cuando se pasa una referencia a la propiedad `.buffer` de una instancia [`TypedArray`], el `Buffer` nuevo creado compartirá la misma memoria asignada que el [`TypedArray`].

Los argumentos opcionales `byteOffset` y `length` especifican un rango de memoria dentro del `arrayBuffer` que será compartido por el `Buffer`.

Ejemplo:

```js
onst arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// Comparte memoria con `arr`
const buf = new Buffer(arr.buffer);

// Imprime: <Buffer 88 13 a0 0f>
console.log(buf);

// Cambiar el Uint16Array original también cambia el Buffer
arr[1] = 6000;

// Imprime: <Buffer 88 13 70 17>
console.log(buf);
```

### new Buffer(size)

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobado: Use [`Buffer.alloc()`] en su lugar (también vea [`Buffer.allocUnsafe()`]).

* `size` {integer} La longitud deseada del nuevo `Buffer`.

Asigna el nuevo `Buffer` de bytes de `size`. El `size` debe ser menor o igual al valor de [`buffer.kMaxLength`]. De lo contrario, se produce un [`RangeError`]. Un `Buffer` de longitud cero será creado si `size <= 0`.

A diferencia de [`ArrayBuffers`][`ArrayBuffer`], la memoria subyacente para instancias de `Buffer` creadas de esta forma *no es inicializada*. Los contenidos de un `Buffer` recién creado son desconocidos y *podrían contener datos sensibles*. Use [`Buffer.alloc(size)`][`Buffer.alloc()`] en su lugar para inicializar un `Buffer` a ceros.

Ejemplo:

```js
const buf = new Buffer(10);

// Imprime: (contents may vary): <Buffer 48 21 4b 00 00 00 00 00 30 dd>
console.log(buf);

buf.fill(0);

// Imprime: <Buffer 00 00 00 00 00 00 00 00 00 00>
console.log(buf);
```

### new Buffer(string[, encoding])

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobado: Use [`Buffer.from(string[, encoding])`] [`Buffer.from(string)`] en su lugar.

* `string` {string} String a codificar.
* `encoding` {string} La codificación de `string`. **Predeterminado:** `'utf8'`

Crea un nuevo `Buffer` que contiene la string `string` de JavaScript dada. Si es proporcionado, el parámetro `encoding` identifica la codificación del carácter de `string`.

Ejemplos:

```js
const buf1 = new Buffer('this is a tést');

// Imprime: this is a tést
console.log(buf1.toString());

// Imprime: this is a tC)st
console.log(buf1.toString('ascii'));


const buf2 = new Buffer('7468697320697320612074c3a97374', 'hex');

// Imprime: this is a tést
console.log(buf2.toString());
```

### Método de Clase: Buffer.alloc(size[, fill[, encoding]])

<!-- YAML
added: v5.10.0
-->

* `size` {integer} La longitud deseada del nuevo `Buffer`.
* `fill` {string|Buffer|integer} Un valor con el cual llenar previamente el nuevo `Buffer`. **Predeterminado:** `0`
* `encoding` {string} Si `fill` es una string, esta es su codificación. **Predeterminado:** `'utf8'`

Asigna el nuevo `Buffer` de bytes de `size`. Si `fill` es `undefined`, el `Buffer` estará *lleno de ceros*.

Ejemplo:

```js
const buf = Buffer.alloc(5);

// Imprime: <Buffer 00 00 00 00 00>
console.log(buf);
```

El `size` debe ser menos o igual al valor de [`buffer.kMaxLength`]. De lo contrario, se produce un [`RangeError`]. Un `Buffer` de longitud cero será creado si `size <= 0`.

Si `fill` es especificado, el `Buffer` asignado será inicializado llamando [`buf.fill(fill)`][`buf.fill()`].

Ejemplo:

```js
const buf = Buffer.alloc(5, 'a');

// Imprime: <Buffer 61 61 61 61 61>
console.log(buf);
```

Si ambos `fill` y `encoding` son especificados, el `Buffer` asignado será inicializado llamando [`buf.fill(fill, encoding)`][`buf.fill()`].

Ejemplo:

```js
const buf = Buffer.alloc(11, 'aGVsbG8gd29ybGQ=', 'base64');

// Imprime: <Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64>
console.log(buf);
```

Llamar a [`Buffer.alloc()`] puede ser significativamente más lento que el [`Buffer.allocUnsafe()`] alternativo pero asegura que el contenido de la instancia de `Buffer` recién creada *nunca contendrá datos sensibles*.

Se producirá un `TypeError` si `size` no es número.

### Método de Clase: Buffer.allocUnsafe(size)

<!-- YAML
added: v5.10.0
-->

* `size` {integer} La longitud deseada del nuevo `Buffer`.

Asigna un nuevo *non-zero-filled* `Buffer` de bytes `size`. El `size` debe ser menor o igual al valor de [`buffer.kMaxLength`]. De lo contrario, se produce un [`RangeError`]. Un `Buffer` de longitud cero será creado si `size <= 0`.

La memoria subyacente para las instancias `Buffer` creada de esta forma *no es inicializada*. Los contenidos del `Buffer` recién creado son desconocidos y *pueden contener datos sensibles*. Use [`Buffer.alloc()`] en su lugar para inicializar instancias de `Buffer` a ceros.

Ejemplo:

```js
const buf = Buffer.allocUnsafe(10);

// Imprime: (contents may vary): <Buffer a0 8b 28 3f 01 00 00 00 50 32>
console.log(buf);

buf.fill(0);

// Imprime: <Buffer 00 00 00 00 00 00 00 00 00 00>
console.log(buf);
```

Se producirá un `TypeError` si `size` no es número.

Note que el módulo `Buffer` asigna previamente una instancia de `Buffer` interna de tamaño [`Buffer.poolSize`] que es usada como un pool para la asignación rápida de nuevas instancias de `Buffer` creadas usando [`Buffer.allocUnsafe()`] y el constructor desaprobado `new Buffer(size)` solo cuando `size` es menor o igual a `Buffer.poolSize >> 1` (piso de [`Buffer.poolSize`] dividido entre dos).

El uso de este pool de memoria interna asignada previamente es una diferencia clave entre `Buffer.alloc(size, fill)` vs. `Buffer.allocUnsafe(size).fill(fill)`. Específicamente, `Buffer.alloc(size, fill)` *nunca* usará el pool de `Buffer` interno, mientras que `Buffer.allocUnsafe(size).fill(fill)` *usará* el pool de `Buffer` interno si `size` es menor o igual a la mitad de [`Buffer.poolSize`]. La diferencia es sutil pero puede ser importante cuando una aplicación requiere el rendimiento adicional que proporciona [`Buffer.allocUnsafe()`].

### Método de Clase: Buffer.allocUnsafeSlow(size)

<!-- YAML
added: v5.12.0
-->

* `size` {integer} La longitud deseada del nuevo `Buffer`.

Asigna un nuevo `Buffer` *no lleno de ceros* y no agrupado de bytes `size`. El `size` debe ser menor o igual al valor de [`buffer.kMaxLength`]. De lo contrario, se produce un [`RangeError`]. Un `Buffer` de longitud cero será creado si `size <= 0`.

La memoria subyacente para las instancias `Buffer` creada de esta forma *no es inicializada*. Los contenidos del `Buffer` recién creado son desconocidos y *pueden contener datos sensibles*. Use [`buf.fill(0)`][`buf.fill()`] para inicializar tal instancia de `Buffer` a ceros.

Al usar [`Buffer.allocUnsafe()`] para asignar nuevas instancias de `Buffer`, asignaciones bajo 4KB son, por defecto, segmentadas desde un solo `Buffer` asignado previamente. Esto permite que las aplicaciones eviten la sobrecarga de la colección de basura que implica crear muchas instancias de `Buffer` asignadas individualmente. Este enfoque mejora el rendimiento y uso de la memoria al eliminar la necesidad de rastrear y limpiar tantos objetos `Persistent`.

Sin embargo, en el caso donde un desarrollador pueda necesitar retener un pedazo pequeño de la memoria de un pool por un período de tiempo predeterminado, puede ser apropiado crear una instancia de `Buffer` un-pooled usando `Buffer.allocUnsafeSlow()` y luego copiar los bits relevantes.

Ejemplo:

```js
// Necesita conservar unos pequeños pedazos de memoria
const store = [];

socket.on('readable', () => {
  const data = socket.read();

  // Asignar para datos retenidos
  const sb = Buffer.allocUnsafeSlow(10);

  // Copiar los datos a la nueva asignación
  data.copy(sb, 0, 0, 10);

  store.push(sb);
});
```

`Buffer.allocUnsafeSlow()` solo debe utilizarse como último recurso *después* de que un desarrollador haya observado la retención indebida de memoria en sus aplicaciones.

Se producirá un `TypeError` si `size` no es número.

### Método de Clase: Buffer.byteLength(string[, encoding])

<!-- YAML
added: v0.1.90
-->

* `string` {string|Buffer|TypedArray|DataView|ArrayBuffer|SharedArrayBuffer} Un valor del cual calcular la longitud.
* `encoding` {string} Si `string` es una string, esta es su codificación. **Predeterminado:** `'utf8'`
* Devuelve: {integer} El número de bytes contenidos dentro de `string`.

Devuelve la longitud real en bytes de una string. Esto no es lo mismo que [`String.prototype.length`] ya que eso devuelve el número de *caracteres* en una string.

*Note* que para `'base64'` y `'hex'`, esta función asume una entrada válida. Para strings que contienen datos no codificados de forma Base64/Hex (p. ej espacio en blanco), el valor devuelto puede ser mayor que la longitud de un `Buffer` creado desde la string.

Ejemplo:

```js
const str = '\u00bd + \u00bc = \u00be';

// Imprime: // Imprime: ½ + ¼ = ¾: 9 caracteres, 12 bytes
console.log(`${str}: ${str.length} characters, ` +
            `${Buffer.byteLength(str, 'utf8')} bytes`);
```

Cuando `string` es un `Buffer`/[`DataView`]/[`TypedArray`]/[`ArrayBuffer`]/ [`SharedArrayBuffer`], la longitud real en bytes es devuelta.

De lo contrario, se convierte en `String` y devuelve la longitud en bytes de la string.

### Método de Clase: Buffer.compare(buf1, buf2)

<!-- YAML
added: v0.11.13
-->

* `buf1` {Buffer}
* `buf2` {Buffer}
* Devuelve: {integer}

Compara `buf1` con `buf2` típicamente con el propósito de clasificar los arrays de instancias de `Buffer`. Esto es el equivalente de llamar a [`buf1.compare(buf2)`][`buf.compare()`].

Ejemplo:

```js
const buf1 = Buffer.from('1234');
const buf2 = Buffer.from('0123');
const arr = [buf1, buf2];

// Imprime: [ <Buffer 30 31 32 33>, <Buffer 31 32 33 34> ]
// (Este resultado es igual a: [buf2, buf1])
console.log(arr.sort(Buffer.compare));
```

### Método de Clase: Buffer.concat(list[, totalLength])

<!-- YAML
added: v0.7.11
-->

* `list` {Array} Lista de instancias de `Buffer` a concatenar.
* `totalLength` {integer} Longitud total de las instancias de `Buffer` en `list` cuando están concatenadas.
* Devuelve: {Buffer}

Devuelve un nuevo `Buffer`, el cual es el resultado de concatenar juntas todas las instancias de `Buffer` en la `list`.

Si la lista no tiene elementos, o si la `totalLength` es 0, entonces un nuevo `Buffer` de longitud cero es devuelto.

Si `totalLength` no es proporcionado, se calcula desde las instancias de `Buffer` en `list`. Sin embargo, esto ocasiona que se ejecute un bucle adicional para calcular la `totalLength`, así es más rápido proporcionar la longitud explícitamente si ya es conocida.

Si `totalLength` es proporcionado, es forzado a ser un entero sin signo. Si la longitud combinada de los `Buffer`s en `list` excede a `totalLength`, el resultado es truncado a `totalLength`.

Ejemplo: Crear un `Buffer` único a partir de una lista de tres instancias de `Buffer`

```js
const buf1 = Buffer.alloc(10);
const buf2 = Buffer.alloc(14);
const buf3 = Buffer.alloc(18);
const totalLength = buf1.length + buf2.length + buf3.length;

// Imprime: 42
console.log(totalLength);

const bufA = Buffer.concat([buf1, buf2, buf3], totalLength);

// Imprime: <Buffer 00 00 00 00 ...>
console.log(bufA);

// Imprime: 42
console.log(bufA.length);
```

### Método de Clase: Buffer.from(array)

<!-- YAML
added: v5.10.0
-->

* `array` {Array}

Asigna un nuevo `Buffer` usando un `array` de octetos.

Ejemplo:

```js
// Crea un Buffer nuevo que contiene bytes UTF-8 de la string 'buffer'
const buf = Buffer.from([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
```

Se producirá un `TypeError` si `array` no es un `Array`.

### Método de Clase: Buffer.from(arrayBuffer[, byteOffset[, length]])

<!-- YAML
added: v5.10.0
-->

* `arrayBuffer` {ArrayBuffer|SharedArrayBuffer} Un [`ArrayBuffer`], [`SharedArrayBuffer`], o la propiedad `.buffer` de un [`TypedArray`].
* `byteOffset` {integer} Índice del primer byte a exponer. **Predeterminado:** `0`
* `length` {integer} Número de bytes a exponer. **Predeterminado:** `arrayBuffer.length - byteOffset`

Esto crea una vista del [`ArrayBuffer`] sin copiar la memoria subyacente. Por ejemplo, cuando se pasa una referencia a la propiedad de `.buffer` de una instancia [`TypedArray`], el `Buffer` recién creado compartirá la misma memoria asignada que el [`TypedArray`].

Ejemplo:

```js
const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// Comparte memoria con `arr`
const buf = Buffer.from(arr.buffer);

// Imprime: <Buffer 88 13 a0 0f>
console.log(buf);

// Cambiar el Uint16Array original también cambia el Buffer
arr[1] = 6000;

// Imprime: <Buffer 88 13 70 17>
console.log(buf);
```

Los argumentos opcionales `byteOffset` y `length` especifican un rango de memoria dentro del `arrayBuffer` que será compartido por el `Buffer`.

Ejemplo:

```js
const ab = new ArrayBuffer(10);
const buf = Buffer.from(ab, 0, 2);

// Imprime: 2
console.log(buf.length);
```

Se producirá un `TypeError` si `arrayBuffer` no es un [`ArrayBuffer`] o un [`SharedArrayBuffer`].

### Método de Clase: Buffer.from(buffer)

<!-- YAML
added: v5.10.0
-->

* `buffer` {Buffer} Un `Buffer` existente del cual copiar datos.

Copia los datos del `buffer` pasado en una nueva instancia de `Buffer`.

Ejemplo:

```js
const buf1 = Buffer.from('buffer');
const buf2 = Buffer.from(buf1);

buf1[0] = 0x61;

// Imprime: buffer
console.log(buf1.toString());

// Imprime: buffer
console.log(buf2.toString());
```

Se producirá un `TypeError` si `buffer` no es un `Buffer`.

### Método de Clase: Buffer.from(string[, encoding])

<!-- YAML
added: v5.10.0
-->

* `string` {string} Una string a codificar.
* `encoding` {string} La codificación de `string`. **Predeterminado:** `'utf8'`

Crea un nuevo `Buffer` que contiene la string `string` de JavaScript dada. Si es proporcionado, el parámetro `encoding` identifica la codificación del carácter de `string`.

Ejemplos:

```js
const buf1 = Buffer.from('this is a tést');

// Imprime: this is a tést
console.log(buf1.toString());

// Imprime: this is a tC)st
console.log(buf1.toString('ascii'));


const buf2 = Buffer.from('7468697320697320612074c3a97374', 'hex');

// Imprime: this is a tést
console.log(buf2.toString());
```

Se producirá un `TypeError` si `string` no es una string.

### Método de Clase: Buffer.isBuffer(obj)

<!-- YAML
added: v0.1.101
-->

* `obj` {Object}
* Devuelve: {boolean}

Devuelve `true` si `obj` es un `Buffer`, de otra forma es `false`.

### Método de Clase: Buffer.isEncoding(encoding)

<!-- YAML
added: v0.9.1
-->

* `encoding` {string} Un nombre de codificación de caracteres a verificar.
* Devuelve: {boolean}

Devuelve `true` si `encoding` contiene una codificación de caracteres permitida, de otra forma es `false`.

### Propiedad de Clase: Buffer.poolSize

<!-- YAML
added: v0.11.3
-->

* {integer} **Predeterminado:** `8192`

Este es el número de bytes usados para determinar el tamaño de instancias internas de `Buffer` pre-asignadas, usadas para realizar pooling. Este valor puede ser modificado.

### buf[index]

<!-- YAML
type: property
name: [index]
-->

El operador de índice `[index]` puede ser usado para obtener y establecer el octeto en la posición `index` en `buf`. Los valores refieren a bytes individuales, por lo que el rango de valor legal está entre `0x00` y `0xFF` (hex) o `0` y `255` (decimal).

Este operador es heredado de `Uint8Array`, por lo que su comportamiento en el acceso fuera de los límites es el mismo que `UInt8Array` - es decir, hacer "get" devuelve `undefined` y "set" no hace nada.

Ejemplo: Copiar una string ASCII en un `Buffer`, un byte a la vez

```js
const str = 'Node.js';
const buf = Buffer.allocUnsafe(str.length);

for (let i = 0; i < str.length; i++) {
  buf[i] = str.charCodeAt(i);
}

// Imprime: Node.js
console.log(buf.toString('ascii'));
```

### buf.compare(target[, targetStart[, targetEnd[, sourceStart[, sourceEnd]]]])

<!-- YAML
added: v0.11.13
-->

* `target` {Buffer} Un `Buffer` para comparar.
* `targetStart` {integer} El offset dentro de `target` en el cual debe comenzar la comparación. **Predeterminado:** `0`
* `targetEnd` {integer} El offset con `target` en el cual debe terminar la comparación (no inclusivo). **Predeterminado:** `target.length`
* `sourceStart` {integer} El offset dentro de `buf` en el cual debe comenzar la comparación. **Predeterminado:** `0`
* `sourceEnd` {integer} El offset dentro de `buf` en el cual debe terminar la comparación (no inclusivo). **Predeterminado:** [`buf.length`]
* Devuelve: {integer}

Compara `buf` con `target` y devuelve un número que indica si `buf` viene antes, después, o está igual que `target` en el orden de clasificación. La comparación basada en la secuencia real de bytes en cada `Buffer`.

* `0` es devuelto si `target` es igual que `buf`
* `1` es devuelto si `target` debe venir *antes* de `buf` cuando se ordenan.
* `-1` es devuelto si `target` debe venir *después* de `buf` cuando se ordenan.

Ejemplos:

```js
const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('BCD');
const buf3 = Buffer.from('ABCD');

// Imprime: 0
console.log(buf1.compare(buf1));

// Imprime: -1
console.log(buf1.compare(buf2));

// Imprime: -1
console.log(buf1.compare(buf3));

// Imprime: 1
console.log(buf2.compare(buf1));

// Imprime: 1
console.log(buf2.compare(buf3));

// Imprime: [ <Buffer 41 42 43>, <Buffer 41 42 43 44>, <Buffer 42 43 44> ]
// (Este resultado es igual a: [buf1, buf3, buf2])
console.log([buf1, buf2, buf3].sort(Buffer.compare));
```

Los argumentos opcionales `targetStart`, `targetEnd`, `sourceStart`, y `sourceEnd` pueden ser usados para limitar la comparación a rangos específicos dentro de `target` y `buf`, respectivamente.

Ejemplos:

```js
const buf1 = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8, 9]);
const buf2 = Buffer.from([5, 6, 7, 8, 9, 1, 2, 3, 4]);

// Imprime: 0
console.log(buf1.compare(buf2, 5, 9, 0, 4));

// Imprime: -1
console.log(buf1.compare(buf2, 0, 6, 4));

// Imprime: 1
console.log(buf1.compare(buf2, 5, 6, 5));
```

Se producirá un `RangeError` si `targetStart < 0`, `sourceStart < 0`, `targetEnd > target.byteLength` o `sourceEnd > source.byteLength`.

### buf.copy(target[, targetStart[, sourceStart[, sourceEnd]]])

<!-- YAML
added: v0.1.90
-->

* `target` {Buffer|Uint8Array} Un `Buffer` o [`Uint8Array`] en el cual copiar.
* `targetStart` {integer} El offset dentro de `target` en el cual comenzar a copiar. **Predeterminado:** `0`
* `sourceStart` {integer} El offset dentro de `buf` en cual comenzar a copiar. **Predeterminado:** `0`
* `sourceEnd` {integer} El offset dentro de `buf` en el cual terminar de copiar (no inclusivo). **Predeterminado:** [`buf.length`]
* Devuelve: {integer} El número de bytes copiados.

Copia datos desde una región de `buf` a una región en `target`, incluso si la región de la memoria de `target` se superpone con `buf`.

Ejemplo: Crea dos instancias de `Buffer`, `buf1` y `buf2`, y copia `buf1` desde el byte 16 hasta el byte 19 en `buf2`, comenzando en el 8º byte en `buf2`

```js
const buf1 = Buffer.allocUnsafe(26);
const buf2 = Buffer.allocUnsafe(26).fill('!');

for (let i = 0; i < 26; i++) {
  // 97 es el valor ASCII decimal para 'a'
  buf1[i] = i + 97;
}

buf1.copy(buf2, 8, 16, 20);

// Imprime: !!!!!!!!qrst!!!!!!!!!!!!!
console.log(buf2.toString('ascii', 0, 25));
```

Ejemplo: Crea una sola copia de `Buffer` y copia datos desde una región a una región superpuesta dentro del mismo `Buffer`

```js
const buf = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 es el valor ASCII decimal para 'a'
  buf[i] = i + 97;
}

buf.copy(buf, 0, 4, 10);

// Imprime: efghijghijklmnopqrstuvwxyz
console.log(buf.toString());
```

### buf.entries()

<!-- YAML
added: v1.1.0
-->

* Devuelve: {Iterator}

Crea y devuelve un [iterador](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) de pares `[index, byte]` desde el contenido de `buf`.

Ejemplo: Registra todos los contenidos de un `Buffer`

```js
const buf = Buffer.from('buffer');

// Imprime:
//   [0, 98]
//   [1, 117]
//   [2, 102]
//   [3, 102]
//   [4, 101]
//   [5, 114]
for (const pair of buf.entries()) {
  console.log(pair);
}
```

### buf.equals(otherBuffer)

<!-- YAML
added: v0.11.13
-->

* `otherBuffer` {Buffer} Un `Buffer` con el cual comparar.
* Devuelve: {boolean}

Devuelve `true` si ambos `buf` y `otherBuffer` tienen exactamente los mismos bytes, de lo contrario `false`.

Ejemplos:

```js
const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('414243', 'hex');
const buf3 = Buffer.from('ABCD');

// Imprime: true
console.log(buf1.equals(buf2));

// Imprime: false
console.log(buf1.equals(buf3));
```

### buf.fill(value\[, offset[, end]\]\[, encoding\])

<!-- YAML
added: v0.5.0
-->

* `value` {string|Buffer|integer} El valor con el cual llenar `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a llenar `buf`. **Predeterminado:** `0`
* `end` {integer} Dónde detener el llenado de `buf` (no inclusivo). **Predeterminado:** [`buf.length`]
* `encoding` {string} Si `value` es una string, esa es su codificación. **Predeterminado:** `'utf8'`
* Returns: {Buffer} Una referencia a `buf`.

Llena `buf` con el `value` especificado. Si el `offset` y `end` no son dados, el `buf` completo se llenará. Esto pretende ser una pequeña simplificación para permitir que la creación y llenado de un `Buffer` sea realizado en una sola línea.

Ejemplo: Llenar un `Buffer` con el carácter ASCII `'h'`

```js
const b = Buffer.allocUnsafe(50).fill('h');

// Imprime: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh
console.log(b.toString());
```

`value` es forzado a ser un valor `uint32` si no es una String o un Entero.

Si la escritura final de una operación `fill()` cae en un carácter multi-byte, entonces solo se escriben los primeros bytes de ese carácter que encajen en `buf`.

Ejemplo: Llena un `Buffer` con un carácter de dos bytes

```js
// Imprime: <Buffer c8 a2 c8>
console.log(Buffer.allocUnsafe(3).fill('\u0222'));
```

### buf.includes(value\[, byteOffset\]\[, encoding\])

<!-- YAML
added: v5.3.0
-->

* `value` {string|Buffer|integer} Qué buscar.
* `byteOffset` {integer} Dónde comenzar la búsqueda en `buf`. **Predeterminado:** `0`
* `encoding` {string} Si `value` es una string, esa es su codificación. **Predeterminado:** `'utf8'`
* Devuelve: {boolean} `true` si `value` fue encontrado en `buf`, de otra forma es `false`.

Equivalente a [`buf.indexOf() !== -1`][`buf.indexOf()`].

Ejemplos:

```js
const buf = Buffer.from('this is a buffer');

// Imprime: true
console.log(buf.includes('this'));

// Imprime: true
console.log(buf.includes('is'));

// Imprime: true
console.log(buf.includes(Buffer.from('a buffer')));

// Imprime: true
// (97 es el valor ASCII decimal para 'a')
console.log(buf.includes(97));

// Imprime: false
console.log(buf.includes(Buffer.from('a buffer example')));

// Imprime: true
console.log(buf.includes(Buffer.from('a buffer example').slice(0, 8)));

// Imprime: false
console.log(buf.includes('this', 4));
```

### buf.indexOf(value\[, byteOffset\]\[, encoding\])

<!-- YAML
added: v1.5.0
-->

* `value` {string|Buffer|integer} Qué buscar.
* `byteOffset` {integer} Dónde comenzar la búsqueda en `buf`. **Predeterminado:** `0`
* `encoding` {string} Si `value` es una string, esa es su codificación. **Predeterminado:** `'utf8'`
* Devuelve: {integer} El índice de la primera aparición de `value` en `buf` o `-1` si `buf` no contiene `value`.

Si `value` es:

* una string, `value` es interpretado de acuerdo a la codificación de caracteres en `encoding`.
* un `Buffer`, `value` será usado en su totalidad. Para comparar un `Buffer` parcial use [`buf.slice()`].
* un número, `value` será interpretado como un valor entero de 8-bit sin signo entre `0` y `255`.

Ejemplos:

```js
const buf = Buffer.from('this is a buffer');

// Imprime: 0
console.log(buf.indexOf('this'));

// Imprime: 2
console.log(buf.indexOf('is'));

// Imprime: 8
console.log(buf.indexOf(Buffer.from('a buffer')));

// Imprime: 8
// (97 es el valor ASCII decimal para 'a')
console.log(buf.indexOf(97));

// Imprime: -1
console.log(buf.indexOf(Buffer.from('a buffer example')));

// Imprime: 8
console.log(buf.indexOf(Buffer.from('a buffer example').slice(0, 8)));


const utf16Buffer = Buffer.from('\u039a\u0391\u03a3\u03a3\u0395', 'ucs2');

// Imprime: 4
console.log(utf16Buffer.indexOf('\u03a3', 0, 'ucs2'));

// Imprime: 6
console.log(utf16Buffer.indexOf('\u03a3', -4, 'ucs2'));
```

Si `value` no es una string, un número, o un `Buffer`, este método producirá un `TypeError`. Si `value` es un número, será forzado a ser un valor byte válido, un entero entre 0 y 255.

Si `byteOffset` no es un número, será forzado a ser un número. Cualquier argumento que fuerce a `NaN` o 0, como `{}`, `[]`, `null` o `undefined`, buscará en todo el buffer. Este comportamiento coincide con [`String#indexOf()`].

```js
const b = Buffer.from('abcdef');

// Pasando un valor que es un número, pero no un valor byte válido
// Imprime: 2, equivalente a buscar 99 o 'c'
console.log(b.indexOf(99.9));
console.log(b.indexOf(256 + 99));

// Pasando un byteOffset que coacciona a NaN o 0
// Imprime: 1, buscando todo el buffer
console.log(b.indexOf('b', undefined));
console.log(b.indexOf('b', {}));
console.log(b.indexOf('b', null));
console.log(b.indexOf('b', []));
```

### buf.includes(value\[, byteOffset\]\[, encoding\])

<!-- YAML
added: v5.3.0
-->

* `value` {String|Buffer|Integer} Qué buscar.
* `byteOffset` {Integer} Dónde comenzar la búsqueda en `buf`. **Predeterminado:** `0`
* `encoding` {String} Si `encoding` es una string, esta es su codificación. **Predeterminado:** `'utf8'`
* Devuelve: {Boolean} `true` si `value` fue encontrado en `buf`, de lo contrario es `false`

Equivalente a [`buf.indexOf() !== -1`][`buf.indexOf()`].

Ejemplos:

```js
const buf = Buffer.from('this is a buffer');

// Imprime: true
console.log(buf.includes('this'));

// Imprime: true
console.log(buf.includes('is'));

// Imprime: true
console.log(buf.includes(Buffer.from('a buffer')));

// Imprime: true
// (97 es el valor ASCII decimal para 'a')
console.log(buf.includes(97));

// Imprime: false
console.log(buf.includes(Buffer.from('a buffer example')));

// Imprime: true
console.log(buf.includes(Buffer.from('a buffer example').slice(0, 8)));

// Imprime: false
console.log(buf.includes('this', 4));
```

### buf.keys()

<!-- YAML
added: v1.1.0
-->

* Devuelve: {Iterator}

Crea y devuelve un [iterador](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) de claves `buf` (índices).

Ejemplo:

```js
const buf = Buffer.from('buffer');

// Imprime:
//   0
//   1
//   2
//   3
//   4
//   5
for (const key of buf.keys()) {
  console.log(key);
}
```

### buf.lastIndexOf(value\[, byteOffset\]\[, encoding\])

<!-- YAML
added: v6.0.0
-->

* `value` {string|Buffer|integer} Qué buscar.
* `byteOffset` {integer} Dónde comenzar la búsqueda en `buf`. **Predeterminado:** [`buf.length`]`- 1`
* `encoding` {string} Si `value` es una string, esa es su codificación. **Predeterminado:** `'utf8'`
* Devuelve: {integer} El índice de la última aparición de `value` en `buf` o `-1` si `buf` no contiene `value`.

Idéntico a [`buf.indexOf()`], excepto que `buf` es buscado de atrás hacia adelante en lugar de delante hacia atrás.

Ejemplos:

```js
const buf = Buffer.from('this buffer is a buffer');

// Imprime: 0
console.log(buf.lastIndexOf('this'));

// Imprime: 17
console.log(buf.lastIndexOf('buffer'));

// Imprime: 17
console.log(buf.lastIndexOf(Buffer.from('buffer')));

// Imprime: 15
// (97 es el valor ASCII decimal para 'a')
console.log(buf.lastIndexOf(97));

// Imprime: -1
console.log(buf.lastIndexOf(Buffer.from('yolo')));

// Imprime: 5
console.log(buf.lastIndexOf('buffer', 5));

// Imprime: -1
console.log(buf.lastIndexOf('buffer', 4));


const utf16Buffer = Buffer.from('\u039a\u0391\u03a3\u03a3\u0395', 'ucs2');

// Imprime: 6
console.log(utf16Buffer.lastIndexOf('\u03a3', undefined, 'ucs2'));

// Imprime: 4
console.log(utf16Buffer.lastIndexOf('\u03a3', -5, 'ucs2'));
```

Si `value` no es una string, un número, o un `Buffer`, este método producirá un `TypeError`. Si `value` es un número, será forzado a ser un valor byte válido, un entero entre 0 y 255.

Si `byteOffset` no es un número, será forzado a ser un número. Cualquier argumento que fuerce a `NaN`, como `{}` o `undefined`, buscará todo el buffer. Este comportamiento coincide con [`String#lastIndexOf()`].

```js
const b = Buffer.from('abcdef');

// Pasando un valor que es un número, pero no un byte válido
// Imprime: 2, equivalente a buscar 99 o 'c'
console.log(b.lastIndexOf(99.9));
console.log(b.lastIndexOf(256 + 99));

// Pasando un byteOffset que fuerza a NaN
// Imprime: 1, buscando todo el buffer
console.log(b.lastIndexOf('b', undefined));
console.log(b.lastIndexOf('b', {}));

// Pasando un byteOffset que fuerza a 0
// Imprime: -1, equivalente a pasar 0
console.log(b.lastIndexOf('b', null));
console.log(b.lastIndexOf('b', []));
```

### buf.length

<!-- YAML
added: v0.1.90
-->

* {integer}

Devuelve la cantidad de memoria asignada para `buf` en bytes. Tenga en cuenta que esto no necesariamente refleja la cantidad de datos "utilizables" dentro de `buf`.

Ejemplo: Crear un `Buffer` y escribir una string ASCII más corta en él

```js
const buf = Buffer.alloc(1234);

// Imprime: 1234
console.log(buf.length);

buf.write('some string', 0, 'ascii');

// Imprime: 1234
console.log(buf.length);
```

Mientras que la propiedad `length` no es inmutable, cambiar el valor de `length` puede resultar en un comportamiento indefinido e incoherente. Las aplicaciones que deseen modificar la longitud de un `Buffer` deberían, por lo tanto, tratar a `length` como solo de lectura y usar [`buf.slice()`] para crear un nuevo `Buffer`.

Ejemplos:

```js
let buf = Buffer.allocUnsafe(10);

buf.write('abcdefghj', 0, 'ascii');

// Imprime: 10
console.log(buf.length);

buf = buf.slice(0, 5);

// Imprime: 5
console.log(buf.length);
```

### buf.readDoubleBE(offset[, noAssert])

### buf.readDoubleLE(offset[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Must satisfy: `0 <= offset <= buf.length - 8`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {number}

Lee un doble de 64-bit desde `buf` en el `offset` especificado con el formato endian especificado (`readDoubleBE()` devuelve big endian, `readDoubleLE()` devuelve little endian).

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8]);

// Imprime: 8.20788039913184e-304
console.log(buf.readDoubleBE());

// Imprime: 5.447603722011605e-270
console.log(buf.readDoubleLE());

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readDoubleLE(1));

// Advertencia: ¡Las lecturas pasaron el final de buffer!
// ¡Esto resultará en una falla de segmentación! ¡No hagas esto!
console.log(buf.readDoubleLE(1, true));
```

### buf.readFloatBE(offset[, noAssert])

### buf.readFloatLE(offset[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 4`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {number}

Lee un float de 32-bit desde `buf` en el `offset` especificado con el formato endian especificado (`readFloatBE()` devuelve big endian, `readFloatLE()` devuelve little endian).

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([1, 2, 3, 4]);

// Imprime: 2.387939260590663e-38
console.log(buf.readFloatBE());

// Imprime: 1.539989614439558e-36
console.log(buf.readFloatLE());

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readFloatLE(1));

// Advertencia: ¡Las lecturas pasaron el final del buffer!
// ¡Esto resultará en una falla de segmentación! ¡No hagas esto!
console.log(buf.readFloatLE(1, true));
```

### buf.readInt8(offset[, noAssert])

<!-- YAML
added: v0.5.0
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 1`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee un entero de 8-bit con signo desde `buf` en el `offset` especificado.

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Los enteros leído desde un `Buffer` son interpretados como valores con signo del complemento de dos.

Ejemplos:

```js
const buf = Buffer.from([-1, 5]);

// Imprime: -1
console.log(buf.readInt8(0));

// Imprime: 5
console.log(buf.readInt8(1));

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readInt8(2));
```

### buf.readInt16BE(offset[, noAssert])

### buf.readInt16BE(offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 2`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee un entero de 16-bit con signo desde `buf` en el `offset` especificado con el formato endian especificado (`readInt16BE()` devuelve big endian, `readInt16LE()` devuelve little endian).

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Los enteros leídos desde un `Buffer` son interpretados como valores con signo del complemento de dos.

Ejemplos:

```js
// Imprime: 5
console.log(buf.readInt16BE());

// Imprime: 1280
console.log(buf.readInt16LE());

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readInt16LE(1));
```

### buf.readInt32BE(offset[, noAssert])

### buf.readInt32BE(offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 4`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee un entero de 32-bit desde `buf` en el `offset` especificado con el formato endian especificado (`readInt32BE()` devuelve big endian, `readInt32LE()` devuelve little endian).

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Los enteros leído desde un `Buffer` son interpretados como valores con signo del complemento de dos.

Ejemplos:

```js
const buf = Buffer.from([0, 0, 0, 5]);

// Imprime: 5
console.log(buf.readInt32BE());

// Imprime: 83886080
console.log(buf.readInt32LE());

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readInt32LE(1));
```

### buf.readIntBE(offset, byteLength[, noAssert])

### buf.readIntLE(offset, byteLength[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - byteLength`.
* `byteLength` {integer} Número de bytes a leer. Debe satisfacer: `0 < byteLength <= 6`.
* `noAssert` {boolean} ¿Saltar validación de `offset` y `byteLength`? **Predeterminado:** `false`.
* Devuelve: {integer}

Lee el número de bytes `byteLength` desde `buf` en el `offset` especificado e interpreta el resultado como un valor con signo del complemento de dos. Soporta hasta 48 bits de precisión.

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

// Imprime: -546f87a9cbee
console.log(buf.readIntLE(0, 6).toString(16));

// Imprime: 1234567890ab
console.log(buf.readIntBE(0, 6).toString(16));

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readIntBE(1, 6).toString(16));
```

### buf.readUInt8(offset[, noAssert])

<!-- YAML
added: v0.5.0
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 1`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee un entero de 8-bit desde `buf` en el `offset` especificado.

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([1, -2]);

// Imprime: 1
console.log(buf.readUInt8(0));

// Imprime: 254
console.log(buf.readUInt8(1));

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readUInt8(2));
```

### buf.readUInt16BE(offset[, noAssert])

### buf.readUInt16LE(offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 2`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee un entero de 16-bit sin signo desde `buf` en el `offset` especificado con el formato endian especificado (`readUInt16BE()` devuelve big endian, `readUInt16LE()` devuelve little endian).

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([0x12, 0x34, 0x56]);

// Imprime: 1234
console.log(buf.readUInt16BE(0).toString(16));

// Imprime: 3412
console.log(buf.readUInt16LE(0).toString(16));

// Imprime: 3456
console.log(buf.readUInt16BE(1).toString(16));

// Imprime: 5634
console.log(buf.readUInt16LE(1).toString(16));

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readUInt16LE(2).toString(16));
```

### buf.readUInt32BE(offset[, noAssert])

### buf.readUInt32LE(offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - 4`.
* `noAssert` {boolean} ¿Saltar validación de `offset`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee un entero de 32-bit sin signo desde `buf` en el `offset` especificado con el formato endian especificado (`readUInt32BE()` devuelve big endian, `readUInt32LE()` devuelve little endian).

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([0x12, 0x34, 0x56, 0x78]);

// Imprime: 12345678
console.log(buf.readUInt32BE(0).toString(16));

// Imprime: 78563412
console.log(buf.readUInt32LE(0).toString(16));

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readUInt32LE(1).toString(16));
```

### buf.readUIntBE(offset, byteLength[, noAssert])

### buf.readUIntLE(offset, byteLength[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `offset` {integer} Número de bytes a omitir antes de comenzar a leer. Debe satisfacer: `0 <= offset <= buf.length - byteLength`.
* `byteLength` {integer} Número de bytes a leer. Debe satisfacer: `0 < byteLength <= 6`.
* `noAssert` {boolean} ¿Saltar validación de `offset` y `byteLength`? **Predeterminado:** `false`
* Devuelve: {integer}

Lee el número de bytes `byteLength` desde `buf` en el `offset` especificado e interpreta el resultado como un entero sin signo. Soporta hasta 48 bits de precisión.

Configurar `noAssert` a `true` permite que `offset` esté más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

// Imprime: 1234567890ab
console.log(buf.readUIntBE(0, 6).toString(16));

// Imprime: ab9078563412
console.log(buf.readUIntLE(0, 6).toString(16));

// Arroja una excepción: RangeError: Índice fuera de rango
console.log(buf.readUIntBE(1, 6).toString(16));
```

### buf.slice([start[, end]])

<!-- YAML
added: v0.3.0
-->

* `start` {integer} Dónde comenzará el nuevo `Buffer`. **Predeterminado:** `0`
* `end` {integer} Dónde terminará el nuevo `Buffer` (no inclusivo). **Predeterminado:** [`buf.length`]
* Devuelve: {Buffer}

Devuelve un nuevo `Buffer` que hace referencia a la misma memoria del original, pero se desplaza y se recorta por los índices `start` y `end`.

**Tenga en cuenta que modificar el nuevo segmento `Buffer` modificará la memoria en el `Buffer` original ya que la memoria asignada de los dos objetos se superpone.**

Ejemplo: Crear un `Buffer` con el alfabeto ASCII, tomar un segmento, y luego modificar un byte del `Buffer` original

```js
const buf1 = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 es el valor ASCII decimal para 'a'
  buf1[i] = i + 97;
}

const buf2 = buf1.slice(0, 3);

// Imprime: abc
console.log(buf2.toString('ascii', 0, buf2.length));

buf1[0] = 33;

// Imprime: !bc
console.log(buf2.toString('ascii', 0, buf2.length));
```

La especificación de índices negativos causa que el segmento se genere en relación al final de `buf`, en lugar de al inicio.

Ejemplos:

```js
const buf = Buffer.from('buffer');

// Imprime: buffe
// (Equivalente a buf.slice(0, 5))
console.log(buf.slice(-6, -1).toString());

// Imprime: buff
// (Equivalente a buf.slice(0, 4))
console.log(buf.slice(-6, -2).toString());

// Imprime: uff
// (Equivalente a buf.slice(1, 4))
console.log(buf.slice(-5, -2).toString());
```

### buf.swap16()

<!-- YAML
added: v5.10.0
-->

* Returns: {Buffer} Una referencia a `buf`.

Interpreta `buf` como un array de enteros de 16-bit sin signos y cambia el orden de bytes *en el lugar*. Arroja un `RangeError` si [`buf.length`] no es un múltiplo de 2.

Ejemplos:

```js
const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

// Imprime: <Buffer 01 02 03 04 05 06 07 08>
console.log(buf1);

buf1.swap16();

// Imprime: <Buffer 02 01 04 03 06 05 08 07>
console.log(buf1);


const buf2 = Buffer.from([0x1, 0x2, 0x3]);

// Arroja una excepción: RangeError: El tamaño del buffer debe ser un múltiplo de 16-bits
buf2.swap16();
```

### buf.swap32()

<!-- YAML
added: v5.10.0
-->

* Returns: {Buffer} Una referencia a `buf`.

Interpreta `buf` como un array de enteros de 32-bits sin signo y cambia el orden de bytes *en el lugar*. Arroja un `RangeError` si [`buf.length`] no es un múltiplo de 4.

Ejemplos:

```js
const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

// Imprime: <Buffer 01 02 03 04 05 06 07 08>
console.log(buf1);

buf1.swap32();

// Imprime: <Buffer 04 03 02 01 08 07 06 05>
console.log(buf1);


const buf2 = Buffer.from([0x1, 0x2, 0x3]);

// Arroja una excepción: RangeError: El tamaño del buffer debe ser un múltiplo de 32-bits
buf2.swap32();
```

### buf.swap64()

<!-- YAML
added: v6.3.0
-->

* Returns: {Buffer} Una referencia a `buf`.

Interpreta `buf` como un array de números de 64-bits y cambia el orden de los bytes *en el lugar*. Arroja un `RangeError` si [`buf.length`] no es un múltiplo de 8.

Ejemplos:

```js
const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

// Imprime: <Buffer 01 02 03 04 05 06 07 08>
console.log(buf1);

buf1.swap64();

// Imprime: <Buffer 08 07 06 05 04 03 02 01>
console.log(buf1);


const buf2 = Buffer.from([0x1, 0x2, 0x3]);

// Arroja una excepción: RangeError: El tamaño del buffer debe ser un múltiplo de 64-bits
buf2.swap64();
```

Tenga en cuenta que JavaScript no puede codificar enteros de 64-bits. Este método está diseñado para trabajar con floats de 64-bits.

### buf.toString([encoding[, start[, end]]])

<!-- YAML
added: v0.1.90
-->

* `encoding` {string} La codificación de caracteres a decodificar. **Predeterminado:** `'utf8'`
* `start` {integer} El offset de bytes en el cual comenzar la decodificación. **Predeterminado:** `0`
* `end` {integer} El offset de bytes en el cual detener la decodificación (no inclusivo). **Predeterminado:** [`buf.length`]
* Devuelve: {string}

Decodifica `buf` en una string de acuerdo a la codificación de caracteres especificados en `encoding`. `start` y `end` pueden pasarse para decodificar solo un subconjunto de `buf`.

Ejemplos:

```js
const buf1 = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 es el valor ASCII decimal para 'a'
  buf1[i] = i + 97;
}

// Imprime: abcdefghijklmnopqrstuvwxyz
console.log(buf1.toString('ascii'));

// Imprime: abcde
console.log(buf1.toString('ascii', 0, 5));


const buf2 = Buffer.from('tést');

// Imprime: 74c3a97374
console.log(buf2.toString('hex'));

// Imprime: té
console.log(buf2.toString('utf8', 0, 3));

// Imprime: té
console.log(buf2.toString(undefined, 0, 3));
```

### buf.toJSON()

<!-- YAML
added: v0.9.2
-->

* Devuelve: {Object}

Devuelve una representación JSON de `buf`. [`JSON.stringify()`] implícitamente llama a esta función al convertir en una string a una instancia de `Buffer`.

Ejemplo:

```js
const buf = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5]);
const json = JSON.stringify(buf);

// Imprime: {"type":"Buffer","data":[1,2,3,4,5]}
console.log(json);

const copy = JSON.parse(json, (key, value) => {
  return value && value.type === 'Buffer' ?
    Buffer.from(value.data) :
    value;
});

// Imprime: <Buffer 01 02 03 04 05>
console.log(copy);
```

### buf.values()

<!-- YAML
added: v1.1.0
-->

* Devuelve: {Iterator}

Crea y devuelve un [iterador](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) para valores de `buf` (bytes). Esta función es llamada automáticamente cuando un `Buffer` es usado en una declaración `for..of`.

Ejemplos:

```js
const buf = Buffer.from('buffer');

// Imprime:
//   98
//   117
//   102
//   102
//   101
//   114
for (const value of buf.values()) {
  console.log(value);
}

// Imprime:
//   98
//   117
//   102
//   102
//   101
//   114
for (const value of buf) {
  console.log(value);
}
```

### buf.write(string\[, offset[, length]\]\[, encoding\])

<!-- YAML
added: v0.1.90
-->

* `string` {string} String a ser escrita en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir la `string`. **Predeterminado:** `0`
* `length` {integer} Número de bytes a escribir. **Predeterminado:** `buf.length - offset`
* `encoding` {string} La codificación de caracteres del `string`. **Predeterminado:** `'utf8'`
* Devuelve: {integer} Número de bytes escritos.

Escribe el `string` en `buf` de `offset` de acuerdo a la codificación de caracteres en `encoding`. El parámetro `length` es el número de bytes a escribir. Si `buf` no contenía suficiente espacio para ajustar el string completo, solo una parte del `string` será escrita. Sin embargo, no se escribirán caracteres codificados parcialmente.

Ejemplo:

```js
const buf = Buffer.allocUnsafe(256);

const len = buf.write('\u00bd + \u00bc = \u00be', 0);

// Imprime: 12 bytes: ½ + ¼ = ¾
console.log(`${len} bytes: ${buf.toString('utf8', 0, len)}`);
```

### buf.writeDoubleBE(value, offset[, noAssert])

### buf.writeDoubleLE(value, offset[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `value` {number} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 8`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe el `value` al `buf` en el `offset` especificado con el formato endian especificado (`writeDoubleBE()` escribe big endian, `writeDoubleLE()` escribe little endian). `value` *debe* ser un doble válido de 64-bits. El comportamiento es indefinido cuando `value` es cualquier otra cosa distinta a un doble de 64-bits.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(8);

buf.writeDoubleBE(0xdeadbeefcafebabe, 0);

// Imprime: <Buffer 43 eb d5 b7 dd f9 5f d7>
console.log(buf);

buf.writeDoubleLE(0xdeadbeefcafebabe, 0);

// Imprime: <Buffer d7 5f f9 dd b7 d5 eb 43>
console.log(buf);
```

### buf.writeFloatBE(value, offset[, noAssert])

### buf.writeFloatLE(value, offset[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `value` {number} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 4`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe el `value` al `buf` en el `offset` especificado con el formato endian especificado (`writeFloatBE()` escribe big endian, `writeFloatLE()` escribe little endian). `value` *debe* ser un float válido de 32-bits. El comportamiento es indefinido cuando `value` es cualquier otra cosa distinta a un float de 32-bits.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(4);

buf.writeFloatBE(0xcafebabe, 0);

// Imprime: <Buffer 4f 4a fe bb>
console.log(buf);

buf.writeFloatLE(0xcafebabe, 0);

// Imprime: <Buffer bb fe 4a 4f>
console.log(buf);
```

### buf.writeInt8(value, offset[, noAssert])

<!-- YAML
added: v0.5.0
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 1`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe el `value` al `buf` en el `offset` especificado. `value` *debe* ser un entero válido de 8-bits con signo. El comportamiento es indefinido cuando `value` es cualquier otra cosa distinta a un entero de 8-bits con signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

`value` es interpretado y escrito como un entero con signo de complemento a dos.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(2);

buf.writeInt8(2, 0);
buf.writeInt8(-2, 1);

// Imprime: <Buffer 02 fe>
console.log(buf);
```

### buf.writeInt16BE(value, offset[, noAssert])

### buf.writeInt16LE(value, offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 2`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe `value` al `buf` en el `offset` especificado con el formato endian especificado (`writeInt16BE()` escribe big endian, `writeInt16LE()` escribe little endian). `value` *debe* ser un entero válido 16-bits con signo. El comportamiento es indefinido cuando `value` es cualquier otra cosa distinta a un entero de 16-bits con signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

`value` es interpretado y escrito como un entero con signo de complemento a dos.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(4);

buf.writeInt16BE(0x0102, 0);
buf.writeInt16LE(0x0304, 2);

// Imprime: <Buffer 01 02 04 03>
console.log(buf);
```

### buf.writeInt32BE(value, offset[, noAssert])

### buf.writeInt32LE(value, offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 4`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe `value` al `buf` en el `offset` especificado con el formato endian especificado (`writeInt32BE()` escribe big endian, `writeInt32LE()` escribe little endian). `value` *debe* ser un entero válido de 32-bits con signo. El comportamiento es indefinido cuando `value` es cualquier otra cosa distinta a un entero de 32-bits con signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

`value` es interpretado y escrito como un entero con signo de complemento a dos.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(8);

buf.writeInt32BE(0x01020304, 0);
buf.writeInt32LE(0x05060708, 4);

// Imprime: <Buffer 01 02 03 04 08 07 06 05>
console.log(buf);
```

### buf.writeIntBE(value, offset, byteLength[, noAssert])

### buf.writeIntLE(value, offset, byteLength[, noAssert])

<!-- YAML
added: v0.11.15
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - byteLength`.
* `byteLength` {integer} Número de bytes a escribir. Debe satisfacer: `0 < byteLength <= 6`.
* `noAssert` {boolean} ¿Saltar validación de `value`, `offset`, y `byteLength`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe `byteLength` bytes de `value` a `buf` en el `offset` especificado. Soporta hasta 48 bits de precisión. El comportamiento es definid cuando `value` es cualquier cosa distinta a un entero con signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(6);

buf.writeUIntBE(0x1234567890ab, 0, 6);

// Imprime: <Buffer 12 34 56 78 90 ab>
console.log(buf);

buf.writeUIntLE(0x1234567890ab, 0, 6);

// Imprime: <Buffer ab 90 78 56 34 12>
console.log(buf);
```

### buf.writeUInt8(value, offset[, noAssert])

<!-- YAML
added: v0.5.0
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 1`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe el `value` al `buf` en el `offset` especificado. `value` *debe* ser un entero válido de 8-bits sin signo. El comportamiento es indefinido cuando `value` es cualquier cosa distinta a un entero de 8-bits sin signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(4);

buf.writeUInt8(0x3, 0);
buf.writeUInt8(0x4, 1);
buf.writeUInt8(0x23, 2);
buf.writeUInt8(0x42, 3);

// Imprime: <Buffer 03 04 23 42>
console.log(buf);
```

### buf.writeUInt16BE(value, offset[, noAssert])

### buf.writeUInt16LE(value, offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 2`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe `value` al `buf` en el `offset` especificado con el formato endian especificado (`writeUInt16BE()` escribe big endian, `writeUInt16LE()` escribe little endian). `value` debe ser un entero válido de 16-bits sin signo. El comportamiento es indefinido cuando `value` es cualquier cosa distinta a un entero de 16-bits sin signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(4);

buf.writeUInt16BE(0xdead, 0);
buf.writeUInt16BE(0xbeef, 2);

// Imprime: <Buffer de ad be ef>
console.log(buf);

buf.writeUInt16LE(0xdead, 0);
buf.writeUInt16LE(0xbeef, 2);

// Imprime: <Buffer ad de ef be>
console.log(buf);
```

### buf.writeUInt32BE(value, offset[, noAssert])

### buf.writeUInt32LE(value, offset[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - 4`.
* `noAssert` {boolean} ¿Saltar validación de `value` y `offset`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe `value` al `buf` en el `offset` especificado con el formato endian especificado (`writeUInt32BE()` escribe big endian, `writeUInt32LE()` escribe little endian). `value` debe ser un entero válido de 32-bits sin signo. El comportamiento es indefinido cuando `value` es cualquier cosa distinta a un entero de 32-bits sin signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(4);

buf.writeUInt32BE(0xfeedface, 0);

// Imprime: <Buffer fe ed fa ce>
console.log(buf);

buf.writeUInt32LE(0xfeedface, 0);

// Imprime: <Buffer ce fa ed fe>
console.log(buf);
```

### buf.writeUIntBE(value, offset, byteLength[, noAssert])

### buf.writeUIntLE(value, offset, byteLength[, noAssert])

<!-- YAML
added: v0.5.5
-->

* `value` {integer} Número a ser escrito en `buf`.
* `offset` {integer} Número de bytes a omitir antes de comenzar a escribir. Debe satisfacer: `0 <= offset <= buf.length - byteLength`.
* `byteLength` {integer} Número de bytes a escribir. Debe satisfacer: `0 < byteLength <= 6`.
* `noAssert` {boolean} ¿Saltar validación de `value`, `offset`, y `byteLength`? **Predeterminado:** `false`
* Devuelve: {integer} `offset` más el número de bytes escritos.

Escribe `byteLength` bytes de `value` a `buf` en el `offset` especificado. Soporta hasta 48 bits de precisión. El comportamiento es indefinido cuando `value` es cualquier cosa distinta a un entero sin signo.

Configurar `noAssert` a `true` permite que la forma codificada de `value` se extienda más allá del final de `buf`, pero el comportamiento resultante es indefinido.

Ejemplos:

```js
const buf = Buffer.allocUnsafe(6);

buf.writeIntBE(0x1234567890ab, 0, 6);

// Imprime: <Buffer 12 34 56 78 90 ab>
console.log(buf);

buf.writeIntLE(0x1234567890ab, 0, 6);

// Imprime: <Buffer ab 90 78 56 34 12>
console.log(buf);
```

## buffer.INSPECT_MAX_BYTES

<!-- YAML
added: v0.5.4
-->

* {integer} **Predeterminado:** `50`

Devuelve el número máximo de bytes que serán devueltos cuando `buf.inspect()` es llamado. Esto puede ser reemplazado por módulos de usuario. Vea [`util.inspect()`] para más detalles sobre el comportamiento de `buf.inspect()`.

Tenga en cuenta que esto es una propiedad en el módulo `buffer` tal como es devuelto por `require('buffer')`, no en el `Buffer` global o una instancia de `Buffer`.

## buffer.kMaxLength

<!-- YAML
added: v3.0.0
-->

* {integer} El tamaño más grande permitido para una sola instancia de `Buffer`.

En arquitecturas de 32-bits, este valor es `(2^30)-1` (~1GB). En arquitecturas de 64-bits, este valor es `(2^31)-1` (~2GB).

## Clase: SlowBuffer

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobado: Use [`Buffer.allocUnsafeSlow()`], en su lugar.

Devuelve un `Buffer` no-agrupado.

Para evitar la sobrecarga de recolección de basura al crear muchas instancias de `Buffer` asignadas individualmente, por defecto, las asignaciones debajo de 4KB se recortan desde un único objecto asignado más grande. Este enfoque mejora tanto el rendimiento como el uso de la memoria, ya que v8 no necesita rastrear y limpiar tantos objetos `Persistent`.

En el caso donde un desarrollador necesite retener una pequeña porción de memoria de un pool por un tiempo indeterminado, puede que sea apropiado crear una instancia de `Buffer` no-agrupado utilizando `SlowBuffer` y luego crear una copia de los bits relevantes.

Ejemplo:

```js
// Necesita mantener unos pequeños pedazos de memoria
const store = [];

socket.on('readable', () => {
  const data = socket.read();

  // Asignar para los datos retenidos
  const sb = SlowBuffer(10);

  // Copiar los datos dentro de la nueva asignación
  data.copy(sb, 0, 0, 10);

  store.push(sb);
});
```

El uso de `SlowBuffer` debe ser empleado solo como último recurso *después* de que un desarrollador haya observado una retención de memoria indebida en sus aplicaciones.

### new SlowBuffer(size)

<!-- YAML
deprecated: v6.0.0
-->

> Estabilidad: 0 - Desaprobado: Use [`Buffer.allocUnsafeSlow()`], en su lugar.

* `size` {integer} La longitud deseada del nuevo `SlowBuffer`.

Asigna un nuevo `SlowBuffer` de `size` bytes. El `size` debe ser menor o igual al valor de [`buffer.kMaxLength`]. De lo contrario, un [`RangeError`] es arrojado. Un `Buffer` de longitud cero será creado si `size <= 0`.

La memoria subyacente para instancias de `SlowBuffer` *no está inicializada*. Los contenidos de un `SlowBuffer` recién creado son desconocidos y pueden contener datos sensibles. Use [`buf.fill(0)`][`buf.fill()`] para inicializar un `SlowBuffer` a ceros.

Ejemplo:

```js
const SlowBuffer = require('buffer').SlowBuffer;

const buf = new SlowBuffer(5);

// Imprime: (contents may vary): <Buffer 78 e0 82 02 01>
console.log(buf);

buf.fill(0);

// Imprime: <Buffer 00 00 00 00 00>
console.log(buf);
```