# Programación híbrida C y ASM
### Enlazado, nombres de símbolos y visibilidad


---

## 1. Introducción

Cuando se trabaja en sistemas embebidos, drivers o cualquier cosa que necesite exprimir al máximo el hardware, tarde o temprano C se queda corto. Hay cosas que simplemente no se pueden hacer desde C: instrucciones específicas del procesador, manejo directo de interrupciones, rutinas optimizadas a mano que el compilador nunca generaría igual de bien. Ahí es donde entra el ensamblador.

El tema es que C y ASM no se entienden solos. Si escribes una función en ensamblador y quieres usarla desde C (o al revés), tienen que ponerse de acuerdo en varias cosas: cómo se llama esa función a nivel de símbolo, si es visible desde afuera o se queda escondida, cómo se pasan los argumentos, y cómo todo eso termina uniéndose en un solo ejecutable. Si algo de esto falla, el resultado casi siempre es el mismo dolor de cabeza: un `undefined reference` que no dice mucho sobre qué salió mal.

Esta investigación va sobre eso: entender qué pasa "por detrás" cuando se mezclan estos dos lenguajes.

### 1.1 ¿Qué es la programación híbrida?

La programación híbrida es básicamente construir un programa usando más de un lenguaje a la vez, donde cada parte se compila (o ensambla) por separado y luego todo se une en un solo binario mediante el enlazador.

En el caso de C y ASM, la idea es escribir en ensamblador solo las partes que realmente lo necesitan —por velocidad, por control del hardware, o porque hay instrucciones que C no puede expresar directamente— y dejar el resto del programa en C, que es mucho más rápido de escribir y de mantener.

Algunas cosas que la caracterizan:

- Cada archivo se compila por su cuenta, con su propia herramienta (`gcc` para C, `as` o `nasm` para ASM).
- La unión ocurre después, en el enlazado, no durante la compilación.
- Como no comparten representación interna, C y ASM necesitan ponerse de acuerdo explícitamente en nombres, visibilidad y forma de pasar datos. No hay manera automática de que "adivinen" esto entre sí.

¿Dónde se usa en la práctica? En sistemas operativos y drivers, en bibliotecas de criptografía o compresión que aprovechan instrucciones SIMD, en motores de videojuegos donde cada ciclo de CPU importa, y en firmware que necesita tocar registros de hardware directamente.

---

## 2. Desarrollo

### 2.1 Cómo se enlaza todo

El proceso, en resumen, es este:

```text
archivo.c   --(gcc -c)-->   archivo.o  --+
                                          +--(ld / gcc)--> ejecutable
archivo.asm --(as / nasm)--> archivo.o  --+
```

Cada archivo fuente termina, por separado, en un archivo objeto (`.o`). Ese objeto trae el código máquina y también una tabla de símbolos: qué funciones/variables define, y cuáles necesita de otros archivos (esas quedan marcadas como indefinidas, `UND`).

Después llega el enlazador y junta todos los `.o`, tratando de emparejar cada símbolo indefinido con su definición en algún otro archivo. Cuando no la encuentra, aparece esto:

```console
/usr/bin/ld: main.o: undefined reference to `suma'
```

Y aquí va lo importante: ese error casi nunca significa que la lógica esté mal. Casi siempre es que el nombre del símbolo no coincide, o que el símbolo nunca se hizo visible hacia afuera.

### 2.2 Nombres de símbolos (name mangling)

El compilador de C, a diferencia de C++, no le cambia el nombre a las funciones. Si escribes `int suma(int, int)`, el símbolo que queda en el objeto es literalmente `suma`... aunque hay una excepción histórica: el prefijo de guion bajo, que varía según la plataforma.

| Plataforma / formato objeto | Símbolo para `int suma()` |
|---|---|
| Linux · ELF · x86-64 | `suma` |
| Windows · PE · 32 bits (`__cdecl`) | `_suma` |
| Windows · PE · 64 bits | `suma` |
| macOS · Mach-O | `_suma` (siempre con guion bajo) |

En C++ la cosa cambia bastante: el compilador sí decora el nombre metiendo información de tipos, clase y namespace (esto se llama *name mangling*). Por ejemplo, `int suma(int,int)` puede terminar como `_Z4sumaii`. Por eso, si el proyecto es C++ y necesita hablar con ASM, hay que forzar el nombre plano:

```cpp
extern "C" int suma(int a, int b);   // evita el name mangling
```

En resumen: el nombre que se exporta desde el `.asm` con `.global` tiene que coincidir letra por letra —incluyendo guiones bajos— con lo que el compilador de C espera para ese `extern`. Un solo carácter distinto y el linker no los va a relacionar.

### 2.3 Visibilidad de símbolos

No todos los símbolos deberían ser visibles desde afuera, y en ASM eso se controla con directivas:

| Directiva ASM (GNU `as`) | Qué hace | Equivalente en C |
|---|---|---|
| `.global` / `.globl` | Exporta el símbolo, queda visible para el linker y otros `.o` | función/variable normal (no `static`) |
| *(sin directiva)* | Por defecto es local, solo se ve dentro del mismo archivo | `static` |
| `.local` | Fuerza el alcance local explícitamente | `static` |
| `.hidden` | Se ve entre los objetos del mismo binario, pero no se exporta afuera de una librería compartida | `__attribute__((visibility("hidden")))` |
| `.weak` | Símbolo "débil": otra definición fuerte lo puede reemplazar sin generar conflicto | `__attribute__((weak))` |

En bibliotecas compartidas esto no es un detalle menor: ocultar los símbolos que no necesitan exponerse evita colisiones de nombres entre librerías, reduce la superficie atacable y hasta acelera la carga dinámica, porque hay menos símbolos que resolver.

### 2.4 La convención de llamada (ABI)

Nombrar bien el símbolo y hacerlo visible no es suficiente si los datos no viajan por donde el otro lado espera. Aquí es donde entra la ABI (interfaz binaria de aplicación), que define en qué registros van los argumentos y el retorno:

| Elemento | System V AMD64 (Linux/macOS) | Microsoft x64 (Windows) |
|---|---|---|
| Argumentos enteros/punteros | `rdi, rsi, rdx, rcx, r8, r9` | `rcx, rdx, r8, r9` |
| Retorno | `rax` | `rax` |
| Registros que debe preservar la función llamada | `rbx, rbp, r12–r15` | `rbx, rbp, rdi, rsi, r12–r15` |
| Extra | zona roja de 128 bytes | *shadow space* de 32 bytes |

Esto explica por qué un mismo archivo `.asm` no funciona igual en Windows que en Linux, aunque el nombre del símbolo sea idéntico: los registros donde se espera cada argumento son distintos.

### 2.5 C vs. ASM, en resumen

| Aspecto | C (compilador) | ASM (ensamblador) |
|---|---|---|
| Decoración de nombres | Ninguna (salvo el prefijo `_` según plataforma) | Ninguna, el nombre escrito es literal |
| Exportar un símbolo | Automático si no es `static` | Hay que hacerlo a mano con `.global` |
| Ocultar un símbolo | `static` o `visibility("hidden")` | No poner `.global`, o usar `.local`/`.hidden` |
| ¿Verifica tipos entre módulos? | Sí, gracias a los prototipos | No, el linker solo mira nombres |
| Convención de llamada | La genera el compilador | Hay que respetarla manualmente |
| Error típico | Prototipo mal declarado | Símbolo mal exportado o registros mal usados |

### 2.6 ASM en archivo separado vs. ASM en línea

Vale la pena mencionar que no siempre se trabaja con archivos `.asm` sueltos: también existe el ensamblador insertado directamente en el código C (`__asm__`). Cambia bastante el panorama:

| Criterio | Archivo separado | ASM en línea |
|---|---|---|
| Manejo de símbolos | Manual, hay que declarar todo | Lo maneja el compilador |
| Convención de llamada | Hay que implementarla a mano | Se resuelve con *constraints* |
| Optimización | El compilador no toca ese código | Se integra con el optimizador |
| Cuándo conviene | Rutinas completas, arranque, manejo de interrupciones | Una o dos instrucciones sueltas (por ejemplo `rdtsc`) |

---

## 3. Conclusión

Al final, mezclar C con ASM no es solo cuestión de escribir código en dos lenguajes distintos y esperar que funcione: hay todo un acuerdo silencioso que tiene que respetarse para que el binario final tenga sentido.

El enlazador no sabe nada de lógica ni de tipos, solo une nombres de símbolos entre archivos objeto, así que cualquier descuido ahí —una letra distinta, un guion bajo faltante— hace que todo se rompa recién al momento de enlazar, cuando ya es tarde para darse cuenta a simple vista. Los nombres de símbolos son literalmente el punto donde C y ASM se encuentran, y por eso hay que tener cuidado con las diferencias entre plataformas y con el name mangling si se está trabajando con C++. La visibilidad, por su parte, decide qué queda expuesto y qué se mantiene privado, algo que en ASM se controla a mano con directivas como `.global` o `.hidden`, pero que en C ya viene resuelto con `static`. Y la ABI completa el cuadro: define cómo se mueven los datos entre funciones, y nadie la verifica automáticamente cuando se combina C con ASM, así que ahí el error es enteramente responsabilidad de quien escribe el código.

En el fondo, entender estos cuatro puntos —enlazado, nombres, visibilidad y ABI— es lo que separa a alguien que solo copia código ensamblador de alguien que realmente entiende por qué ese código funciona (o por qué deja de funcionar apenas cambia algo pequeño). Y más allá de evitar errores raros de compilación, esto también ayuda a escribir código de bajo nivel más ordenado, más seguro y, cuando se puede, más portable entre plataformas.

## Referencias

- *System V AMD64 ABI Specification*
- Manual de GNU `as` (GNU Assembler)
- *Itanium C++ ABI* — name mangling
- Documentación de `ld` (GNU linker)
