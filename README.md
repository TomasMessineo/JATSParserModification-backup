Documentación del Plugin jatsParser para OJS
Introducción

Lo que se estuvo desarrollando fue sobre un plugin ya existente llamado jatsParser. Este plugin es para OJS. Se desarrollaron nuevas herramientas y funcionalidades. En esta documentación se abordarán cuestiones más técnicas sobre las modificaciones realizadas en el plugin.
Funcionalidad del Plugin

Este plugin se encarga de generar un documento PDF a partir de un archivo XML con el estándar JATS.

Inicialmente, el PDF generado contaba con una plantilla predefinida. Sin embargo, se realizaron cambios para poder:

    Cargar más metadatos de OJS.
    Considerar las traducciones de dichos metadatos (por ejemplo, título del artículo, subtítulo, resúmenes o palabras clave).
    Citar una referencia de manera correcta, dependiendo del estilo de citación que se esté utilizando.

Estas y otras funcionalidades serán detalladas más adelante.
Proceso de Generación del PDF

La generación del PDF en este plugin se puede dividir en dos partes:
1. Conversión del XML a HTML

    Proceso:
    El DOM del XML JATS, que se selecciona desde la interfaz "JatsParser" ubicada en la etapa de "Publicación" del artículo, se convierte en un DOM HTML.
    Resultado:
    Este nuevo DOM contendrá los datos del contenido del artículo (lo que estaba en el DOM del XML ahora estará en el DOM del HTML) y se utilizará para generar en el PDF el contenido del artículo científico.

2. Generación de la Plantilla PDF

    Proceso:
    Se utiliza una plantilla que obtiene los metadatos desde OJS y los imprime en el PDF.
    Presentación:
    La plantilla es lo primero que se ve al abrir el PDF generado por el plugin; posteriormente se muestra el contenido generado a partir del HTML (el contenido del XML JATS convertido a HTML).
    Herramienta:
    Para la creación de esta plantilla se emplea la librería de PHP TCPDF, la cual permite crear un documento PDF y agregarle los datos y estilos necesarios.

Estructura de la Plantilla PDF

La plantilla se compone de tres secciones:
Body

Contiene los siguientes metadatos:

    Logo de la revista.
    Logo de la institución.
    Número de la revista.
    DOI del artículo.
    ISSN de la revista.
    Link de la revista.

En este apartado se tuvieron en cuenta las traducciones disponibles para algunos metadatos cargados en OJS, tales como el título del artículo, el subtítulo, palabras clave y resúmenes. Además, se consideró que tanto las palabras clave como los resúmenes deben estar acompañados de un encabezado que indique qué es cada cosa (por ejemplo: "Resúmen: --resúmen en español--", "Abstract: --resúmen en inglés--", "Palabras clave: --palabras clave en español--", "Keywords: --palabras clave en inglés--"). Esto se solucionó declarando un arreglo clave-valor que contiene como claves los localekey (es_ES, en_US, etc.) y como valores los ID de las traducciones; se detallará mejor esta parte más adelante.
Header

Contiene los metadatos correspondientes al número de la revista y al DOI del artículo.
Footer

Contiene el metadato referido a la licencia del artículo. Este dato debe cargarse en el campo "Licencia URL" en la sección "Permisos y divulgación" de la etapa de Publicación, ingresando la URL de la licencia Creative Commons (por ejemplo: https://creativecommons.org/licenses/by/4.0/).
Flujo del Plugin y Detalle del Código

Para entender cómo se genera la plantilla PDF, se debe revisar el archivo JatsParserPlugin.php ubicado en la carpeta raíz "jatsParser" del plugin. En este archivo se encuentra la clase JatsParserPlugin, que es la base para comprender el flujo del plugin y su funcionamiento.
Funciones Clave

    register()
    Esta función se encarga de llamar a algunos de los hooks que brinda OJS y de asignarles una función específica.

    initPublicationCitation
    (Modificación aplicada)
    Se utiliza en el hook publication::add (cuando se acepta el envío de un artículo). Su función es agregar una nueva fila a la tabla publicationsettings de la base de datos. En el campo "setting_name" de esta fila se insertará el valor jatsParser::citationTableData. La función de esta tabla se explicará más adelante.
    Cabe mencionar que, si se quiere agregar una nueva fila a la tabla publication_settings en la base de datos, ésta debe añadirse al esquema (ubicado en la función addToSchema) con un nombre específico. En esta función se ha agregado al esquema la nueva fila ya mencionada: jatsParser::citationTableData.

    editPublicationFullText
    Esta función llama a otra función denominada getFullTextFromJats(), la cual se encarga de convertir el DOM del XML JATS en un DOM de HTML.

    createPdfGalley
    Esta función es la encargada de crear el PDF y de hacerlo aparecer en la sección "Galeradas" de OJS, en la etapa de Publicación. Además, llama a la función pdfCreation, sobre la cual se han realizado modificaciones.

Modificaciones en la Función pdfCreation

La función pdfCreation se modificó para separar la creación del PDF de sus metadatos específicos. Anteriormente, en esta función se instanciaba una clase que extendía de TCPDFDocument, la cual creaba cada parte del PDF de una forma “mezclada” (una parte se generaba llamando a métodos de esa clase desde esta función, y la otra parte se creaba dentro de la clase instanciada). Esto mezclaba la creación del PDF, cuando la idea era delegar toda la creación a la clase correspondiente.

Con la modificación, ahora se realiza lo siguiente:

    Se llama a la función getMetadata(), que retorna un arreglo de la forma ['clave' => 'valor'] con los metadatos a utilizar para la creación del PDF.
    Una vez obtenido el arreglo con los metadatos, se envían a una nueva clase denominada Configuration, que los almacena internamente en un atributo llamado $config (esto se explicará más adelante).
    Se instancia el objeto TemplateStrategy, que recibe el nombre de la plantilla a utilizar y la configuración cargada previamente.

En esta clase se aplicó el patrón de diseño Strategy, lo que posibilita la creación de múltiples plantillas en el futuro. Es cuestión de manejar qué nombre de plantilla se envía como parámetro desde la función pdfCreation a TemplateStrategy. En esta clase se instancia un objeto cuyo nombre es idéntico al nombre de la plantilla recibido como parámetro; en el momento, la única plantilla creada es TemplateOne, por lo que se instanciará TemplateOne.
Normas para la Creación de Nuevas Plantillas

Para implementar nuevas plantillas de forma correcta, se deben seguir estas normas:

    En el directorio del plugin jatsParser/JATSParser/src/JATSParser/PDF/Templates debe crearse un archivo .php que contenga el nombre de la plantilla deseada.
    En la nueva clase creada en el paso 1, se debe indicar como namespace:
    JATSParser\PDF\Templates
    En la nueva clase creada, se debe agregar el siguiente require_once para importar la librería TCPDF, que sirve para crear el PDF:

require_once(__DIR__ . '/../../../../vendor/tecnickcom/tcpdf/tcpdf.php');

Además, se deben incluir los siguientes use:

use JATSParser\PDF\PDFConfig\Configuration;
use JATSParser\PDF\PDFConfig\Translations;

En la clase TemplateStrategy, se debe agregar un require_once indicando la ruta de la nueva clase para la plantilla, de la siguiente forma:

    require_once __DIR__ . '/Templates/{nombre de la plantilla}.php';

    El nombre de la plantilla en esta ruta debe ser exactamente el mismo que el de la clase creada en el paso 1.
    Tomar como ejemplo la plantilla TemplateOne para ver cómo se utiliza la librería TCPDF.

Configuración en TemplateOne y Detalles de la Clase Configuration

Notese que en TemplateOne se trabaja con una configuración recibida como parámetro. Esta clase Configuration se encuentra en la carpeta PDFConfig.

Como se puede observar, se han definido varios arreglos. Uno de ellos contendrá la configuración ($config), que se utiliza para acceder a los metadatos y a la configuración propia de la plantilla PDF, así como al estilo del artículo (recordemos que, por una parte, se genera la PLANTILLA que contiene los metadatos, y por otra, el artículo científico mediante un HTML).

Este arreglo se compone de:

    header: contiene los estilos para los metadatos del HEADER.
    footer: contiene los estilos para los metadatos del FOOTER.
    body: contiene los estilos para el BODY, que corresponde al artículo científico.
    template_body: contiene los estilos para los metadatos del body de la PLANTILLA.
    metadata: contiene todos los metadatos que se utilizarán en la creación de la plantilla. Estos se reciben como parámetro en el constructor de la configuración.

Desde la plantilla (por ejemplo, TemplateOne) se puede acceder a los datos de la configuración llamando a los métodos get(NombreParte)Config. Por ejemplo, con $this->config->getHeaderConfig() se retorna un arreglo con la configuración del header y los metadatos, similar a:

[
    'config' => {datos para el header del arreglo $config de Configuration},
    'metadata' => todos los metadatos
]

Esto se repite para todas las partes (TemplateBodyConfig, Footer y Body).

Otro aspecto que se define en la clase Configuration son los arreglos $supportedCustomCitationStyles y $numberedReferencesCitationStyles:

    $supportedCustomCitationStyles: Define aquellos estilos de citación (APA, AMA, IEEE, etc.) que mostrarán una tabla para conectar las citas con las referencias en el formato deseado. (Por el momento, solo se soporta APA).
    $numberedReferencesCitationStyles: En este arreglo se definen los estilos de citación en los cuales las referencias aparecerán numeradas en el PDF. Por ejemplo, en IEEE las referencias deben estar numeradas; en APA, en cambio, no deben numerarse, por lo que el valor "apa" no debe estar definido en este arreglo.

Función Body() en la Plantilla

Algo importante a mencionar es la función Body() que se llama en el constructor de la plantilla. En esta función se invoca el método _prepareForPdfGalley de la clase PDFBodyHelper. Esta clase recorre el DOM HTML (el artículo científico) y lo adapta para su posterior generación en el PDF. Se realizan consultas con XPath para acomodar las figuras y tablas contenidas en el HTML. Además, si el estilo de citación está soportado (definido en el arreglo $supportedCustomCitationStyles de la clase Configuration), se utiliza la clase CustomPublicationSettingsDAO para hacer una petición GET a la base de datos y obtener el valor del campo "settingvalue" de la tabla publication_settings, cuyo "settingname" es jatsParser::citationTableData.
Traducciones en la Plantilla

En la plantilla se utiliza un arreglo de traducciones definido en la clase Translations (ubicada en la carpeta PDFConfig). Este arreglo define, para cada idioma, la traducción de determinadas palabras. Por ejemplo:

'en_EN' => [
    'abstract' => 'Abstract',
    'received' => 'Received',
    'accepted' => 'Accepted',
    'published' => 'Published',
    'keywords' => 'Keywords',
    'license_text' => 'This work is under a Creative Commons License',
    'references_sections_separator' => '&'
],
'en_US' => [
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
],

Estas traducciones se utilizan para generar el PDF, ya que los metadatos pueden estar cargados en diferentes idiomas. Se pueden agregar más traducciones según se requiera. Por ahora, los idiomas soportados son: inglés, español y portugués.
Citación de Referencias y la Tabla de Citas

Durante el desarrollo surgió un problema respecto a la forma de citar una referencia, ya que la citación en IEEE difiere de la citación en APA.

    En IEEE, para citar se utilizan corchetes ([]), y las referencias deben estar numeradas. Por ejemplo, se mostrará [1] en el texto.
    En APA, la cita no debe aparecer como [1] o [2], sino que debe mostrarse como (Giménez, 2025) o (Giménez, 2025, pp. 15).

Debido a ello, se optó por desarrollar una nueva funcionalidad en el plugin jatsParser: la Tabla de Citas.

Esta tabla de citas, por el momento, solo aparece si en la configuración de jatsParser (en la sección de Módulos instalados de OJS) se indica que se está trabajando con APA. A futuro se pretende implementar una tabla para cada estilo de citación.

La Tabla de Citas está conformada por:

    Contexto:
    Es una porción de texto que hace referencia al lugar donde se está insertando la cita. Se muestran, si existen, las 50 palabras anteriores a la ubicación de la cita, para ofrecer una ayuda visual sobre su contexto en el artículo.

    Referencias:
    Son aquellas referencias que se están citando. Por ejemplo, si en una cita se mencionan 4 referencias, estas aparecerán en la tabla bajo un mismo contexto.

    Estilo de Citación:
    Se muestra un menú desplegable en el que se indica el formato que se desea para la cita. En APA hay tres opciones posibles: (Apellido, año), (año) y "Otro". Al seleccionar "Otro", se abrirá un campo de texto para especificar un formato personalizado.

Al seleccionar el estilo de cita para cada cita, se deben guardar los cambios haciendo clic en el botón "Guardar Citas". Esto guardará un JSON en la base de datos, específicamente en la fila jatsParser::citationTableData de la tabla publication_settings. Es importante destacar que cada artículo tendrá su propia tabla y configuración guardada, y que al guardar el JSON se utiliza el ID de la publicación (indicado en la tabla bajo el nombre "publication_id").

En la tabla se podrán observar los siguientes indicadores visuales en la columna "Estilo de Cita":

    Color verde: Es la opción predeterminada. Si aún no se han guardado citas para un artículo, se mostrarán las opciones por defecto (por ejemplo, "Apellido, nombre").
    Al guardar las citas, al recargar la página se cargarán nuevamente desde la base de datos y se seleccionará la última opción guardada para cada cita.
    Color amarillo: Aparece cuando se cambia una opción, es decir, cuando la opción seleccionada no coincide con la opción predeterminada obtenida de la base de datos.
    Color rojo: Indica un problema. Este color aparece, por ejemplo, cuando se intentan enviar campos vacíos después de seleccionar "Otro" en el menú de Estilo de Cita. Se mostrará un mensaje de error y los bordes del campo de texto se marcarán en rojo para señalar el error.

Al generar el PDF, una vez establecidos los estilos de cita para todas las citas, estos se reflejarán en el documento. Es decir, en el artículo, donde antes se mostraban [1] o [2], aparecerá el texto configurado en la tabla.

Importante: Cada vez que se realice un cambio en el estilo de cita en la tabla, se deben guardar los cambios para que se reflejen correctamente al generar el PDF.
Desarrollo y Funcionamiento de la Tabla de Citas (Código)

Este desarrollo se puede ver en la ruta JATSParser/classes. En este directorio se encuentran dos carpetas principales: components y daos.
Estructura del Directorio

    components:
    Contiene dos clases principales:

        PublicationJATSUploadForm:
        En esta clase (que ya formaba parte del plugin original) se gestiona la sección "JATSParser" en la etapa de publicación del artículo. Aquí se implementa la creación de botones y fields específicos que se mostrarán al usuario.
        Modificación: Se ha implementado un nuevo field, denominado FieldHTML, encargado de mostrar la Tabla de Citas.

        Uso de la clase Configuration: Para mostrar la tabla de citas se llama a un método estático de la clase Configuration. Se verifica que el lenguaje de citación seleccionado (almacenado en la variable $citationStyle, recibida desde un metadato en OJS) exista en el arreglo de lenguajes soportados para la tabla ($supportedCitationStyles).
            Si existe, se muestra la tabla en la sección "JATSParser" de la etapa de publicación.
            Si no, no se muestra nada.

    daos:
    Contiene la clase CustomPublicationSettingsDao, que cuenta con dos métodos fundamentales:
    Métodos en CustomPublicationSettingsDao

        getSetting:
            Parámetros: ID de la publicación (artículo), nombre de la configuración a buscar en la tabla "publication_settings" y el localekey (por ejemplo, es_ES).
            Funcionalidad: Busca una coincidencia en la fila con jatsParser::citationTableData considerando el ID de la publicación y el localekey.
                Si se encuentra coincidencia, retorna un arreglo (originalmente un JSON que se convierte a array mediante json_decode).
                Nota: Si para un artículo en un idioma determinado no se han guardado las citas, se mostrarán los valores por defecto (por ejemplo, "Apellido, nombre"). Si se han guardado, se mostrará la última opción seleccionada.

        updateSetting:
            Se ejecuta al hacer clic en el botón "Guardar Citas" en la Tabla de Citas.
            Funcionalidad: Inserta o actualiza en la base de datos la configuración necesaria para ese artículo en el idioma específico.
                Si ya existe una entrada (basada en el ID de publicación y el localekey), se actualiza el campo setting_value.
                Si no existe, se inserta una nueva fila con el setting_name de jatsParser::citationTableData.

        Uso en el Archivo:
        El método updateSetting es llamado desde el archivo process_citations.php.

Explicación de la Clase TableHTML

La clase TableHTML se encarga de procesar y crear un arreglo que se utilizará para renderizar el HTML de la Tabla de Citas. El proceso se realiza en varios pasos:
1. Inicialización del DOM y Preparación del XPath

    Se instancia un objeto DOMDocument y se carga en él el archivo XML (la ruta del XML se recibe como parámetro).
    Se utiliza el DOMDocument para instanciar un DOMXPath (almacenado en la variable $xpath), que se usará para realizar consultas en el XML JATS.

2. Extracción de las Citas: Función extractXRefs()

    Propósito:
    Realizar una consulta XPath para buscar en el XML todas las citas, que se encuentran en el elemento <body> bajo etiquetas <xref>.
    Proceso:
        Cada <xref> contiene atributos como id (identificador único) y rid (referencia a las citas; por ejemplo, "parser0 parser1 parser2" indica que se están citando tres referencias).
        Para cada cita, se extraen las 50 palabras anteriores (definidas en la constante CITATION_MARKER, cuyo valor por defecto es 50) para obtener el "Contexto" de la cita.
        Problema resuelto:
        Si en el mismo párrafo (<p>) existen dos <xref> con el mismo atributo rid, se implementa una marca para diferenciar cada cita y evitar que se ignore alguna.
    Resultado:
    Se guarda un arreglo $xrefsArray con la siguiente estructura:

    [
        'xref_id_unico' => [
            'context' => 'Contexto obtenido',
            'rid' => 'Valor del atributo rid',
            'originalText' => 'Texto original del xref sin la marca'
        ],
        // ...
    ]

3. Extracción de las Referencias: Función extractReferences()

    Propósito:
    Realizar una consulta XPath para obtener todas las referencias definidas en el elemento <back> del XML.
    Proceso:
        Cada referencia se encuentra en un elemento <ref> y tiene un atributo id (por ejemplo, "parser0").
        Se extrae:
            El texto completo de la referencia mediante <mixed-citation>.
            Los detalles específicos (fecha, autores, etc.) mediante <element-citation>.
    Resultado:
    Se genera un arreglo $referencesArray con una estructura similar a:

    $referencesArray['parser0'] = [
        'reference' => 'Referencia completa obtenida',
        'authors' => [
            'data_1' => [
                'surname' => 'Messineo',
                'year' => '2025'
            ],
            // ...
        ]
    ];

4. Fusión de Arreglos: Método mergeArrays()

    Propósito:
    Combinar los arreglos $xrefsArray y $referencesArray para generar un único arreglo que contenga toda la información necesaria para renderizar la Tabla de Citas.
    Resultado:
    Se crea un arreglo tipo diccionario con la siguiente estructura:

[
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
                    'data_1' => [
                        'surname' => 'Smith',
                        'year' => '2020'
                    ],
                    'data_2' => [
                        'surname' => 'Johnson',
                        'year' => '2020'
                    ]
                ]
            ],
            [
                'id' => 'parser_1',
                'reference' => 'Reference 2',
                'authors' => [
                    'data_1' => [
                        'surname' => 'Doe',
                        'year' => '2019'
                    ]
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
                    'data_1' => [
                        'surname' => 'Doe',
                        'year' => '2019'
                    ]
                ]
            ]
        ]
    ]
]

Este arreglo se utilizará para renderizar la Tabla de Citas en HTML.
