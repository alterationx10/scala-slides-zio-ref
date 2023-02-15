---
title: ZIO Ref + FiberRef
description: A quick talk about Refs from the Zionomicon
author: mark Rudolph
keywords: scala, zio
url: https://alterationx10.com
marp: true
---

# ZIO Ref + FiberRef

A quick talk about Refs from the Zionomicon

---

## Refs

`Ref`s:

- model mutable state in a functional way.
- are the purely functional equivalent of an AtomicReference.
- are safe for concurrent access.
- DO NOT SAFELY COMPOSE

---

## Compose Note

### Good:

```scala
for {
    ref <- Ref.make(0)
    _ <- ZIO.foreachParDiscard((1 to 10000).toList) { _ =>
            ref.update(_ + 1)
        }
    result <- ref.get
} yield result
```

### Bad:

```scala
for {
    ref <- Ref.make(0)
    _ <- ZIO.foreachParDiscard((1 to 10000).toList) { _ =>
            ref.get.flatMap(n => ref.set(n + 1))
        }
    result <- ref.get
} yield result
```

---

# Ref Tips

- Put all pieces of state that need to be “consistent” with each other in a
  single Ref.
- Always modify the Ref through a single operation.

---

## Making Refs

```scala
val makeHighScore: UIO[Ref[Int]] = Ref.make[Int](0)
def makeInitials: UIO[Ref[String]] = Ref.make[String]("AAA")
def insertCoins(mny: Int) = Ref.make[Int](mny)
```

---

## Let's Play A Game

```scala
case class Game(
    credits: Ref[Int],
    highScore: Ref[Int],
    highScoreInitials: Ref[String],
    playerInitials: String
) {
  val playGame: Task[Int] = for {
    _ <- Console.printLine(s"$playerInitials playing:")
    currentHighScore <- highScore.get
    currentInitials <- highScoreInitials.get
    _ <- Console.printLine(
      s"High Score to beat: $currentInitials $currentHighScore"
    )
    generatedScore <- zio.Random.nextIntBetween(0, currentHighScore + 1000)
    _ <- Console
      .printLineError(s"Try again!")
      .when(generatedScore <= currentHighScore)
    _ <- Console
      .printLineError(s"New high score: $generatedScore")
      .when(generatedScore > currentHighScore)
    _ <- highScore.set(generatedScore).when(generatedScore > currentHighScore)
    _ <- highScoreInitials
      .set(playerInitials)
      .when(generatedScore > currentHighScore)
    _ <- credits.update(_ - 1)
  } yield generatedScore
}
```

---

## The Game In Action

```scala
val gameServer = for {
  credits <- insertCoins(4)
  highScore <- makeHighScore
  highInitials <- makeInitials
  _ <- Game(credits, highScore, highInitials, "MAR").playGame
    .repeatWhileZIO(_ => credits.get.map(_ > 0))
  _ <- highScore.get
    .flatMap(s =>
      Console.printLine(s"Final high score: $s\nPlease insert more coins.")
    )
} yield ()
```

---

## Game Output

```
MAR playing:
High Score to beat: AAA 0
New high score: 562
MAR playing:
High Score to beat: MAR 562
Try again!
MAR playing:
High Score to beat: MAR 562
New high score: 1296
MAR playing:
High Score to beat: MAR 1296
Try again!
Final high score: 1296
Please insert more coins.
```

---

## Multiplayer Game!

```scala
val onlineGameServer = for {
  player1 <- insertCoins(4)
  player2 <- insertCoins(4)
  highScore <- makeHighScore
  highInitials <- makeInitials
  g1 <- Game(player1, highScore, highInitials, "MAR").playGame
    .repeatWhileZIO(_ => player1.get.map(_ > 0))
    .fork
  g2 <- Game(player2, highScore, highInitials, "JER").playGame
    .repeatWhileZIO(_ => player2.get.map(_ > 0))
    .fork
  _ <- g1.join
  _ <- g2.join
  hs <- highScore.get
  init <- highInitials.get
  _ <- Console.printLine(
    s"Final high score: $init $hs\nPlease insert more coins."
  )
} yield ()
```

---

## Multiplayer Output

```
MAR playing:
High Score to beat: AAA 0
JER playing:
New high score: 445
High Score to beat: AAA 0
New high score: 211
JER playing:
High Score to beat: JER 211
New high score: 836
JER playing:
High Score to beat: JER 836
New high score: 959
MAR playing:
High Score to beat: JER 959
JER playing:
New high score: 1391
High Score to beat: JER 959
New high score: 1270
MAR playing:
High Score to beat: JER 1270
New high score: 1980
MAR playing:
High Score to beat: MAR 1980
Try again!
Final high score: MAR 1980
Please insert more coins.
```

---

## Ref.Synchronized

- Used when you want to run an effect while working with state.
- Each operation on Ref.Synchronized is guaranteed to be executed atomically.
- Implemented using a Semaphore to ensure that only one fiber can be interacting
  with the reference at the same time.

---

## GameV2

```scala
case class ServerState(
    highScore: Int
)
val makeServerState = Ref.Synchronized.make(ServerState(0))

case class GameV2(
    currentHighScore: Int
) {
  val playGame: Task[Int] = for {
    generatedScore <- zio.Random.nextIntBetween(0, currentHighScore + 1000)
  } yield generatedScore
}
```

---

## GameV2 Example

```scala
val gameServerV2 = for {
  credits <- insertCoins(4)
  server <- makeServerState
  _ <- server
    .getAndUpdateZIO { state =>
      credits.getAndUpdate(_ - 1) *>
        GameV2(state.highScore).playGame
          .debug("Player score")
          .map { playerScore =>
            state.copy(highScore = scala.math.max(state.highScore, playerScore))
          }
    }
    .repeatWhileZIO(_ => credits.get.map(_ > 0))
  _ <- server.get.flatMap(hs =>
    Console.printLine(
      s"Final high score: $hs\nPlease insert more coins."
    )
  )

} yield ()
```

---

## GameV2 Output

```
Player score: 764
Player score: 118
Player score: 797
Player score: 375
Final high score: ServerState(797)
Please insert more coins.
```

---

## FiberRef

- maintain some state that is local to each fiber
- has `fork` behavior
- has `join` behavior

---

## Request Context

```scala
case class User(name: String)

def makeRequestContext(userOpt: Option[User]) =
  FiberRef
    .make(
      userOpt,
      fork = parent => parent,
      join = (parent, child) => parent
    )
    .provideLayer(Scope.default)
```

---

## Request Context Example

```scala
def handleRequest(ref: FiberRef[Option[User]]) = for {
  userOpt <- ref.get
  _ <- ZIO
    .fromOption(userOpt)
    .flatMap(u => Console.printLine(s"Welcome back, $u!"))
    .ignore
  _ <- Console.printLine("Forbidden!").when(userOpt.isEmpty)
} yield ()

val serve = for {
  ref <- makeRequestContext(Option.empty)
  f1 <- (ref.set(Some(User("Mark"))) *> handleRequest(ref)).fork
  _ <- f1.join
  _ <- handleRequest(ref)
} yield ()
```

```
Welcome back, User(Mark)!
Forbidden!
```

---

## Danger

```scala
val serveDANGER = for {
  ref <- makeRequestContext(Option.empty)
  f1 <- (ref.set(Some(User("Mark"))) *> handleRequest(ref))
  _ <- handleRequest(ref)
} yield ()
```

```
Welcome back, User(Mark)!
Welcome back, User(Mark)!
```

---

## Default Danger

```scala
def makeBadRequestContext(userOpt: Option[User]) =
  FiberRef
    .make(
      userOpt,
      fork = parent => parent,
      join = (parent, child) => child
    )
    .provideLayer(Scope.default)

val serveBAD = for {
  ref <- makeBadRequestContext(Option.empty)
  f1 <- (ref.set(Some(User("Mark"))) *> handleRequest(ref)).fork
  _ <- f1.join
  _ <- handleRequest(ref)
} yield ()
```

```
Welcome back, User(Mark)!
Welcome back, User(Mark)!
```

---

## Locally

```scala
val serveSNEAKY = for {
  ref <- makeRequestContext(Option.empty)
  f1 <- (ref.set(Some(User("Mark"))) *> handleRequest(ref)).fork
  _ <- f1.join
  _ <- ref
    .locally(Some(User("Admin"))) {
      handleRequest(ref)
    }
  _ <- handleRequest(ref)
} yield ()
```

```
Welcome back, User(Mark)!
Welcome back, User(Admin)!
Forbidden!
```