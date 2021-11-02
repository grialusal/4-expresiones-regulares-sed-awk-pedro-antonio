# Sesión IV - Expresiones regulares: grep, sed y awk

Herramientas computacionales para bioinformática: UNIX, expresiones regulares y shell script

======================================

Nota: Crea enlaces simbólicos a los directorios con GTFs y BEDs que creaste en la anterior sesión antes de comenzar. Añade el fichero "cromosomas.txt" a la carpeta "genes". 

* * *

### 1. Grep avanzado

En la anterior práctica, vimos que el filtro `grep` servía para encontrar patrones en las líneas del stream de texto que recibe por su entrada estándar. 
En esta sesión, continuaremos explorando las posibilidades que ofrece esta herramienta para realizar búsquedas un poco más complejas. 


#### 1.1 Expresiones regulares
Como ya comentamos en la sesión anterior, para definir patrones en `grep` emplearemos __expresiones regulares__  (o "regexps" para acortar). De hecho, `grep` es un acrónimo de "**g**lobal **r**egular **e**xpression **p**rint"! Las expresiones regulares son cadenas que contienen caracteres con significado especial, que son los que nos permiten buscar patrones. Estos caracteres especiales se denominan *metacaracteres*. 

He aquí una tabla con todos los metacaracteres soportados y su descripción:


| Metacaracter(es) | Descripción                                                                                                                                                                                            |
|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| .                | Casa con exactamente un caracter                                                                                                                                                                             |
| [ ]              | Casa con exactamente un caracter de una clase de caracteres (ver siguiente apartado)                                                                                                                                          |
| ^                | Casa con el principio de una línea o palabra                                                                                                                                   |
| \$                | Casa con el final de una línea o palabra                                                                                                                                                          |
| *                | Casa con **cero o más** ocurrencias del caracter (o grupo de caracteres/expresiones) inmediatamente a la izquierda                                                                                                           |
| ?                | Casa con **cero o una** ocurrencias del caracter (o grupo de caracteres/expresiones) inmediatamente a la izquierda                                                                                                            |
| +                | Casa con **una o más** ocurrencias del caracter (o grupo de caracteres/expresiones) inmediatamente a la izquierda                                                                                                            |
| {n}              | Casa **exactamente con n** ocurrencias del caracter (o grupo de caracteres/expresiones) inmediatamente a la izquierda                                                                                                                      |
| {m,n}            | Casa **m a n** ocurrencias (intervalo cerrado) del caracter (o grupo de caracteres/expresiones) inmediatamente a la izquierda                                                                                                 |
| \|               | Combina expresiones regulares con un OR                                                                                                                                                                 |
| \                | Escapar metacaracteres. Se usa cuando se quiere casar con un metacaracter literalmente. Por ejemplo, `\.` casa con un punto, y `\?` casa con un símbolo de cierre de interrogación.                                                                   |
| ( )              | Usado para agrupar caracteres en una única unidad, a los que se le podrán aplicar modificadores como `*` o `+`. También se usa para delimitar el alcance de una expresión OR (`|`), o en expresiones avanzadas para insertar reemplazos. |



#### 1.2 Clases de caracteres
Como muestra la tabla del apartado anterior, las clases de caracteres se definirán entre corchetes `[]`. Una clase de caracteres simplemente indica "casar con cualquiera de estos caracteres". Por ejemplo, `[abcABC]` expresa "casar con las letras `a`, `b` o `c` en mayúsculas o minúsculas. Cuando usemos los corchetes para definir clases de caracteres, los símbolos `^` y `-` adquirirán significado especial:

- Cuando se use `-` entre números o letras, expresa un rango. Por ejemplo, `[a-zA-Z]` quiere decir cualquier letra; `[0-5]` quiere decir cualquier número entre 0 y 5. 
- Un `-` al principio de la secuencia de caracteres dentro de los corchetes, significará el símbolo en sí, es decir, `[-;ab]` significará casar con cualquiera de los caracteres `-`, `;`, `a` o `b`. 
- Un `^` al principio de la secuencia de caracteres dentro de los corchetes, tendrá el significado de *negación*. Es decir, casa con cualquier cosa que no esté en la clase. Por ejemplo, `[^ABC]` significa "cualquier caracter que no sea `A`, ni `B`, ni `C`." 
- Cuando `^` *no esté* al principio de la secuencia de caracteres dentro de los corchetes, entonces no tiene significado especial. Representará el mismo caracter `^`. Ejemplo: `[:;^%]` quiere decir: "casar con `:`, `;`, `^`, o `%`." 
- Todos los demás caracteres, incluyendo `?`, `*`, `(`, `)`, o `+` se interpretan _literalmente_ (en otras palabras, no como metacaracteres), como sucedería en `[?*()+]`. 


#### 1.3. BRE vs ERE

En el principio de los tiempos, `grep` admitía sólo el conjunto de metaracteres `^`, `$`, `.` y `*`. Después, se añadieron algunos más, como `()`, `{}`, `?`, `+` y `|`. 

Las expresiones regulares que sólo emplean el primer grupo de caracteres originales se denominan *Expresiones Regulares Básicas* (Basic Regular Expressions o BREs), mientras que las que utilizan también el segundo conjunto más nuevo se conocen como *Expresiones Regulares Extendidas* (Extended Regular Expressions o EREs). Para mantener la [retrocompatibilidad](https://es.wikipedia.org/wiki/Retrocompatibilidad) con programas escritos para las versiones más antiguas de `grep` (aquellas que sólo soportaban BRE), los desarrolladores optaron por implementar las ERE en las nuevas versiones de `grep` de manera que los metacaracteres específicos de BRE tendrían que ser escapados (con el símbolo `\`) para asegurar que los programas basados en BRE funcionarían igual en las nuevas versiones. Este sería el caso por ejemplo de aquellos programas que usasen los nuevos metacaracteres en sentido literal. Si no se hubiese adoptado esta medida, las expresiones regulares habrían perdido su significado original! 

Vamos a ver cómo esto nos afecta con un ejemplo. Imagina que queremos buscar todas las líneas en el fichero `Drosophila_melanogaster.BDGP6.28.102.gtf` relativas a cualquier gen con un id que contenga exactamente seis ceros. Siguiendo la sintaxis explicada en la tabla de arriba, tendríamos que buscar líneas que contuviesen `gene_id` (literal), un espacio en blanco seguido de un id entre comillas `"`, que tendría seis ceros `0{6}` entre 0 o más caracteres `.*`. Fíjate que usamos la comilla simple para marcar la expresión regular y usamos `wc -l` para contar el número de líneas que coincidan. 

```
abenito@cpg3:~/sesion-iv/gtfs$ grep 'gene_id ".*0{6}.*"' Drosophila_melanogaster.BDGP6.28.102.gtf | wc -l
0
```

Vaya! No existe ninguno así? Sí. Lo que pasa es que grep no lo encuentra porque está tratando de casar exactamente la secuencia `0{6}` (`0`, `{`, `6`, `}`) en el fichero la cual, por supuesto, no existe. 

Para arreglarlo, tenemos que indicarle a `grep` que `{` y `}` son caracteres especiales (recuerda, en BRE estos caracteres no existían), ya que usa BRE por defecto. Ahora, en vez de usar `wc -l`, le indicamos a `grep` que cuente con la opción `-c`. 

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -c 'gene_id ".*0\{6\}.*"' Drosophila_melanogaster.BDGP6.28.102.gtf
100
```

Ahora sí! `grep` entiende que con `0\{6\}` queremos decir `000000` y podemos saber que hay 100 líneas de este tipo. Sin embargo, como esto de andar escapando caracteres es un poco engorroso, podemos usar la opción `-E` (extended) para indicarle a `grep` que use ERE y omitir los `\`. 

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -cE 'gene_id ".*0{6}.*"' Drosophila_melanogaster.BDGP6.28.102.gtf
100
```

__Nota__: existe el comando `egrep` que incorpora ya la opción -E por defecto. Sin embargo, existe en la actualidad sólo por motivos históricos (otra vez, la retrocompatibilidad) y su uso está desaconsejado. Así que acostúmbrate a usar `grep -E`. 


**Hacer ejercicio 1**


#### 1.3. Alternancia
Imagina ahora que quisiéramos buscar líneas referentes a `gene_ids` que contuviesen **seis ceros o seis unos**. ¿Cómo lo haríamos? Tendríamos que usar el operador de alternancia `|` (OR) para capturar las dos alternativas (cuidado con introducir espacios antes y/o después de `|`). 

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -cE 'gene_id ".*(0{6}|1{6}).*"' Drosophila_melanogaster.BDGP6.28.102.gtf | wc -l
113
```


#### 1.4. Otras opciones
Existen algunas otras opciones interesantes a usar con grep. 

La primera es `-o`, que sacará sólo la parte coincidente de cada línea con el patrón especificado:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -E -o 'gene_id "\w+"' Drosophila_melanogaster.BDGP6.28.102.gtf | head -n5
gene_id "FBgn0267431"
gene_id "FBgn0267431"
gene_id "FBgn0267431"
gene_id "FBgn0267431"
gene_id "FBgn0267431"
```

La opción `-i` hará una búsqueda `case-insensitive`:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -i 'FBGN' Drosophila_melanogaster.BDGP6.28.102.gtf | head -n5
3R	FlyBase	gene	567076	2532932	.	+	.	gene_id "FBgn0267431"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding";
3R	FlyBase	transcript	567076	2532932	.	+	.	gene_id "FBgn0267431"; transcript_id "FBtr0392909"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding";
3R	FlyBase	exon	567076	567268	.	+	.	gene_id "FBgn0267431"; transcript_id "FBtr0392909"; exon_number "1"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding"; exon_id "FBtr0392909-E1";
3R	FlyBase	exon	835376	835491	.	+	.	gene_id "FBgn0267431"; transcript_id "FBtr0392909"; exon_number "2"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding"; exon_id "FBtr0392909-E2";
3R	FlyBase	CDS	835378	835491	.	+	0	gene_id "FBgn0267431"; transcript_id "FBtr0392909"; exon_number "2"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding"; protein_id "FBpp0352251";
```

Finalmente, `-B` y `-A`, mostrarán un número fijo de lineas antes (before) y después (after) de cada ocurrencia encontrada:

```
abenito@cpg3:~/sesion-iv$ grep -B 2 -A 2 --color "veces" voluntad.txt
indigna de llegar a tus oídos,
pues de ornamento y gracia va desnuda;
mas a las veces son mejor oídos
el puro ingenio y lengua casi muda,
testigos limpios de ánimo inocente,
--
porque de todo aquesto y cada cosa
estaba Nise ya tan lnformada,
que llorando el pastor, mil veces ella
se enterneció escuchando su querella.

--
tanto, que sin mudarse las oían,
y al son de las zampoñas escuchaban
dos pastores a veces que cantaban.
```


### 2. Procesamiento de textos con awk
Como ya sabes, una tarea que realizamos muy a menudo en bioinformática es el [análisis exploratorio de datos](https://es.wikipedia.org/wiki/An%C3%A1lisis_exploratorio_de_datos), esto es, vamos a sacar estadísticos, recuentos, etc. preliminares que nos van a ayudar a comprender los datos y a plantear la mejor forma de analizarlos. Esta tarea fundamental, si no se plantea bien, puede consumirnos muchísimo tiempo debido a tener que convertir los datos a otros formatos (p.ej. MS Excel), limpiarlos, etc. 

En muchos de estos casos, crear un script en Python para esto sería como _matar moscas a cañonazos_, ya que no nos hace falta emplear todo el poder que ofrece este lenguaje para realizar tareas que normalmente son sencillas y/o repetitivas. Sin embargo, las herramientas UNIX que hemos visto hasta ahora son un poco "tontas": hacen las tareas para las que fueron diseñadas muy eficientemente, pero no las saques de ahí. Estaría bien que pudiésemos añadir algo de lógica a nuestros pipelines de edición de textos para que pudieran realizar tareas más complejas, como por ejemplo ejecutar condiciones o cosas así. Para suplir esta carencia se creó precisamente `awk`.

`awk` es un lenguaje de programación muy pequeño orientado al procesamiento de textos. Es menos expresivo que Python pero completo, ya que permite realizar un montón de tareas con mucha facilidad. Esto de que sea menos expresivo significa que cuando la lógica de nuestros programas se vaya complicando, será un dolor hacerlo con `awk`. Este será el momento de evaluar la posibilidad de movernos a Python. En concreto, usaremos `awk` para este tipo de análisis rápidos que haremos en datos tabulares. 


#### 2.1 Funcionamiento básico de awk 
`awk` procesa los datos de entrada fila a fila. Cada fila estará compuesta a su vez de una o varias columnas o campos, que `awk` tratará de distinguir automáticamente. De esta manera, `awk` asignará un registro o fila a la variable `$0`, mientras que los campos de dicho registro los asignará a las variables `$1`, `$2`, `$3`, etc.

Los programas `awk` usarán una o varias de la siguiente estructura: `patrón { acción }` para operar, donde el `patrón` será una **expresión (ej.: $1>100) o una expresión regular**. Cuando este `patrón` se evalúe a cierto, o la expresión regular case con el texto en cuestión, `awk` ejecutará la `acción` correspondiente definida por nosotras. Podremos añadir varias de estas estructuras patrón-acción a nuestros mini-scripts separándolas con el símbolo de punto y coma `;`. 

Si omitimos el `patrón`, `awk` por defecto ejecutará la acción en todos las filas que le pasemos por la entrada. 
De manera similar, si omitimos la `acción` pero especificamos un `patrón`, `awk` imprimirá todos los campos o registros que casen con el patrón. 

Por ejemplo, podemos emular `cat` con la siguiente orden:

```
abenito@cpg3:~/sesion-iv/gtfs$ awk '{ print $0 }' Drosophila_melanogaster.BDGP6.28.102.gtf
<muchas cosas>
...
```

De manera análoga, también podemos emular el comportamiento de `cut`. Vamos a convertir un fichero .gtf a un .bed mínimo (sólo 3 campos), algo que también podríamos hacer con cut:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf | cut -f 1,4,5 > Drosophila_melanogaster.BDGP6.28.102.bed
abenito@cpg3:~/sesion-iv/gtfs$ head Drosophila_melanogaster.BDGP6.28.102.bed -n5
3R	567076	2532932
3R	567076	2532932
3R	567076	567268
3R	835376	835491
3R	835378	835491
```

Usando grep+awk:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf | awk '{ print $1 " " $4 " " $5 }' | head -n5
3R 567076 2532932
3R 567076 2532932
3R 567076 567268
3R 835376 835491
3R 835378 835491
```

Si quisiéramos usar sólo `awk` para eliminar las líneas pertenecientes a la cabecera del .gtf, tendríamos que hacer algún tipo de comparación en la parte del `patrón`. `awk` aceptará los siguientes tipos de comparaciones:


|   Comparación |   Descripción                           |
|---------------|-----------------------------------------|
|   a == b      |   a es igual a b                        |
|   a != b      |   a no es igual a b                     |
|   a < b       |   a es menor que b                      |
|   a > b       |   a es mayor que b                      |
|   a <= b      |   a es menor o igual a b                |
|   a >= b      |   a es mayor o igual a b                |
|   a ~ b       |   a casa con la expresión regular b     |
|   a !~ b      |   a no casa con la expresión regular b  |
|   a && b      |   operador lógico AND                   |
|   a \|\| b    |   operador lógico OR                    |
|   !a          |   negación lógica                       |


Por lo tanto, una expresión equivalente en awk diría: _en las líneas que no empiezan por el caracter `#`, imprime el primer, cuarto y quinto campos separados por espacios:

```
abenito@cpg3:~/sesion-iv/gtfs$ awk '$0 !~ "^#" { print $1 " " $4 " " $5 }' Drosophila_melanogaster.BDGP6.28.102.gtf | head -n5
3R 567076 2532932
3R 567076 2532932
3R 567076 567268
3R 835376 835491
3R 835378 835491
```


Imagina ahora que sólo estuviésemos interesadas en imprimir aquellas líneas que se refieresen a anotaciones genómicas de longitud 200 o más. Podríamos encadenar ambas partes de la expresión con el operador `AND (&&)`. Fíjate en que la expresión regular la escribiremos entre barras inclinadas `/` y en que podemos imprimir también el resultado de operaciones en la correspondiente acción. 

```
abenito@cpg3:~/sesion-iv/gtfs$ awk '$0 !~ /^#/ && $5 - $4 >= 200 { print $1 "\t" $4 "\t" $5 "\t" $5-$4}' Drosophila_melanogaster.BDGP6.28.102.gtf | head -n5
3R	567076	2532932	1965856
3R	567076	2532932	1965856
3R	1025040	1025614	574
3R	1025040	1025614	574
3R	1138730	1138972	242
```

Como ves, las posibilidades que ofrece `awk` para implementar este tipo de lógicas sencillas son casi ilimitadas.


En el anterior ejemplo, hemos sustituido los espacios en blanco por tabuladores para generar datos tabulares listos para ser consumidos por otros programas (recuerda que la tabulación es el formato preferido por las máquinas para procesar tablas). Sin embargo, `awk` por defecto va a trabajar con espacios en blanco. Para cambiar esta opción, emplearemos la opción `-F fs (--field-separator fs)`, por ejemplo: `awk -F"\t"`. 


#### 2.2 Uso avanzado
Hasta ahora hemos visto dos casos de uso para `awk`:

1. Filtrar datos combinando expresiones regulares y operadores aritmético-lógicos
2. Reformatear columnas usando aritmética básica.

Vamos a ver ahora dos casos de uso más avanzados que nos van a permitir refinar estos compartimientos usando dos patrones especiales: `BEGIN` y `END`. 

- El patrón `BEGIN` especifica qué hacer antes de que se lea la primera fila. Útil para inicializar variables. 
- El patrón `END` especifica qué hacer después que el procesamiento de la última fila haya terminado. Útil para imprimir estadísticos y otros resúmenes de los datos.

Continuando un poco con el anterior ejemplo, imagina que ahora queremos calcular la longitud media de las regiones anotadas en un GTF. Podríamos hacerlo con un bloque `BEGIN` y otro bloque `END`. 

- Con el bloque `BEGIN` inicializamos un acumulador en la variable `s`, donde iremos guardando la suma de todas las longitudes que vayamos calculando (esto último lo hacemos dentro de la acción siguiendo el procedimiento visto anteriormente). 
- Con el bloque `END` imprimiremos la división del acumulador `s` por el número de anotaciones encontradas en el fichero para calcular la media aritmética. Para ello, nos valeremos de la variable interna `NR`, que alberga el número del último registro procesado por `awk`. Como el bloque `END` se ejecuta después de haber procesado el último registro, esta variable albergará al final el número total de registros o líneas del fichero.

```
abenito@cpg3:~/sesion-iv/gtfs$ awk 'BEGIN{ s=0 }; $0 !~ /^#/ { s+=$5-$4 }; END{ print "media: " s/NR };' Drosophila_melanogaster.BDGP6.28.102.gtf
media: 1141.32
```

También podemos usar `NR` para seleccionar rangos de líneas con una condición. Por ejemplo, calcula la longitud media para las líneas `5000 a 30000`: 

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf | awk 'BEGIN{ s=0 }; NR >= 5000 && NR <= 30000 { s+=$5-$4 }; END{ print "media: " s/NR };'
media: 50.3017
```

#### 2.3 Conversión de formatos inteligente

Anteriormente, hemos usado una combinación de `grep` y `cut` para generar un fichero bed a partir de un gtf. A pesar de que la expresión que usamos extrae correctamente los campos del primer fichero para crear el segundo, el fichero bed generado no es correcto debido a que en el formato GTF, los índices comienzan en 1, mientras que el formato BED comienzan en 0:

- Especificación del formato GTF: [Ver](http://www.ensembl.org/info/website/upload/gff.html)
- Especificación del formato BED: [Ver](http://www.ensembl.org/info/website/upload/bed.html)

Antes no podíamos arreglar este error de conversión de manera sencilla, pero con `awk` es extremadamente fácil: Simplemente tenemos que restarle 1 al campo annotation_start (columna 4):

```
abenito@cpg3:~/sesion-iv/gtfs$ awk '$0 !~ "^#" { print $1 " " $4-1 " " $5 }' Drosophila_melanogaster.BDGP6.28.102.gtf | head -n5
3R 567075 2532932
3R 567075 2532932
3R 567075 567268
3R 835375 835491
3R 835377 835491
```

### 3. Edición de textos con sed
Para terminar con este bloque, vamos a presentar la última herramienta para procesamiento de streams de texto que veremos en este curso: `sed` (**s**tream **ed**itor). 

Esta herramienta nos va a permitir editar de manera rápida nuestros streams para adaptarlos a los formatos esperados por otras herramientas. Para ello, y como viene siendo habitual, `sed` va a leer los datos desde un fichero o desde su entrada estándar y va a permitirnos editar las líneas una por una de acuerdo a una expresión regular que le proporcionemos. 

Por ejemplo, considera el fichero `genes/cromosomas.txt`:

```
abenito@cpg3:~/sesion-iv/genes$ head cromosomas.txt -n5
chrom1	3214482	3216968
chrom1	3216025	3216968
chrom1	3216022	3216024
chrom1	3671349	3671498
chrom1	3214482	3216021
```

E imagina ahora que quisiéramos cambiar los valores de la primera columna, que denota el cromosoma (`chrom1`)  al formato `chr1`. Quizás te haya tocado hacer algo parecido en el pasado y lo resolviste abriendo el fichero en Excel y usando Editar->Buscar y reemplazar. Además de ser un proceso muy lento (descargar el fichero desde el servidor, abrirlo, editarlo, volverlo a subir...), podrías tener problemas si el fichero es muy grande (hacer esto con un fichero de texto de varios gigas es probable que cuelgue tu ordenador). 

Lo bueno que tiene `sed` es que sigue el estilo de las herramientas UNIX vistas hasta ahora, y no necesita abrir todo el fichero para realizar las sustituciones, por lo cual es muy eficiente en memoria:

```
abenito@cpg3:~/sesion-iv/genes$ sed 's/chrom/chr/' cromosomas.txt | head -n5
chr1	3214482	3216968
chr1	3216025	3216968
chr1	3216022	3216024
chr1	3671349	3671498
chr1	3214482	3216021
```

Date cuenta de que si el fichero contuviese otros cromosomas (chrom12, chrom13, chrom2, etc.) nuestra sustitución funcionaría igual. El patrón que hemos usado quiere decir: "sustituye, fila a fila, las ocurrencias de `chrom` por `chr`". En general, usaremos la sintaxis de sustitución de `sed`: `s/patron/reemplazo`. 

Por defecto, `sed` sólo va a reemplazar la primera ocurrencia del patrón (si hubiese más, las dejaría sin tratar). Podemos cambiar este comportamiento añadiendo `g`(global) a la expresión: 

- Búsqueda global: `s/patron/reemplazo/g`

Además, si queremos que la búsqueda del patrón no tenga en cuenta mayúsculas/minúsculas, emplearemos el flag `i`:

- Búsqueda case-insensitive: `s/patron/reemplazo/i`

Como ocurría con `grep`, `sed` emplea `BRE` por defecto para especificar las expresiones regulares. Si queremos usar `ERE`, emplearemos la opción `-E`. 

#### 3.1. Grupos de captura
Muy importante es la capacidad de `sed` para capturar trozos de texto que encajen con un patrón dado, y usar estos mismos trozos para realizar el reemplazo que queramos. 

Por ejemplo, puede ser que tengamos un fichero con regiones genómicas anotadas en el formato `chr1:3214482-3216968`, y queremos pasarlo a un formato tabular de tres columnas. ¿Cómo lo podríamos hacer? Usando las herramientas vistas hasta ahora sería complicado, pero podemos implementarlo fácilmente con `sed`:

```
abenito@cpg3:~/sesion-iv$ echo "chr1:3214482-3216968" | sed -E 's/^(chr[^:]+):([0-9]+)-([0-9]+)/\1\t\2\t\3/'
chr1	3214482	3216968
```

¿Qué ha ocurrido? En el ejemplo, empleamos tres grupos de captura (lo que ponemos entre paréntesis) que luego referenciamos en la segunda parte de la expresión de reemplazo (lo que va después del `/`).

- `^(chr[^:]+)`: texto al comienzo de la línea (carácter `^`) que empieza por `chr`, y va seguido de uno o más (`+`) caracteres que no (`^`) son `:` (nuestro delimitador). Captura lo que vaya entre paréntesis en el grupo 1. 
-  seguido del delimitador `:` (hasta aquí tendríamos el nombre del cromosoma).
-  ([0-9]+) seguido de una secuencia de 1 o más (`+`) dígitos del 0 al 9 (`[0-9]`). Captura lo que vaya entre paréntesis al grupo 2. Este es el comienzo de la región.
-  seguido del caracter `-`.
-  ([0-9]+) seguido de una secuencia de 1 o más (`+`) dígitos del 0 al 9 (`[0-9]`). Captura lo que vaya entre paréntesis al grupo 3. Este es el final de la región.

En la sustitución, simplemente referenciamos los tres grupos capturados con `\1`, `\2`, etc. y los separamos con tabuladores `\t` para obtener nuestro resultado final.

**Ejercicio**: Existen, sin embargo, otras maneras (quizás más sencillas pero menos ilustrativas del poder de `sed`) en las que podríamos haber obtenido el mismo resultado final. ¿Se te ocurren algunas? Recuerda que puedes usar el flag `g`, o puedes encadenar distintas llamadas a `sed` con tuberías si ves que meterlo todo en una única expresión regular se te antoja complicado. 

#### 3.2. tr
Una versión simplificada de sed es el comando `tr` (**tr**anslate o **tr**ansliterate). `tr` va a operar a nivel de caracter, y no a nivel de cadenas o líneas como `sed`. Su funcionamiento es simple: sólo convierte entre dos grupos de caracteres definidos por el usuario. 

Por ejemplo, podemos decirle que cambie todas las ocurrencias de `:` y `-` por un tabulador `\t`, resolviendo el anterior ejemplo:

```
abenito@cpg3:~/sesion-iv$ echo "chr1:3214482-3216968" | tr ':-' '\t'
chr1	3214482	3216968
```

Pero no puede hacer sustituciones a nivel de palabra: 

```
abenito@cpg3:~/sesion-iv/genes$ cat gene-1.bed | tr '1' 'chr1' | head -n5
c	6206c97	6206270	GENE00000025907
c	6223599	6223745	GENE00000025907
c	6227940	6228049	GENE00000025907
c	6229959	6230073	GENE00000025907
c	6230003	6230005	GENE00000025907
``` 

Consulta el manual con `man tr` para ver todas las opciones que admite este comando. Especialmente interesante es la opción `-s` (squeeze) que va a comprimir varias apariciones seguidas del mismo caracter en un único representante:

```
abenito@cpg3:~/sesion-iv$ echo -e "linea 1: esta es la línea 1\n\nlinea 2: esta es la línea 2\n\n\nlínea 3: esta es la línea 3"
linea 1: esta es la línea 1

linea 2: esta es la línea 2


línea 3: esta es la línea 3
```

```
abenito@cpg3:~/sesion-iv$ echo -e "linea 1: esta es la línea 1\n\nlinea 2: esta es la línea 2\n\n\nlínea 3: esta es la línea 3" | tr -s '\n'
linea 1: esta es la línea 1
linea 2: esta es la línea 2
línea 3: esta es la línea 3
```

#### 3.3. Controlando la salida
Por defecto, `sed` va a imprimir todas y cada una de las líneas que le pasemos por la entrada estándar. Pero a veces no es lo que queremos. Imagina que queremos capturar todos los nombres de las transcripciones que aparecen en un gtf (campo `transcript_id`). Podríamos implementar un pipeline así:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf  | sed -E 's/.*transcript_id "([^"]+)".*/\1/' | head -n5
3R	FlyBase	gene	567076	2532932	.	+	.	gene_id "FBgn0267431"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding";
FBtr0392909
FBtr0392909
FBtr0392909
FBtr0392909
```

Desafortunadamente (y esto es bastante habitual), la composición de cada fila en este tipo de ficheros no es muy homogénea y grep imprime toda la línea cuando no encuentra el patrón que estamos buscando (como es el caso de la primera línea, que no contiene el campo `transcript_id`). Podríamos solucionar este problema añadiendo un segundo `grep` que filtrase por `transcript_id` pero sed ofrece una manera más clara de conseguir esto mismo con la opción `-n`, que hace que `grep` **no** imprima todas las líneas:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf  | sed -nE 's/.*transcript_id "([^"]+)".*/\1/'
```

Si ejecutas la línea anterior, verás que `sed` no imprime nada. Tenemos que añadir una `p` al final de la expresión para hacer que imprima todas aquellas líneas en las que haya hecho sustituciones:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf  | sed -E -n 's/.*transcript_id "([^"]+)".*/\1/p' | head -n5
FBtr0392909
FBtr0392909
FBtr0392909
FBtr0392909
FBtr0392909
```

Fíjate ahora en qué hace la expresión regular `/.*transcript_id "([^"]+)".*/`

1. Primero, captura cero o más caracteres (`.*`).
2. Luego, encuentra el literal `transcript_id`, seguido de un espacio y comillas (` "`).
3. Luego, encuentra una o más ocurrencias de cualquier caracter que no sea comillas (`"`) y captúralo en el primer grupo. (`[^"]+`)
4. Finalmente, encuentra cero o más caracteres de nuevo (`.*`).

Podríamos haber estado tentados de sustituir la captura del `transcript_id` por algo así como: `"(.*)"`. Es decir, encuentra las comillas, luego casa con 0 o más caracteres hasta que encuentres otra vez las comillas. Pero si lo ejecutamos...

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf  | sed -E -n 's/.*transcript_id "(.*)".*/\1/p' | head -n5
FBtr0392909"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding
FBtr0392909"; exon_number "1"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding"; exon_id "FBtr0392909-E1
FBtr0392909"; exon_number "2"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding"; exon_id "FBtr0392909-E2
FBtr0392909"; exon_number "2"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding"; protein_id "FBpp0352251
FBtr0392909"; exon_number "2"; gene_name "Myo81F"; gene_source "FlyBase"; gene_biotype "protein_coding"; transcript_source "FlyBase"; transcript_biotype "protein_coding
```

...no funciona como esperamos. ¿Por qué? Bueno, resulta que la búsqueda por defecto que hace `sed` es avariciosa ("greedy"). Esto quiere decir que `sed` sabe qué caracteres "quedan" por delante y siempre va a tratar de casar con todo el texto restante que pueda. Si te fijas bien, en el ejemplo anterior `sed` coge hasta la última ocurrencia de las comillas (que vendrá con el valor del último campo). Por eso, con la regex original, "obligamos" a `sed` a casar **sólo con todo lo que no sea `"`**. De esta manera, `sed` parará de intentar incluir más caracteres en el grupo cuando vea otra vez `"`. 

#### 3.4. Reemplazo de sólo algunas líneas
Finalmente, la sintaxis  de `sed` nos permite operar sólo con un rango de líneas que definamos. Por ejemplo, si quisiéramos sacar la distribución de ids de transcripción en el rango 1000-2000 podríamos hacer:

```
abenito@cpg3:~/sesion-iv/gtfs$ grep -v "^#" Drosophila_melanogaster.BDGP6.28.102.gtf | sed -E -n '1000,2000s/.*transcript_id "([^"]+)".*/\1/p' | sort | uniq -c | sort -nr | head -n5
     27 FBtr0078907
     25 FBtr0336483
     25 FBtr0100617
     23 FBtr0336484
     21 FBtr0309257
```


### 4. Trabajando con datos comprimidos
En informática, comprimir es el proceso de condensar los datos de tal manera que ocupen menos espacio en disco. Dependiendo de los algoritmos que se empleen, esta compresión puede ser **con pérdida** o **sin pérdida**. En la compresión con pérdida, se sacrifica la fidelidad de los datos originales para favorecer un mayor ahorro de espacio en disco (como ocurre por ejemplo con los ficheros mp3), lo cual hace imposible la reconstrucción de los mismos a posteriori. En la compresión sin pérdida, sin embargo, la información original es exactamente restaurada al término de la descompresión. 

En bioinformática, vamos a emplear algoritmos de compresión sin pérdida debido a que manejaremos principalmente ficheros de texto plano, en ocasiones muy grandes. Por esta razón, la mayoría de pipelines que escribamos deberían permitir el uso de datos comprimidos de manera nativa que no obliguen a la usuaria a tener que descomprimir los datos antes de usarlos. Para ello, muchas de las herramientas UNIX que hemos visto cuentan con variantes diseñadas para aceptar datos comprimidos (como `cat-zcat` o `grep-zgrep`). Y si no existe dicha variante, siempre puedes enlazar la salida de `zcat` con la herramienta a través de una tubería. 

#### 4.1. gzip, bzip2

Los dos algoritmos/sistemas de compresión más famosos en el mundo UNIX son `gzip` y `bzip2`. Algunas diferencias:

- `gzip` es más rápido que `bzip2`. 
- `bzip2` tiene un ratio más alto de compresión (comprime más los datos) que `gzip`. 
- De manera orientativa, `gzip` se emplea en bioinformática para comprimir la mayoría de los datos con los que se trabaje "a diario", mientras que `bzip2` se usa más para comprimir datos que están destinados a ser almacenados por largos periodos de tiempo. 

`gzip` puede comprimir cualquier cosa que le pasemos por la entrada estándar. Muy útil para almacenar los resultados de nuestros pipelines:

```
comando_1 datos | comando_2 | comando_3 | ... | comando_n | gzip > resultados.gz
```

...o también comprimirá ficheros de nuestro disco duro, cuando así se lo indiquemos. Por defecto, gzip reemplazará el fichero a comprimir por un nuevo fichero cuyo nombre resultará de añadir la extensión .gz al nombre del fichero original:

```
abenito@cpg3:~/sesion-iv/gtfs$ gzip Drosophila_melanogaster.BDGP6.28.102.gtf
abenito@cpg3:~/sesion-iv/gtfs$ ls -a Drosophila_melanogaster.BDGP6.28.102.gtf*
Drosophila_melanogaster.BDGP6.28.102.gtf.gz
```

Para descomprimir, podremos usar el comando `gunzip`, que revertiría los resultados del proceso anterior:

```
abenito@cpg3:~/sesion-iv/gtfs$ gunzip Drosophila_melanogaster.BDGP6.28.102.gtf.gz
abenito@cpg3:~/sesion-iv/gtfs$ ls -a Drosophila_melanogaster.BDGP6.28.102.gtf*
Drosophila_melanogaster.BDGP6.28.102.gtf
```

También podemos usar la opción `-c` con ambos para que vuelquen sus resultados a la salida estándar:

```
abenito@cpg3:~/sesion-iv/gtfs$ gunzip -c Homo_sapiens.GRCh38.102.gtf.gz | head -n5
#!genome-build GRCh38.p13
#!genome-version GRCh38
#!genome-date 2013-12
#!genome-build-accession NCBI:GCA_000001405.28
#!genebuild-last-updated 2020-09
```

o comprimir múltiples ficheros a la vez (usando por ejemplo comodines el comodín *: `gzip *.gtf`. 

#### 4.2. tar.gz
Cuando queramos comprimir directorios enteros, es mucho mejor utilizar el formato `tar.gz`. En realidad, este tipo de ficheros codifica el resultado de la herramienta `tar`, que es luego comprimido con `gunzip`. 

La herramienta `tar` (**ta**pe **ar**chive) fue creada originalmente para almacenar datos en dispositivos en serie de entrada/salida (cintas o tapes en inglés). Lo bueno de esta herramienta es que no sólo crea un fichero con los contenidos de una carpeta o carpetas, sino que además es capaz de almacenar diferentes parámetros del sistema de ficheros como el nombre, tiempos de modificación y acceso, permisos, etc. El fichero .tar resultante es habitualmente comprimido para ahorrar espacio extra. Esta combinación es tan famosa que el comando `tar` ya incorpora una opción (`-z`) para comprimir directamente el tar a gunzip. 

Por ejemplo, para comprimir los contenidos de la carpeta actual en un fichero .tar.gz haremos `tar <ruta_fichero_targz> <directorio a comprimir>`:

```
abenito@cpg3:~/sesion-iv$ tar -cvzf ../cuarta_sesion.tar.gz .
```
 
Las opciones que le estamos pasando son:
- c: create. Queremos crear un fichero tar nuevo.
- v: verbose. Queremos que tar nos vaya informando de todos los ficheros/directorios que incluye en el tar.
- z: gunzip. Queremos que tar comprima el tar una vez creado. 
- f: file. Le especificaremos dónde ha de almacenar el fichero que cree. 

Y ahora, para descomprimir, puedes hacerlo con:

 **Mucho cuidado** porque `tar` va a sobreescribir cualquier fichero/directorio que ya exista. Si tienes dudas y no quieres cargarte algo, la opción `-k` hará que tar te pregunta si va a sobreescribir algo. 

```
abenito@cpg3:~/sesion-iv$ tar -zxvf cuarta_sesion.tar.gz
./
./genes
./esta_sesion.tar.gz
./fasta
./voluntad.txt
./gtfs
```


### Órdenes de la shell vistas en esta sesión:

`egrep`: Atajo para invocar `grep` con la opción `-E` (no recomendado)

`awk`: Herramienta/lenguaje para escanear y procesar textos.

`sed`: edición de streams de texto.

`tr`: traduce o elimina caracteres

`gzip`: Comprime datos (rápido y eficiente).

`gunzip`: Descomprime ficheros `gzip`.

`bzip2`: Comprime datos (lento y eficaz).

`zcat`: como `cat` pero acepta ficheros comprimidos con `gzip`.

`zgrep`: como `grep` pero acepta ficheros comprimidos con `gzip`.

`tar`: archiva carpetas y ficheros.
