# From App to IOApp in a jiffy

When I've started my recent pet-project, I didn't knew anything about functional paradigm (except for the fact that it is very aesthetic way to write code). So I've started writing my application "by the book" as in defining `Main` class as seen in [docs](https://www.scala-lang.org/api/current/scala/App.html):

```scala
object Main extends App 
```

This particular application was designed as REST-service with database. Since I planned to implement business-logic using actors, other framework-wise choses were pretty obvious: [Akka HTTP](https://doc.akka.io/docs/akka-http/current/index.html) server and [Slick](https://scala-slick.org/) for accessing the database. So my code (simplified) looked like:

```scala
object Main extends App {
  def routes(implicit system: ActorSystem, db: Database, profile: JdbcProfile): Route = // REST API routes

  implicit val system: ActorSystem = ActorSystem("my-project")
  val dbConfig = DatabaseConfig.forConfig[JdbcProfile]("my.db")
  implicit val db: Database = Database.forConfig("my.db.properties")
  implicit val profile: JdbcProfile = dbConfig.profile
  
  val bind: Future[Http.ServerBinding] = Http().bindAndHandle(routes, "0.0.0.0", "80")
  
  bind.map(b => {
    StdIn.readLine()
    StdIn.readLine()
    b.terminate(1 second)
    db.close()
    system.terminate()
  })
}
```

The first run was faulty - I've messed up the DB configuration and the initialization of `db` threw an exception. So I started to write `try - catch` blocks. With `ActorSystem` and `Database` everything is pretty straightforward:

```scala
object Main extends App {
  try {
    implicit val system: ActorSystem = ActorSystem("my-project")
    try {            
      implicit val db: Database = Database.forConfig("my.db.properties")
      implicit val profile: JdbcProfile = DatabaseConfig.forConfig[JdbcProfile]("my.db").profile
      try {
        // Start HTTP, wait for two Enters and close the app
      }
      catch {
        case _: Throwable => 
      }
    }        
    finally {
      system.terminate()
    }
  }
  catch {
    case _: Throwable => ()
  }
}
```

But  `Http().bindAndHandle` returns `Future` and it complicates things a little. Also the code itself with this nesting of `try - catch` is not something that I want to write and maintain. Don't know about anyone else's preferences, but this is not the code that I want to work with. 

Just to recap the problem: I have `ActorSystem`, `Database` and `Route` (of Akka HTTP), which depends on both of them. Each one of them may fail to initialize, in which case I would like to gracefully dispose of the previously initialized objects. 

Fortunately, functional library [Cats-effect](https://typelevel.org/cats-effect) has a data type which solves this problem perfectly - [Resource](https://typelevel.org/cats-effect/datatypes/resource.html). `Resource` can be acquired and release (see [Acquiring and releasing Resource](https://typelevel.org/cats-effect/tutorial/tutorial.html#acquiring-and-releasing-resources)), and they can be combined in for-comprehension to form single `Resource`. Although implicit values will be impossible to use with Resource, the benefits are bigger, than the disadvantages of passing the parameters explicitly. And thus the code of initialization was change to:

```scala
object IOStarter {
  // REST API routes
  def routes(system: ActorSystem, db: Database, profile: JdbcProfile): Route = ???
    
  def getSystem: Resource[IO, ActorSystem] = Resource.make {
    IO(ActorSystem("my-project")
  } {
    s => IO.fromFuture(IO(s.terminate())).map(_ => ())
  }
     
  def getDb: Resource[IO, Database] = Resource.make {
    IO(Database.forConfig("my.db.properties"))
  } {
    d => IO(db.close())
  }
     
  def getProfile: Resource[IO, JdbcProfile] = Resource.liftF(
    IO(DatabaseConfig.forConfig[JdbcProfile]("my.db").profile)
  )
     
  def getHttp(system: ActorSystem, db: Database, profile: JdbcProfile): Resource[IO, Http.ServerBinding] = Resource.make {
    IO.fromFuture(IO(Http().bindAndHandle(routes(system, db, profile), "0.0.0.0", "80")))
  } {
    srv => IO.fromFuture(IO(srv.terminate(1 second)))
  }
       
  def getAllResources: Resource[IO, (ActorSystem, Database, Http.ServerBinding)] = for {
    system <- getSystem
    profile <- getProfile
    db <- getDb
    srv <- getHttp(system, db, profile)
  } yield (system, db, srv)
     
  def start: IO[Unit] = getAllResources.map(_ => for {
      _ <- IO(StdIn.readLine)
      _ <- IO(StdIn.readLine)
    } yield ()
  })
}
```

This code can be used in regular `scala.App`:

```scala
object Main extends App {
  IOStarter.start.unsafeRunSync()
}
```

Or it can be changed to `cats.effect.IOApp`:

```scala
object Main extends IOApp {
  def run(args: List[String]): IO[ExitCode] =
    IOStarter.start.map(_ => ExitCode.Success)
}
```

The rest of the code may very well be non-functional, but at least resource management became less clogged with `try-catch`. 

From my personal experience of such rewrite, after implementing `IOApp`:

* with `scala.App` I had weired exception every time while shutting down the app (something to do with the `sbt` thread not being released). They disappeared when I switched to `IOApp`

* with `scala.App` sometimes I had exceptions at start up. They also were gone after the switch.

  All in all, the startup and shutdown of the application runs smoother, that with `scala.App` and all the `try-catch` nesting hell (which reminds me a lot of JavaScript's [callback hell](http://callbackhell.com/)) is no more.

  