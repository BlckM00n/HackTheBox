# LootStash - HackTheBox

## Descripcion
Un binario que simula un monton de items. Hay que encontrar el item especifico que contiene la flag.

---

## Fase 1: Identificar el archivo

`file stash`

**Resultado:**
`stash: ELF 64-bit LSB pie executable, x86-64, dynamically linked, not stripped`

---

## Fase 2: Dar permisos de ejecucion

`chmod +x stash`

---

## Fase 3: Ejecutar el programa varias veces

`./stash`

**Salida 1:**
`Diving into the stash - let's see what we can find.`
`.....`
`You got: 'Nexus, Trinket of the Basilisk'. Now run, before anyone tries to steal it!`

`./stash`

**Salida 2:**
`Diving into the stash - let's see what we can find.`
`.....`
`You got: 'Clemence, Breaker of Cunning'. Now run, before anyone tries to steal it!`

`./stash`

**Salida 3:**
`Diving into the stash - let's see what we can find.`
`.....`
`You got: 'Nightkiss, Terror of the South'. Now run, before anyone tries to steal it!`

Cada ejecucion da un item diferente aleatorio.

---

## Fase 4: Buscar la flag con strings

`strings stash | grep -i "flag"`

**Resultado:**
(No aparece nada)

`strings stash | grep "HTB"`

**Resultado:**
`HTB{n33dl3_1n_a_l00t_stack}`

La flag esta dentro de la lista de items del binario.

---

## Fase 5: Intentar obtener la flag ejecutando el programa (opcional)

`for i in {1..500}; do ./stash; done | grep "HTB"`

Esto ejecuta el programa 500 veces y filtra la salida buscando "HTB".

---

## Fase 6: Flag

`HTB{n33dl3_1n_a_l00t_stack}`

---

## Comandos utiles

`file stash`

`chmod +x stash`

`./stash`

`strings stash | grep "HTB"`

`strings stash | grep -i "flag"`

`for i in {1..500}; do ./stash; done | grep "HTB"`