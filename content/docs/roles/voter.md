---
title: "Votante"
slug: voter
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
## Designación del Votante

El rol de Votante recae en cada persona que esté habilitada por el padrón
electoral para participar en una elección dada. La habilitación es
responsabilidad del [Administrador]({{< ref "/docs/roles/admin.md" >}}) al
momento de configurar la elección. Los datos necesarios para caracterizar a
un votante (nombre, ponderación, grupo y su identificador en el servidor de
autentificación) están descritos en [Caracterización de
Votantes]({{< ref "/docs/functionalities/about_voters.md" >}}).

## Rol del Votante

Durante la jornada electoral, cada Votante puede ingresar a la plataforma
para emitir su sufragio. Para ello, el Votante debe primero autenticarse en
el servidor externo, conforme al protocolo descrito en [Autentificación
Externa]({{< ref "/docs/functionalities/auth.md" >}}). Validadas sus
credenciales, el sistema verifica si la persona está habilitada en el
padrón y, en caso afirmativo, le permite el acceso a la Cabina de Votación.

Dentro de la Cabina de Votación, el Votante emite su voto siguiendo el
procedimiento descrito en [Envío de
Votos]({{< ref "/docs/voting-steps/vote_casting.md" >}}). El Votante puede
ingresar a votar las veces que estime necesario durante la jornada, pero
solamente se contabilizará el último voto emitido. Esta mecánica funciona
también como [medida anti
coerción]({{< ref "/docs/functionalities/coercion.md" >}}) y permite,
además, rectificar un voto enviado por error.

Una vez emitido cada voto, el Votante recibe un Certificado de Votación que
acredita que el servidor recibió correctamente su voto encriptado, sin
revelar el contenido de sus preferencias. Adicionalmente, el Votante puede
verificar en cualquier momento que su voto encriptado se encuentre
desplegado en la Urna Electrónica del [Portal de
Información]({{< ref "/docs/functionalities/stats_public_info.md" >}}).

Después de publicado el resultado de la elección, el Votante (al igual que
cualquier otra persona) puede ejercer también el rol de [Verificador
Externo]({{< ref "/docs/roles/verifier.md" >}}), descargando el archivo de
verificación y ejecutando el script descrito en [Verificación de la
Elección]({{< ref "/docs/voting-steps/verification.md" >}}). De esta manera,
puede convencerse de forma independiente de que el resultado publicado es
consistente con los votos efectivamente emitidos.

Durante toda la jornada electoral, el Votante cuenta con el apoyo de la
[Mesa de Ayuda]({{< ref "/docs/functionalities/help_desk.md" >}}),
disponible mediante chat para resolver dudas o problemas técnicos.
