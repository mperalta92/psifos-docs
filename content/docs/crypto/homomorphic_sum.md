---
title: "Suma Homomórfica"
slug: homomorphic-property
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 830
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
La **suma homomórfica** es la propiedad criptográfica que permite a
UParticipa contar los votos sin necesidad de descifrarlos
individualmente. Es la propiedad que habilita el conteo de las
votaciones de [tipo
simple]({{< ref "/docs/functionalities/voting_types.md" >}}), donde se
aplica directamente sobre los votos encriptados para producir el
precómputo descrito en [Cierre de Elección y
Precómputo]({{< ref "/docs/voting-steps/tally_compute.md" >}}).

## Propiedad homomórfica de ElGamal

UParticipa cifra los votos con **ElGamal exponencial**, una variante del
esquema [ElGamal]({{< ref "/docs/crypto/encryption.md" >}}) en la que
el plaintext `v` se codifica como `g^v` antes de cifrar (`g` es el
generador del grupo elegido para la elección). Bajo esta codificación,
el producto componente a componente de dos ciphertexts corresponde al
ciphertext de la **suma** de sus plaintexts.

Concretamente, si dos votos se cifran como

```
C₁ = (g^r₁, g^v₁ · pk^r₁)
C₂ = (g^r₂, g^v₂ · pk^r₂)
```

su producto queda

```
C₁ · C₂ = (g^(r₁+r₂), g^(v₁+v₂) · pk^(r₁+r₂))
```

que es precisamente el ciphertext del plaintext `v₁ + v₂` bajo la
misma clave pública, con randomness `r₁ + r₂`. Repitiendo esta
multiplicación sobre todos los votos válidos, se obtiene un único
ciphertext que contiene la **suma total** en el exponente sin haber
expuesto ningún voto individual.

## Aplicación al escrutinio

En las votaciones de tipo simple, cada votante envía, para cada
candidatura disponible, un ciphertext que codifica un bit
(`0` si no marcó esa candidatura, `1` si la marcó). El servidor
multiplica todos los ciphertexts asociados a una misma candidatura y
obtiene el ciphertext del **número de votos** que esa candidatura
recibió. Si la elección utiliza ponderaciones, cada bit se eleva (en el
exponente) a la ponderación correspondiente del votante antes de
multiplicar.

Para asegurar que sólo entran al cómputo votos bien formados, cada voto
va acompañado de una prueba criptográfica
([Zero-Knowledge Proof]({{< ref "/docs/crypto/zkp.md" >}})) que
demuestra, sin revelar la selección, que el plaintext está dentro del
rango admitido por la pregunta.

## Recuperación del resultado y limitaciones

El resultado de la multiplicación homomórfica no es directamente un
número, sino una potencia `g^total`. Para recuperar `total`, el sistema
utiliza una **tabla de logaritmos discretos** precomputada que mapea
cada `g^k` con su correspondiente `k`, recorriendo el rango de valores
plausibles para la elección (que depende del número total de votantes y
de la ponderación máxima).

### Por qué un mayor número de candidaturas obliga a usar Mixnet

La suma homomórfica funciona haciendo crecer un cómputo independiente
**por cada candidatura**: cada votante envía un ciphertext por
candidatura (que codifica el bit `0` o `1`), y el servidor los
multiplica entre sí para obtener la suma cifrada de votos que recibe
esa candidatura. Hasta ahí, agregar candidaturas no rompe el modelo;
lo que sí crece de forma incómoda es la maquinaria de **validación
criptográfica** que tiene que acompañar a cada voto:

- Por cada candidatura, el votante debe adjuntar una [prueba en cero
  conocimiento]({{< ref "/docs/crypto/zkp.md" >}}) demostrando que su
  ciphertext codifica un `0` o un `1`, y no, por ejemplo, un `1000`
  que multiplicaría artificialmente el resultado de esa candidatura.
  La cantidad y el tamaño de estas pruebas crecen linealmente con el
  número de candidaturas.
- A eso se suma una prueba adicional, de carácter **disjuntivo**, que
  demuestra que el total de selecciones que hizo el votante respeta el
  rango admitido por la pregunta (por ejemplo, "seleccionar entre 1 y
  3 candidaturas"). Cuando ese rango se hace amplio (algo natural si
  hay muchas candidaturas), el número de "ramas" de la prueba
  disjuntiva aumenta y, con él, el costo de generarla y verificarla.

Conviene detenerse en cómo funciona esta segunda prueba. Una **prueba
disjuntiva** (también llamada *OR proof*) permite al votante demostrar
que su selección satisface **una de varias condiciones posibles**, sin
revelar cuál. Cada condición admisible (por ejemplo, "el votante
seleccionó 1 candidatura", "el votante seleccionó 2 candidaturas", …)
constituye una **rama** independiente de la prueba. Bajo la
construcción estándar, descrita por R. Cramer, I. Damgård y B.
Schoenmakers en *Proofs of Partial Knowledge and Simplified Design of
Witness Hiding Protocols* (CRYPTO '94, LNCS 839, pp. 174–187, DOI:
[10.1007/3-540-48658-5_19](https://doi.org/10.1007/3-540-48658-5_19)),
cada rama consta de un compromiso `a`, un desafío `c` y una respuesta
`z` que deben satisfacer una ecuación pública verificable.

La idea central es que el votante puede **simular** una rama falsa
siempre que pueda elegir libremente su desafío: elige `c` y `z` al
azar y deriva `a` "hacia atrás", de modo que la ecuación se cumpla por
construcción. Para la única rama que efectivamente vale, en cambio,
únicamente él (que conoce el secreto del cifrado) puede producir una
respuesta `z` consistente con un `c` que ya no controla. Lo que
amarra todas las ramas entre sí es un **desafío global**, calculado
con la heurística de [Fiat–Shamir]({{< ref "/docs/crypto/zkp.md" >}})
sobre los compromisos `a` y los datos públicos de la elección: la
suma de los `c` parciales debe coincidir con ese desafío global, lo
que obliga a derivar el `c` de la rama verdadera a partir de los `c`
elegidos para las ramas simuladas. El resultado es una prueba **no
interactiva** en la que un verificador (el servidor o cualquier
[Verificador Externo]({{< ref "/docs/roles/verifier.md" >}})) puede
chequear la ecuación de cada rama por separado, sin descubrir cuál
fue la verdadera.

La consecuencia para la operación es directa: cada rama adicional
añade un bloque propio de compromisos, desafíos y respuestas al voto,
de modo que tanto la generación como la verificación cuestan **lineal
en el número de ramas**. Por eso, a medida que una pregunta admite
más candidaturas (y con ellas más selecciones posibles que cubrir),
las pruebas disjuntivas que sostienen cada voto se vuelven
progresivamente más pesadas en tiempo y en tamaño.

Estas operaciones se ejecutan en el **navegador del votante**, durante
la fase de cifrado dentro de la Cabina de Votación. Mantener una
experiencia fluida en navegadores y dispositivos modernos ha llevado a
fijar el tope de la Votación Simple en torno a una docena de
candidaturas. Más allá de ese punto, el tiempo de generación de las
pruebas en el cliente y el costo de verificarlas en el servidor (y en
cada [Verificador Externo]({{< ref "/docs/roles/verifier.md" >}}) que
las revise post-elección) hacen que el enfoque homomórfico deje de
ser práctico, aun cuando matemáticamente sigue siendo correcto.

[Mixnet]({{< ref "/docs/crypto/mixnet.md" >}}) elude este crecimiento
porque cambia la naturaleza del cómputo. En lugar de **sumar
ciphertexts por candidatura** con pruebas que escalan con el número de
opciones, se **baraja y re-cifra el conjunto completo de votos** y
luego se desencripta cada uno por separado. El costo del escrutinio
pasa a depender principalmente del número de votos —no del número de
candidaturas por pregunta— y se vuelven posibles codificaciones del
voto que la suma homomórfica binaria no puede representar de forma
económica, como un *ranking* de candidaturas.
