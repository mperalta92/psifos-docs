---
title: "Código Fuente"
slug: source-code
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 501
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
Todo el código del proyecto UParticipa está publicado de manera abierta
en GitHub, bajo la organización [clcert](https://github.com/clcert). A
continuación se listan los repositorios públicos que componen el
ecosistema, agrupados por su función dentro del sistema.

## Plataforma de votación

- **[psifospoll](https://github.com/clcert/psifospoll)**: librería en
  Python que implementa el algoritmo *Single Transferable Vote (STV)*
  utilizado en las elecciones de [tipo
  ranking]({{< ref "/docs/functionalities/voting_types.md" >}}).
- **[psifos-crypto](https://github.com/clcert/psifos-crypto)**:
  primitivos criptográficos del backend (encriptación ElGamal, propiedad
  homomórfica, generación distribuida de claves, mixnet, zero-knowledge
  proofs). Incluye un componente en Python para el servidor y otro en
  JavaScript que se ejecuta en la Cabina de Votación del navegador.

## Aplicación móvil para Custodios

- **[psifos_mobile_app](https://github.com/clcert/psifos_mobile_app)**:
  aplicación móvil (Flutter) para los [Custodios de
  Clave]({{< ref "/docs/roles/trustee.md" >}}), pensada para reemplazar
  el flujo desktop de la [Ceremonia de Creación de
  Claves]({{< ref "/docs/voting-steps/key_generation.md" >}}) y del
  [envío de desencriptaciones
  parciales]({{< ref "/docs/voting-steps/partial_decryptions.md" >}}).
- **[psifos_mobile_crypto](https://github.com/clcert/psifos_mobile_crypto)**:
  librería en Dart consumida por la aplicación móvil. Implementa el
  subconjunto de primitivos criptográficos necesarios para los
  Custodios.

## Verificación independiente

- **[pyrios](https://github.com/clcert/pyrios)**: verificador externo
  offline (en Go), derivado del verificador del sistema Helios y
  adaptado a Psifos. Es la herramienta que un [Verificador
  Externo]({{< ref "/docs/roles/verifier.md" >}}) ejecuta sobre el
  archivo de verificación, conforme al procedimiento descrito en
  [Verificación de la
  Elección]({{< ref "/docs/voting-steps/verification.md" >}}).

## Sitios web públicos

- **[uparticipa-home](https://github.com/clcert/uparticipa-home)**:
  sitio institucional del proyecto UParticipa (Next.js).
- **[uparticipa-uchile-web](https://github.com/clcert/uparticipa-uchile-web)**:
  sitio público específico de la implementación en la Universidad de
  Chile ([participa.uchile.cl](https://participa.uchile.cl)), donde se
  muestran las elecciones vigentes y se redirige al votante a la Cabina
  de Votación.

## Pruebas y documentación

- **[psifos-testing](https://github.com/clcert/psifos-testing)**: suite
  *end-to-end* (Selenium) que ejerce el ciclo electoral completo
  (administrador, custodios, votantes, escrutinio, verificación) contra
  una instancia real del sistema.
- **[psifos-docs](https://github.com/clcert/psifos-docs)**: la presente
  documentación, generada con Hugo y desplegada en Netlify.

## Licencias y contribuciones

Cada repositorio publica su propia licencia en el archivo `LICENSE` de
su raíz. Las contribuciones se gestionan a través de *issues* y *pull
requests* en el repositorio correspondiente. Para conocer las
instrucciones de instalación, ejecución y desarrollo local de un
componente específico, se recomienda consultar el `README.md` del
repositorio respectivo.
