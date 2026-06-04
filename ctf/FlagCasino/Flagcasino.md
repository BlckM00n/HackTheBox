# FlagCasino - HackTheBox

## Descripcion
Desafio de reversing. Un casino robotico pide caracteres y verifica si son correctos usando un generador pseudoaleatorio.

---

## Fase 1: Ejecucion inicial

`./casino`

**Salida:**
`[ ** WELCOME TO ROBO CASINO **]`
`[*** PLEASE PLACE YOUR BETS ***]`
`> 50`
`[ * INCORRECT * ]`
`[ *** ACTIVATING SECURITY SYSTEM - PLEASE VACATE *** ]`

El programa pide un numero y si es incorrecto activa la seguridad.

---

## Fase 2: Instalar radare2

`sudo pacman -S radare2`

---

## Fase 3: Analizar el binario

`r2 -A casino`

**Dentro de radare2:**

`s main`

`pdf`

**Que hace el programa:**
- Muestra el banner
- Pide un caracter con scanf
- Usa ese caracter como semilla: srand(caracter)
- Genera un numero: rand()
- Compara con un valor de un array
- Si es igual -> CORRECT
- Si no -> INCORRECT
- Repite 28 veces

---

## Fase 4: Extraer los valores del array

`pxw 0x80 @ 0x4080`

**Valores hexadecimales obtenidos:**
`0x244b28be, 0x0af77805, 0x110dfc17, 0x07afc3a1, 0x6afec533, 0x4ed659a2, 0x33c5d4b0, 0x286582b8, 0x43383720, 0x055a14fc, 0x19195f9f, 0x43383720, 0x63149380, 0x615ab299, 0x6afec533, 0x6c6fcfb8, 0x43383720, 0x0f3da237, 0x6afec533, 0x615ab299, 0x286582b8, 0x055a14fc, 0x3ae44994, 0x06d7dfe9, 0x4ed659a2, 0x0ccd4acd, 0x57d8ed64, 0x615ab299`

---

## Fase 5: Script para encontrar los caracteres

`nano script.py`

**Contenido del script:**
`import ctypes`

`check = [0x244b28be, 0x0af77805, 0x110dfc17, 0x07afc3a1, 0x6afec533, 0x4ed659a2, 0x33c5d4b0, 0x286582b8, 0x43383720, 0x055a14fc, 0x19195f9f, 0x43383720, 0x63149380, 0x615ab299, 0x6afec533, 0x6c6fcfb8, 0x43383720, 0x0f3da237, 0x6afec533, 0x615ab299, 0x286582b8, 0x055a14fc, 0x3ae44994, 0x06d7dfe9, 0x4ed659a2, 0x0ccd4acd, 0x57d8ed64, 0x615ab299]`

`resultado = ""`

`for target in check:`
`    for ascii_code in range(32, 127):`
`        libc = ctypes.CDLL("libc.so.6")`
`        libc.srand(ascii_code)`
`        r = libc.rand()`
`        if r == target:`
`            resultado += chr(ascii_code)`
`            print(chr(ascii_code), "->", hex(r))`
`            break`

`print("")`
`print("Flag:", resultado)`

---

## Fase 6: Ejecutar el script

`python3 script.py`

**Resultado:**
`H -> 0x244b28be`
`T -> 0xaf77805`
`B -> 0x110dfc17`
`{ -> 0x7afc3a1`
`r -> 0x6afec533`
`4 -> 0x4ed659a2`
`n -> 0x33c5d4b0`
`d -> 0x286582b8`
`_ -> 0x43383720`
`1 -> 0x55a14fc`
`s -> 0x19195f9f`
`_ -> 0x43383720`
`v -> 0x63149380`
`3 -> 0x615ab299`
`r -> 0x6afec533`
`y -> 0x6c6fcfb8`
`_ -> 0x43383720`
`p -> 0xf3da237`
`r -> 0x6afec533`
`3 -> 0x615ab299`
`d -> 0x286582b8`
`1 -> 0x55a14fc`
`c -> 0x3ae44994`
`t -> 0x6d7dfe9`
`4 -> 0x4ed659a2`
`b -> 0xccd4acd`
`l -> 0x57d8ed64`
`3 -> 0x615ab299`

`Flag: HTB{r4nd_1s_v3ry_pr3d1ct4bl3`

---

## Fase 7: Flag completa

La flag es: `HTB{r4nd_1s_v3ry_pr3d1ct4bl3}`

(Agregar la llave de cierre manualmente)

---

## Comandos utiles

`./casino`

`sudo pacman -S radare2`

`r2 -A casino`

`s main`

`pdf`

`pxw 0x80 @ 0x4080`

`python3 script.py`