### Assistant (before) — 2026-11-21T03:23:32.076Z

Escribí el borrador de A en `deep-dive-A.md`.

## La idea: tres barreras independientes
1. **Por construcción.** Cada pack tiene un plan que lista lo que el LP **puede** ver. Los datos de una empresa excluida nunca llegan al armado, así que no hay nada que "olvidarse de borrar".
2. **Por inspección.** Un escáner lee el archivo final completo: cada XML, los Excel de los gráficos, las notas, la metadata e incluso los nombres de archivo de las imágenes (`pellory_logo.png`). Ahí busca rastros de lo excluido.
3. **Por trazabilidad.** Cada pack entregado guarda qué reglas se aplicaron, el plan, el resultado del escaneo y quién lo aprobó. Es lo que se le muestra al abogado del LP si llama.

**Regla general: ante la duda, se bloquea.** Si un nombre no coincide con ninguna empresa, si dos reglas chocan o si aparece una empresa nueva, el pack se detiene y escala. Nunca se genera "por si acaso".

## Las filas del Appendix B
- **Qamar (excusado de Pellory):** "Pellory Payments" tiene que coincidir con la empresa *antes* de armar nada. Si no coincide, se bloquean todos sus packs. Es la falla más peligrosa del sistema: una exclusión que no coincide se ignora en silencio. Además puse una regla nueva: **ningún commentary menciona otras empresas del portfolio**.
- **Hoshikawa (excusado de Vantrel en Opps I):** el capital account se toma tal cual del administrador. El look-through, en cambio, lo calculamos nosotros: hay que excluir Vantrel. Para validarlo, el total del look-through tiene que ser igual a la parte del NAV que el administrador le asigna a este LP.
- **Caldera (FOI contra la cláusula s.4.2):** hay conflicto en la valoración, así que sus 2 packs se bloquean. Se genera una vista previa con la opción más restrictiva para que IR y el abogado del fondo decidan. Lo que decidan queda guardado con quién, cuándo y por qué, y vale para los trimestres siguientes.
- **Oskervale (sin joinder, 3 fondos, PPTX):** las 9 empresas restringidas van sin KPIs ni gráfico, con el commentary en su variante sin financials.
- **Northmoor:** su pack no puede quedar "listo" hasta que llegue el cover note de los partners.

## Dos ideas que salieron al armarlo
- **IR revisa solo lo que cambió respecto del trimestre anterior**, no 214 planes: LPs nuevos, side letters modificados, empresas nuevas. Si el fondo hace una inversión nueva, todos los LPs con alguna exclusión en ese fondo quedan en revisión, porque nadie sabe todavía si están excusados de ella.
- **El incidente probablemente vino de copiar una página de otro deck**, que es lo que IR hace hoy ("copiamos la anterior"). Por eso el escaneo se repite cada vez que IR sube un PPTX editado, y sin un escaneo aprobado después de la última edición, el pack no se puede entregar.

## Supuestos para cuestionar
Estos supuestos son los más riesgosos, así que cuestiónalos:
1. **Agregados del fondo:** a un LP excusado se le muestran como los reporta el fondo. Ojo: Pellory es la única nota convertible de 750k, así que Qamar podría deducir su valor. Es el supuesto de mayor riesgo y no lo preguntamos.
2. **"Financials"** significa los KPIs y el revenue que reporta la empresa. La valoración no cuenta como financials.
3. **El look-through de Oskervale incluye a las empresas restringidas**, porque un agregado no es "financials de una empresa".
4. **Moneda del LP:** se aplica a las cifras del fondo, al capital account y al schedule. Los KPIs de cada empresa quedan en su propia moneda.
5. **Las imágenes de Finance no traen nombres de empresa.** El escáner no puede leer imágenes.

¿Qué te parece? Si estás de acuerdo, el siguiente paso sería `ALTERNATIVES.md`: los 3 enfoques que consideramos y por qué elegimos este. Ahí entra el tema de Python que dejamos pendiente.

### User — 2026-11-21T03:33:42.045Z

tengo una duda: imagina que ejecutamos nuestro query inicial y traemos un listado de packs que tiene o que cada lp puede ver..... para que las barreras tengan sentido lo unico que serviria seria que esa consulta traiga lo que PUEDE y lop que NO PUEDE!

entonces ahi si cobra sentido que la #1 se haga con lo que puede ver y la barrera #2 con lo que NO puede ver. de lo contrario estarian validando lo mismo

### Assistant (after)

Tienes razón, y además mi diseño tenía exactamente ese defecto. La lista de cosas prohibidas que usaba el escáner salía **del mismo plan** que arma el pack. Si el plan tiene un error, por ejemplo no aplicó la exclusión de Pellory para Qamar, ese error desaparece también de la lista de búsqueda. El escáner terminaba validando el plan contra sí mismo.

Le agregué un matiz a tu idea, que es lo que la hace funcionar: no basta con que la consulta devuelva las dos listas. **Las dos tienen que calcularse por caminos de código distintos.**
- **Lista de lo que PUEDE ver** (barrera 1): la función completa, con reglas, holdings, overrides y defaults. Con ella se arma el pack.
- **Lista de lo que NO PUEDE ver** (barrera 2): una función mínima que solo lee las restricciones del side letter. Son las exclusiones, el joinder sin firmar para las 9 empresas restringidas y el FOI. Al ser tan simple es fácil de auditar, y con ella trabaja el escáner.

Con las dos listas aparece además un **chequeo cruzado antes de armar**:
- **No se superponen:** nada puede estar en las dos. Si algo está, hay un error en el plan y el pack se bloquea.
- **Cubren todo:** cada holding del fondo, campo por campo, está en exactamente una de las dos. Nada queda "sin decidir", que es donde se esconden los olvidos.

Actualicé `deep-dive-A.md` con esto y con una falla nueva (#11: error en el cálculo del plan). La registré en `corrections-log.md` como la #5, y es de las fuertes. Ya hay 4 fuertes o medias más la débil, así que estás por encima del mínimo.

¿Seguimos con `ALTERNATIVES.md`?

