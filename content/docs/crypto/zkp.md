---
title: "Zero-Knowledge Proofs"
slug: zero-knowledge-proofs
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 850
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
Las **pruebas en cero conocimiento** (*zero-knowledge proofs*, ZKP)
son el ingrediente que permite a UParticipa verificar cada paso del
proceso electoral **sin exigir confianza ciega en ninguna de sus
partes**. Cada voto encriptado lleva adjunta una prueba de
bien-formación; cada desencriptación parcial enviada por un Custodio
lleva una prueba de correctitud; cada barajado del Mixnet va
acompañado de una prueba verificable. Esta página describe la
estructura común a todas esas pruebas y las construcciones específicas
que UParticipa utiliza.

## Propiedades fundamentales

Un protocolo de prueba en cero conocimiento involucra dos partes: un
**probador** que conoce un secreto y quiere convencer a un
**verificador** de que cierta afirmación matemática es cierta. Para
que el protocolo sea una ZKP válida debe satisfacer tres propiedades:

- **Completitud.** Si la afirmación es cierta y el probador es
  honesto, el verificador siempre acepta.
- **Solidez** (*soundness*). Si la afirmación es falsa, ningún
  probador puede convencer al verificador salvo con probabilidad
  despreciable, bajo los supuestos criptográficos del esquema.
- **Cero conocimiento.** El verificador no aprende nada sobre el
  secreto del probador, más allá del hecho de que la afirmación es
  cierta.

En UParticipa, las afirmaciones que se prueban son siempre relaciones
algebraicas sobre el grupo `G` definido en
[Encriptación]({{< ref "/docs/crypto/encryption.md" >}}): por ejemplo,
"el ciphertext `C = (A, B)` codifica un valor en `{0, 1}`", o "el
elemento `A^{s_j}` enviado por el Custodio `j` fue calculado con la
share `s_j` que le fue asignada".

## Estructura de un protocolo sigma

La mayoría de las pruebas usadas en UParticipa siguen el patrón
canónico de los **protocolos sigma**, así llamados por la forma del
intercambio. Probador y verificador intercambian tres mensajes:

1. **Compromiso** (`a`). El probador elige una aleatoriedad fresca y
   construye un elemento del grupo que "compromete" su secreto sin
   revelarlo.
2. **Desafío** (`c`). El verificador elige un escalar aleatorio `c`
   y lo envía al probador.
3. **Respuesta** (`z`). El probador combina su secreto con `c` y la
   aleatoriedad del compromiso para producir un escalar `z` que
   satisface una **ecuación pública** verificable.

El verificador acepta si y sólo si la ecuación pública se cumple para
la terna `(a, c, z)` y los datos públicos del protocolo (afirmación,
clave pública, etc.). Completitud y solidez se demuestran a partir de
las propiedades algebraicas de `G`; el cero conocimiento, del hecho
de que toda terna `(a, c, z)` válida se puede **simular** sin conocer
el secreto, lo que hace que una transcripción honesta sea
estadísticamente indistinguible de una transcripción simulada.

## Schnorr: conocer un logaritmo discreto

El protocolo más simple, base de varios otros, es el descrito por
C. P. Schnorr para demostrar **conocimiento de un logaritmo
discreto**: el probador conoce `x ∈ Z_q` tal que `y = g^x` y quiere
convencer al verificador sin revelar `x`.

1. Compromiso: el probador elige `r ∈ Z_q` al azar y envía `a = g^r`.
2. Desafío: el verificador envía `c ∈ Z_q` aleatorio.
3. Respuesta: el probador envía `z = r + c · x (mod q)`.

El verificador acepta si `g^z = a · y^c`. La igualdad sólo se cumple
si `z = r + c · x`, lo que requiere que el probador conozca `x`. La
construcción aparece en *Efficient Identification and Signatures for
Smart Cards* (CRYPTO '89, LNCS 435, pp. 239–252, DOI:
[10.1007/0-387-34805-0_22](https://doi.org/10.1007/0-387-34805-0_22)).

## Chaum–Pedersen: igualdad de logaritmos discretos

Una extensión del protocolo de Schnorr permite probar la **igualdad
de dos logaritmos discretos**: el probador conoce `x` tal que
`y₁ = g^x` y `y₂ = h^x`, con `g, h ∈ G`, y quiere convencer al
verificador de que ambas relaciones se cumplen con el **mismo** `x`
sin revelarlo.

1. Compromiso: el probador elige `r` y envía `a₁ = g^r`, `a₂ = h^r`.
2. Desafío: el verificador envía `c`.
3. Respuesta: el probador envía `z = r + c · x (mod q)`.

El verificador acepta si `g^z = a₁ · y₁^c` y `h^z = a₂ · y₂^c`. La
construcción fue descrita por D. Chaum y T. P. Pedersen en *Wallet
Databases with Observers* (CRYPTO '92, LNCS 740, pp. 89–105, DOI:
[10.1007/3-540-48071-4_7](https://doi.org/10.1007/3-540-48071-4_7)),
y es la herramienta central que usan los Custodios para demostrar
que cada [desencriptación
parcial]({{< ref "/docs/voting-steps/partial_decryptions.md" >}}) fue
calculada con la share correcta: el Custodio prueba que el exponente
con el que calculó `A^{s_j}` es el mismo que aparece en el
compromiso público `g^{s_j}` derivable de los compromisos
publicados durante la [Generación Distribuida de
Claves]({{< ref "/docs/crypto/dkg.md" >}}).

## Composición disjuntiva

Para probar afirmaciones del tipo "alguna de varias condiciones se
cumple, sin revelar cuál" —fundamental para que un voto demuestre
estar bien formado sin filtrar la preferencia del votante— se utiliza
una **composición disjuntiva** de protocolos sigma. La técnica
estándar, descrita por R. Cramer, I. Damgård y B. Schoenmakers en
*Proofs of Partial Knowledge and Simplified Design of Witness Hiding
Protocols* (CRYPTO '94, LNCS 839, pp. 174–187, DOI:
[10.1007/3-540-48658-5_19](https://doi.org/10.1007/3-540-48658-5_19)),
combina varias instancias del esquema sigma de modo que el probador
puede ejecutar honestamente una rama y **simular** las restantes, sin
que el verificador pueda distinguir cuál fue cuál. La mecánica
concreta —cómo cada rama se ata a un desafío parcial cuya suma debe
coincidir con un desafío global— está desarrollada en [Suma
Homomórfica]({{< ref "/docs/crypto/homomorphic_sum.md" >}}).

## De interactivo a no interactivo: Fiat–Shamir

Los protocolos descritos hasta aquí son **interactivos**: el
verificador envía un desafío genuinamente aleatorio durante la
ejecución. En un sistema de votación pública, el verificador no es
una entidad única ni está disponible en tiempo real; lo que se
publica es un **archivo de verificación** que cualquier persona
debe poder revisar de forma autónoma.

La **heurística de Fiat–Shamir**, descrita por A. Fiat y A. Shamir
en *How to Prove Yourself: Practical Solutions to Identification
and Signature Problems* (CRYPTO '86, LNCS 263, pp. 186–194, DOI:
[10.1007/3-540-47721-7_12](https://doi.org/10.1007/3-540-47721-7_12)),
convierte cualquier protocolo sigma en no interactivo: el desafío
`c` se reemplaza por el resultado de aplicar una función de hash
criptográfica al compromiso `a` y a los datos públicos del
protocolo. Como el hash es determinista pero impredecible para el
probador, su rol equivale al de un desafío genuinamente aleatorio.

Bajo esta transformación, una ZKP se compone únicamente de los
elementos `(a, z)` —el desafío se reconstruye— y se publica junto
con el dato al que acompaña. Cualquier [Verificador
Externo]({{< ref "/docs/roles/verifier.md" >}}) puede recalcular el
hash y chequear la ecuación de verificación sin interacción con el
probador.

## Aplicaciones en UParticipa

A lo largo del proceso electoral, las pruebas en cero conocimiento
verifican cada paso criptográfico del protocolo:

- **Validez de cada voto.** Cada ciphertext que ingresa a la urna va
  acompañado de una composición disjuntiva que demuestra (i) que el
  ciphertext encripta un valor en `{0, 1}` para cada candidatura, y
  (ii) que la suma de las selecciones del votante respeta el rango
  admitido por la pregunta. El desglose del costo por candidatura
  está en [Suma
  Homomórfica]({{< ref "/docs/crypto/homomorphic_sum.md" >}}).
- **Correctitud de cada desencriptación parcial.** Cada Custodio
  acompaña su valor `A^{s_j}` con una prueba de Chaum–Pedersen que
  ata el exponente al compromiso público derivado de la ceremonia
  de claves, evitando que pueda enviar un valor arbitrario que dañe
  el resultado.
- **Correctitud de cada barajado del Mixnet.** Cada nodo del mixnet
  publica una prueba en cero conocimiento de **shuffle** que
  demuestra haber permutado y re-cifrado el lote sin alterar el
  multiset de plaintexts. La mecánica concreta (identidad polinomial
  + Schwartz–Zippel + sigma de Schnorr, debida a C. A. Neff) está
  descrita en [Mixnet]({{< ref "/docs/crypto/mixnet.md" >}}).
- **Consistencia del DKG.** Los compromisos verificables de Feldman
  sobre los coeficientes del polinomio de cada Custodio cumplen un
  rol análogo al de las ZKP: garantizan que las shares distribuidas
  son consistentes con un polinomio publicado, sin revelar el
  polinomio en sí. Ver [Generación Distribuida de
  Claves]({{< ref "/docs/crypto/dkg.md" >}}).

## Garantías y supuestos

La seguridad del conjunto de pruebas se apoya en tres supuestos
estándar:

- **Dureza del logaritmo discreto e hipótesis Decisional
  Diffie–Hellman** en el grupo `G`, ya enunciados en
  [Encriptación]({{< ref "/docs/crypto/encryption.md" >}}). Garantizan
  la solidez de los protocolos sigma sobre ese grupo.
- **Modelo del oráculo aleatorio** para el hash usado en
  Fiat–Shamir: asume que la función de hash se comporta como una
  función aleatoria desde el punto de vista del adversario. Es el
  supuesto que vuelve no interactivos los protocolos sin sacrificar
  la solidez.
- **Aleatoriedad fresca** del lado del probador. Cada compromiso debe
  usar una aleatoriedad nueva e independiente; reutilizar la `r` del
  compromiso entre dos ejecuciones de un mismo protocolo puede
  revelar el secreto.

Todas las pruebas se publican junto con el archivo de verificación
de la elección y pueden ser revisadas por el servidor, por los demás
[Custodios]({{< ref "/docs/roles/trustee.md" >}}) y por cualquier
[Verificador Externo]({{< ref "/docs/roles/verifier.md" >}}) como
parte del proceso descrito en [Verificación de la
Elección]({{< ref "/docs/voting-steps/verification.md" >}}).
