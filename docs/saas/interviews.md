# Guion de entrevistas — Phase 0 Stage B (roadmap 0.3)

> 15 entrevistas: ~10 pymes usuarias de Odoo (16–19, Community o Enterprise) + 5 agencias/partners.
> Duración objetivo: 30 min. Idioma: ES (usar EN si el entrevistado lo prefiere).
> Regla de oro (Mom Test): preguntar por **hechos pasados**, no por intenciones futuras. No hacer demo hasta el bloque 3.

## Hipótesis a validar

| # | Hipótesis | Se valida si… |
| --- | --- | --- |
| H1 | Extraer respuestas/informes de Odoo es un dolor recurrente y caro | ≥60% relata un caso real del último mes con tiempo/dinero perdido |
| H2 | Las escrituras con aprobación humana + auditoría son el desbloqueador de confianza | La objeción "no dejo que una IA toque mi ERP" desaparece al explicar el gate |
| H3 | Hay disposición a pagar en el rango Pro/Team ($49–149/mes) | ≥3 dan señal de pricing sin rechazo frontal y ≥3 se comprometen a piloto |
| H4 | (D11) Hay demanda de MCP-only: conectar su propio cliente IA a Odoo sin nuestro chat | Algún entrevistado ya usa Claude/ChatGPT/Cursor y quiere conectarlo a Odoo directamente |
| H5 | (Agencias) La consola multi-cliente es diferencial | Agencias describen trabajo manual recurrente cruzando datos de varios clientes |

## Screener (filtrar antes de agendar)

1. ¿Usáis Odoo en producción? ¿Versión y edición? → descartar <16 o solo pruebas.
2. ¿Cuántos usuarios activos? ¿Quién consulta datos (finanzas, ventas, operaciones)?
3. ¿Quién resuelve hoy una pregunta tipo "¿cuánto nos deben los clientes?" y cómo?
4. Agencias: ¿cuántas BBDD de clientes gestionáis?
5. ¿Disponibilidad para 30 min esta semana o la próxima?

## Guion (30 min)

### Bloque 1 — Contexto (5 min)

- Rol, tamaño de empresa, versión/edición de Odoo, módulos principales.
- ¿Quién NO usa Odoo directamente pero pide datos de Odoo a otros?

### Bloque 2 — Dolor real, sin mencionar la solución (10 min)

- Cuéntame la última vez que necesitaste un dato o informe de Odoo y no fue inmediato. ¿Qué hiciste? ¿Cuánto tardaste?
- ¿Qué informes pedís cada mes sí o sí? ¿Quién los prepara y cuánto le cuesta?
- ¿Habéis probado ya IA (ChatGPT, Copilot…) con datos de empresa? ¿Qué pasó?
- ¿Qué pasaría si mañana esa persona que "sabe sacar cosas de Odoo" no está?
- Agencias: ¿qué tarea repetís en todas las BBDD de clientes? ¿Cuánto os lleva al mes?

### Bloque 3 — Reacción a la solución (10 min)

Presentar en una frase: *"Chat con tu Odoo: conectas tu instancia en minutos con credenciales que ya tienes, preguntas en lenguaje natural, y cualquier cambio en el ERP requiere aprobación humana explícita y queda auditado."*

- Primera reacción espontánea. ¿Qué pregunta harías tú primero al sistema?
- Probar H2: ¿dejarías que escribiera en tu Odoo con ese flujo de aprobación? ¿Qué te frena?
- Probar H4: ¿usáis ya Claude/ChatGPT/Cursor? ¿Preferirías conectar **tu** asistente a Odoo (sin nuestro chat) pagando solo la conexión? *(línea MCP-only, D11)*
- Agencias: reacción a "un panel que consulta las BBDD de todos tus clientes a la vez".
- ¿Dónde NO usarías esto nunca? (límites de confianza, datos sensibles, RGPD/residencia EU)

### Bloque 4 — Señal de pricing y cierre (5 min)

- ¿Cuánto os cuesta hoy resolver esto (horas, consultora, módulos)? Anclar contra ese coste, no contra "lo que pagarías".
- Reacción a rangos: Pro $49/mes · Team $149/mes · MCP-only (precio menor, por conexión). ¿Cuál encaja y por qué?
- **Cierre-compromiso** (elegir uno, de menor a mayor): ¿te apunto a la waitlist? · ¿30 min de piloto con tu Odoo real? · ¿pagarías el piloto?
- ¿A quién más con Odoo me presentarías? (referidos = señal de dolor real)

## Plantilla de síntesis (una por entrevista)

```
fecha / empresa / rol / versión-edición Odoo / nº usuarios / ¿agencia?
H1 dolor:        [caso concreto + coste estimado]        señal: fuerte|media|nula
H2 writes:       [reacción al gate de aprobación]        señal: fuerte|media|nula
H3 pricing:      [ancla de coste actual + rango aceptado] señal: fuerte|media|nula
H4 mcp-only:     [cliente IA propio sí/no + interés]     señal: fuerte|media|nula
H5 fleet (agencias): [tarea cross-cliente + horas/mes]   señal: fuerte|media|nula
Compromiso:      waitlist | piloto | piloto pagado | nada
Cita literal más valiosa: "..."
Riesgo/objeción nueva: ...
```

**Gate G0 (recordatorio):** G0a ✅ · ≥3 pre-comprometidos a piloto · waitlist viva.
