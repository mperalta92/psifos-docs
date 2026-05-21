---
title: "Encriptación"
slug: message-encryption
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 810
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
La encriptación de votos es el primer eslabón criptográfico de toda
elección en UParticipa: garantiza que las preferencias del votante
permanezcan en secreto desde el momento en que abandonan su navegador,
viajan al servidor, son acumuladas para el escrutinio, y son
desencriptadas únicamente cuando se combinan las contribuciones de
todos los [Custodios de Clave]({{< ref "/docs/roles/trustee.md" >}}).
Esta página describe el esquema utilizado, su procedimiento y la
relación con el resto de la sección Criptografía.

## ElGamal y parámetros de la elección

UParticipa cifra los votos con **ElGamal**, un esquema de criptografía
de clave pública definido sobre un grupo cíclico de orden primo. En
particular, se utiliza un subgrupo `G` de orden primo `q` dentro del
grupo multiplicativo `(Z/pZ)*`, con `p` primo grande, `q` divisor de
`p − 1` y `g ∈ G` un generador del subgrupo. La elección de `p`, `q`
y `g` queda fijada en la configuración de la elección y se publica
como parte de los datos públicos del [Portal de
Información]({{< ref "/docs/functionalities/stats_public_info.md" >}}).

La **clave pública** de la elección, `pk`, se construye durante la
[Ceremonia de Creación de
Claves]({{< ref "/docs/voting-steps/key_generation.md" >}}) a partir
de las contribuciones de todos los Custodios: en abstracto, equivale
a `pk = g^x mod p`, donde `x` (la **clave privada**) está repartido
secretamente entre los Custodios mediante [Generación Distribuida de
Claves]({{< ref "/docs/crypto/dkg.md" >}}). Ningún Custodio por
separado conoce `x`.

UParticipa adopta este esquema del sistema
[Helios](https://www.usenix.org/legacy/event/sec08/tech/full_papers/adida/adida.pdf),
propuesto por B. Adida en *Helios: Web-based Open-Audit Voting* (17th
USENIX Security Symposium, 2008, pp. 335–348). Helios fue uno de los
primeros sistemas operacionales en combinar ElGamal exponencial,
pruebas en cero conocimiento y un flujo de votación accesible desde el
navegador para ofrecer simultáneamente privacidad del voto y
verificabilidad pública. UParticipa hereda de él tanto el esquema
criptográfico descrito en esta página como las construcciones que
aparecen en las demás páginas de esta sección, y lo extiende para
soportar los casos de uso de las organizaciones universitarias
chilenas.

## Procedimiento de cifrado

Para cifrar un voto cuyo plaintext es un valor `v` del grupo, el
votante —a través de la Cabina de Votación, en su propio navegador—
realiza los siguientes pasos:

1. Genera una **aleatoriedad** `r` uniformemente en `Z_q`. Esta
   aleatoriedad es nueva en cada cifrado y nunca se reutiliza entre
   votos.
2. Calcula los dos componentes del ciphertext:

   ```
   A = g^r mod p
   B = v · pk^r mod p
   ```

3. Envía al servidor el par `C = (A, B)` junto con una [prueba en
   cero conocimiento]({{< ref "/docs/crypto/zkp.md" >}}) que demuestra
   que `C` está bien formado y que `v` se encuentra dentro del rango
   admitido por la pregunta.

El par `(A, B)` es el voto encriptado. Sin conocer `r` ni `x`, ningún
observador del par puede determinar el contenido del voto: dos votos
idénticos producen ciphertexts distintos (porque la aleatoriedad
cambia en cada cifrado), y dos votos distintos resultan
indistinguibles para quien no posea la clave privada.

## Codificación exponencial

Para que el ciphertext sea **homomórficamente aditivo** —propiedad
que UParticipa explota en el conteo de las votaciones simples— el
plaintext del voto no se cifra directamente como un número, sino
codificado como `g^v`. Antes de aplicar el cifrado descrito arriba, el
sistema eleva el valor `v` (típicamente un bit `0` o `1`) al
generador del grupo:

```
C = (g^r, g^v · pk^r)
```

Esta variante se conoce como **ElGamal exponencial** y es la que
habilita las propiedades discutidas en [Suma
Homomórfica]({{< ref "/docs/crypto/homomorphic_sum.md" >}}). El precio
que se paga por ella es que recuperar `v` durante el escrutinio
requiere invertir un logaritmo discreto, lo que se resuelve mediante
la tabla precomputada descrita en esa misma página.

## Desencriptación

Como `x` está repartido entre los Custodios, no existe ninguna
operación de descifrado que produzca el plaintext a partir de un
único ciphertext en un solo paso. En su lugar, cada Custodio aporta
una **desencriptación parcial** sobre el resultado acumulado del
precómputo. La combinación de un número suficiente de
desencriptaciones parciales —por encima del umbral fijado en la
elección— produce el resultado final. El procedimiento se describe
en [Envío de Desencriptaciones
Parciales]({{< ref "/docs/voting-steps/partial_decryptions.md" >}}) y
[Escrutinio de la
Elección]({{< ref "/docs/voting-steps/final_result.md" >}}).

## Garantías y supuestos

La seguridad del esquema de cifrado depende de tres supuestos
criptográficos estándar:

- **Dureza del problema del logaritmo discreto** en el subgrupo `G`,
  que asegura que `x` no se puede recuperar a partir de `pk`, ni `r`
  a partir de `A`.
- **Hipótesis Decisional Diffie–Hellman (DDH)** en `G`, que asegura
  que ningún observador puede distinguir el cifrado de un mensaje del
  cifrado de otro sin la clave privada.
- **Calidad e independencia de la aleatoriedad** en el cliente, que
  asegura que cada `r` sea estadísticamente independiente del resto.
  La Cabina de Votación obtiene esta aleatoriedad del generador
  criptográfico del navegador.

A estos supuestos se suman las garantías que aportan las pruebas en
cero conocimiento (la integridad de cada voto y la corrección del
escrutinio) y la separación de roles entre el servidor, los Custodios
y el [Verificador Externo]({{< ref "/docs/roles/verifier.md" >}}), que
permite reejecutar el chequeo de todas las propiedades de forma
independiente.
