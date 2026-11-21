### Assistant (before) — 2026-11-21T02:57:15.433Z

De acuerdo, A + B, con B como el núcleo. Escribí el borrador largo de B en `deep-dive-B.md`, en español, para pensarlo. La versión final irá en inglés dentro de `DESIGN.md`, con unas 900 palabras.

## Deep dive B en una frase
El template se lee **una vez por versión** y se convierte en un **manifest**: un contrato que dice qué slides modelo existen y qué dato recibe cada hueco. Con eso, armar un pack es un programa en TypeScript que siempre produce el mismo archivo para el mismo input. Ningún LLM toca el XML ni los números.

## Los 5 pasos
0. **Preparar el template.** Hacemos un inventario de las slides y las clasificamos en tipos: cover, company page, schedule, etc. Las slides ocultas se marcan para no copiarlas nunca. Después asociamos cada hueco a un dato: un script lo propone cruzando un pack del archive con los datos de ese trimestre, el LLM solo sugiere los casos dudosos y **un humano confirma**. Para validar, volvemos a armar un pack del archive y lo comparamos con el original. Si llega una v8 que rompe el manifest, la generación con v8 queda bloqueada.
1. **Lista de slides del pack**, a partir del plan de permisos que sale del deep dive A. Por ejemplo, Qamar recibe Fund III sin Pellory.
2. **Datos ya calculados y formateados** antes de armar. El armado no decide nada. Con las filas del Appendix B:
   - **Vantrel en Fund III**: la company page suma las series B y C (costo 8,5m, valor 17,35m), pero el schedule mantiene las 2 filas del administrador.
   - **Vantrel en Opps I**: misma empresa, con los números de ese fondo.
   - **Tiles de KPIs**: 3 para Vantrel y 2 para Dunmarrow. Dunmarrow se muestra en GBP, sin convertir.
   - **Pellory**: se muestra su periodo fiscal tal cual, con aviso de que no coincide con Q3.
   - **Ångström**: está en 0 y no envió datos, así que va solo en el schedule (es un supuesto).
   - **Schedule de Fund III**: 31 filas se reparten en 14/14/3, y para Qamar en 14/14/2.
   - **Footer de LP-0031**: el nombre legal no cabe, así que se usa un nombre corto aprobado por IR.
3. **Armado en TypeScript.** Copiamos cada slide modelo y le llenamos el texto conservando el formato.
   - **Tiles de KPIs**: si se borra un tile, también se borra su animación.
   - **Gráficos**: se reescriben los números **y** se vuelve a crear desde cero el Excel que llevan adentro, para que no quede data de otra empresa. La barra verde se recalcula.
   - **Menú lateral**: se vuelve a enlazar por sección, y si una sección no está en el pack, se quita su item.
   - **Índice y limpieza**: el índice se numera por código. Se eliminan notas, slides ocultas, comentarios y metadata.
4. **Validación de estructura.** Revisamos que no haya referencias internas rotas y validamos contra el esquema oficial del formato, porque PowerPoint es estricto y LibreOffice no. Por cada versión que liberamos, se abre una muestra de packs **en PowerPoint real** para confirmar que no aparece el aviso de "reparar".

## Cómo falla
El archivo tiene una tabla con 12 formas de fallar, cada una con su ejemplo del Appendix B, cómo se detecta y cómo se mitiga. Las más importantes:
- **Deriva del template:** una shape renombrada que coincide con la shape equivocada.
- **Aviso de "reparar" en PowerPoint** por IDs duplicados al copiar 30 company pages.
- **Gráfico con data ajena** dentro del Excel incrustado.
- **Waterfall que no se puede editar.**
- **Preview engañoso:** LibreOffice muestra la fuente sustituta, y IR aprueba algo que en PowerPoint se ve distinto.

## Alerta para el plan
B solo se lleva unas **9 de las 15 semanas de ingeniería**. Quedan unas 6 para todo lo demás: ingesta, permisos (A), commentary, pantalla de revisión, correcciones y entrega. Hay que recortar. Lo primero que sacaría es la pantalla de onboarding del template: para Q4 mapeamos la v7 a mano en un JSON. También el waterfall, pasando al plan B de barras apiladas.

Todo esto es una propuesta mía, así que **cuestiónala**. Si algo te parece mal, esas correcciones valen mucho más para `CORRECTIONS.md` que la de las preguntas. Hay tres puntos donde no estoy seguro:
- Si el waterfall se puede editar igual que un gráfico clásico.
- Si PowerPoint recalcula el ajuste automático de texto al abrir el archivo.
- Si los nombres de shape sobreviven a las ediciones de IR.

¿Revisamos B juntos o pasamos a A?

### User — 2026-11-21T03:12:17.456Z

falta agregar una herramienta de validacion para confirmar que la data generada sea la correcta! 
algo que quisiera cambiar es que me parece que preparar el template de primero es perder el timpo.. yo primero filtraria la data (paso 1) y ya despues veo si vale la pena procesar todo lo del paso 0. lo invertiria! 


no se si esto sea un correccion pero:
1. a mi parecer aunque se que el template y el armado son mucho mas dificiles, hablando de planificacion por semanas, estamos subestimando el A que abarca permisos y reglas para mostrar/ocultar. siento que este es el core del producto y legalmente es lo que mas impactaria (desde mi punto de vista.. no se mucho de tema tampoco)
2. me parece que el llm en el paso 0 sobra. y romperia la regla de solo usarlo para commentary

### Assistant (after)

Tienes razón en tres de los cuatro puntos. En el orden tienes razón a medias. Voy uno por uno.

**1. Validador de datos: tienes razón, faltaba.** El paso 4 de B solo revisaba que el archivo abriera bien. Agregué un **paso 5** que es otro módulo, sin código compartido con el armado:
- Lee el PPTX final como lo leería un LP, incluyendo los Excel de los gráficos y el footer.
- Compara cada número contra los datos originales del administrador y de las submissions, no contra lo que calculamos nosotros. Así, si el cálculo intermedio salió mal, el error igual aparece.
- Recalcula las sumas por su cuenta: la suma de filas del schedule contra el total y el NAV, y Vantrel B + C en la company page.
- Si algo no cuadra, el pack queda en `needs_human` con la slide, el valor esperado y el encontrado.
- Se vuelve a correr cuando IR sube el PPTX editado.

El escaneo de fugas de A usa el mismo lector del PPTX.

**2. Invertir el orden: de acuerdo para el plan de construcción, con dos matices.**
- En el flujo de cada ciclo, el paso 0 no va "primero": se hace una sola vez por versión del template, fuera del ciclo. Lo dejé aclarado en `architecture-draft.md`.
- Al construir, sí: datos y permisos van primero, porque no dependen del template y se pueden probar sin él.
- Lo que no postergaría es un **spike de una semana** sobre los riesgos del template: el waterfall y el aviso de "reparar" de PowerPoint. Si eso falla, hay que saberlo en la semana 1 y no en la 6.
- Tu orden además tiene un beneficio extra. El reporte de permisos por LP le sirve a IR **aunque el armado no llegue a Q4**: les dice exactamente qué incluir en cada pack, que es justo lo que falló en el incidente.

**3. Subestimé A: sí, y es una corrección real.** Le había dado 9 de las 15 semanas a B y dejé A como "lo que sobra". Rehice el plan en `plan-draft.md`: A tiene ahora 3,5 semanas propias y B baja a ~6. Para eso B escribe a mano el mapeo del template de la v7 en un JSON y no tiene pantalla de onboarding. El total da unas 15 semanas, y al final anoté lo que no llega a Q4.

**4. El LLM en el paso 0 sobra: también es una corrección real, y la más fuerte.** Yo mismo había dicho "el LLM solo para prosa" y después lo metí en dos lugares más: el mapeo del template y la unificación de nombres. El problema no es solo que contradice la regla. Un mapeo equivocado no produce un error puntual, produce el mismo error en los 214 packs. Y no hace falta: son unos 12 tipos de slide, que se mapean a mano en un día. Lo quité de los dos lugares y saqué esas filas de `cost-estimate.md`. El total casi no cambia.

**Para `CORRECTIONS.md`:** el log ahora tiene 4 entradas.
- **#2 (el LLM en el paso 0) y #3 (subestimar A):** son fuertes. Tu mensaje muestra el error y lo que cambió por él.
- **#4 (faltaba el validador):** es media, porque los checks ya estaban en `questions.md`, aunque B no los usaba.
- **#1 (la de las preguntas):** sigue marcada como débil.

Con estas cuatro llegas al mínimo, y conviene conseguir una o dos más fuertes en el camino.

¿Pasamos al deep dive A?

