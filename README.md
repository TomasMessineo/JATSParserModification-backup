# Documentación del Plugin jatsParser para OJS

## Introducción

Lo que se estuvo desarrollando fue sobre un plugin ya existente llamado jatsParser. Este plugin es para OJS. Se desarrollaron nuevas herramientas y funcionalidades. En esta documentación se abordarán cuestiones más técnicas sobre las modificaciones realizadas en el plugin.

## Funcionalidad del Plugin

Este plugin se encarga de generar un documento PDF a partir de un archivo XML con el estándar JATS.

Inicialmente, el PDF generado contaba con una plantilla predefinida. Sin embargo, se realizaron cambios para poder:

- Cargar más metadatos de OJS.
- Considerar las traducciones de dichos metadatos (por ejemplo, título del artículo, subtítulo, resúmenes o palabras clave).
- Citar una referencia de manera correcta, dependiendo del estilo de citación que se esté utilizando.

Estas y otras funcionalidades serán detalladas más adelante.

## Proceso de Generación del PDF

La generación del PDF en este plugin se puede dividir en dos partes:

### 1. Conversión del XML a HTML

- **Proceso:**
  - El DOM del XML JATS, que se selecciona desde la interfaz "JatsParser" ubicada en la etapa de "Publicación" del artículo, se convierte en un DOM HTML.
- **Resultado:**
  - Este nuevo DOM contendrá los datos del contenido del artículo y se utilizará para generar el PDF con el contenido del artículo científico.

### 2. Generación de la Plantilla PDF

- **Proceso:**
  - Se utiliza una plantilla que obtiene los metadatos desde OJS y los imprime en el PDF.
- **Presentación:**
  - La plantilla es lo primero que se ve al abrir el PDF generado por el plugin; posteriormente se muestra el contenido generado a partir del HTML.
- **Herramienta:**
  - Para la creación de esta plantilla se emplea la librería de PHP TCPDF.

## Estructura de la Plantilla PDF

La plantilla se compone de tres secciones:

### Body

Contiene los siguientes metadatos:

- Logo de la revista.
- Logo de la institución.
- Número de la revista.
- DOI del artículo.
- ISSN de la revista.
- Link de la revista.

Se tuvieron en cuenta las traducciones disponibles para algunos metadatos cargados en OJS, tales como el título del artículo, el subtítulo, palabras clave y resúmenes.

### Header

Contiene los metadatos correspondientes al número de la revista y al DOI del artículo.

### Footer

Contiene el metadato referido a la licencia del artículo, el cual debe cargarse en el campo "Licencia URL" en la sección "Permisos y divulgación" de la etapa de Publicación.

## Flujo del Plugin y Detalle del Código

Para entender cómo se genera la plantilla PDF, se debe revisar el archivo `JatsParserPlugin.php` ubicado en la carpeta raíz "jatsParser" del plugin.

### Funciones Clave

#### register()

Esta función se encarga de llamar a algunos de los hooks que brinda OJS y de asignarles una función específica.

#### initPublicationCitation (Modificación aplicada)

Se utiliza en el hook `publication::add`. Su función es agregar una nueva fila a la tabla `publicationsettings` de la base de datos con el valor `jatsParser::citationTableData`.

#### editPublicationFullText

Esta función llama a `getFullTextFromJats()`, que convierte el DOM del XML JATS en un DOM de HTML.

#### createPdfGalley

Esta función es la encargada de crear el PDF y de hacerlo aparecer en la sección "Galeradas" de OJS.

### Modificaciones en la Función pdfCreation

La función `pdfCreation` se modificó para separar la creación del PDF de sus metadatos específicos. Con la modificación:

1. Se llama a `getMetadata()`, que retorna un arreglo con los metadatos.
2. Se envían a la nueva clase `Configuration`, que los almacena internamente.
3. Se instancia el objeto `TemplateStrategy`, que recibe el nombre de la plantilla y la configuración cargada previamente.

Se aplicó el patrón de diseño **Strategy**, lo que posibilita la creación de múltiples plantillas en el futuro.

## Normas para la Creación de Nuevas Plantillas

1. En `jatsParser/JATSParser/src/JATSParser/PDF/Templates` debe crearse un archivo .php con el nombre de la plantilla.
2. En la nueva clase creada, indicar como namespace:
   ```php
   namespace JATSParser\PDF\Templates;
   ```
3. Importar TCPDF con:
   ```php
   require_once(__DIR__ . '/../../../../vendor/tecnickcom/tcpdf/tcpdf.php');
   ```
4. Incluir los `use`:
   ```php
   use JATSParser\PDF\PDFConfig\Configuration;
   use JATSParser\PDF\PDFConfig\Translations;
   ```
5. En `TemplateStrategy`, agregar:
   ```php
   require_once __DIR__ . '/Templates/{nombre de la plantilla}.php';
   ```
6. Tomar como ejemplo la plantilla `TemplateOne`.

## Configuración en TemplateOne y Detalles de Configuration

La clase `Configuration` define varios arreglos que contienen:

- `header`, `footer`, `body`, `template_body` con los estilos de cada parte.
- `metadata`, que almacena todos los metadatos necesarios.

Se han definido los arreglos:

- `$supportedCustomCitationStyles`: define los estilos de citación con tabla (ej. APA).
- `$numberedReferencesCitationStyles`: define los estilos que requieren referencias numeradas (ej. IEEE).

Desde la plantilla (ejemplo, `TemplateOne`) se accede a la configuración con:
```php
$this->config->getHeaderConfig()
```

Esto devuelve un arreglo con la configuración y los metadatos correspondientes.

---

Este documento detalla las modificaciones y mejoras realizadas en el plugin jatsParser para OJS, asegurando una correcta generación de PDFs con metadatos y citaciones adecuadas.
