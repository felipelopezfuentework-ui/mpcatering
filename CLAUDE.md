# MP Catering — sistema interno (Mariana Pages Catering)

Toda la app está en **`index.html`** (~11k líneas: HTML + CSS + un único `<script>` al final). Sin build. Firebase (Firestore + Auth Google) por CDN; jsPDF 2.5.1, pdf.js, SheetJS por CDN.
Usuario: dueño del negocio, no técnico. Hablar en español rioplatense, resúmenes breves, sin jerga. Si un pedido es ambiguo, preguntar con opciones antes de programar.

## Cómo trabajar (ahorrar tokens)
- Buscar por **nombre de función** con Grep (los de abajo) y leer solo ese tramo. No leer el archivo entero.
- Ediciones grandes: script Python con `io.open(..., newline='')` + `assert s.count(a)==1`. Evitar `sed` con regex/`$`/`\n` (se rompe el escapado).
- Verificar sintaxis: extraer el último `<script>` a un .js y `node --check`.
- PDFs: arnés en el scratchpad que extrae `presuDibujarPDF` + constantes y lo corre con `jspdf@2.5.1` en node; revisar con `pymupdf` (texto y PNG).
- Al terminar: commit en español + `git push origin main` sin preguntar. Actualizar este archivo si cambia algo de lo que describe.

## Datos
- `DB` global en memoria. Firestore: doc `mp_data/main` (`FB_DOC`, guarda el resto vía `sbSave`, que **excluye** las colecciones propias) y colecciones `insumos`, `proveedores`, `subproductos`, `platos`, `maestro_subs`/`maestro_platos` (doc `lista`), `comanda_periodos`, `comanda_eventos`, `maestro_clientes`, `stock_conteos`, `presupuestos`. Listeners `iniciarListeners*` en `entrarComo()`.
- Recetas se identifican por **nombre con prefijo**: `INSUMO - `, `SUB - `, `PLATO - ` (`nombreDocSeguro` para el id del doc).
- Autoguardado genérico: eventos `change`/Enter → `sbSave(true)` o `guardarEventoActual()` (si está dentro de `#modal-evento-detalle`). Modales con cierre que guarda: `CIERRE_SEGURO_MODAL`.
- Tab nueva → agregarla en `refrescarVistaActual` y en el switch de `switchTab`.
- Acceso: `EMAILS_AUTORIZADOS` (login); Presupuestos solo `EMAILS_PRESUPUESTOS` (Mariana, admmppcatering y felipelopezfuentework).

## Módulos (`cambiarModulo(perfil)`; tabs en `TABS_PROD/COT/COM/STOCK/PRESU`)
- **Producción** (insumos, subproductos, platos, proveedores): `renderInsumos`, `abrirModalSub`/`guardarSub`→`_guardarSubReal`, `abrirModalPlato`/`guardarPlato`→`_guardarPlatoReal`, vistas `_renderListaRecetas/_renderCardsRecetas/_renderTablaRecetas` (`_tipoMeta`), **sin merma** (decisión del cliente: los kg son lo que se compra y la pérdida queda en el rinde): costo ingrediente `calcCostoIng` = kg × costo unitario (`precio_kg`), costo por porción = suma ÷ porciones; `recetasCorregirCostoMerma` limpia merma/kgBruto/merma_receta de recetas viejas al cargar; rendimiento `rendUpdate`/`rendUpdateCosto`, totalizadores discretos `_totReceta`/`_totFilaHtml`/`_totBarraHtml` (pie de tabla en modales y vista Lista) y `_totPestanaRecetas`/`ins-tot` (resumen por pestaña), `moverRecetaEntreListas` (pasar/duplicar sub⇄plato), import Excel `importRecetas`, cascada de costos `recalcularRecetasPorInsumo`, actualizar precios desde lista de proveedor `handleListaPrecios` (equivalencias `lpCandidatos`/`LP_ABREV`, más barato por kg, C/DTO ÷ 1,21, los seguros se aplican y guardan solos al cargar (`lpAplicarVarios`: guarda insumos en lotes y cada receta afectada una sola vez vía `_recetasDiferidas`), en el modal quedan solo los dudosos/variación >40% para confirmar ✓ o descartar ✕; único botón de carga en Insumos, el viejo "Importar Excel" se quitó). Marca de precio: campos `precio_actualizado`/`precio_fuente`/`precio_producto`, columna "Actualizado" (`insMarcaActualizado`; editar a mano marca "✎"), PDF receta `descargarRecetaPDFPorNombre`.
- **Cotización**: Cotizador `costeoRows` (en memoria, no se guarda), `renderCosteoTable`, `cotizRecetaInfo` (costo/porción desde la receta; sin porciones → alerta, no suma), `exportarCoteoPDF`; Precio Final (conformación del precio, por ítem): `pfItems` (costo, precio de venta unitario `row.precioVenta` escrito a mano o por markup del concepto `pfMarkups`/`PF_ORDER`, ganancia $ y % sobre costo), `pfSetPrecio`, `aplicarMarkupGlobal(val,pisar)`, `exportarPrecioFinalPDF` (interno, por ítem). Tipos: plato, sub, vajilla, personal, seguro, descartables, flete.
- **Fichas Técnicas / Comanda**: períodos y eventos (`abrirPeriodo`, `abrirEvento`, `renderEventoSheet`, `guardarEventoActual`), import desde PDF de presupuesto (`extraerDatosPDF`), menú por secciones, checklists, PDF del evento (`exportarEventoPDFById`). Maestro de clientes: `abrirModalCliente`/`guardarCliente` (hace `set()` del doc completo: **todo campo nuevo va en `datos`**), `getHistorialCliente`.
- **Stock**: conteos (`stock_conteos`), importación de planilla.
- **Presupuestos**: `abrirModalPresupuesto`, `presuRecolectarDatosDesdeModal`, `presuGuardarDesdeModal` (guarda inmediato; Ver/Descargar PDF también guardan), `cerrarModalPresupuesto`/`cancelarModalPresupuesto`, `duplicarPresupuesto`, `presuMigrar` (corrige textos por defecto viejos en docs guardados), `presuDibujarPDF(d, 'ver'|'descargar')` con `pagePortada/pageEvento/pageMenu/pagePresupuesto/pagesCondiciones/pageGracias`, corrector `presuRevisarOrtografiaTodo` (LanguageTool API), `presuFormatoPesos`.

## Reglas de negocio y decisiones del cliente
- Subproductos llevan **solo insumos**; platos llevan insumos y subs. Un sub usado en platos no se "pasa" a plato (sí se duplica).
- Presupuesto nuevo: precargados solo fotos, intro del menú, términos y cierre; estaciones/precios/incluye vacíos.
- PDF presupuesto: fecha del evento siempre "(a confirmar)"; cartel "Seleccionar 1 opción para todos los comensales" por estación (casilla, tildada por defecto); hojas de menú: hasta 3 estaciones, ≤3 platos sobrantes no abren hoja (se compacta); frase fija de validez 7 días + IPC; servicio de salón "$ X por cada camarero (se sugiere 1 cada 10 comensales)" (sin cantidad); precios en formato `$ 55.000`; cliente en la tapa; "Solicitante" (campo `referentes`); firma "Mariana Pages Palenque" (sin tilde).
- Doc de Firestore máx. 1 MiB: fotos de presupuestos se recomprimen (`presuAjustarTamano`).
- Pendiente de definir: módulo "Pendientes" (propuesta hecha, sin implementar); facturación ARCA (solo charla).
