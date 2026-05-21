---
title: "Arquitectura"
slug: software-arch
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 500
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
UParticipa es un sistema distribuido compuesto por varios componentes
que colaboran para conducir una elección segura, verificable y
respetuosa del secreto del voto. Esta página presenta los componentes
del sistema, los flujos que los integran y las decisiones de diseño que
sostienen las garantías criptográficas descritas en la sección de
[Criptografía]({{< ref "/docs/crypto/" >}}).

## Componentes

- **Backend Psifos**: servidor que recibe los votos encriptados,
  administra el padrón electoral, opera con la información de la
  elección y expone tanto la **Cabina de Votación** como las APIs
  públicas para consulta y verificación.
- **Cabina de Votación**: aplicación cliente que se ejecuta en el
  navegador del [Votante]({{< ref "/docs/roles/voter.md" >}}) y realiza
  el cifrado del voto localmente, antes de enviarlo al servidor.
- **Servidor externo de autentificación**: instancia de [OpenID
  Connect]({{< ref "/docs/functionalities/auth.md" >}}) (por ejemplo,
  Google) que valida la identidad de votantes y custodios. Es
  independiente del backend Psifos por diseño, para separar la
  identificación del voto.
- **Portal del Custodio de Clave** (web) y **aplicación móvil para
  Custodios** (en desarrollo): interfaces que utilizan los [Custodios
  de Clave]({{< ref "/docs/roles/trustee.md" >}}) durante la ceremonia
  de creación de claves y el envío de las desencriptaciones parciales.
  El [Código Fuente]({{< ref "/docs/software/source_code.md" >}}) de la
  aplicación móvil se publica en los repositorios `psifos_mobile_app`
  y `psifos_mobile_crypto`.
- **Servidores Mixnet**: en las elecciones configuradas con conteo por
  mixnet, una serie de servidores externos reciben los votos
  encriptados, los re-encriptan y los permutan para romper la
  asociación voto ↔ votante antes del escrutinio.
- **Sitios web públicos**: `uparticipa-home` (presentación general del
  proyecto) y `uparticipa-uchile-web` (sitio específico de la
  Universidad de Chile, que consulta el estado de las elecciones
  vigentes vía la API pública del backend y redirige al votante a la
  Cabina de Votación).
- **Verificador Externo**: programa offline (por ejemplo,
  [pyrios](https://github.com/clcert/pyrios)) que cualquier persona
  puede ejecutar sobre el bundle público de la elección para validar
  de manera independiente todas las pruebas criptográficas. Ver el rol
  de [Verificador Externo]({{< ref "/docs/roles/verifier.md" >}}).

La siguiente representación esquemática es **provisional**: ilustra
cómo se relacionan los componentes mientras se desarrolla el diagrama
definitivo en el estilo gráfico del proyecto (ver [issue
#3](https://github.com/clcert/psifos-docs/issues/3)).

```text
                          ┌────────────────────┐
                          │  Servidor OIDC     │  ← autentificación
                          │  (externo)         │     externa
                          └─────────┬──────────┘
                                    │
                       (1) auth     │
                                    ▼
   ┌─────────────┐            ┌─────────────────┐
   │  Sitios     │            │  Votante        │
   │  públicos   │ ─────────► │                 │
   │ (uparticipa │            └────────┬────────┘
   │   -*)       │                     │
   └─────────────┘                     │ (2) ingresa a votar
                                       ▼
                            ┌─────────────────┐
                            │  Cabina de      │  ← cifra el voto
                            │  Votación       │     en el navegador
                            │  (JS, cliente)  │
                            └────────┬────────┘
                                     │ (3) voto cifrado + ZKP
                                     ▼
   ┌────────────┐          ┌─────────────────┐          ┌──────────┐
   │ Custodios  │ ───────► │ Backend Psifos  │ ◄─────── │  Admin   │
   │ (web /     │          │ urna · padrón · │          │          │
   │  móvil)    │          │ escrutinio      │          │          │
   └────────────┘          └────┬─────────┬──┘          └──────────┘
                                │         │
                 (4) si aplica  │         │ (5) publica
                       ┌────────┘         └────────┐
                       ▼                           ▼
              ┌────────────────┐           ┌─────────────────┐
              │  Servidores    │           │  Portal de      │
              │  Mixnet        │           │  Información    │
              │                │           │  (público)      │
              └────────────────┘           └────────┬────────┘
                                                    │ (6) bundle público
                                                    ▼
                                           ┌─────────────────┐
                                           │  Verificador    │
                                           │  Externo        │
                                           │  (pyrios)       │
                                           └─────────────────┘
```

## Flujos principales

### Antes de la elección

El [Administrador]({{< ref "/docs/roles/admin.md" >}}) configura la
elección y registra a los Custodios. Estos últimos participan
sincrónicamente en la [Ceremonia de Creación de
Claves]({{< ref "/docs/voting-steps/key_generation.md" >}}), que
produce la **clave pública** que se utilizará para cifrar los votos y
las **claves privadas** que cada Custodio resguarda. Generadas las
claves, el Administrador abre la elección.

### Durante la jornada electoral

Cada Votante se autentica frente al servidor OIDC y, si está habilitado
en el padrón, accede a la Cabina de Votación. La Cabina cifra las
preferencias del Votante con la clave pública de la elección y envía el
voto encriptado al backend, junto con las pruebas de correctitud
descritas en [Zero-Knowledge
Proofs]({{< ref "/docs/crypto/zkp.md" >}}). El backend valida el voto y
lo expone públicamente en la **Urna Electrónica** del [Portal de
Información]({{< ref "/docs/functionalities/stats_public_info.md" >}}).
Si el Votante reenvía un voto, solamente se contabilizará el último
(ver [medidas anti
coerción]({{< ref "/docs/functionalities/coercion.md" >}})).

### Cierre y escrutinio

El Administrador cierra la elección y gatilla el **precómputo** sobre
los votos almacenados, utilizando suma homomórfica o mixnet según el
tipo de elección (ver [Cierre de Elección y
Precómputo]({{< ref "/docs/voting-steps/tally_compute.md" >}})). Una
vez listo el precómputo, los Custodios envían sus desencriptaciones
parciales y, al combinarlas, se obtiene el resultado final (ver
[Escrutinio de la
Elección]({{< ref "/docs/voting-steps/final_result.md" >}})), que el
Administrador publica.

### Verificación

Publicado el resultado, cualquier persona puede asumir el rol de
[Verificador Externo]({{< ref "/docs/roles/verifier.md" >}}), descargar
el archivo público de verificación y validar todas las propiedades
criptográficas del proceso (ver [Verificación de la
Elección]({{< ref "/docs/voting-steps/verification.md" >}})).

## Decisiones de diseño

- **Cifrado en el cliente.** El voto se cifra en el navegador del
  Votante antes de salir de su equipo. El backend nunca observa
  preferencias en claro.
- **Separación entre identidad y voto.** La autentificación se delega a
  un servidor OIDC externo, distinto del que recibe los votos
  encriptados. Ningún componente conoce simultáneamente la identidad
  del votante y su preferencia en claro.
- **Urna Electrónica pública.** Cada voto encriptado se expone en
  tiempo real para que cualquier persona, incluido el propio votante,
  pueda verificar su presencia antes del escrutinio.
- **Custodios distribuidos.** La clave que permite descifrar los
  resultados está repartida entre múltiples Custodios mediante
  [Generación Distribuida de
  Claves]({{< ref "/docs/crypto/dkg.md" >}}); ningún Custodio individual
  puede descifrar votos por su cuenta.
- **Verificación independiente.** El protocolo está diseñado para que
  un tercero pueda re-ejecutar todo el escrutinio y rechazar cualquier
  manipulación, sin depender de la operación del sistema.
