La capacidad de mapear las relaciones entre entidades está  ampliamente soportada por muchas herramientas de O/RM. Sin embargo, por  alguna razón, muchos desarrolladores se encuentran con problemas cuando  intentan mapear una relación muchos-a-muchos entre entidades. Aunque ya  se ha escrito mucho sobre los aspectos tecnológicos de esto, pensé en  tomar una perspectiva más arquitectónica / DDD sobre esto aquí.

![img](./header-image.png)

## **Los objetos de valor no cuentan**

Aunque el ejemplo canónico presentado es Cliente -> Dirección Postal, y tiene un buen tratamiento [aquí](http://devlicio.us/blogs/billy_mccafferty/archive/2008/07/11/when-to-use-many-to-one-s-vs-many-to-many-with-nhibernate.aspx) para nHibernate, no es arquitectónicamente representativo.

Las direcciones postales son Value Objects. Esto significa que si  tenemos dos instancias de la clase Address, y ambas tienen los mismos  datos de negocio, son semánticamente equivalentes. Los clientes, en  cambio, no son Value Objects, son entidades. Si tenemos dos clientes con los mismos datos de negocio (ambos llamados Bob Smith), eso no  significa que sean semánticamente equivalentes – no son la misma  persona.

## **Todas las entidades**

Por lo tanto, para nuestros propósitos aquí usaremos algo diferente.  Digamos que tenemos una entidad llamada Job que es algo que una empresa  quiere contratar. Tiene un título, una descripción, un nivel de  habilidad y un montón de otros datos. Digamos que también tenemos otra  entidad llamada JobBoard que es donde la empresa publica los Job para  que los solicitantes puedan verlos, como moster.com. Una JobBoard tiene  un nombre, una descripción, un sitio web, una cuota de referencia y un  montón de otros datos.

Un Job puede publicarse en varias JobBoard. Y una JobBoard puede  tener varias ofertas publicadas. Una relación regular de muchos a  muchos. En este punto, ni siquiera vamos a complicar la asociación.

Simplemente se representa en la BD con una tabla de asociación que  contiene dos columnas para cada uno de los ids de las tablas de  entidades.

En el modelo de dominio, los desarrolladores también pueden  representar esto con la clase Job que contiene una lista de instancias  de JobBoard, y la clase JobBoard que contiene una lista de Job.

Es intuitivo. Sencillo. Fácil de implementar. Y erróneo.

Para hacer elecciones inteligentes de DDD, vamos a tomar primero lo  que puede parecer un curso tangencial, pero te aseguro que tus raíces  agregadas dependen de ello.

## **Avanzando con nuestro ejemplo**

Digamos que el usuario elige un Job, y luego marca las JobBoard donde quiere que se publique el Job, y hace clic en enviar.

Para simplificar, en este punto, vamos a ignorar la comunicación con  los sitios de empleo reales, asumiendo que si podemos conseguir la  asociación en la DB, la magia sucederá más tarde haciendo que el trabajo aparezca en todos los sitios.

Nuestro bienintencionado desarrollador toma el ID del Job, y todos  los IDs de las JobBoard, abre una transacción, obtiene el objeto del  Job, obtiene los objetos de las JobBoard, añade todos los objetos de las JobBoards al Job, y lo confirma, como sigue:

```
public void PostJobToBoards(Guid jobId, params Guid[] boardIds)
{
    using (ISession s = this.SessionFactory.OpenSession())
    using (ITransaction tx = s.BeginTransaction())
    {
        var job = s.Get<Job>(jobId);
        var boards = new List<JobBoard>();
        foreach(Guid id in boardIds)
            boards.Add(s.Get<JobBoard>(id));
        job.PostTo(boards);
        tx.Commit();
    }
}
```

En este código, Jobo es nuestro Aggregate Root. Puedes ver que este  es el caso ya que Job es el punto de entrada que el código de la capa de servicio utiliza para interactuar con el modelo de dominio. Pronto  veremos por qué esto es incorrecto.

***\*** Fíjate que en este código de la capa de servicio, nuestro bienintencionado desarrollador está siguiendo la regla de que  aunque puedes obtener tantos objetos como quieras, sólo se te permite  una llamada a un método en un objeto de dominio. El código llamado en la línea 12 es lo que más o menos se espera:

```
public void PostTo(IList<JobBoard> boards)
{
    foreach(JobBoard jb in boards)
    {
        this.JobBoards.Add(jb);
        jb.Jobs.Add(this);
    }
}
```

Sólo que mientras estábamos comprometidos, alguien borró una de las  JobBoard justo en ese momento. O que alguien haya actualizado la  JobBoard causando un conflicto de concurrencia. O cualquier cosa que  causara que una sola asociación no se creara.

Eso haría que toda la transacción fallara y que todos los cambios se revirtieran.

Con razón, piensa nuestro bienintencionado desarrollador.

Pero los usuarios no piensan como los desarrolladores bien intencionados.

## **Fallos parciales**

Si yo fuera al supermercado con la lista que me ha dado mi mujer,  encontrara que se han acabado las avellanas (el último artículo de la  lista), NO compraría el resto de la compra y volvería a casa con las  manos vacías, ¿qué crees que pasaría?

¡Exacto! Así es como los usuarios nos ven a los desarrolladores.  Antes de salir corriendo a escribir un montón de código, tenemos que  entender la semántica de negocio de las acciones de los usuarios,  incluyendo la pregunta sobre los fallos parciales.

La lista no es una unidad de trabajo que necesita tener éxito o  retroceder atómicamente. En realidad son muchas unidades de trabajo. Es  decir, no querría que mi mujer me enviara a la tienda 10 veces para  comprar 10 artículos, así que la lista es realmente una especie de atajo del usuario. Por lo tanto, en el escenario de la bolsa de trabajo, cada conexión entre la bolsa de trabajo es su propia transacción.

Esto es más común de lo que se cree.

Una vez que busques casos en los que el dominio es indulgente con los fallos parciales, puedes empezar a ver más y más de ellos.

## **Aggreagte Roots**

En la transacción original en la que intentamos conectar muchas  JobBoards a un solo Job, vimos que el sólo Job es el Aggregate Root. Sin embargo, una vez que tenemos múltiples transacciones, cada una  conectando un Job y un JobBoard, el JobBoard es tan Aggregate Root como  el Job.

Podemos hacer `jobBoard.Post(job);` o `job.PostTo(jobBoard);`

Pero necesitamos un poco más de análisis para llegar a la decisión correcta.

Aunque podríamos dejar la dependencia bidireccional/circular entre  ellos, sería preferible que la hiciéramos unidireccional. Para ello,  tenemos que entender su relación:

Si no existiera el término «Job», ¿tendría sentido la «JobBoard»? Probablemente no.

Si no existiera «JobBoard», ¿tendría sentido «Job»? Probablemente.  Sí. Nuestra empresa puede gestionar el proceso de contratación de un  puesto de trabajo independientemente de que el candidato haya entrado a  través de Monster.com o no.

A partir de esto entendemos que:

1. La relación unidireccional puede ser modelada como uno-a-muchos de la bolsa de trabajo al empleo. 
2. La clase Job ya no tendría una colección de objetos Job Board. De  hecho, podría incluso estar en un ensamblaje separado de Job Board y no  hacer referencia a Job Board de ninguna manera. 
3. Job Board seguiría teniendo una colección de objetos Job.

Volviendo al código anterior vemos que la opción correcta es `jobBoard.Post(job);`

JobBoard es lel Aggregate Root en este caso. Además, el mapeo  muchos-a-muchos se ha disuelto, dejando atrás un único mapeo  uno-a-muchos.

Deja que eso se hunda un segundo.

## **Pero espere…**

Mientras que la interfaz gráfica de usuario que muestra qué puestos  de trabajo se publican en una determinada bolsa de trabajo está bien  servida por la decisión anterior (simplemente atravesando el gráfico de  objetos desde la bolsa de trabajo a su colección de puestos de trabajo), esa no es toda la historia. Otra GUI necesita mostrar a los usuarios  administrativos en qué Bolsas de Trabajo se ha publicado un determinado  Trabajo. Como ya no tenemos la conexión a nivel de dominio, no podemos  recorrer miTrabajo.TablasDeTrabajo.

Nuestra única opción es realizar una consulta. Eso no es tan malo, pero no es tan bonito como el recorrido de objetos.

El beneficio real está en cortar el nudo gordiano de mapeo M-a-N y obtener un modelo de dominio más limpio y bien factorizado.

Eso nos da una mayor ventaja para una mayor descomposición a nivel de sistema.

Ahora estamos preparados para pasar a una solución pub/sub entre  estas raíces agregadas, convirtiéndolas efectivamente en Bounded  Contexts. A partir de ahí, podemos pasar a un almacenamiento en caché a  escala de Internet con REST para obtener una escalabilidad adicional al  mostrar una bolsa de trabajo con todas sus ofertas.

## **Para terminar**

A menudo vemos las relaciones muchos-a-muchos como cualquier otra  relación. Y desde una perspectiva puramente técnica, no nos equivocamos. Sin embargo, la realidad empresarial en torno a estas relaciones es a  menudo muy diferente: perdona los fallos parciales, hasta el punto de  exigirlos.

Dado que la gente de negocios que nos proporciona los requisitos rara vez piensa en escenarios de fallo, no especifican que «entre estas dos  entidades de aquí, no quiero atomicidad transaccional» (rodando nuestros ojos técnicos – los idiotas [sarcasmo, sólo para asegurarse de que no  me malinterpreten]).

Sin embargo, si explicamos lo que hará el sistema en condiciones de  fallo cuando sea atómico transaccionalmente, esos mismos empresarios nos pondrán los ojos en blanco.

Lo que he encontrado sorprende a algunos practicantes de DDD es lo  crítico que es este tema para llegar a los aggregate Roots correctos y a los Bounded Contexts.

También es simple, y práctico, así que no ofenderás a la policía de YAGNI.

Traducción por interés personal de [DDD & Many to Many Object Relational Mapping](https://udidahan.com/2009/01/24/ddd-many-to-many-object-relational-mapping/)