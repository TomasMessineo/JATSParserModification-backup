<div align="center">
  <h1>Plugin jatsParser para OJS</h1>
  <p>Documentación Técnica y Detallada</p>
  <img src="https://img.shields.io/badge/OJS-Compatible-blue" alt="OJS Compatible">
</div>

---

## Índice

- [Introducción](#introducción)
- [Funcionalidad del Plugin](#funcionalidad-del-plugin)
- [Proceso de Generación del PDF](#proceso-de-generación-del-pdf)
  - [1. Conversión del XML a HTML](#1-conversión-del-xml-a-html)
  - [2. Generación de la Plantilla PDF](#2-generación-de-la-plantilla-pdf)
- [Estructura de la Plantilla PDF](#estructura-de-la-plantilla-pdf)
  - [Body](#body)
  - [Header](#header)
  - [Footer](#footer)
- [Flujo del Plugin y Detalle del Código](#flujo-del-plugin-y-detalle-del-código)
  - [Funciones Clave](#funciones-clave)
  - [Modificaciones en la Función pdfCreation](#modificaciones-en-la-función-pdfcreation)
- [Normas para la Creación de Nuevas Plantillas](#normas-para-la-creación-de-nuevas-plantillas)
- [Configuración en TemplateOne y la Clase Configuration](#configuración-en-templateone-y-la-clase-configuration)
- [Citación de Referencias y la Tabla de Citas](#citación-de-referencias-y-la-tabla-de-citas)
- [Desarrollo y Funcionamiento de la Tabla de Citas (Código)](#desarrollo-y-funcionamiento-de-la-tabla-de-citas-código)
  - [Estructura del Directorio](#estructura-del-directorio)
  - [Explicación de la Clase TableHTML](#explicación-de-la-clase-tablehtml)

---

## Introducción

Lo que se estuvo desarrollando fue sobre un plugin ya existente llamado **jatsParser**. Este plugin es para **OJS**. Se desarrollaron nuevas herramientas y funcionalidades. En esta documentación se abordarán cuestiones más técnicas sobre las modificaciones realizadas en el plugin.

---

## Funcionalidad del Plugin

Este plugin se encarga de generar un documento **PDF** a partir de un archivo **XML** con el estándar **JATS**.

Inicialmente, el PDF generado contaba con una plantilla predefinida. Sin embargo, se realizaron cambios para poder:

- **Cargar más metadatos** de OJS.
- **Considerar las traducciones** de dichos metadatos (por ejemplo, título del artículo, subtítulo, resúmenes o palabras clave).
- **Citar una referencia** de manera correcta, dependiendo del estilo de citación que se esté utilizando.

Estas y otras funcionalidades serán detalladas más adelante.

---

## Proceso de Generación del PDF

### 1. Conversión del XML a HTML

- **Proceso:**  
  El DOM del XML JATS, que se selecciona desde la interfaz "JatsParser" ubicada en la etapa de "Publicación" del artículo, se convierte en un DOM HTML.

- **Resultado:**  
  Este nuevo DOM contendrá los datos del contenido del artículo (lo que estaba en el DOM del XML ahora estará en el DOM del HTML) y se utilizará para generar en el PDF el contenido del artículo científico.

### 2. Generación de la Plantilla PDF

- **Proceso:**  
  Se utiliza una plantilla que obtiene los metadatos desde OJS y los imprime en el PDF.

- **Presentación:**  
  La plantilla es lo primero que se ve al abrir el PDF generado por el plugin; posteriormente se muestra el contenido generado a partir del HTML (el contenido del XML JATS convertido a HTML).

- **Herramienta:**  
  Se emplea la librería de PHP **TCPDF** para crear el PDF y aplicarle los estilos necesarios.

---

## Estructura de la Plantilla PDF

### Body

Contiene los siguientes metadatos:

- Logo de la revista.
- Logo de la institución.
- Número de la revista.
- DOI del artículo.
- ISSN de la revista.
- Link de la revista.

Se tuvieron en cuenta las traducciones de metadatos (por ejemplo, título, subtítulo, palabras clave y resúmenes) y se implementaron encabezados para cada uno, mediante un arreglo clave-valor.

### Header

Contiene los metadatos correspondientes al número de la revista y al DOI del artículo.

### Footer

Contiene el metadato referido a la licencia del artículo.  
*Nota:* Este dato debe cargarse en el campo "Licencia URL" en la sección "Permisos y divulgación" de la etapa de Publicación (por ejemplo: [Creative Commons 4.0](https://creativecommons.org/licenses/by/4.0/)).

---

## Flujo del Plugin y Detalle del Código

Para comprender el flujo del plugin, se revisa el archivo **JatsParserPlugin.php** ubicado en la carpeta raíz `jatsParser`.

### Funciones Clave

- **register()**  
  Llama a algunos de los hooks de OJS y les asigna funciones específicas.

- **initPublicationCitation** *(Modificación aplicada)*  
  Se utiliza en el hook **publication::add** para agregar una nueva fila a la tabla **publicationsettings**.  
  Se inserta el valor `jatsParser::citationTableData` en el campo `setting_name`.

- **editPublicationFullText**  
  Llama a la función **getFullTextFromJats()** para convertir el DOM del XML JATS en un DOM HTML.

- **createPdfGalley**  
  Es la función que crea el PDF y lo agrega a la sección "Galeradas".  
  Además, llama a **pdfCreation**.

### Modificaciones en la Función pdfCreation

Esta función se modificó para separar la creación del PDF de sus metadatos:

1. **getMetadata()**  
   Retorna un arreglo `['clave' => 'valor']` con los metadatos.

2. **Configuration**  
   Se envía el arreglo a la clase **Configuration**, que lo almacena en el atributo `$config`.

3. **TemplateStrategy**  
   Se instancia con el nombre de la plantilla a utilizar y la configuración cargada.  
   *(Por el momento, la única plantilla es **TemplateOne**.)*

---

## Normas para la Creación de Nuevas Plantillas

1. Crear un archivo `.php` en `jatsParser/JATSParser/src/JATSParser/PDF/Templates` con el nombre de la plantilla deseada.
2. Indicar el namespace: `JATSParser\PDF\Templates`.
3. En la clase, agregar el siguiente bloque de código (para importar la librería TCPDF) de forma integrada:
   ```php
   require_once(__DIR__ . '/../../../../vendor/tecnickcom/tcpdf/tcpdf.php');

y los siguientes use:

use JATSParser\PDF\PDFConfig\Configuration;
use JATSParser\PDF\PDFConfig\Translations;

    En la clase TemplateStrategy, agregar:

    require_once __DIR__ . '/Templates/{nombre de la plantilla}.php';

    donde el nombre de la plantilla debe ser idéntico al nombre de la clase creada.
    Tomar como ejemplo la plantilla TemplateOne.

Configuración en TemplateOne y la Clase Configuration

En TemplateOne se trabaja con una configuración recibida como parámetro. La clase Configuration (en la carpeta PDFConfig) define varios arreglos:

    header: Estilos para los metadatos del HEADER.
    footer: Estilos para los metadatos del FOOTER.
    body: Estilos para el BODY (el artículo científico).
    template_body: Estilos para los metadatos del body de la PLANTILLA.
    metadata: Todos los metadatos que se utilizarán en la plantilla, recibidos en el constructor.

Desde la plantilla se accede a estos datos mediante métodos como getHeaderConfig(), getBodyConfig(), etc.

Además, se definen:

    $supportedCustomCitationStyles: Estilos de citación (APA, AMA, IEEE, etc.) que mostrarán una tabla para conectar citas con referencias (por ahora, solo se soporta APA).
    $numberedReferencesCitationStyles: Estilos en los cuales las referencias se numeran (por ejemplo, IEEE).
    Nota: En APA, no deben numerarse.

Función Body() en la Plantilla

En el constructor de la plantilla se llama a Body(), que invoca _prepareForPdfGalley de la clase PDFBodyHelper. Este método adapta el DOM HTML (el artículo científico) para el PDF, acomodando figuras y tablas, y, si el estilo de citación está soportado, utiliza la clase CustomPublicationSettingsDAO para obtener datos de la base de datos.
Traducciones en la Plantilla

Las traducciones se definen en la clase Translations (en PDFConfig). Un ejemplo del arreglo de traducciones:

'en_EN' => [
    'abstract' => 'Abstract',
    'received' => 'Received',
    'accepted' => 'Accepted',
    'published' => 'Published',
    'keywords' => 'Keywords',
    'license_text' => 'This work is under a Creative Commons License',
    'references_sections_separator' => '&'
],
'es_ES' => [
    'abstract' => 'Resumen',
    'received' => 'Recibido',
    'accepted' => 'Aceptado',
    'published' => 'Publicado',
    'keywords' => 'Palabras clave',
    'license_text' => 'Esta obra está bajo una Licencia Creative Commons',
    'references_sections_separator' => 'y'
],
// ... otros idiomas

Estos arreglos permiten que los metadatos se muestren correctamente según el idioma. Actualmente se soportan inglés, español y portugués.
Citación de Referencias y la Tabla de Citas

Durante el desarrollo surgió un problema en la forma de citar:

    IEEE: Se utilizan corchetes ([1]), y las referencias se numeran.
    APA: La cita debe mostrarse como (Giménez, 2025) o (Giménez, 2025, pp. 15).

Por ello, se desarrolló la Tabla de Citas.
Esta tabla aparece (por el momento) solo si se trabaja con APA en la configuración de jatsParser.

Elementos de la Tabla de Citas:

    Contexto:
    Porción de texto que muestra las 50 palabras anteriores a la cita para dar contexto.

    Referencias:
    Listado de las referencias citadas.

    Estilo de Citación:
    Menú desplegable con opciones: (Apellido, año), (año) y "Otro".
    Al seleccionar "Otro", se permite definir un formato personalizado.

Indicadores Visuales en la Tabla:

    Color verde:
    Opción predeterminada (por defecto "Apellido, nombre"). Se carga la última opción guardada.
    Color amarillo:
    Aparece cuando se cambia la opción, diferente de la opción predeterminada.
    Color rojo:
    Indica un error (por ejemplo, campos vacíos al seleccionar "Otro").

Al generar el PDF, el texto de la cita se reemplaza por el valor configurado en la tabla.

Importante: Cada cambio en el estilo de cita debe guardarse para que se refleje al generar el PDF.
Desarrollo y Funcionamiento de la Tabla de Citas (Código)

Este desarrollo se encuentra en la ruta JATSParser/classes. En este directorio se distinguen dos carpetas: components y daos.
Estructura del Directorio

    components:
        PublicationJATSUploadForm:
        Maneja la sección "JatsParser" en la etapa de publicación del artículo.
        Modificación: Se añadió un nuevo field (FieldHTML) para mostrar la Tabla de Citas.
        Uso de Configuration:
        Se llama a un método estático de la clase Configuration para verificar que el lenguaje de citación (almacenado en $citationStyle) se encuentre en $supportedCitationStyles. Si es así, se muestra la tabla; de lo contrario, no se muestra nada.

    daos:

        CustomPublicationSettingsDao:
        Contiene dos métodos fundamentales:
        Métodos:

            getSetting:
            Recibe el ID de la publicación, el nombre de la configuración y el localekey (ej., es_ES).
            Busca en la tabla publication_settings la fila jatsParser::citationTableData y, si existe, retorna un arreglo (obtenido desde un JSON decodificado).
            Nota: Si no se han guardado citas para un artículo en un idioma, se muestran valores por defecto.

            updateSetting:
            Se invoca al hacer clic en "Guardar Citas".
            Inserta o actualiza en la base de datos la configuración para el artículo y el idioma.

            Uso en el Archivo:
            Este método es llamado desde process_citations.php.

Explicación de la Clase TableHTML

La clase TableHTML procesa y crea un arreglo para renderizar el HTML de la Tabla de Citas, siguiendo los siguientes pasos:
1. Inicialización del DOM y del XPath

    Se instancia un DOMDocument y se carga el archivo XML proporcionado como parámetro.
    Se crea un DOMXPath (almacenado en la variable $xpath) para realizar consultas sobre el XML JATS.

2. Extracción de las Citas (extractXRefs())

Propósito: Buscar todas las citas dentro del XML (elementos dentro de <xref>).

Proceso:

    Se extraen atributos como id y rid.
    Se obtienen las 50 palabras anteriores (definidas en la constante CITATION_MARKER) para obtener el "Contexto".
    Se implementa una marca para diferenciar citas con el mismo rid en el mismo párrafo.

Resultado: Se crea el arreglo $xrefsArray con la siguiente estructura:

$xrefsArray = [
    'xref_id_unico' => [
        'context' => 'Contexto obtenido',
        'rid' => 'Valor del atributo rid',
        'originalText' => 'Texto original del xref sin la marca'
    ],
    // ... más citas
];

3. Extracción de las Referencias (extractReferences())

Propósito: Obtener todas las referencias dentro del elemento del XML.

Proceso:

    Se obtiene cada referencia de un <ref> con su atributo id.
    Se extrae el texto completo mediante el atributo reference y se obtienen los detalles con los autores.

Resultado: Se crea el arreglo $referencesArray con una estructura similar a:

$referencesArray['parser0'] = [
    'reference' => 'Referencia completa obtenida',
    'authors' => [
        'data_1' => [
            'surname' => 'Messineo',
            'year' => '2025'
        ],
        // ... más autores
    ]
];

4. Fusión de Arreglos (mergeArrays())

Propósito: Combinar los arreglos $xrefsArray y $referencesArray en un solo arreglo.

Resultado: Se genera un arreglo tipo diccionario con la siguiente estructura:

$mergedArray = [
    'xref_id1' => [
        'status' => 'default', // o 'not-default'
        'citationText' => '',  // Texto de la cita (modificable)
        'context' => 'Context 1',
        'rid' => 'parser_0 parser_1',
        'references' => [
            [
                'id' => 'parser_0',
                'reference' => 'Reference 1',
                'authors' => [
                    'data_1' => ['surname' => 'Smith', 'year' => '2020'],
                    // ... más autores
                ]
            ],
            [
                'id' => 'parser_1',
                'reference' => 'Reference 2',
                'authors' => [
                    'data_1' => ['surname' => 'Doe', 'year' => '2019']
                ]
            ]
        ]
    ],
    'xref_id2' => [
        'status' => 'not-default',
        'citationText' => '(Smith y Johnson, 2020; Doe et al, 2019)',
        'context' => 'Context 2',
        'rid' => 'parser_1',
        'references' => [
            [
                'id' => 'parser_1',
                'reference' => 'Reference 1',
                'authors' => [
                    'data_1' => ['surname' => 'Doe', 'year' => '2019']
                ]
            ]
        ]
    ]
];

Este arreglo final se utiliza para renderizar la Tabla de Citas en HTML.
