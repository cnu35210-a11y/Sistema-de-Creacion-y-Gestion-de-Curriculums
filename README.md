# CV-Builder FX — Documentación del Proyecto

**Sistema de Creación y Gestión de Currículums** — aplicación de escritorio JavaFX que permite a un usuario registrarse, crear uno o varios currículums mediante un asistente por pasos, elegir una plantilla visual, previsualizarlos en tiempo real y exportarlos a HTML.

---

## 1. Stack tecnológico

| Componente | Detalle |
|---|---|
| Lenguaje | Java 11 |
| UI | JavaFX 13 (`javafx-controls`, `javafx-fxml`, `javafx-web`) |
| Serialización | Gson 2.10.1 (JSON) |
| Build | Maven (`javafx-maven-plugin`) |
| Clase principal | `com.creadorcv.App` |
| Persistencia | Archivos JSON locales en disco (sin base de datos) |

No usa `module-info.java` con dependencias estrictas más allá de lo básico (el archivo existe pero el `pom.xml` está configurado en modo classpath vía el plugin de JavaFX).

## 2. Flujo de la aplicación

```
Login.fxml
   │  (iniciar sesión / registrarse)
   ▼
Home.fxml  ──(tarjeta "+")──►  Plantillas.fxml
   │  (doble clic en un CV)          │ (elige plantilla: Clásico/ATS/Foto)
   ▼                                 ▼
Formulario.fxml  ◄────────────────────
   │  (asistente de 6 pasos)
   │   1. Datos Personales
   │   2. Experiencia Laboral
   │   3. Formación Académica
   │   4. Capacitación
   │   5. Habilidades
   │   6. Idiomas
   ▼
VistaPrevia.fxml
   (guardar / exportar a HTML / eliminar / volver a Home)
```

La navegación entre pantallas la gestiona `Router`, que reemplaza la raíz de la `Scene` única de la aplicación (patrón "single stage, swap root").

## 3. Estructura de paquetes

```
com.creadorcv
├── App.java                     → punto de entrada (extends Application)
├── controlador/                 → controladores JavaFX + estado de la app
├── modelo/                      → clases de datos (POJOs)
├── persistencia/                → guardado/carga en disco y exportación HTML
│   └── plantillas/               → generadores de HTML por plantilla
└── vista/                       → archivos .fxml y hoja de estilos .css
    └── pasos/                    → fragmentos FXML de cada paso del asistente
```

## 4. Capa `modelo` (datos)

- **`CurriculumVitae`** — raíz del modelo: agrupa `DatosPersonales` y las listas de `ExperienciaLaboral`, `FormacionAcademica`, `Capacitacion`, `Habilidad`, `Idioma`, más el campo `plantilla` (`"CLASICO" | "ATS" | "FOTO"`).
- **`DatosPersonales`** — nombre, apellido, correo, teléfono, dirección, enlace profesional (LinkedIn/GitHub), perfil profesional y foto (Base64 + MIME type).
- **`ExperienciaLaboral`** — empresa, puesto, fechas, flag "trabajo actual", logros.
- **`FormacionAcademica`** — institución, título, estado (Graduado/En curso/Incompleto), período.
- **`Capacitacion`** — nombre + institución (cursos complementarios).
- **`Habilidad`** — nombre simple (sin tipo ni nivel).
- **`Idioma`** — nombre + nivel (escala MCER: A1–C2, Nativo).
- **`Usuario`** — usuario + hash de contraseña (SHA-256).

## 5. Capa `controlador`

### Estado global y navegación
- **`Router`** — navega reemplazando el `root` de la `Scene`; captura cualquier excepción (no solo `IOException`) y muestra un `Alert` de error con la causa, para que ningún fallo de navegación quede en silencio.
- **`Sesion`** (singleton) — usuario autenticado, el `ContextoCV` en edición y el nombre de archivo actual (`null` si el CV es nuevo y no se ha guardado).
- **`ContextoCV`** — envuelve el `CurriculumVitae` en edición junto con listas `ObservableList` (una por cada sección) que alimentan las tablas del formulario; se comparte por referencia entre todos los sub-controladores del asistente.
- **`NotificadorUI`** — centraliza mensajes de estado (barra inferior) y errores (`Alert`) para desacoplar a los sub-controladores de la UI concreta.
- **`Utilidades`** — validación de correo y helper `orEmpty`.

### Pantallas principales
- **`LoginController`** — alterna entre modo "Iniciar sesión" y "Registrarme" en la misma pantalla; valida campos y usa `GestorUsuarios`.
- **`HomeController`** — pinta una grilla de "tarjetas" (tiles) dibujadas a mano con `Shape`s de JavaFX: una tarjeta especial con "+" para crear CV nuevo y una tarjeta por cada CV guardado (icono de documento con esquina doblada). Un clic selecciona (habilita eliminar), doble clic abre el CV en el formulario.
- **`PlantillasController`** — galería con `ToggleButton`/`ToggleGroup` para elegir la plantilla (Clásico por defecto) antes de pasar al formulario.
- **`FormularioController`** — orquesta el asistente de 6 pasos usando `fx:include`; cada paso vive en su propio fragmento FXML con su propio sub-controlador (`DatosPersonalesController`, `ExperienciaController`, `FormacionController`, `CapacitacionController`, `HabilidadController`, `IdiomaController`). Dibuja un indicador de progreso con círculos (activo/hecho/pendiente) y controla botones Atrás/Siguiente/Cancelar.
- **`VistaPreviaController`** + **`PreviewController`** — renderiza el HTML generado dentro de un `WebView`; permite guardar (pidiendo nombre si es nuevo), exportar a archivo `.html` vía `FileChooser`, eliminar el CV o volver a Home.

## 6. Capa `persistencia`

- **`GestorUsuarios`** — registro/login de usuarios en `usuarios.json` (raíz del proyecto); contraseñas cifradas con **SHA-256** (nunca en texto plano).
- **`RepositorioCV`** — guarda cada CV como `mis_cvs/<usuario>/<nombre>.json`; cada usuario tiene su propia carpeta; expone listar/cargar/guardar/eliminar.
- **`GestorJSON`** — serialización/deserialización del `CurriculumVitae` con Gson (pretty-print, `serializeNulls`).
- **`MotorExportacion`** — punto de entrada único para generar el HTML final: delega según `cv.getPlantilla()` en una de las tres clases de plantilla.
- **`plantillas/`**:
  - `PlantillaClasica` — barra lateral oscura + contenido en blanco ("Clásico Ejecutivo").
  - `PlantillaATS` — minimalista, sin foto, una sola columna, pensada para lectores ATS.
  - `PlantillaFoto` — foto circular arriba + contenido en dos columnas.
  - `HtmlUtil` — helpers compartidos de escape HTML y verificación de texto no vacío.

## 7. Vistas (`vista/*.fxml`)

| Archivo | Pantalla |
|---|---|
| `Login.fxml` | Acceso / registro |
| `Home.fxml` | Listado de CVs guardados (tiles) |
| `Plantillas.fxml` | Selección de plantilla visual |
| `Formulario.fxml` | Contenedor del asistente de 6 pasos |
| `vista/pasos/Paso*.fxml` | Los 6 fragmentos del asistente (Datos Personales, Experiencia, Formación, Capacitación, Habilidades, Idiomas) |
| `VistaPrevia.fxml` | Previsualización (WebView) + acciones finales |
| `estilos.css` | Hoja de estilos compartida por toda la app |

## 8. Puntos destacados de diseño

- **Un solo `Stage`, `root` intercambiable**: evita el overhead de abrir ventanas nuevas; toda la navegación es un simple reemplazo de nodo raíz.
- **Estado compartido por referencia** (`ContextoCV`): todos los pasos del asistente escriben sobre el mismo objeto, por lo que no hace falta "recolectar" datos al final.
- **Separación asistente/plantilla**: la plantilla elegida se guarda como un string en el propio `CurriculumVitae`, y el motor de exportación es agnóstico de la UI — el mismo HTML se usa tanto para la vista previa (`WebView`) como para el archivo exportado.
- **Manejo de errores de navegación robusto**: `Router.ir()` atrapa cualquier excepción (no solo `IOException`) y se la muestra al usuario en vez de fallar en silencio.
- **Sin base de datos**: toda la persistencia es JSON plano en el sistema de archivos, organizado por carpeta de usuario.

## 9. Cómo ejecutar

```bash
mvn clean javafx:run
```

(Requiere Maven y JDK 11+; las dependencias de JavaFX 13 se resuelven automáticamente vía el `javafx-maven-plugin`.)

---

*Documento generado a partir del código fuente subido (`CV_Builder_FX.zip`).*
