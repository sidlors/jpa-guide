# jpa-guide
guia de JPA 


1)  Nearly all interaction with the JPA is done through the EntityManager. To obtain an instance of an EntityManager, we have to create an instance of the EntityManagerFactory. Normally we only need one EntityManagerFactory for one “persistence unit” per application. A persistence unit is a set of JPA classes that is managed together with the database configuration in a file called persistence.xml:

<?xml version="1.0" encoding="UTF-8" ?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://java.sun.com/xml/ns/persistence
 http://java.sun.com/xml/ns/persistence/persistence_1_0.xsd" version="1.0">

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


This file is created in the src/main/resource/META-INF folder of the maven project. As you can see, we define one persistence-unit with the name PersistenceUnit that has the transaction-type RESOURCE_LOCAL. The transaction-type determines how transactions are handled in the application.

In our sample application we want to handle them on our own, hence we specify here RESOURCE_LOCAL. When you use a JEE container then the container is responsible for setting up the EntityManagerFactory and only provides you he EntityManager. The container then also handles the begin and end of each transaction. In this case you would provide the value JTA.


2) In persistence.xml is to inform the JPA provider about the database we want to use. This is done by specifying the JDBC driver that Hibernate should use. As we want to use the H2 database (www.h2database.com), the property connection.driver_class is set to the value org.h2.Driver.

3) We have to tell Hibernate is the JDBC dialect that it should use. As Hibernate provides a dedicated dialect implementation for H2, we choose this one with the property hibernate.dialect. With this dialect Hibernate is capable of creating the appropriate SQL statements for the H2 database.

Last but not least we provide three options that come handy when developing a new application, but that would not be used in production environments. The first of them is the property hibernate.hbm2ddl.auto that tells Hibernate to create all tables from scratch at startup. If the table already exists, it will be deleted. In our sample application this is a good feature as we can rely on the fact that the database is empty at the beginnig and that all changes we have made to the schema since our last start of the application are reflected in the schema.

The second option is hibernate.show_sql that tells Hibernate to print every SQL statement that it issues to the database on the command line. With this option enabled we can easily trace all statements and have a look if everything works as expected. And finally we tell Hibernate to pretty print the SQL for better readability by setting the property hibernate.format_sql to true.


4) Returning to code...


After having obtained an instance of the EntityManagerFactory and from it an instance of EntityManager we can use them in the method persistPerson to save some data in the database. Please note that after we have done our work we have to close both the EntityManager as well as the EntityManagerFactory.



4.1) Transactions*

The EntityManager represents a persistence unit and therefore we will need in RESOURCE_LOCAL applications only one instance of the EntityManager. A persistence unit is a cache for the entities that represent parts of the state stored in the database as well as a connection to the database. In order to store data in the database we therefore have to pass it to the EntityManager and therewith to the underlying cache. In case you want to create a new row in the database, this is done by invoking the method persist() on the EntityManager as demonstrated in the following code:

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

But before we can call persist() we have to open a new transaction by calling transaction.begin() on a new transaction object we have retrieved from the EntityManager. If we would omit this call, Hibernate would throw a IllegalStateException that tells us that we have forgotten to run the persist() within a transaction:

After calling persist() we have to commit the transaction, i.e. send the data to the database and store it there. In case an exception is thrown within the try block, we have to rollback the transaction we have begun before. But as we can only rollback active transactions, we have to check before if the current transaction is already running, as it may happen that the exception is thrown within the transaction.begin() call.


4.2) Tables

The class Person is mapped to the database table T_PERSON by adding the annotation @Entity:

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

 	
 	
On the other hand you can specify more information for each column by using the other attributes that the @Column annotation provides:

@Column(name = "FIRST_NAME", length = 100, nullable = false, unique = false)

Trying to insert null as first name into this table would provoke a constraint violation on the database and cause the current transaction to roll back.

The two annotations @Id and @GeneratedValue tell JPA that this value is the primary key for this table and that it should be generated automatically. 	


In the example code above we have added the JPA annotations to the getter methods for each field that should be mapped to a database column. Another way would be to annotate the field directly instead of the getter method:


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
    
The two ways are more or less equal, the only difference they have plays a role when you want to override annotations for fields in subclasses. As we will see in the further course of this tutorial, it is possible to extend an existing entity in order to inherit its fields. When we place the JPA annotations at the field level, we cannot override them as we can by overriding the corresponding getter method.

One also has to pay attention to keep the way to annotate entities the same for one entity hierarchy. You can mix annotation of fields and methods within one JPA project, but within one entity and all its subclasses is must be consistent. If you need to change the way of annotatio within a subclass hierarchy, you can use the JPA annotation Access to specify that a certain subclass uses a different way to annotate fields and methods:


@Entity

@Table(name = "T_GEEK")

@Access(AccessType.PROPERTY)

public class Geek extends Person {

...


The code snippet above tells JPA that this class is going to use annotations on the method level, whereas the superclass may have used annotations on field level.
