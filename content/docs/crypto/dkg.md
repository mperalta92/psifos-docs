---
title: "Generación Distribuida de Claves"
slug: distributed-key-generation
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 820
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
La **Generación Distribuida de Claves** (DKG, por su sigla en inglés
*Distributed Key Generation*) es el procedimiento criptográfico que
permite a UParticipa generar las claves de una elección **sin que
ninguna parte conozca la clave privada completa**. Lo que ocurre
operativamente durante la [Ceremonia de Creación de
Claves]({{< ref "/docs/voting-steps/key_generation.md" >}}) está
descrito desde el punto de vista de los Custodios y del
Administrador; esta página describe la mecánica criptográfica que
sostiene esa ceremonia.

## El problema

UParticipa cifra los votos con
[ElGamal]({{< ref "/docs/crypto/encryption.md" >}}) sobre un grupo
cíclico `G` de orden primo `q`. La clave pública `pk = g^x mod p`
exige la existencia de una clave privada `x`. Si una única entidad
poseyera `x`, su compromiso (o su captura por un adversario) bastaría
para descifrar todos los votos. Tampoco es viable exigir que
**todos** los Custodios de Clave participen al momento del
escrutinio: cualquier indisponibilidad o pérdida de la clave privada
por parte de un Custodio dejaría a la elección sin posibilidad de
ser desencriptada.

El DKG resuelve ambos problemas al mismo tiempo: produce una clave
pública `pk` cuya clave privada `x` queda **repartida** entre los `n`
[Custodios de Clave]({{< ref "/docs/roles/trustee.md" >}}) bajo un
esquema de umbral. Un subconjunto cualquiera de `t ≤ n` Custodios
puede colaborar para realizar el escrutinio; menos de `t` Custodios
no obtienen información alguna sobre `x`.

## Reparto por polinomios

La pieza central del DKG es el **secreto compartido por polinomio**
(Shamir): cada Custodio `i` (`i = 1, …, n`) elige al azar un
polinomio

```
f_i(z) = a_{i,0} + a_{i,1} z + … + a_{i,t-1} z^{t-1}
```

de grado `t − 1` con coeficientes en `Z_q`. Su **aporte privado** a
la clave es el término constante `a_{i,0} = f_i(0)`, que nadie más
conoce. Las **shares** que envía a los demás Custodios son los
valores `f_i(j)` para `j = 1, …, n`, transmitidos privadamente bajo
encripción a la clave del Custodio receptor.

Al recibir los aportes del resto, cada Custodio `j` acumula sus
shares para producir su parte final de la clave:

```
s_j = ∑_i f_i(j) = F(j),   donde   F(z) = ∑_i f_i(z)
```

`s_j` es entonces la evaluación en `j` del polinomio agregado `F`,
cuyo término constante es `F(0) = ∑_i a_{i,0}`. Ese término
constante, **la suma de todos los aportes privados**, es la clave
privada de la elección `x`. Nadie la conoce —cada Custodio sólo
conoce su propio `a_{i,0}` y la share acumulada `s_j`— pero queda
implícitamente definida y repartida entre todos.

La clave pública correspondiente se calcula públicamente: cada
Custodio publica `g^{a_{i,0}}` y el sistema las combina como

```
pk = ∏_i g^{a_{i,0}} = g^{∑_i a_{i,0}} = g^x
```

sin revelar los exponentes individuales.

## Compromisos verificables

Para que ningún Custodio pueda enviar shares incorrectas (sea por
error o por malicia), el procedimiento incluye **compromisos
verificables** a los coeficientes del polinomio. Cada Custodio `i`
publica, junto con sus shares, los valores `g^{a_{i,k}}` para
`k = 0, …, t − 1`. Cualquier receptor `j` puede entonces verificar
que la share `f_i(j)` que le llegó es consistente con esos
compromisos, comprobando la identidad

```
g^{f_i(j)} = ∏_{k=0}^{t-1} (g^{a_{i,k}})^{j^k}
```

La igualdad sólo se cumple si la share corresponde efectivamente al
polinomio cuyas potencias del generador fueron publicadas, sin que
los coeficientes mismos hayan sido revelados.

La técnica proviene del esquema de **secreto compartido verificable**
de P. Feldman (*A Practical Scheme for Non-interactive Verifiable
Secret Sharing*, 28th Annual Symposium on Foundations of Computer
Science, FOCS '87, pp. 427–438, DOI:
[10.1109/SFCS.1987.4](https://doi.org/10.1109/SFCS.1987.4)). Su
adaptación al DKG sobre criptosistemas basados en logaritmo discreto,
en el formato que UParticipa hereda de Helios, fue descrita por
T. P. Pedersen en *A Threshold Cryptosystem without a Trusted Party*
(EUROCRYPT '91, LNCS 547, pp. 522–526, DOI:
[10.1007/3-540-46416-6_47](https://doi.org/10.1007/3-540-46416-6_47)).

## Denuncia y exclusión durante la ceremonia

El procedimiento descrito hasta aquí asume que todos los Custodios
siguen el protocolo. En la práctica, el DKG debe poder lidiar con
Custodios que se desvían, sea por error de implementación o
deliberadamente, y resolver las inconsistencias sin que la ceremonia
quede bloqueada.

Cuando un Custodio `j` recibe una share `f_i(j)` que no verifica
contra los compromisos publicados por el Custodio `i` (es decir, que
no satisface la identidad
`g^{f_i(j)} = ∏_k (g^{a_{i,k}})^{j^k}`), `j` emite una **denuncia
pública** dentro de la ceremonia. La denuncia incluye el índice `i`
del Custodio acusado y la share inválida recibida, de modo que
cualquier participante pueda re-ejecutar la verificación.

El Custodio acusado debe entonces revelar **públicamente** la share
en disputa. La revelación puede tener dos desenlaces:

- Si la share publicada en público sí verifica contra los
  compromisos de `i`, la denuncia se descarta: el receptor `j` debe
  usar esa share corregida.
- Si la share tampoco verifica, o si el Custodio acusado se niega a
  responder, el Custodio `i` queda **excluido** de la ceremonia y
  su aporte `a_{i,0}` no entra al cómputo final de la clave.

Al final del proceso queda definido un **conjunto calificado**
`Q ⊆ {1, …, n}` de Custodios no excluidos. La clave privada
implícita y la clave pública pasan a calcularse sobre `Q`:

```
x = ∑_{i ∈ Q} a_{i,0}     pk = ∏_{i ∈ Q} g^{a_{i,0}}
```

y las shares acumuladas `s_j` se recalculan análogamente sumando
sólo sobre `i ∈ Q`. Para que la elección sea viable, el conjunto
calificado debe seguir cumpliendo `|Q| ≥ t`; en caso contrario la
ceremonia se aborta y debe volver a iniciarse.

## Distribución uniforme y la corrección de GJKR

La construcción de Pedersen, tal como fue publicada originalmente,
admite un ataque sutil sobre la **distribución** de la clave pública
resultante. Un Custodio malicioso que sea el último en publicar su
aporte puede aprovechar haber visto los `g^{a_{i,0}}` ajenos para
decidir si participar o, por medio de denuncias falsas o de no
responder a una acusación legítima, salir del conjunto calificado.
Esa elección adaptativa le permite influir, dentro de un margen
pequeño pero no nulo, en el valor final de `pk` y, por extensión,
en la distribución estadística de `x`.

R. Gennaro, S. Jarecki, H. Krawczyk y T. Rabin describieron este
ataque y propusieron una corrección en *Secure Distributed Key
Generation for Discrete-Log Based Cryptosystems* (Journal of
Cryptology, vol. 20, no. 1, 2007, pp. 51–83, DOI:
[10.1007/s00145-006-0347-3](https://doi.org/10.1007/s00145-006-0347-3);
versión original en EUROCRYPT '99). La corrección, conocida como
**GJKR DKG**, divide la ceremonia en dos rondas con
*commit-then-reveal*. Los Custodios se comprometen primero a sus
contribuciones mediante un segundo esquema de compromisos —al estilo
Pedersen, **perfectamente ocultantes**— que esconden por completo el
valor comprometido hasta su apertura. Sólo después de fijar el
conjunto calificado se publican los `g^{a_{i,k}}` (los compromisos
de Feldman) que determinan `pk`, lo que impide que un atacante
decida adaptativamente entre participar o ser excluido en función
del resultado parcial observado.

GJKR es la opción canónica cuando se requiere una distribución
estrictamente uniforme de `pk`. Para los casos de uso de UParticipa
—donde los Custodios son explícitamente designados por la
organización electoral y los efectos prácticos del sesgo identificado
por GJKR son acotados (no rompen el secreto del voto ni la corrección
del escrutinio)— la elección entre la versión clásica de Pedersen y
la corrección GJKR es una decisión de configuración del protocolo
que puede tomarse según el contexto de cada elección.

## Desencriptación por umbral e interpolación de Lagrange

Una vez cerrada la elección y producido el precómputo descrito en
[Cierre de Elección y
Precómputo]({{< ref "/docs/voting-steps/tally_compute.md" >}}), cada
Custodio `j` envía su **desencriptación parcial**: sobre el
ciphertext agregado `C = (A, B)` calcula `A^{s_j}`, donde `s_j` es la
share que acumuló al final del DKG. Este envío se realiza en el paso
descrito en [Envío de Desencriptaciones
Parciales]({{< ref "/docs/voting-steps/partial_decryptions.md" >}}),
acompañado de una [prueba en cero
conocimiento]({{< ref "/docs/crypto/zkp.md" >}}) que demuestra que la
desencriptación parcial fue calculada correctamente con la share
asignada y no con un valor arbitrario.

Cuando el sistema dispone de al menos `t` desencriptaciones
parciales válidas, las combina mediante **interpolación de
Lagrange** en el exponente. Si las desencriptaciones provienen del
subconjunto `J ⊆ {1, …, n}` con `|J| = t`, los **coeficientes de
Lagrange** son

```
λ_j = ∏_{j' ∈ J, j' ≠ j}  j' / (j' − j)   (mod q)
```

y satisfacen, por construcción del polinomio interpolador, la
identidad `F(0) = ∑_{j ∈ J} λ_j · s_j`. Aplicada en el exponente
sobre las desencriptaciones parciales, la combinación produce

```
∏_{j ∈ J} (A^{s_j})^{λ_j} = A^{∑_{j ∈ J} λ_j · s_j} = A^{F(0)} = A^x
```

y de ahí se recupera el plaintext del agregado homomórfico (o de
cada ciphertext del lote barajado por la
[Mixnet]({{< ref "/docs/crypto/mixnet.md" >}})), como describe
[Escrutinio de la
Elección]({{< ref "/docs/voting-steps/final_result.md" >}}).

## Garantías

El DKG aporta al sistema, en combinación con los demás primitivos
criptográficos:

- **Confidencialidad bajo colusión parcial.** Mientras menos de `t`
  Custodios cooperen, la clave privada `x` permanece desconocida y
  los votos no pueden descifrarse. Esta garantía es de naturaleza
  **información-teórica** sobre los aportes privados `a_{i,0}`: no
  depende de la dureza de ningún problema computacional, sólo del
  hecho de que `t − 1` evaluaciones de un polinomio de grado `t − 1`
  no determinan al polinomio.
- **Disponibilidad bajo ausencias.** Cualquier subconjunto de `t`
  Custodios alcanza para reconstruir `x` (en el exponente) y
  desencriptar el resultado: la elección no queda bloqueada si
  algunos Custodios se vuelven indisponibles después de la
  ceremonia.
- **Auditabilidad.** Los compromisos `g^{a_{i,k}}` son públicos, de
  modo que cualquier desviación del protocolo durante la ceremonia
  puede ser detectada en su momento y revisada después por cualquier
  [Verificador Externo]({{< ref "/docs/roles/verifier.md" >}}).

El valor `t` se fija al configurar la elección. La práctica habitual
en UParticipa es elegir `t = ⌊n/2⌋ + 1` —la mitad de los Custodios
más uno— como recoge la nota correspondiente en [Escrutinio de la
Elección]({{< ref "/docs/voting-steps/final_result.md" >}}).
