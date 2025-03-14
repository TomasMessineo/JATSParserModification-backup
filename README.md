# Documentación del Plugin jatsParser para OJS

## Introducción

Se ha desarrollado sobre un plugin ya existente llamado **jatsParser**, el cual es utilizado en **OJS** (Open Journal Systems). En este trabajo se han implementado nuevas herramientas y funcionalidades. Esta documentación aborda aspectos técnicos sobre las modificaciones realizadas en el plugin.

## Funcionalidad del Plugin

El propósito de este plugin es generar un documento **PDF** a partir de un archivo **XML** que sigue el estándar **JATS**.

Inicialmente, el PDF generado tenía una plantilla predefinida, la cual fue modificada para:
- Cargar más metadatos desde OJS.
- Considerar las traducciones de metadatos como el título del artículo, subtítulo, resúmenes y palabras clave.
- Permitir la citación de referencias de acuerdo con el estilo de citación utilizado.
- Otras mejoras que se detallarán más adelante.

## Proceso de Generación del PDF

La generación del PDF en este plugin se divide en dos partes:

1. **Conversión del XML JATS a HTML:**
   - El **DOM** del XML JATS seleccionado desde la interfaz "JatsParser" (ubicada en la etapa de "Publicación" del artículo) es convertido a un **DOM HTML**.
   - Este nuevo **DOM HTML** contendrá los datos del contenido del artículo, que luego serán utilizados en la generación del PDF.

2. **Plantilla del PDF:**
   - La plantilla obtiene los **metadatos** desde OJS y los imprime en el PDF.
   - Es la primera sección visible del PDF generado antes de mostrar el contenido del artículo.
   - Se utiliza la librería **TCPDF** en PHP para la creación del documento PDF.

### Secciones de la Plantilla

La plantilla del PDF se divide en tres secciones principales:

#### **Body**

Contiene los siguientes metadatos:
- Logo de la revista e institución.
- Número de la revista.
- DOI del artículo.
- ISSN de la revista.
- Link de la revista.
- Datos de los autores (nombre, ORCID, email, afiliación).
- Fechas de recepción, aceptación y publicación del artículo (esta última debe cargarse en "Fecha de publicación" dentro de la sección "Número" en la etapa de Publicación).

Se tuvo en cuenta la traducción de metadatos como título, subtítulo, resúmenes y palabras clave. Se implementó un **arreglo clave-valor** para manejar estas traducciones correctamente.

#### **Header**
- Número de la revista.
- DOI del artículo.

#### **Footer**
- Licencia del artículo (se debe cargar la **URL de la licencia Creative Commons** en el campo "Licencia URL" dentro de la sección "Permisos y divulgación" en la etapa de Publicación).

Ejemplo de URL de licencia:
> https://creativecommons.org/licenses/by/4.0/

## Estructura del Código

Para entender cómo se genera el PDF, debemos revisar el archivo `JatsParserPlugin.php`, ubicado en la carpeta raíz `jatsParser` del plugin.

Este archivo contiene la clase `JatsParserPlugin`, la cual gestiona el flujo del plugin. En esta clase se encuentra la función `register()`, encargada de registrar los **hooks** de OJS y asignarles funciones específicas.

### Hooks Modificados

#### **initPublicationCitation**
Se aplica al hook `publication::add`, que se ejecuta al aceptar un artículo. Esta función:
- Agrega una nueva fila en la tabla `publicationsettings` de la base de datos.
- En el campo `setting_name`, se almacena el valor `jatsParser::citationTableData`.
- La función de esta tabla se explicará más adelante.

#### **editPublicationFullText**
Esta función invoca `getFullTextFromJats()`, encargada de convertir el **DOM XML JATS** en **DOM HTML**.

#### **createPdfGalley**
Esta función:
- Crea el PDF y lo agrega en la sección "Galeradas" de OJS en la etapa de Publicación.
- Llama a `pdfCreation()`, sobre la cual se realizaron modificaciones.

## Modificación de la Función `pdfCreation()`

Antes, `pdfCreation()` instanciaba una clase que extendía `TCPDFDocument`, generando el PDF de manera desordenada. Se reorganizó el código para **separar la creación del PDF de la gestión de metadatos**.

### Nuevo Flujo de `pdfCreation()`

1. Llamar a `getMetadata()`, que devuelve un arreglo `['clave' => 'valor']` con los metadatos necesarios para el PDF.
2. Pasar estos metadatos a la clase `Configuration`, donde se almacenan en el atributo `$config`.
3. Instanciar `TemplateStrategy`, la cual recibe el **nombre de la plantilla** y la **configuración de metadatos**.
4. Aplicar el patrón de diseño **Strategy**, permitiendo la creación de múltiples plantillas en el futuro.
5. Instanciar un objeto con el nombre de la plantilla correspondiente (`TemplateOne`, por ahora la única plantilla disponible).

## Creación de Nuevas Plantillas

Para agregar nuevas plantillas correctamente, seguir estos pasos:

1. Crear un nuevo archivo `.php` en `jatsParser/JATSParser/src/JATSParser/PDF/Templates` con el **nombre de la plantilla**.
2. Indicar el `namespace` en la nueva clase:
   ```php
   namespace JATSParser\PDF\Templates;
   ```
3. Agregar el siguiente `require_once` en la nueva clase para importar la librería TCPDF:
   ```php
   require_once(__DIR__ .'/../../../../vendor/tecnickcom/tcpdf/tcpdf.php');
   ```
   También incluir:
   ```php
   use JATSParser\PDF\PDFConfig\Configuration;
   use JATSParser\PDF\PDFConfig\Translations;
   ```
4. En `TemplateStrategy`, agregar el siguiente `require_once`:
   ```php
   require_once __DIR__ . '/Templates/{nombre de la plantilla}.php';
   ```
   **Nota:** El nombre de la plantilla en esta ruta **debe coincidir** con el nombre de la clase creada en el paso 1.
5. Tomar como referencia `TemplateOne` para entender el uso de TCPDF.

Con esta estructura, el plugin `jatsParser` permite la correcta generación de PDFs personalizados en OJS, asegurando flexibilidad y escalabilidad en futuras mejoras.

---

### TemplateOne y la Configuración de PDF

En `TemplateOne`, se trabaja con una configuración recibida como parámetro. Esta clase, `Configuration`, está dentro de la carpeta `PDFConfig`.

Se encuentran definidos tres arreglos clave:

- **`$config`**: Contiene la configuración general utilizada para acceder a los metadatos y la configuración propia de la plantilla PDF y el estilo del artículo.
- **`metadata`**: Contiene todos los metadatos utilizados en la creación de la plantilla.
- **`template_body`**: Contiene los estilos para los metadatos del cuerpo de la plantilla.

#### Estructura de `$config`

```php
'header' => Contiene los estilos para los metadatos del HEADER
'footer' => Contiene los estilos para los metadatos del FOOTER
'body' => Contiene los estilos para el BODY (artículo científico)
'template_body' => Contiene los estilos para los metadatos del body de la plantilla
```

#### Acceso a la Configuración

Desde la plantilla (`TemplateOne`), se puede acceder a la configuración mediante métodos `get(NombreParte)Config`.

Por ejemplo, para obtener la configuración del encabezado:

```php
$this->config->getHeaderConfig();
```

Esto retornará un arreglo con la configuración del `header` y los metadatos:

```php
[
    'config' => { datos para el header del arreglo $config de Configuration },
    'metadata' => { todos los metadatos }
]
```

Este mismo patrón se repite para:

- `getTemplateBodyConfig()`
- `getFooterConfig()`
- `getBodyConfig()`

### Estilos de Citación Soportados

La clase `Configuration` define dos arreglos relacionados con los estilos de citación:

- **`$supportedCustomCitationStyles`**: Define los estilos de citación personalizados que mostrarán una tabla para conectar las citas con las referencias en el formato deseado (actualmente solo soporta APA).
- **`$numberedReferencesCitationStyles`**: Contiene los estilos de citación que tendrán referencias numeradas en el PDF (por ejemplo, IEEE usa referencias numeradas, mientras que APA no).

### Funcionalidad de `Body()`

La función `Body()` es llamada en el constructor de la plantilla. Dentro de esta función, se invoca el método `_prepareForPdfGalley()` de la clase `PDFBodyHelper`.

Este método:

- Recorre el DOM HTML del artículo científico.
- Adapta el contenido para su generación en PDF.
- Realiza consultas con `XPath` para acomodar figuras y tablas.
- Si el lenguaje de citación está soportado en `$supportedCustomCitationStyles`, usa `CustomPublicationSettingsDAO` para obtener datos de la base de datos, consultando la tabla `publication_settings`.

### Traducciones en `PDFConfig`

La clase `Translations` en `PDFConfig` contiene un arreglo con traducciones para diferentes idiomas.

#### Estructura del Arreglo de Traducciones

```php
[
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
    'pt_BR' => [
        'abstract' => 'Resumo',
        'received' => 'Recebido',
        'accepted' => 'Aceito',
        'published' => 'Publicado',
        'keywords' => 'Palavras chave',
        'license_text' => 'Este trabalho está sob uma licença Creative Commons',
        'references_sections_separator' => 'e'
    ]
];
```

### Importancia de las Traducciones

Las traducciones son utilizadas para generar el PDF en distintos idiomas. Los metadatos pueden estar cargados en diferentes idiomas, por lo que estas traducciones son necesarias para generar correctamente cada versión.

Ejemplo:

- En español: `Resumen`
- En inglés: `Abstract`
- En portugués: `Resumo`

Actualmente, los idiomas soportados son:

- **Inglés**
- **Español**
- **Portugués**

Se pueden agregar más idiomas según se requiera en futuras versiones del sistema.




