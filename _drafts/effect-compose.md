---
title: How to compose Effect?
tags:
  - Scala
---
In **What is Effect?**[^1], we talk about the definition of Effect and how to use it in pure way.
In this blog, we will talk about how to use two(even more) effects together in pure way.

# What is the problem?

## The first scenario

```scala
def sqrt(v:Double):Option[Double] = 
  if (v <0) None else Some(Math.sqrt(v))

def div(numerator:Double, denominator:Double):Option[Double] =
  if (denominator == 0) None else Some(numerator/denominator)

def sum(left:Double, right:Double):Either[String, Double] = Right(left + right)

def sub(left:Double, right:Double):Either[String, Double] = Right(left - right)
```

$$
\sqrt{x1 \div x2+x3}-x4
$$

```scala
val divResult:Option[Double] = div(x1,x2)
val result:Either[String, Double] = divResult match {
    case Some(v) =>
       val sumResult:Either[String, Double] = sum(v, x3)
       val sqrtResult:Option[Double] = sumResult match {
           case Right(v1) => sqrt(v1)
           case Left(e) => None
       }
       sqrtResult match {
           case Some(v1) => sub(v1,x4)
           case None => Left("error")
       }
    case None => Left("error")
}
```

```scala
type Composed[A] = Either[String, Option[A]]

def map[A](v:Composed[A])(f: A => B):Composed[B] = v match {
    case Right(Some(v1)) => Right(Some(f(v1)))
    case _ => v
}

def flatMap[A](v:Composed[A])(f: A => Composed[B]):Composed[B] = v match {
    case Right(Some(v1)) => f(v1)
    case _ => v
}

def liftOption[A](v:Option[A]):Composed[A] = Right(v)

def liftEither[A](v:Either[String, A]):Composed[A] = v.map(Some(_))
```

```scala
val divResult:Composed[Double] = liftOption(div(x1,x2))
val result:Either[String, Double] = divResult.flatMap(y => {
    val sumResult:Composed = liftEither(sum(y, x3))
    val sqrtResult:Composed = sumResult.flatMap(x=>liftOption(sqrt(x)))
    sqrtResult.flatMap(x=>liftEither(sub(x,x4)))
}) 
```

## The second scenario
```scala
sealed trait Rocket[A]
case object CheckFuel extends Rocket[String]
case class Launch(target:(Double, Double)) extends Rocket[String]
def rocketInterpreter:Rocket~>Id = new FunctionK[Rocket,Id] {
  def apply[A](v:Rocket[A]):Id[A] = {
    rocket match {
      case CheckFuel => "Fuel is ready"
      case Launch(target) => "Rocket has been launched, target is ${target}"
    }
  }
}

type RocketFree[A] = Free[Rocket, A]
def checkFuel:RocketFree[String] = lift(CheckFuel)
def launch(target:(Double,Double)):RocketFree[String] = lift(Launch(target))

sealed trait Gun[A]
case object CheckBullet extends Rocket[String]
case class Fire(target:(Double, Double)) extends Gun[String]
def gunInterpreter:Gun~>Id = new FunctionK[Gun,Id] {
  def apply[A](v:Gun[A]):Id[A] = {
    gun match {
      case CheckBullet => "Bullet is ready"
      case Fire(target) => "Gun has been fired, target is ${target}"
    }
  }
}

type GunFree[A] = Free[Gun, A]
def checkBullet:GunFree[String] = lift(CheckBullet)
def fire(target:(Double,Double)):GunFree[String] = lift(Fire(target))
```

Steps:
* CheckBullet
* CheckFuel
* Fire
* Launch

```scala
val checkBulletResult = checkBullet.foldMap(gunInterpreter)
val checkFuelResult = checkFuel.foldMap(rocketInterpreter)
val fireResult = fire.foldMap(gunInterpreter)
val launchResult = launch.foldMap(rocketInterpreter)
val result = s"${checkBulletResult};${checkFuelResult};${fireResult};${launchResult}"
```

```scala
case class EitherK[F[_],G[_],A](run: Either[F[A], G[A]])
def leftc[F[_],G[_],A](v:F[A]):EitherK[F,G,A] = EitherK[F,G,A](Left(v))
def rightc[F[_],G[_],A](v:G[A]):EitherK[F,G,A] = EitherK[F,G,A](Right(v))
```

```scala
type RocketAndGun[A] = EitherK[Rocket,Gun,A]
type RocketAndGunFree[A] = Free[RocketAndGun, A]

def checkFuel:RocketAndGunFree[String] = lift(leftc(CheckFuel))
def launch(target:(Double,Double)):RocketAndGunFree[String] = lift(leftc(Launch(target)))
def checkBullet:RocketAndGunFree[String] = lift(rightc(CheckBullet))
def fire(target:(Double,Double)):RocketAndGunFree[String] = lift(rightc(Fire(target)))

val interpreter:RocketAndGun ~> Id = rocketInterpreter or gunInterpreter

val program = for {
  checkBulletResult <- checkFuel
  checkBulletResult <- checkFuel
  launchResult <- checkFuel
  fireResult <- checkFuel
} yield s"${checkBulletResult};${checkFuelResult};${fireResult};${launchResult}"

val result = program.foldMap(interpreter)
```

# Further discussion

## The first scenario

```scala
type ComposeEffect[A] = Reader[String, Either[String,Option[A]]]
```

## The second scenario

```scala
sealed trait Effec1[A]
sealed trait Effec2[A]
sealed trait Effec3[A]
sealed trait Effec4[A]
```



# References
