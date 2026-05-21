---
title: "Mixnet"
slug: mixnet
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 840
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
La **Mixnet** es el método de escrutinio que UParticipa utiliza cuando
la [suma homomórfica]({{< ref "/docs/crypto/homomorphic_sum.md" >}})
deja de ser práctica: en las votaciones de [tipo masivo o por
ranking]({{< ref "/docs/functionalities/voting_types.md" >}}), donde
o bien el número de candidaturas excede la cota práctica del enfoque
homomórfico, o bien el voto no puede expresarse como una colección de
bits independientes. Esta página describe el principio detrás de los
mixnets, cómo se mantiene la verificabilidad de su salida y cómo se
incorporan al [cómputo del
precómputo]({{< ref "/docs/voting-steps/tally_compute.md" >}}) de la
elección.

## Concepto

Un **mixnet** es una secuencia de servidores —los *mix servers* o
*nodos de mezcla*— que recibe un conjunto de ciphertexts, los procesa
nodo por nodo, y al final entrega un conjunto reordenado y de aspecto
totalmente distinto que ya no puede correlacionarse con la entrada
original. Cada nodo, al recibir el lote del nodo anterior, ejecuta
dos operaciones:

- **Re-cifra** cada ciphertext con nueva aleatoriedad, modificando
  los bytes del ciphertext sin alterar el plaintext que codifica.
- **Permuta** los ciphertexts re-cifrados en un orden secreto, y pasa
  el lote al siguiente nodo.

Mientras al menos uno de los nodos sea honesto y no comparta su
permutación con los demás, el ordenamiento conjunto del lote queda
ininteligible: la asociación voto ↔ votante, que sólo existe a
través del orden y el aspecto de los ciphertexts a la entrada,
desaparece antes de que ningún voto sea desencriptado.

La técnica fue introducida por D. Chaum en *Untraceable Electronic
Mail, Return Addresses, and Digital Pseudonyms* (Communications of
the ACM, vol. 24, no. 2, 1981, pp. 84–90, DOI:
[10.1145/358549.358563](https://doi.org/10.1145/358549.358563)) como
mecanismo de anonimato para correo electrónico, y fue adaptada
posteriormente al escrutinio de elecciones electrónicas remotas.

## Re-cifrado en ElGamal

UParticipa cifra los votos con [ElGamal
exponencial]({{< ref "/docs/crypto/encryption.md" >}}), un esquema
que admite **re-cifrar** un ciphertext sin desencriptarlo. Dado un
ciphertext `C = (A, B) = (g^r, v · pk^r)`, un nodo que elige una
aleatoriedad fresca `s` puede calcular

```
C' = (A · g^s, B · pk^s) = (g^(r+s), v · pk^(r+s))
```

que es un ciphertext válido del mismo plaintext `v` bajo la misma
clave pública, con aleatoriedad `r + s`. Visualmente, `C` y `C'`
son elementos distintos del grupo y resulta imposible (sin la clave
privada) determinar que codifican el mismo voto. Este es el
ingrediente algebraico que permite a un nodo "modificar el aspecto"
de los ciphertexts antes de permutarlos.

## Pruebas verificables de shuffle

Re-cifrar y permutar es sencillo; lo difícil es **demostrar** que un
nodo lo hizo correctamente, es decir, que el conjunto de salida es
una permutación válida del conjunto de entrada bajo re-cifrado, sin
alteraciones al multiset de plaintexts subyacente. Un nodo malicioso
podría, en principio, sustituir, descartar o duplicar votos durante
su barajado.

Para evitarlo, cada nodo acompaña su salida con una **prueba en cero
conocimiento de shuffle**, que demuestra exactamente eso —que los
ciphertexts de salida son re-cifrados del conjunto de entrada bajo
alguna permutación— sin revelar cuál es esa permutación ni cuáles
fueron las aleatoriedades nuevas. La construcción estándar para este
tipo de prueba fue descrita por C. A. Neff en *A Verifiable Secret
Shuffle and its Application to E-Voting* (8th ACM Conference on
Computer and Communications Security, CCS '01, pp. 116–125, DOI:
[10.1145/501983.502000](https://doi.org/10.1145/501983.502000)).

La construcción de Neff se apoya en una observación algebraica
elegante: una permutación válida del lote no cambia el **multiset**
de plaintexts subyacente, sólo redistribuye sus posiciones y refresca
las aleatoriedades del cifrado. Esto puede expresarse como una
identidad polinomial: si se construye un polinomio
`P_X(t) = ∏ (t − x_i)` cuyas raíces son los plaintexts de entrada, y
otro polinomio `P_Y(t) = ∏ (t − y_i)` cuyas raíces son los plaintexts
de salida, entonces `P_X = P_Y` si y sólo si ambos multisets
coinciden.

El nodo, sin revelar las raíces, demuestra esta igualdad de
polinomios mediante una variante encriptada del **lema de
Schwartz–Zippel**. El verificador —o, en la versión no interactiva,
la heurística de [Fiat–Shamir]({{< ref "/docs/crypto/zkp.md" >}})—
fija un escalar aleatorio `t`; el nodo construye en el grupo ElGamal
una serie de **compromisos** sobre las evaluaciones parciales de los
productos `(t − x_i)` y `(t − y_i)`, y responde a desafíos sobre
ellos al estilo de los protocolos sigma de Schnorr. Si los multisets
de entrada y salida difieren, los polinomios son distintos: por
Schwartz–Zippel, dos polinomios distintos de grado `n` coinciden en
un punto aleatorio con probabilidad a lo sumo `n / |G|`, despreciable
para los tamaños prácticos del grupo `G`. Un nodo malicioso, por lo
tanto, no puede "salvarse" por casualidad sin haber preservado
genuinamente el multiset.

La prueba completa ocupa `O(n)` elementos del grupo y se verifica en
tiempo `O(n)`, lineal en el número `n` de ciphertexts del lote. La
estructura algebraica —compromisos, desafíos y respuestas— es la
misma que sostiene las pruebas disjuntivas descritas en [Suma
Homomórfica]({{< ref "/docs/crypto/homomorphic_sum.md" >}}); lo que
cambia es la afirmación que se prueba, no la maquinaria
criptográfica que la sostiene.

Las pruebas de shuffle se publican junto con el lote de salida de
cada nodo y pueden ser verificadas por el servidor, por los
[Custodios de Clave]({{< ref "/docs/roles/trustee.md" >}}) y por
cualquier [Verificador
Externo]({{< ref "/docs/roles/verifier.md" >}}) como parte del
proceso descrito en [Verificación de la
Elección]({{< ref "/docs/voting-steps/verification.md" >}}).

## Mixnet en UParticipa

Cuando una elección utiliza el conteo por mixnet, una vez cerrada la
elección los ciphertexts almacenados en la Urna Electrónica se envían
a una secuencia de **servidores mixnet independientes**, operados
idealmente por entidades distintas. UParticipa utiliza actualmente
una cadena de tres nodos: con esa configuración basta que **uno** de
ellos sea honesto para que la asociación voto ↔ votante quede
definitivamente rota.

A la salida del último nodo, el lote final de ciphertexts —ya
desligado del orden de envío original— es desencriptado uno por uno
mediante la combinación de [desencriptaciones
parciales]({{< ref "/docs/voting-steps/partial_decryptions.md" >}})
aportadas por los Custodios. El resultado de cada desencriptación es
el plaintext de un voto, sin ninguna información identificatoria del
votante que lo emitió. El conjunto de todos esos plaintexts se
totaliza para producir el [resultado de la
elección]({{< ref "/docs/voting-steps/final_result.md" >}}).

## Supuestos de seguridad y modelo de adversario

La seguridad de un mixnet se sostiene sobre dos garantías
independientes que conviene distinguir:

- **Integridad (correctitud del escrutinio).** Se sostiene en las
  pruebas de shuffle. Aun si **todos** los nodos del mixnet fueran
  maliciosos y coludidos, cualquier sustitución, descarte o
  duplicación de votos durante el barajado quedaría detectada por la
  verificación de las pruebas. La integridad no depende de la
  honestidad de ningún nodo en particular: depende únicamente de la
  solidez de los argumentos criptográficos —dureza del logaritmo
  discreto en `G`, hipótesis Decisional Diffie–Hellman, e
  idealización del oráculo aleatorio para Fiat–Shamir— ya enunciados
  en [Encriptación]({{< ref "/docs/crypto/encryption.md" >}}).
- **Anonimato (desligar voto y votante).** Depende, en cambio, del
  supuesto de **no colusión** entre nodos. Si todos los nodos
  compartieran su permutación y sus aleatoriedades, la composición
  resultante sería conocida y los ciphertexts de salida podrían
  re-asociarse con los de entrada. Para que el anonimato se sostenga
  basta con que **un** nodo guarde fielmente su permutación; de ahí
  la práctica de operar los nodos por entidades distintas e
  independientes.

El modelo de adversario considerado es el habitual en este tipo de
protocolos: un atacante activo que observa todos los mensajes
públicos (incluyendo la Urna Electrónica y las pruebas de shuffle),
puede coludir con hasta `N − 1` de los `N` nodos del mixnet y puede
intentar inyectar votos inválidos en el sistema. Lo que el modelo
**no** cubre es la compromisión de los componentes adyacentes: la
seguridad del servidor de autentificación, del cifrado en la Cabina
de Votación y del manejo de las claves privadas por parte de los
[Custodios]({{< ref "/docs/roles/trustee.md" >}}) está descrita en
las páginas correspondientes de esta documentación y constituye un
complemento necesario al modelo de seguridad del mixnet.

## Trade-offs frente a la suma homomórfica

Mixnet y suma homomórfica son las dos vías que UParticipa tiene para
producir el resultado de una elección. Su costo y aplicabilidad
escalan en direcciones distintas:

- **Suma homomórfica** combina todos los votos en un solo agregado
  cifrado y produce el resultado con una única desencriptación final.
  Su costo crece con el número de candidaturas por pregunta y con la
  complejidad de las pruebas en cero conocimiento del lado del
  votante. Encaja bien cuando los votos son colecciones cortas de
  bits y el número de opciones es moderado.
- **Mixnet** desencripta los votos uno por uno tras un barajado
  verificable. Su costo crece principalmente con el número de votos
  (no con el número de opciones por pregunta) y demanda una
  infraestructura adicional —los nodos de mezcla y sus pruebas de
  shuffle—. A cambio, admite codificaciones de voto arbitrariamente
  ricas (subconjuntos amplios, *rankings*) y elimina la cota
  práctica de candidaturas que pesa sobre la Votación Simple.

La elección entre uno y otro método se realiza al configurar el tipo
de votación, antes de la apertura de la elección.
