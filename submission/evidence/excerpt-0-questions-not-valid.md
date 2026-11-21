### Assistant (before) — 2026-11-21T00:57:16.844Z

Creé `` con el brief (PDF y texto extraído) y cuatro archivos de trabajo:

- `README.md`: entregables, límites de palabras y restricciones duras.
- `glossary.md`: los términos del brief explicados, más los que el brief usa sin definir (fondos, PowerPoint por dentro, render).
- `findings.md`: las trampas de cada fila del Appendix B y del Appendix A, y las tensiones entre lo que dice cada persona.
- `questions.md`: evaluación de tus 5 preguntas y una versión final propuesta.
- `corrections-log.md`: plantilla para ir anotando material para `CORRECTIONS.md`.

## Lo más importante que encontré

1. **El LP-0058 tiene una contradicción plantada.** Está marcado como reduced disclosure (sujeto a FOI), pero su side letter s.4.2 le da "company-level valuation detail". Tu pregunta 2 apunta justo ahí; conviene preguntarla con ese caso concreto.
2. **El riesgo de fuga más grande está en el PPTX, no en el PDF.** 31 LPs reciben el archivo PowerPoint. Los datos de una empresa excusada pueden quedar en lugares que no se ven al presentar: notas de los partners, slides ocultas con versiones viejas, el Excel que va dentro de cada gráfico, los totales del schedule y los links del menú. Qamar (LP-0102) es exactamente ese caso: excusado de Pellory y recibe el PPTX.
3. **El spike del ingeniero choca con las restricciones de Head of Engineering.** Usa python-pptx, que sería un segundo lenguaje en el stack. Además dejó que el LLM editara números, una de las tres slides falló, y solo se revisó con LibreOffice. A eso se suman la fuente Hartwell Sans (instalada solo en 5 laptops), el waterfall, el SmartArt y el texto que se encoge solo. Generar el PDF es la decisión técnica más cara.
4. **Corregir números choca con las ediciones manuales.** Hay correcciones después del sign-off en 3 de cada 4 trimestres, e IR seguirá retocando los decks en PowerPoint. Si regeneramos un pack, perdemos sus retoques.
5. **Hay que cruzar nombres entre fuentes.** La misma empresa aparece escrita distinto: "Ångström & Søn ApS" / "Angstrom and Son", o "PELLORY PAYMENTS CO" / "Pellory". Además Pellory usa otro año fiscal (FY27 Q1 = Aug-Oct) y Vantrel tiene dos series en el mismo fondo.

## Sobre tus 5 preguntas
- **Mantener 1, 2 y 5**, afinadas. La 5 la uní con el problema de las ediciones manuales.
- **Quitar la 3:** las citas del brief ya la responden (IR revisa todo y retoca).
- **Quitar la 4:** el archivo de packs de los últimos 8 trimestres está en la sección 4 y sirve de banco de pruebas.
- **Agregar dos:** quién produce el PDF final y con qué fuente; y si "nada sobre esa empresa" también cubre los agregados del fondo (NAV, TVPI, totales). Esa la puse con el ejemplo de Hoshikawa (LP-0166), que está excusado de Vantrel.

El borrador en inglés está en `questions.md`. Quedan dos decisiones tuyas. La primera: ¿cambiamos alguna por una pregunta de alcance sobre lo que prometió Head of Product? Eso es el commentary por tipo de LP en Q4 y los templates propios de los clientes de Q1. La segunda: ¿recortamos las sub-partes, por si cada una cuenta como pregunta aparte?

## Dos avisos
- **No sé qué son "[internal tool]" ni "[internal tool]".** No encontré nada con esos nombres en `/git`. ¿Es una herramienta tuya o un MCP? Por ahora todo está en la carpeta local.
- **`CORRECTIONS.md` pide pruebas de que tú corregiste a la IA.** Necesitas al menos 4 casos, y tiene que verse tu mensaje. Corrígeme explícitamente cuando me equivoque y guarda los transcripts de estas sesiones. En `corrections-log.md` dejé una lista de errores probables que vale la pena vigilar.

¿Seguimos cerrando las 5 preguntas o pasamos a elegir los dos deep dives?

### User — 2026-11-21T01:06:45.534Z

NVM con respecto a [internal tool].. es mi tool y esta en otro location.

respecto a las preguntas.. con la #3 me refiero a que necesito saber si ellos validan los resultados o solo como luce. La idea es que podamos realizar unitarios para validar esos resultados simepre, sin embargo, necesito un flag que escale a un humano cuando algo no cuadre.

y con la #4 es porque el master template no se puede compartir, entonces como validamos que nuestros resultados esten ok? data de prueba? mock ? etc.. esa era mi inquietud 
de hecho como no tengo claro el cambio horario del equipo de perry ya yas envie XD.. no queria perder la oportunidad.. entonces asume que esas 5 son las preguntas por que lo son!

### Assistant (after)

Tienes razón: leí la #3 y la #4 de forma literal y no entendí para qué las habías escrito. Las cinco quedan como las enviaste, y actualicé `questions.md`.

**Qué hay ahora en `questions.md`:** cada pregunta enviada tiene su intención real, qué cambia en el diseño según lo que respondan, y un supuesto de trabajo para no frenarnos mientras llega la respuesta.

**Para la #3 (validar resultados + escalar a un humano):** el pack no le llega a IR hasta pasar checks automáticos. Si alguno falla, el pack pasa a un estado `needs_human` con el motivo. Los checks son:
- Los totales del schedule y del capital account cuadran con el NAV del administrador.
- Cada número del pack tiene su fuente, y si alguno no la tiene, el check falla.
- Se revisa el PPTX completo, incluidas notas, slides ocultas y el Excel de cada gráfico, buscando empresas que ese LP no puede ver.
- El tipo de cambio aplicado es el de cierre del trimestre.
- Los números que cita el texto generado por el LLM existen en los datos.
- Los datos que faltan o llegan tarde (Ångström, Pellory) y las cifras corregidas (Dunmarrow) salen como aviso.
- Cuando dos reglas del LP se contradicen (LP-0058), el pack se bloquea.

Si IR solo mira el diseño, estos checks son la única barrera para los números. Si también revisa números, se suma una pantalla de revisión que muestra cada cifra junto a su fuente.

**Para la #4 (cómo probar sin el master template):**
1. **El archive casi equivale al template.** Los packs hechos a mano salen de la v6 y la v7, así que traen su estructura real y sirven de referencia de cómo debe verse un pack correcto.
2. **Un template sintético nuestro** que reproduce todo lo del Appendix A: el waterfall, el SmartArt, los links del menú, las slides ocultas, las notas y la fuente que no está instalada.
3. **Los 6 LPs del Appendix B como datos de prueba**, cada uno cubriendo una regla distinta.
4. **Tests de reglas generales** que se cumplen para cualquier LP: no aparece ninguna empresa excluida, los totales cuadran y los links llevan a slides que existen.
5. **Comparación visual de imágenes.** LibreOffice en CI solo sirve para detectar cambios entre versiones; la fidelidad real se revisa en PowerPoint con un grupo chico de packs.
6. **Un ensayo general:** regenerar Q3 2026 con nuestro sistema y compararlo con los packs que IR hizo a mano ese trimestre.

**Lo que no preguntamos queda como supuesto** en la tabla de claims. El PDF lo exporta IR desde PowerPoint después de revisar. Los agregados del fondo se muestran como los reporta el fondo, pero sin la empresa excusada; es el supuesto más riesgoso.

**Para `CORRECTIONS.md`:** esto ya es la primera corrección válida. Me equivoqué al interpretar tus preguntas, tú me corregiste y eso cambió el diseño. Quedó registrada en `corrections-log.md`.

Lo siguiente sería elegir los dos deep dives. Mis candidatos son el filtrado por permisos de cada LP, desde las reglas hasta la verificación del PPTX final (con Qamar, Hoshikawa y Caldera), y el ciclo de corrección después del sign-off cuando IR ya editó el pack (con Dunmarrow). ¿Te sirven o pensabas en otros?

