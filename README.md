# GUIA JPA

El API de Persistence Java (JPA) es una especificación independiente de proveedor  para el mapeo de objetos Java a las tablas de bases de datos relacionales. Implementaciones de esta especificación permite a los desarrolladores de aplicaciones abstraer del producto de base de datos específica con la que están trabajando y les permiten implementar operaciones CRUD (crear, leer, actualizar y eliminar) las operaciones de tal manera que el mismo código funciona en diferentes productos de base de datos. Estos marcos no sólo manejan el código que interactúa con la base de datos (el código JDBC), sino también  mapear los tipos de estructuras de datos utilizadas por la aplicación.

Los 3 componentes de JPA son:

*  Entidades(Entities): En las versiones actuales las entidades JPA son POJO's. Las versiones anteriores de JPA se obligados a extender como subclase  de las clases proporcionadas por JPA, pero este enfoque hacía más difíciles realizar pruebas debido a dichas dependecies, las nuevas versiones de JPA no requieren que las entidades sean subclase de alguna clase de Framework.

* Metadatos objeto-relacional: El desarrollador de la aplicación debe proporcionar la asignación entre las clases Java y sus atributos a las tablas y columnas de la base de datos. Esto se puede hacer cualquiera de los archivos de configuración mínimas dedicados o en la versión más reciente también por anotaciones.

* Java Persistence Query Language (JPQL): Como JPA tiene como objetivo abstracto a partir del producto de base de datos específica, el framework también proporciona un langauge consultas dedicado que se puede utilizar en lugar de SQL. Esta traducción de JPQL a SQL permite que las implementaciones del framework de soporte a diferentes dialectos de bases de datos y permite que el desarrollador ejecutar consultas en una base de datos de forma independiente asu vendor.

En este tutorial vamos a través de diferentes aspectos del framework y desarrollaremos una sencilla aplicación Java SE que almacena y recupera datos desde una base de datos relacional. Usaremos las siguientes bibliotecas/entornos:

* maven >= 3.0 como tool de Build
* JPA 2.0 contenida en en Java Enterprise Edition (JEE) 6.0
* Framework Hibernate como una implementacion de JPA (4.3.8.Final)
* H2 como base relacional version 1.3.176


###Project setup

Como primer paso vamos a crear un proyecto simple maven desde linea de comandos:


>mvn archetype:create -DgroupId=com.javacodegeeks.ultimate -DartifactId=jpa

```
01|-- src
02|   |-- main
03|   |   `-- java
04|   |       `-- com
05|   |           `-- javacodegeeks
06|   |                `-- ultimate
07|   `-- test
08|   |   `-- java
09|   |       `-- com
10|   |           `-- javacodegeeks
11|   |                `-- ultimate
12`-- pom.xml
```

The libraries our implementation depends on are added to the dependencies section of the pom.xml file in the following way:
```xml
<properties>
    <jee.version>7.0</jee.version>
    <h2.version>1.3.176</h2.version>
    <hibernate.version>4.3.8.Final</hibernate.version>
</properties>
<dependencies>
    <dependency>
        <groupId>javax</groupId>
        <artifactId>javaee-api</artifactId>
        <version>${jee.version}</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <version>${h2.version}</version>
    </dependency>
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-entitymanager</artifactId>
        <version>${hibernate.version}</version>
    </dependency>
</dependencies>
```

Para tener una mejor visión de conjunto de las versiones separadas, definimos cada versión como una propiedad Maven y referencia más adelante en la sección de dependencias.

3.1. EntityManager and Persistence Unit

Ahora empezamos a implementar nuestra primera funcionalidad JPA. Vamos a empezar con una clase simple que proporciona un método run() que se invoca en el método principal de la aplicación:

```java
public class Main {
    private static final Logger LOGGER = Logger.getLogger("JPA");
    public static void main(String[] args) {
        Main main = new Main();
        main.run();
    }


    public void run() {
        EntityManagerFactory factory = null;
        EntityManager entityManager = null;
        try {
            factory = Persistence.createEntityManagerFactory("PersistenceUnit");
            entityManager = factory.createEntityManager();
            persistPerson(entityManager);
        } catch (Exception e) {
            LOGGER.log(Level.SEVERE, e.getMessage(), e);
            e.printStackTrace();
        } finally {
            if (entityManager != null) {
                entityManager.close();
            }
            if (factory != null) {
                factory.close();
            }
        }
    }
    ...

```

###1. Casi toda la interacción con JPA se hace a través del EntityManager. Para obtener una instancia de un EntityManager, tenemos que crear una instancia de la EntityManagerFactory. Normalmente sólo necesitamos una EntityManagerFactory por  "unidad de persistencia" por aplicación. Una unidad de persistencia es un conjunto de clases de la JPA que se gestiona junto con la configuración de base de datos en un archivo llamado persistence.xml


```xml
<persistence xmlns="http://java.sun.com/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://java.sun.com/xml/ns/persistence http://java.sun.com/xml/ns/persistence/persistence_1_0.xsd" version="1.0">
<persistence-unit name="PersistenceUnit" transaction-type="RESOURCE_LOCAL">
        <provider>org.hibernate.ejb.HibernatePersistence</provider>
        <properties>
            <property name="connection.driver_class" value="org.h2.Driver"/>
            <property name="hibernate.connection.url" value="jdbc:h2:~/jpa"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
            <property name="hibernate.hbm2ddl.auto" value="create"/>
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
        </properties>
    </persistence-unit>
    </persistence>
```

Este archivo se crea en la carpeta src/main/resource/META-IN del proyecto Maven. Como se puede ver, definimos una unidad de persistencia  con el nombre *PersistenceUnit* que tiene el tipo de transacción RESOURCE_LOCAL. El tipo de transacción determina cómo las transacciones se manejan en la aplicación.

En nuestra aplicación de ejemplo no tenemos contenedor JEE por lo que tenemos que manejar las transacciones nosotros mismos, de ahí que se especifique  **RESOURCE_LOCAL**. Cuando se utiliza un contenedor JEE entonces el contenedor es responsable de la creación de la EntityManagerFactory y sólo le proporciona que EntityManager. El contenedor también se encarga del comienzo y final de cada transacción. En ese caso se proporcionará el valor **JTA**.
  

###2. En persistence.xml 

Se informa al proveedor de JPA sobre la base de datos que queremos utilizar. Esto se hace mediante la especificación del controlador JDBC que Hibernate debe utilizar. Como queremos usar la base de datos [H2](www.h2database.com), la propiedad **connection.driver_class** se establece en el valor org.h2.Driver.
 
  + Tenemos que decirle a Hibernate el dialecto JDBC que debe utilizar. Como Hibernate proporciona una implementación de dialecto dedicado para H2, elegimos éste con la propiedad **hibernate.dialect**. Con este dialecto de Hibernate es capaz de crear las sentencias SQL apropiados para la base de datos de H2.


Por último, pero no menos importante ofrecemos tres opciones que vienen muy útil en el desarrollo de una nueva aplicación, pero que no sería utilizado en entornos de producción. El primero de ellos es la propiedad **hibernate.hbm2ddl.auto** que le dice a Hibernate como crear todas las tablas a partir de cero desde el inicio. Si ya existe la tabla, se eliminará. En nuestra aplicación de ejemplo esta es una buena característica que podemos confiar en el hecho de que la base de datos está vacía en la a principios y que todos los cambios que hemos hecho en el esquema desde nuestra último inicio de la aplicación se reflejan en el esquema.

La segunda opción es **hibernate.show_sql** que se le dice a Hibernate para que imprima cada declaración SQL que se emite a la base de datos en la línea de comandos. Con esta opción habilitada podemos rastrear fácilmente todas las declaraciones y echar un vistazo si todo funciona como se esperaba. Y finalmente le decimos a Hibernate que imprima de una manera agradable la salida SQL para una mejor legibilidad estableciendo la  propiedad hibernate.format_sql en true.


 ###3. Regresando al la tecla...
 
Después de haber obtenido una instancia de la **EntityManagerFactory** y de ella una instancia de EntityManager podemos utilizarlos en el método **persistPerson** para salvar algunos datos en la base de datos. Ten en cuenta que después de lo que hemos hecho nuestro trabajo tenemos que cerrar tanto el EntityManager así como la EntityManagerFactory.
   + 4.1) Transacciones

El EntityManager representa una unidad de persistencia y por lo tanto vamos a necesitar en la aplicacion **RESOURCE_LOCAL** sólo una instancia del EntityManager. Una unidad de persistencia es una memoria caché para las entidades que representan partes del estado almacenados en la base de datos, así como una conexión a la base de datos. Con el fin de almacenar datos en la base de datos, por lo tanto tenemos que pasarlo al EntityManager y con ello a la caché subyacente. En caso de que quiera crear una nueva fila en la base de datos, esto se hace invocando el método persist () en el EntityManager como se demuestra en el siguiente código:

```java
 private void persistPerson(EntityManager entityManager) {
 	EntityTransaction transaction = entityManager.getTransaction();
	try {
		transaction.begin();
		Person person = new Person();
		person.setFirstName("Homer");
		person.setLastName("Simpson");
		entityManager.persist(person);
		transaction.commit();
	} catch (Exception e) {
		if (transaction.isActive()) {
			transaction.rollback();
		}
	}
 }
 
 ```
 
 
 Pero antes de que podamos llamar a **persist()** tenemos que abrir una nueva transacción llamando **transaction.begin()** en un nuevo objeto de transacciones que hemos recuperado del EntityManager. Si omitimos este llamado, Hibernate podría lanzar una **IllegalStateException** que nos dice que nos hemos olvidado de ejecutar el persisten() dentro de una transacción:

Después de llamar a persistir () tenemos que confirmar (*commit*) la transacción, es decir, enviar los datos a la base de datos y almacenarla allí. En caso de que sea lanzada una excepción dentro del bloque try, tenemos que deshacer (*Rollback*) la transacción hemos comenzado antes. Pero como sólo podemos deshacer transacciones activas, tenemos que comprobar antes si la transacción actual ya está en marcha, ya que puede ocurrir que la excepción se produce dentro de la convocatoria **transaction.begin ()**.

###5. Tables

La clase Person es mapeada para a la tabla T_PERSON agregando la anotacion @Entity:

```java
@Entity
@Table(name = "T_PERSON")
public class Person {
	private Long id;
	private String firstName;
	private String lastName;
	@Id
	@GeneratedValue
	public Long getId() {
		return id;
	}
	public void setId(Long id) {
		this.id = id;
	}
	@Column(name = "FIRST_NAME")
	public String getFirstName() {
		return firstName;
	}
	public void setFirstName(String firstName) {
		this.firstName = firstName;
	}
	@Column(name = "LAST_NAME")
	public String getLastName() {
		return lastName;
	}
	public void setLastName(String lastName) {
		this.lastName = lastName;
	}
	}
```
 	
Por otro lado se puede especificar más información para cada columna usando los otros atributos que la anotación @Column ofrece:

```java
@Column(name = "FIRST_NAME", length = 100, nullable = false, unique = false)
```


Intentar insertar nulo en "FIRST_NAME" en esta tabla provocaría una violación de constraint en la base de datos y hacer que la transacción actual haga un rollback.

Las dos anotaciones @Id y @GeneratedValue dicen a JPA que este valor es la clave principal de esta tabla y que debe ser generado de forma automática


En el código de ejemplo anterior, hemos añadido las anotaciones JPA a los métodos getter para cada campo que se debe asignar a una columna de base de datos. Otra forma sería anotando el campo directamente en lugar de su método getter.


```java
@Entity
@Table(name = "T_PERSON")
public class Person {
    @Id
    @GeneratedValue
    private Long id;
    @Column(name = "FIRST_NAME")
    private String firstName;
    @Column(name = "LAST_NAME")
    private String lastName;
    ...
```

Las dos formas son más o menos iguales, la única diferencia que tienen juega un papel cuando se desea anular anotaciones para los campos en subclases. Como veremos en el curso ulterior de este tutorial, es posible extender una entidad existente con el fin de heredar sus campos. Cuando ponemos las anotaciones JPA sobre el terreno, no podemos ignorar que lo que podamos reemplazando el método getter correspondiente.

Uno también tiene que prestar atención para guardar el camino para anotar entidades del mismo para jerarquía de una entidad. se puede mezclar la anotación de los campos y métodos dentro de un proyecto JPA, pero dentro de una entidad y todas sus subclases se debe ser consistente. Si tiene que cambiar la forma de anotación dentro de una jerarquía subclase, puede utilizar el acceso de anotaciones JPA para especificar que una determinada subclase utiliza de una manera diferente para anotar campos y métodos:
    
```java
@Entity
@Table(name = "T_GEEK")
@Access(AccessType.PROPERTY)

public class Geek extends Person {
...
```


El fragmento de código anterior le dice a JPA que esta clase va a utilizar las anotaciones en el nivel de método, mientras que la superclase puede tener anotaciones a nivel campo.


```SQL
Hibernate: drop table T_PERSON if exists

Hibernate: create table T_PERSON (id bigint generated by default as identity, FIRST_NAME varchar(255), LAST_NAME varchar(255), primary key (id))

Hibernate: insert into T_PERSON (id, FIRST_NAME, LAST_NAME) values (null, ?, ?)

```

Como podemos ver, Hibernate *Dropea* la tabla T_PERSON  si existe y vuelve a crearla después. Se crea la tabla con dos columnas de tipo varchar (255) (FIRST_NAME, LAST_NAME) y una columna llamada Identificación de tipo *big int*. La última columna se define como clave principal y sus valores son generadas automáticamente por la base de datos cuando insertamos un nuevo valor.

Podemos comprobar que todo es correcto con el Shell que se incluye con H2. Para utilizar esta Shell sólo tenemos la h2-1.3.176.jar archivo jar:

>java -cp h2-1.3.176.jar org.h2.tools.Shell -url jdbc:h2:~/jpa

```sql

sql> select * from T_PERSON;

ID | FIRST_NAME | LAST_NAME
1  | Homer      | Simpson

(4 rows, 4 ms)

```

El resultado del query anterior muestra que la tabla T_PERSON realmente contiene un registro con id 1 y con valores  en first name y lastname


###4. Herencia

Después de haber llevado a cabo la configuracio0n en este caso de uso fácil, nos vamos ahora a considerar casos de uso más complejos. 

Supongamos que queremos almacenar junto a personas también información sobre los aficiones-geek y de su lenguaje de programación favorito. Como los *geeks* también son personas, nos modelamos esto en el mundo Java como relación subclase de persona:


```java
@Entity
@Table(name = "T_GEEK")
public class Geek extends Person {

	private String favouriteProgrammingLanguage;
	
	private List<Project> projects = new ArrayList<Project>();
	
	@Column(name = "FAV_PROG_LANG")
	
	public String getFavouriteProgrammingLanguage() {
	
			return favouriteProgrammingLanguage;
			
	}
	
	public void setFavourit frrzfdxeProgrammingLanguage(String favouriteProgrammingLanguage) {
	
		this.favouriteProgrammingLanguage = favouriteProgrammingLanguage;
		
	}
	...
	}
```

Agregando las anotaciones @Entity y @Table a la clase le deja a Hibernate crear la nueva tabla T_GEEK:

>Hibernate: create table T_PERSON (DTYPE varchar(31) not null, id bigint generated by default as identity, FIRST_NAME varchar(255), LAST_NAME varchar(255), FAV_PROG_LANG varchar(255), primary key (id))

Podemos ver que Hibernate crea una tabla para ambas entidades y pone la información si hemos almacenado una persona o un *Geek* Dentro de un nuevo nombre de la columna DTYPE. Vamos a persistir algunos geeks en nuestra base de datos (para facilitarla lectura he omitido el bloque que cacha cualquier excepción y deshace la transacción):

```java	
private void persistGeek(EntityManager entityManager) {

  EntityTransaction transaction = entityManager.getTransaction();
  transaction.begin();
  Geek geek = new Geek();
  geek.setFirstName("Gavin");
  geek.setLastName("Coffee");
  geek.setFavouriteProgrammingLanguage("Java");
  entityManager.persist(geek);
  geek = new Geek();
  geek.setFirstName("Thomas");
  geek.setLastName("Micro");
  geek.setFavouriteProgrammingLanguage("C#");
  entityManager.persist(geek);
  geek = new Geek();
  geek.setFirstName("Christian");
  geek.setLastName("Cup");
  geek.setFavouriteProgrammingLanguage("Java");
  entityManager.persist(geek);
  transaction.commit();

}
```

Después de haber ejecutado este método, el la tabla T_PERSON contiene los siguientes filas (junto con la persona que ya hemos insertado):

```
sql> select * from t_person;
```

|DTYPE  | ID | FIRST_NAME | LAST_NAME | FAV_PROG_LANG|
|---|---|---|---|---|
|Person | 1  | Homer      | Simpson   | null|
|Geek   | 2  | Gavin      | Coffee    | Java|
|Geek   | 3  | Thomas     | Micro     | C#|
|Geek   | 4  | Christian  | Cup       | Java|


Como era de esperar la nueva columna DTYPE determina qué tipo de persona que tenemos. La columna FAV_PROG_LANG tiene el valor null para las personas que son no geeks.

Si no te gusta el nombre o tipo de la columna de discriminador, se puede cambiar con la anotación correspondiente. A continuación queremos que la columna tiene el nombre PERSON_TYPE y es una columna entera en lugar de una columna de serie:

```java
DiscriminatorColumn (Name = "PERSON_TYPE", discriminatorType = DiscriminatorType.INTEGER)

```

Esto produce para el siguiente resultado:


>sql> select * from t_person;


|PERSON_TYPE | ID | FIRST_NAME | LAST_NAME | FAV_PROG_LANG|
|---|---|---|---|---|
|-1907849355 | 1  | Homer      | Simpson   | null|
|2215460     | 2  | Gavin      | Coffee    | Java|
|2215460     | 3  | Thomas     | Micro     | C#|
|2215460     | 4  | Christian  | Cup       | Java|


No en todas las situaciones que desea tener una tabla para todos los tipos diferentes que desea almacenar en su base de datos. Este es especialmente el caso cuando los diferentes tipos no tienen casi todas las columnas en común. Por lo tanto JPA permite especificar cómo diseñar las diferentes columnas. Estas tres opciones disponibles:

* SINGLE_TABLE: Esta estrategia mapas de todas las clases a una sola tabla. Como consecuencia de que cada fila tiene todas las columnas para todos los tipos de la base de datos de necesidades de almacenamiento adicional para las columnas vacías. Por otro lado esta estrategia trae la ventaja de que una consulta nunca tiene que utilizar una combinación y por lo tanto puede ser mucho más rápido.

* JOINED: Esta estrategia crea para cada tipo de una tabla separada. Cada tabla por lo tanto sólo contiene el estado de la entidad asignada. Para cargar una sola entidad, el proveedor JPA tiene que cargar los datos de una entidad de todas las mesas de la entidad está asignado. Este enfoque reduce el espacio de almacenamiento, pero por otro lado introduce unirse a las consultas que pueden disminuir la velocidad de consulta de manera significativa.

* TABLE_PER_CLASS: Al igual que la estrategia ACUMULADOS, esta estrategia crea una tabla separada para cada tipo de entidad. Pero en contraste con la estrategia theJOINED estas tablas contienen toda la información necesaria para cargar esta entidad. Por lo tanto no se unen a las consultas son necesarias para la carga de las entidades pero introduce en situaciones donde la subclase concreta no se conoce quries SQL adicionales con el fin de determinar él.
Para cambiar nuestra aplicación para utilizar la estrategia ACUMULADOS, todo lo que tenemos que hacer es añadir la siguiente anotación a la clase base:

>Inheritance (Estrategia = InheritanceType.JOINED)

Ahora Hibernate crea dos tablas para las personas y los geeks:

>Hibernate: create table T_GEEK (FAV_PROG_LANG varchar(255), id bigint not null, primary key (id))

>Hibernate: create table T_PERSON (id bigint generated by default as identity, FIRST_NAME varchar(255), LAST_NAME varchar(255), primary key (id))


Después de haber agregado la persona y los geeks que obtenemos el siguiente resultado:

1

sql> select * from t_person;

02

ID | FIRST_NAME | LAST_NAME


 

03

1  | Homer      | Simpson

04

2  | Gavin      | Coffee


 

05

3  | Thomas     | Micro

06

4  | Christian  | Cup


 

07

(4 rows, 12 ms)

08

sql> select * from t_geek;


 

09

FAV_PROG_LANG | ID

10

Java          | 2


 

11

C#            | 3

12

Java          | 4


 

13

(3 rows, 7 ms)

As expected the data is distributed over the two tables. The base table T_PERSON contains all the common attributes whereas the tableT_GEEK only contains rows for each geek. Each row references a person by the value of the column ID.

When we issue a query for persons, the following SQL is send to the database:

01

select

02

    person0_.id as id1_2_,


 

03

    person0_.FIRST_NAME as FIRST_NA2_2_,

04

    person0_.LAST_NAME as LAST_NAM3_2_,


 

05

    person0_1_.FAV_PROG_LANG as FAV_PROG1_1_,

06

    case


 

07

        when person0_1_.id is not null then 1

08

        when person0_.id is not null then 0


 

09

    end as clazz_

10

from


 

11

    T_PERSON person0_

12

left outer join


 

13

    T_GEEK person0_1_

14

        on person0_.id=person0_1_.id

We see that a join query is necessary to include the data from the table T_GEEK and that Hibernate encodes the information if one row is a geek or by returning an integer (see case statement).

The Java code to issue such a query looks like the following:

01

TypedQuery<Person> query = entityManager.createQuery("from Person", Person.class);

02

List<Person> resultList = query.getResultList();


 

03

for (Person person : resultList) {

04

    StringBuilder sb = new StringBuilder();


 

05

    sb.append(person.getFirstName()).append(" ").append(person.getLastName());

06

    if (person instanceof Geek) {


 

07

        Geek geek = (Geek)person;

08

        sb.append(" ").append(geek.getFavouriteProgrammingLanguage());


 

09

    }

10

    LOGGER.info(sb.toString());


 

11

}

First of all we create a Query object by calling EntityManager's createQuery() method. The query clause can omit the select keyword. The second parameter helps to parameterize the method such that the Query is of type Person. Issuing the query is simply done by calling query.getResultList(). The returned list is iterable, hence we can just iterate over the Person objects. If we want to know whether we have a Person or a Geek, we can just use Java's instanceof operator.

Running the above code leads to the following output:

1

Homer Simpson

2

Gavin Coffee Java


 

3

Thomas Micro C#

4

Christian Cup Java

5. Relationships

Until now we have not modelled any relations between different entities except the extends relation between a subclass and its superclass. JPA offers different relations between entities/tables that can be modelled:

OneToOne: In this relationship each entity has exactly one reference to the other entity and vice versa.
OneToMany / ManyToOne: In this relationship one entity can have multiple child entities and each child entity belongs to one parent entity.
ManyToMany: In this relationship multiple entites of one type can have multiple references to entities from the other type.
Embedded: In this relationship the other entity is stored in the same table as the parent entity (i.e. we have two entites for one table).
ElementCollection: This relationship is similar to the OneToMany relation but in contrast to it the referenced entity is anEmbedded entity. This allows to define OneToMany relationships to simple objects that are stored in contrast to the "normal"Embedded relationship in another table.
5.1. OneToOne

Let's start with an OneToOne relationship by adding a new entity IdCard:

01

@Entity

02

@Table(name = "T_ID_CARD")


 

03

public class IdCard {

04

    private Long id;


 

05

    private String idNumber;

06

    private Date issueDate;


 

07

  

08

    @Id


 

09

    @GeneratedValue

10

    public Long getId() {


 

11

        return id;

12

    }


 

13

  

14

    public void setId(Long id) {


 

15

        this.id = id;

16

    }


 

17

  

18

    @Column(name = "ID_NUMBER")


 

19

    public String getIdNumber() {

20

        return idNumber;


 

21

    }

22

  


 

23

    public void setIdNumber(String idNumber) {

24

        this.idNumber = idNumber;


 

25

    }

26

  


 

27

    @Column(name = "ISSUE_DATE")

28

    @Temporal(TemporalType.TIMESTAMP)


 

29

    public Date getIssueDate() {

30

        return issueDate;


 

31

    }

32

  


 

33

    public void setIssueDate(Date issueDate) {

34

        this.issueDate = issueDate;


 

35

    }

36

}

Note that we have used a common java.util.Date to model the issue date of the ID card. We can use the annotation @Temporal to tell JPA how we want the Date to be serialized to the database. Depending on the underlying database product this column is mapped to an appropriate date/timestamp type. Possible values for this annotation are next to TIMESTAMP: TIME and DATE.

We tell JPA that each person has exactly one ID card:

01

@Entity

02

@Table(name = "T_PERSON")


 

03

public class Person {

04

    ...


 

05

    private IdCard idCard;

06

    ...


 

07

  

08

    @OneToOne


 

09

    @JoinColumn(name = "ID_CARD_ID")

10

    public IdCard getIdCard() {


 

11

        return idCard;

12

    }

The column in the table T_PERSON that contains the foreign key to the table T_ID_CARD is stored in the additional column ID_CARD_ID. Now Hibernate generates these two tables in the following way:

01

create table T_ID_CARD (

02

    id bigint generated by default as identity,


 

03

    ID_NUMBER varchar(255),

04

    ISSUE_DATE timestamp,


 

05

    primary key (id)

06

)


 

07

  

08

create table T_PERSON (


 

09

    id bigint generated by default as identity,

10

    FIRST_NAME varchar(255),


 

11

    LAST_NAME varchar(255),

12

    ID_CARD_ID bigint,


 

13

    primary key (id)

14

)

An important fact is that we can configure when the ID card entity should be loaded. Therefore we can add the attribute fetch to the@OneToOne annotation:

1

@OneToOne(fetch = FetchType.EAGER)

The value FetchType.EAGER is the default value and specifies that each time we load a person we also want to load the ID card. On the other hand we can specify that we only want to load the ID when we actually access it by calling person.getIdCard():

1

@OneToOne(fetch = FetchType.LAZY)

This result in the following SQL statements when loading all persons:

01

Hibernate:

02

    select


 

03

        person0_.id as id1_3_,

04

        person0_.FIRST_NAME as FIRST_NA2_3_,


 

05

        person0_.ID_CARD_ID as ID_CARD_4_3_,

06

        person0_.LAST_NAME as LAST_NAM3_3_,


 

07

        person0_1_.FAV_PROG_LANG as FAV_PROG1_1_,

08

        case


 

09

            when person0_1_.id is not null then 1

10

            when person0_.id is not null then 0


 

11

        end as clazz_

12

    from


 

13

        T_PERSON person0_

14

    left outer join


 

15

        T_GEEK person0_1_

16

            on person0_.id=person0_1_.id


 

17

Hibernate:

18

    select


 

19

        idcard0_.id as id1_2_0_,

20

        idcard0_.ID_NUMBER as ID_NUMBE2_2_0_,


 

21

        idcard0_.ISSUE_DATE as ISSUE_DA3_2_0_

22

    from


 

23

        T_ID_CARD idcard0_

24

    where


 

25

        idcard0_.id=?

We can see that we now have to load each ID card separately. Therefore this feature has to be used wisely, as it can cause hundreds of additional select queries in case you are loading a huge number of persons and you know that you are loading each time also the ID card.

5.2. OneToMany

Another important relationship is the @OneToMany relationship. In our example every Person should have one ore more phones:

01

@Entity

02

@Table(name = "T_PHONE")


 

03

public class Phone {

04

    private Long id;


 

05

    private String number;

06

    private Person person;


 

07

  

08

    @Id


 

09

    @GeneratedValue

10

    public Long getId() {


 

11

        return id;

12

    }


 

13

  

14

    public void setId(Long id) {


 

15

        this.id = id;

16

    }


 

17

  

18

    @Column(name = "NUMBER")


 

19

    public String getNumber() {

20

        return number;


 

21

    }

22

  


 

23

    public void setNumber(String number) {

24

        this.number = number;


 

25

    }

26

  


 

27

    @ManyToOne(fetch = FetchType.LAZY)

28

    @JoinColumn(name = "PERSON_ID")


 

29

    public Person getPerson() {

30

        return person;


 

31

    }

32

  


 

33

    public void setPerson(Person person) {

34

        this.person = person;


 

35

    }

36

}

Each phone has an internal id as well as the number. Next to this we also have to specify the relation to the Person with the@ManyToOne, as we have "many" phones for "one" person. The annotation @JoinColumn specifies the column in the T_PHONE table that stores the foreign key to the person.

On the other side of the relation we have to add a List of Phone objects to the person and annotate the corresponding getter method with @OneToMany as we have "one" person with "many" phones:

1

private List<Phone> phones = new ArrayList<>();

2

...


 

3

@OneToMany(mappedBy = "person", fetch = FetchType.LAZY)

4

public List<Phone> getPhones() {


 

5

    return phones;

6

}

The value of the attribute mappedBy tells JPA which list on the other side of the relation (here Phone.person) this annotation references.

As we do not want to load all phones every time we load the person, we set the relationship to be fetched lazy (although this is the default value and we would have to set it explicitly). Now we get an additional selecte statment each time we load the phones for one person:

01

select

02

    phones0_.PERSON_ID as PERSON_I3_3_0_,


 

03

    phones0_.id as id1_4_0_,

04

    phones0_.id as id1_4_1_,


 

05

    phones0_.NUMBER as NUMBER2_4_1_,

06

    phones0_.PERSON_ID as PERSON_I3_4_1_


 

07

from

08

    T_PHONE phones0_


 

09

where

10

    phones0_.PERSON_ID=?

As the value for the attribute fetch is set at compile time, we unfortunately cannot change it at runtime. But if we know that we want to load all phone numbers in this use case and in other use cases not, we can leave the relation to be loaded lazy and add the clauseleft join fetch to our JPQL query in order to tell the JPA provider to also load all phones in this specific query even if the relation is set to FetchType.LAZY. Such a query can look like the following one:

1

TypedQuery<Person> query = entityManager.createQuery("from Person p left join fetch p.phones", Person.class);

We give the Person the alias p and tell JPA to also fetch all instances of phones that belong to each person. This results with Hibernate in the following select query:

01

select

02

    person0_.id as id1_3_0_,


 

03

    phones1_.id as id1_4_1_,

04

    person0_.FIRST_NAME as FIRST_NA2_3_0_,


 

05

    person0_.ID_CARD_ID as ID_CARD_4_3_0_,

06

    person0_.LAST_NAME as LAST_NAM3_3_0_,


 

07

    person0_1_.FAV_PROG_LANG as FAV_PROG1_1_0_,

08

    case


 

09

        when person0_1_.id is not null then 1

10

        when person0_.id is not null then 0


 

11

    end as clazz_0_,

12

    phones1_.NUMBER as NUMBER2_4_1_,


 

13

    phones1_.PERSON_ID as PERSON_I3_4_1_,

14

    phones1_.PERSON_ID as PERSON_I3_3_0__,


 

15

    phones1_.id as id1_4_0__

16

from


 

17

    T_PERSON person0_

18

left outer join


 

19

    T_GEEK person0_1_

20

        on person0_.id=person0_1_.id


 

21

left outer join

22

    T_PHONE phones1_


 

23

        on person0_.id=phones1_.PERSON_ID

Please note that without the keyword left (i.e. only join fetch) Hibernate will create an inner join and only load persons that actually have at least one phone number.

5.3. ManyToMany

Another interesting relationship is the @ManyToMany one. As one geek can join many projects and one project consists of many geeks, we model the relationship between Project and Geek as @ManyToMany relationship:

01

@Entity

02

@Table(name = "T_PROJECT")


 

03

public class Project {

04

    private Long id;


 

05

    private String title;

06

    private List<Geek> geeks = new ArrayList<Geek>();


 

07

  

08

    @Id


 

09

    @GeneratedValue

10

    public Long getId() {


 

11

        return id;

12

    }


 

13

  

14

    public void setId(Long id) {


 

15

        this.id = id;

16

    }


 

17

  

18

    @Column(name = "TITLE")


 

19

    public String getTitle() {

20

        return title;


 

21

    }

22

  


 

23

    public void setTitle(String title) {

24

        this.title = title;


 

25

    }

26

  


 

27

    @ManyToMany(mappedBy="projects")

28

    public List<Geek> getGeeks() {


 

29

        return geeks;

30

    }


 

31

  

32

    public void setGeeks(List<Geek> geeks) {


 

33

        this.geeks = geeks;

34

    }


 

35

}

Our project has an internal id, a title of type String and a list of Geeks. The getter method for the geeks attribute is annotated with@ManyToMany(mappedBy="projects"). The value of the attribute mappedBy tells JPA the class member on the other side of the relation this relation belongs to as there can be more than one list of projects for a geek. The class Geek gets a list of projects and :

01

private List<Project> projects = new ArrayList<>();

02

...


 

03

@ManyToMany

04

@JoinTable(


 

05

        name="T_GEEK_PROJECT",

06

        joinColumns={@JoinColumn(name="GEEK_ID", referencedColumnName="ID")},


 

07

        inverseJoinColumns={@JoinColumn(name="PROJECT_ID", referencedColumnName="ID")})

08

public List<Project> getProjects() {


 

09

    return projects;

10

}

For a @ManyToMany we need an additional table. This table is configured by the @JoinTable annotation that describes the table used to store the geek's assignments to the different projects. It has the name GEEK_PROJECT and stores the geek's id in the columnGEEK_ID and the project's id in the column PROJECT_ID. The referenced column is on both sides just ID as we have named the internal id in both classes ID.

A @ManyToMany is also per default fetched lazy, as in most cases we do not want to load all projects assignments when we load a single geek.

As the relation @ManyToMany is equal on both sides, we could have also annotated the two lists in both classes the other way round:

1

@ManyToMany

2

@JoinTable(


 

3

        name="T_GEEK_PROJECT",

4

        joinColumns={@JoinColumn(name="PROJECT_ID", referencedColumnName="ID")},


 

5

        inverseJoinColumns={@JoinColumn(name="GEEK_ID", referencedColumnName="ID")})

6

public List<Geek> getGeeks() {


 

7

    return geeks;

8

}

And on the other Geek side:

1

@ManyToMany(mappedBy="geeks")

2

public List<Project> getProjects() {


 

3

    return projects;

4

}

In both cases Hibernate creates a new table T_GEEK_PROJECT with the two colums PROJECT_ID and GEEK_ID:

1

sql> select * from t_geek_project;

2

PROJECT_ID | GEEK_ID


 

3

1          | 2

4

1          | 4


 

5

(2 rows, 2 ms)

The Java code to persist these relations is the following one:

01

List<Geek> resultList = entityManager.createQuery("from Geek g where g.favouriteProgrammingLanguage = :fpl", Geek.class).setParameter("fpl", "Java").getResultList();

02

EntityTransaction transaction = entityManager.getTransaction();


 

03

transaction.begin();

04

Project project = new Project();


 

05

project.setTitle("Java Project");

06

for (Geek geek : resultList) {


 

07

    project.getGeeks().add(geek);

08

    geek.getProjects().add(project);


 

09

}

10

entityManager.persist(project);


 

11

transaction.commit();

In this example we only want to add geeks to our "Java Project" whose favourite programming language is of course Java. Hence we add a where clause to our select query that restricts the result set to geeks with a specific value for the column FAV_PROG_LANG. As this column is mapped to the field favouriteProgrammingLanguage, we can reference it directly by its Java field name in the JPQL statement. The dynamic value for the query is passed into the statement by calling setParameter() for the corresponding variable in the JPQL query (here: fpl).

5.4. Embedded / ElementCollection

It can happen that you want to structure your Java model more fine-grained than your database model. An example for such a use case is the Java class Period that models the time between a start and an end date. This construct can be reused in different entities as you do not want to copy the two class fields startDate and endDate to each entity that has a period of time.

For such cases JPA offers the ability to model embedded entities. These entities are modeled as separate Java classes with the annotation @Embeddable:

01

@Embeddable

02

public class Period {


 

03

    private Date startDate;

04

    private Date endDate;


 

05

  

06

    @Column(name ="START_DATE")


 

07

    public Date getStartDate() {

08

        return startDate;


 

09

    }

10

  


 

11

    public void setStartDate(Date startDate) {

12

        this.startDate = startDate;


 

13

    }

14

  


 

15

    @Column(name ="END_DATE")

16

    public Date getEndDate() {


 

17

        return endDate;

18

    }


 

19

  

20

    public void setEndDate(Date endDate) {


 

21

        this.endDate = endDate;

22

    }


 

23

}

This entity can then be included in our Project entity:

01

private Period projectPeriod;

02

  


 

03

@Embedded

04

public Period getProjectPeriod() {


 

05

    return projectPeriod;

06

}


 

07

  

08

public void setProjectPeriod(Period projectPeriod) {


 

09

    this.projectPeriod = projectPeriod;

10

}

As this entity is embedded, Hibernate creates the two columns START_DATE and END_DATE for the table T_PROJECT:

1

create table T_PROJECT (

2

    id bigint generated by default as identity,


 

3

    END_DATE timestamp,

4

    START_DATE timestamp,


 

5

    projectType varchar(255),

6

    TITLE varchar(255),


 

7

    primary key (id)

8

)

Altough these two values are modelled within a separate Java class, we can query them as part of the project:

1

sql> select * from t_project;

2

ID | END_DATE                | START_DATE              | PROJECTTYPE       | TITLE


 

3

1  | 2015-02-01 00:00:00.000 | 2016-01-31 23:59:59.999 | TIME_AND_MATERIAL | Java Project

4

(1 row, 2 ms)

A JPQL query has to reference the embedded period in order to formulate a condition that restricts the result set to projects that started on a certain day:

1

entityManager.createQuery("from Project p where p.projectPeriod.startDate = :startDate", Project.class).setParameter("startDate", createDate(1, 1, 2015));

This yields to the following SQL query:

01

select

02

    project0_.id as id1_5_,


 

03

    project0_.END_DATE as END_DATE2_5_,

04

    project0_.START_DATE as START_DA3_5_,


 

05

    project0_.projectType as projectT4_5_,

06

    project0_.TITLE as TITLE5_5_


 

07

from

08

    T_PROJECT project0_


 

09

where

10

    project0_.START_DATE=?

Since version 2.0 of JPA you can even use @Embeddable entities in one-to-many relations. This is accomplished by using the new annotations @ElementCollection and @CollectionTable as shown in the following example for the class Project:

01

private List<Period> billingPeriods = new ArrayList<Period>();

02

  


 

03

@ElementCollection

04

@CollectionTable(


 

05

        name="T_BILLING_PERIOD",

06

        joinColumns=@JoinColumn(name="PROJECT_ID")


 

07

)

08

public List<Period> getBillingPeriods() {


 

09

    return billingPeriods;

10

}


 

11

  

12

public void setBillingPeriods(List<Period> billingPeriods) {


 

13

    this.billingPeriods = billingPeriods;

14

}

As Period is an @Embeddable entity we cannot just use a normal @OneToMany relation.
