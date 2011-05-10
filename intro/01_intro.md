!SLIDE smbullets
# Squeryl
## A Scala ORM and DSL for talking with Databases with minimum verbosity and maximum type safety

!SLIDE small
# Defining Persistent Classes

	@@@ Scala
	  import org.squeryl.PrimitiveTypeMode._

	  class Author(val id: Long, 
				   val firstName: String, 
				   val lastName: String,
				   val email: Option[String])
				   
	  class Book(val id: Long, 
				 var title: String,
				 @Column("AUTHOR_ID")
				 var authorId: Long,
				 var coAuthorId: Option[Long])

!SLIDE small
# Defining a Schema
	@@@ Scala	  
	  object Library extends Schema {
	  
		val authors = table[Author]
		
		val books = table[Book]
		
		val borrowals = table[Borrowal]	
		
		val sciFiBooks = 
		  books.where(_.category === Categories.SciFi)
	  
		on(authors)(s => declare(
			s.email      is(unique,indexed("idxEmail")),
			s.firstName  is(indexed),
			columns(s.firstName, s.lastName) are(indexed)
		))

!SLIDE small
# Selecting
	@@@ Scala	  
	
	import Library._ 
	
	val forDummies = 
	  from(books)(b => 
	    where(b.title like "% for dummies") 
		select(b)
	  )
	
	// Queries are composable : 
	
	val recentForDummies = 
	  from(forDummies)(b => 
	    where(b.yearPublished gte 2010) 
		select(b)
	  )
	
	val recentForDummiesCount = recentForDummies.Count

	
!SLIDE small
# Selecting
## shorter form
	@@@ Scala	  
	
	import Library._ 
	
	val forDummies = 
	  books.where(_.title like "% for dummies")
	
	
!SLIDE small
# Joins
## Reuse by composability, 

	@@@Scala	  
		
	val dummies = 
	  from(forDummies, borrowals, borrowers)((b, br, p) => 
	    where(b.id === br.bookId and br.borrowerId === p.id) 
		select(p)
	  ).distinct


!SLIDE small
# Subqueries	  
	@@@Scala	  

	val titlesOfBooksBorrowedByBob =
	  from(books)(book =>
		where(book.id in (
		  from(borrowals)(b =>
		    where(b.borrowerId === 1234) 
		    select(b.bookId)
		  )
    	)
		select(book.title)
	  )

## It would *not* compile if book.id and borrowal.bookId where not of compatible types

!SLIDE small
# Partial Updates
	@@@Scala	  
	update(songs)(s =>
	  where(s.title === "Watermelon Man")
	  set(s.title := "The Watermelon Man",
	      s.year  := s.year plus 1)
	)   
	
	
!SLIDE small
# Agregate Queries
## Type inference at work : the compiler _"knows"_ that the result is a scalar

	@@@Scala
	val avgHeight: Option[Float] = 
	  from(people)(p => 
	    compute(avg(p.heightInCentimeters)))	

## the sql is :

	@@@SQL
	select avg(people1.height) 
	from people people1
  
!SLIDE small
# More type inference
## Why is the result a Option[BigDecimal] ?
    
	@@@Scala
	class LotteryGame(
	  val probabilityOfWining: Float, 
	  val cost: Float, 
	  val prize: BigDecimal)
	  
	val probableLossOrWorstGame: Option[BigDecimal] = 
	  from(games)(g => 
	    compute(min(
		  (g.probabilityOfWining times g.prize) minus 
		  ((1 minus g.probabilityOfWining) times g.cost)
		))
	)
## the sql is :

	@@@SQL
    select min((g.probabilityOfWining * g.prize) -
		       ((1 - g.probabilityOfWining) * g.cost))
    from games
		  
!SLIDE small
# Agregate Queries

	@@@Scala
	val avgHeightByAge: Query[Tuple2[Int,Option[Float]]] = 
	  from(people)(p => 
	    groupBy(p.age)
	    compute(avg(p.heightInCentimeters)))	

## the sql is :

	@@@SQL
	select people1.age, AVG(people1.height) 
	from people people1 
	group by people1.age
	
	
## thanks to the collection API magic, can easily turn the result into a Map[Int,Option[Float]]

	@@@Scala
	val m = avgHeightByAge.toMap
	// Here's why, a Query[A] is an Iterable[A], and :
	trait Iterable[T] {
	...
	  def toMap [T, U] (implicit ev: <:<[A, (T, U)]): 
	    Map[T, U] 
	
	
!SLIDE small
# JPA ? Hibernate ?
## No composability
## Ceremonious ! ;-)

	@@@Java
	Query q = entityManager.createQuery(
	  "SELECT AVG(g.scoreInPercentage) " +
	  "FROM Grades g "+
	  "WHERE g.subjectId = :subjectId");
	  
	q.setParameter(1, mathId);
	
	Number avg = (Number) q.getSingleResult();	 
	
	avg.floatValue(); 	
	
!SLIDE small
# JPA ? Hibernate ?
## No type safety

	@@@Java
	// Equivalent JPA query
	Query q = entityManager.createQuery(
	  //We'll get an SQLException if there's a typo here :
	  "SELECT AVG(g.scoreInPercentage) " +
	  "FROM Grades g "+
	  "WHERE g.subjectId = :subjectId");
	 
	// a runtime exception if mathId is of the wrong type
	q.setParameter(1, mathId); 
	// or if 1 is not the right index 
	   
	// ClassCastException possible 
	Number avg = (Number) q.getSingleResult();
	 
	// NullPointerException if the query returns null
	//(ex.: if there are no math Grades in the table)
	avg.floatValue(); 		

!SLIDE small
#But wait, JPA 2.0 is going type safe
## it is indeed, ...since 2009... Lets look at some code :

	@@@Java
	EntityManager em = ...
	CriteriaBuilder qb = em.getCriteriaBuilder();
	CriteriaQuery<Person> c = qb.createQuery(Person.class);
	Root<Person> p = c.from(Person.class);
	Predicate condition = qb.gt(p.get(Person_.age), 20);
	c.where(condition);
	TypedQuery<Person> q = em.createQuery(c); 
	List<Person> olderThan20s = q.getResultList();
	
## Type safety in a Java query API == Extreeme verbosity *and* code generation
## Compare this with Squeryl :
	
	@@@Scala 
	val olderThan20s = people.where(_.age gt 20)
	
!SLIDE small	
# Arbitrary select expression

	@@@Scala 	
    val q: Query[String] = 
	  from(people)(p => select(&(
	    p.firstName || p.lastName || ', age :' || p.age
	  )))
	  
## sql (oracle) : 
	@@@SQL
	select p.firstName || p.lastName || ', age :' || p.age
	from people p
	  
!SLIDE small
#Relations
##OneToMany
    @@@Scala
    object SchoolDb extends Schema {	 
	  val courses = table[Course]	 
	  val subjects = table[Subject] 	 
	  val subjectToCourses =
	    oneToManyRelation(subjects, courses).
	    via((s,c) => s.id === c.subjectId) 
	}
	   
	class Course(val subjectId: Long, time: Date) 
	  extends KeyedEntity[Long] {	 
	  lazy val subject: ManyToOne[Subject] = 
		SchoolDb.subjectToCourses.right(this)
	}
	 
	class Subject(val name: String) 
	  extends KeyedEntity[Long] {
	  lazy val courses: OneToMany[Course] = 
		SchoolDb.subjectToCourses.left(this)
	}
	
	
!SLIDE small
#Relations

	@@@Scala
	def printCourseSchedule(subjectId: Long) = {
	
	  val subject = subjects.lookup(subjectId).get
	  
	  println("Schedule of " + subject.name)
	  
	  for(course <- subject.courses)
	    println(" - " + course.time)
    }

!SLIDE code_size70
##ManyToMany
    @@@Scala
	class CourseSubscription(val courseId: Int, val studentId: Int, val grade: Float) extends KeyedEntity[CompositeKey2[Int,Int]] {
	  def id = compositeKey(courseId, studentId)
	}
	
	object SchoolDb extends Schema {
	  val students = table[Student]
	  val courses = table[Course]
	  //courseSubscriptions is a ManyToManyRelation, 
	  //it extends Table[CourseSubscriptions]
	  val courseSubscriptions =
		manyToManyRelation(courses, students).
		via[CourseSubscription]((c,s,cs) => 
		  (cs.studentId === s.id, c.id === cs.courseId))  
	}
	  
	class Course(val subjectId: Long) extends KeyedEntity[Long] {
	  lazy val students = SchoolDb.courseSubscriptions.left(this)
	}

	class Student(val firstName: String, val lastName: String) 
	  extends KeyedEntity[Long] {
	  lazy val courses = SchoolDb.courseSubscriptions.right(this)  
	}

!SLIDE small
#ManyToMany

    @@@Scala
	
	val subscription = aCourse.associate(aStudent)
	
	def printStudentsInCourse(c: Course) = {
	
	  println("Students taking " + c.subject.name)
	  
	  for(s <- c.students)
	    println(" - " + s.firstName)
	}
	
	
!SLIDE small
# Performance profiling
## Built in, just turn on the switch !

## <a href='http://squeryl.org/performance-profiling.html'>check out how<a/>