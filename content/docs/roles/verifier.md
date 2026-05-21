---
title: "Verificador Externo"
slug: external-verifier
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
## Designación del Verificador Externo

A diferencia de los otros roles del sistema, el Verificador Externo no
requiere ser designado previamente por la organización que conduce la
elección. Cualquier persona, independiente del sistema y de la Autoridad
Electoral, puede asumir este rol descargando el archivo de verificación
que se publica al cierre de la elección y ejecutando sobre él un programa
de verificación independiente.

La existencia de este rol es central para la confianza pública en el
sistema: el resultado de una elección no requiere ser aceptado por el
solo hecho de haber sido publicado en la plataforma, sino que puede ser
auditado de forma reproducible por terceros que no participaron de la
operación del proceso.

## Rol del Verificador Externo

El Verificador Externo descarga el archivo de verificación desde el
[Portal de
Información]({{< ref "/docs/functionalities/stats_public_info.md" >}}) de
la elección y ejecuta sobre él un programa que comprueba todas las
propiedades criptográficas necesarias, según el procedimiento detallado
en [Verificación de la
Elección]({{< ref "/docs/voting-steps/verification.md" >}}). Las
comprobaciones incluyen las pruebas asociadas a cada voto encriptado, la
pertenencia de cada voto al padrón, la consistencia del precómputo con
la información pública, y la correcta combinación de las
[desencriptaciones
parciales]({{< ref "/docs/voting-steps/partial_decryptions.md" >}}) hasta
el resultado final.

UParticipa pone a disposición la implementación de referencia
[pyrios](https://github.com/clcert/pyrios), un verificador externo
offline escrito en Go, basado en el verificador del sistema Helios y
adaptado a Psifos. El uso de esta implementación no es obligatorio:
cualquier persona u organización puede desarrollar su propio verificador,
en otro lenguaje y de manera independiente, siempre que respete el
protocolo descrito en la sección de [Criptografía]({{< ref "/docs/crypto/" >}}).
