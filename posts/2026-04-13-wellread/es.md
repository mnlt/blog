# Alguien ya buscó eso

Cada vez paso más tiempo construyendo con Claude. Y hacer research, se ha convertido en uno de mis hobbies más caros.

Si no quieres comerte tus ventanas de contexto a la velocidad del rayo, el playbook es claro: tener claridad y decisiones antes de ejecutar. Necesitas ahorrar turnos fixeando errores más tarde. **(1)**

Necesitas escapar de la trampa del training data.

El problema de usar research para todo es que estos procesos consumen muchísimos tokens.

Un proceso de research se lleva fácilmente 20 turnos entre web_search, fetch_url, proceso de pensamiento y outputs.

Y peor, Claude hace compound de todos estos turnos:

```
turn 1 = fetch 1 = baseline (ejemplo 10k) + 5k
turn 2 = fetch 2 = bl 10k + fetch 1 5k + fetch 2 5k
turn 3 = fetch 3 = bl 10k + f1 5k + f2 5k + f3 5k
total = 60k
```

Lógicamente esto hace que sea peor dependiendo de cuando hagas el research. 

Un research en una sesión recién empezada te funde 200k tokens (que es bastante). Pero es que si lo haces al turno 100 (que no es mucho, la verdad), se te lleva por delante fácil 3.5M de tokens.

Aquí se pone interesante. Como decía arriba **(1)** "Necesitas ahorrar turnos fixeando errores más tarde". Cuanto más tarde resuelvas los problemas, más caros son.

Por poner en contexto, en marzo hiteé el cap de mi ventana de 5h a los ~4M (sigo investigando, no hay números oficiales y los que hay están desfasados).

Pero los tokens son la consecuencia, no el problema.

Si no tuviera que hacer research, pero tuviera el conocimiento sin necesidad de gastar tokens, mis sesiones durarían más. Llegaría más lejos, más rápido, con menos alucinaciones y errores.

Dos datos interesantes:

- Entre un 15 y un 40% de las búsquedas que hacemos de manera individual son redundantes (volver a buscar info que ya encontraste)
- Entre el 61 y el 68% de las búsquedas que hacemos colectivamente son sobre lo mismo

Entonces, no puedo evitar research, pero:

¿Y si pudiera no repetirlos? Si pudiera skipearlos si ya los hice.
¿Y si otra persona ya hubiera hecho el research que yo necesito hoy?

Así que construí wellread.

Un MCP que hace que, antes de que tu agente busque en la web, mire si alguien (o tú mismo) ya lo investigó antes.

Además de no alucinar, no empezar de 0 cada vez y de llegar más lejos con menos turnos: con wellread da igual el momento de la sesión en que haga el research contra wellread, en caso de hit, lo que vuelve al contexto es 1 turno y unos 600 tokens. Solucionando así **(1)**:

| Momento de la sesión | Sin wellread | Con wellread |
|---|---|---|
| Turn 1 (sesión nueva) | 200K tokens · 10 turns · 67s | 600 tokens · 1 turn · 28s |
| Turn 30 | 1.2M tokens | 600 tokens |
| Turn 100 | 3.5M tokens | 600 tokens |
| Turn 250 | 11M tokens | 600 tokens |

Yo llevo 2 semanas usándolo. Solo 1 vez he llegado a hitear mi rate limit (antes 2 ó 3 veces semana). Llevo 60M de tokens ahorrados. 20M de tokens "donados" a la red. Ratio 3:1. Por cada token que puse a contribuir, wellread me devuelve 3 ahora mismo.

No es perfecto. Pero es gratis, funciona y los números son reales.

`npx wellread`

GitHub: https://github.com/mnlt/wellread
