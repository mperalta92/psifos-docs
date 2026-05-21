---
title: "Interfaz de Usuario"
slug: user-interface
description: ""
summary: ""
date: 2023-09-07T16:04:48+02:00
lastmod: 2023-09-07T16:04:48+02:00
draft: false
weight: 502
toc: true
seo:
  title: "" # custom title (optional)
  description: "" # custom description (recommended)
  canonical: "" # custom canonical URL (optional)
  noindex: false # false (default) or true
---
La plataforma UParticipa expone varias interfaces de usuario distintas,
cada una orientada a un rol específico del proceso electoral. A
continuación se describen las principales, indicando para cada caso qué
acciones permite y en qué etapa de la elección se utiliza.

## Portal de Información

El Portal de Información es la interfaz **pública** de cada elección.
Cualquier persona (votante, custodio, miembro de la Junta Electoral o
público en general) puede consultarlo sin autenticarse. Sus secciones,
descritas en detalle en [Estadísticas e Información
Pública]({{< ref "/docs/functionalities/stats_public_info.md" >}}), son:

- **Urna Electrónica**: muestra cada uno de los votos encriptados
  recibidos por el servidor, sin revelar las preferencias.
- **Estadísticas**: cantidad de votos recibidos, participación, y
  distribución de los votos en el tiempo.
- **Eventos**: registro cronológico de los hitos relevantes de la
  elección (apertura, modificaciones del padrón, cierre, etc.).
- **Resultados**: una vez publicados, los conteos por opción.
- **Verificación**: instrucciones y archivos necesarios para que
  cualquier persona pueda asumir el rol de [Verificador
  Externo]({{< ref "/docs/roles/verifier.md" >}}).

## Cabina de Votación

Es la interfaz que utiliza cada [Votante]({{< ref "/docs/roles/voter.md" >}})
durante la jornada electoral. Solamente se accede a ella después de la
[Autentificación
Externa]({{< ref "/docs/functionalities/auth.md" >}}) y de la
verificación de habilitación en el padrón. La cabina guía al votante a
través de los cinco pasos descritos en [Envío de
Votos]({{< ref "/docs/voting-steps/vote_casting.md" >}}): selección de
preferencias, encriptación del voto en el navegador, revisión, envío
final y descarga del Certificado de Votación.

## Portal del Custodio de Clave

Esta interfaz está reservada a las personas designadas como [Custodios
de Clave]({{< ref "/docs/roles/trustee.md" >}}). Permite las dos
operaciones críticas que cada Custodio debe realizar:

- Participar en la [Ceremonia de Creación de
  Claves]({{< ref "/docs/voting-steps/key_generation.md" >}}) antes del
  inicio de la elección, generando su clave privada y sincronizándose
  con los demás Custodios.
- Una vez cerrada la elección y realizado el precómputo, enviar su
  [desencriptación
  parcial]({{< ref "/docs/voting-steps/partial_decryptions.md" >}}) para
  contribuir al cómputo del resultado final.

## Panel del Administrador

Es la interfaz que utiliza el [Administrador]({{< ref "/docs/roles/admin.md" >}})
para configurar y operar la elección. Concentra las tareas de la fase
previa (carga del padrón, definición de preguntas y candidaturas,
registro de Custodios), de la jornada (apertura, monitoreo, eventuales
modificaciones del padrón) y del cierre (cierre de la elección, gatillo
del precómputo, publicación de resultados, eliminación posterior de la
elección). Todas las acciones del Administrador quedan registradas en
la sección **Eventos** del Portal de Información.

## Aplicación móvil para Custodios

Adicionalmente a las interfaces web, se encuentra en desarrollo una
aplicación móvil destinada a los Custodios de Clave. Su objetivo es
reemplazar el flujo de escritorio actual de la Ceremonia de Creación de
Claves y del envío de desencriptaciones parciales por un flujo móvil
nativo, simplificando la operación y reduciendo el equipamiento
requerido por cada Custodio. El [Código
Fuente]({{< ref "/docs/software/source_code.md" >}}) de esta aplicación
se publica en los repositorios `psifos_mobile_app` y
`psifos_mobile_crypto`.
