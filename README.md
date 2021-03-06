# kotlin-result

[![Release](https://api.bintray.com/packages/michaelbull/maven/kotlin-result/images/download.svg)](https://bintray.com/michaelbull/maven/kotlin-result/_latestVersion) [![Build Status](https://travis-ci.org/michaelbull/kotlin-result.svg?branch=master)](https://travis-ci.org/michaelbull/kotlin-result) [![License](https://img.shields.io/github/license/michaelbull/kotlin-result.svg)](https://github.com/michaelbull/kotlin-result/blob/master/LICENSE)

[`Result<V, E>`][result] is a monad for modelling success ([`Ok`][result-ok]) or
failure ([`Err`][result-err]) operations.

## Installation

```groovy
repositories {
    maven { url = 'https://dl.bintray.com/michaelbull/maven' }
}

dependencies {
    compile 'com.michael-bull.kotlin-result:kotlin-result:1.1.0'
}
```

## Introduction

The [`Result`][result] monad has two subtypes, [`Ok<V>`][result-ok] 
representing success and containing a `value`, and [`Err<E>`][result-err],
representing failure and containing an `error`. 

Scott Wlaschin's article on [Railway Oriented Programming][swalschin-rop] is a great
introduction to the benefits of modelling operations using the `Result` type.

Mappings are available on the [wiki][wiki] to assist those with experience 
using the `Result` type in other languages:

- [Elm](https://github.com/michaelbull/kotlin-result/wiki/Elm)
- [Haskell](https://github.com/michaelbull/kotlin-result/wiki/Haskell)
- [Rust](https://github.com/michaelbull/kotlin-result/wiki/Rust)
- [Scala](https://github.com/michaelbull/kotlin-result/wiki/Scala)

### Creating Results

To begin incorporating the `Result` type into an existing codebase, you can 
wrap functions that may fail (i.e. throw an `Exception`) with
[`Result.of`][result-of]. This will execute the block of code and `catch` any
`Exception`, returning a `Result<T, Exception>`.

```kotlin
val result: Result<Customer, Exception> = Result.of { 
    customerDb.findById(id = 50) // could throw SQLException or similar 
}
```

The idiomatic approach to modelling operations that may fail in Railway
Oriented Programming is to avoid throwing an exception and instead make the 
return type of your function a `Result`.

```kotlin
fun checkPrivileges(user: User, command: Command): Result<Command, CommandError> {
    return if (user.rank >= command.mininimumRank) {
        Ok(command)
    } else {
        Err(CommandError.InsufficientRank(command.name))
    }
}
```

Nullable types, such as the `find` method in the example below, can be 
converted to a `Result` using the `toResultOr` extension function.

```kotlin
val result: Result<Customer, String> = customers
    .find { it.id == id } // returns Customer?
    .toResultOr { "No customer found" }
```

### Transforming Results

Both success and failure results can be transformed within a stage of the
railway track. The example below demonstrates how to transform an internal
program error (`UnlockError`) into an exposed client error 
(`IncorrectPassword`).

```kotlin
val result: Result<Treasure, UnlockResponse> = 
    unlockVault("my-password") // returns Result<Treasure, UnlockError>
    .mapError { IncorrectPassword } // transform UnlockError into IncorrectPassword
```

### Chaining

Results can be chained to produce a "happy path" of execution. For example, the
happy path for a user entering commands into an administrative console would 
consist of: the command being tokenized, the command being registered, the user
having sufficient privileges, and the command executing the associated action.
The example below uses the `checkPrivileges` function we defined earlier.

```kotlin
tokenize(command.toLowerCase())
    .andThen(::findCommand)
    .andThen { cmd -> checkPrivileges(loggedInUser, cmd) }
    .andThen { execute(user = loggedInUser, command = cmd, timestamp = LocalDateTime.now()) }
    .mapBoth(
        { output -> printToConsole("returned: $output") },
        { error  -> printToConsole("failed to execute, reason: ${error.reason}") }
    )
```

## Inspiration

Inspiration for this library has been drawn from other languages in which the
Result monad is present, including:

- [Elm](http://package.elm-lang.org/packages/elm-lang/core/latest/Result)
- [Haskell](https://hackage.haskell.org/package/base-4.10.0.0/docs/Data-Either.html)
- [Rust](https://doc.rust-lang.org/std/result/)
- [Scala](http://www.scala-lang.org/api/2.12.4/scala/util/Either.html)

It also iterates on other Result libraries written in Kotlin, namely:

- [danneu/kotlin-result](https://github.com/danneu/kotlin-result)
- [kittinunf/Result](https://github.com/kittinunf/Result)
- [npryce/result4k](https://github.com/npryce/result4k)

Improvements on the existing solutions include:

- Feature parity with Result types from other languages including Elm, Haskell,
     & Rust
- Lax constraints on `value`/`error` nullability
- Lax constraints on the `error` type's inheritance (does not inherit from
    `Exception`)
- Top level `Ok` and `Err` classes avoids qualifying usages with
    `Result.Ok`/`Result.Err` respectively
- Higher-order functions marked with the `inline` keyword for reduced runtime
    overhead
- Extension functions on `Iterable` & `List` for folding, combining, partitioning
- Consistent naming with existing Result libraries from other languages (e.g.
    `map`, `mapError`, `mapBoth`, `mapEither`, `and`, `andThen`, `or`, `orElse`,
    `unwrap`)
- Extensive test suite with over 50 [unit tests][unit-tests] covering every library method

## Example

The [example][example] module contains an implementation of Scott's
[example application][swalschin-example] that demonstrates the usage of `Result`
in a real world scenario.

It hosts a [ktor][ktor] server on port 9000 with a `/customers` endpoint. The
endpoint responds to both `GET` and `POST` requests with a provided `id`, e.g.
`/customers/100`. Upserting a customer id of 42 is hardcoded to throw an 
[`SQLException`][customer-42] to demonstrate how the `Result` type can [map 
internal program errors][update-customer-error] to more appropriate 
user-facing errors.

### Payloads

#### Fetch customer information

```
$ curl -i -X GET  'http://localhost:9000/customers/5'
```

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Content-Length: 93

{
  "id": 5,
  "firstName": "Michael",
  "lastName": "Bull",
  "email": "example@email.com"
}
```

#### Add new customer

```
$ curl -i -X POST \
   -H "Content-Type:application/json" \
   -d \
'{
  "firstName": "Your",
  "lastName": "Name",
  "email": "your@email.com"
}' \
 'http://localhost:9000/customers/200'
```

```
HTTP/1.1 201 Created
Content-Type: text/plain; charset=UTF-8
Content-Length: 16

Customer created
```

## Contributing

Bug reports and pull requests are welcome on [GitHub][github].

## License

This project is available under the terms of the ISC license. See the
[`LICENSE`](LICENSE) file for the copyright information and licensing terms.

[result]: https://github.com/michaelbull/kotlin-result/blob/master/src/main/kotlin/com/github/michaelbull/result/Result.kt#L10
[result-ok]: https://github.com/michaelbull/kotlin-result/blob/master/src/main/kotlin/com/github/michaelbull/result/Result.kt#L30
[result-err]: https://github.com/michaelbull/kotlin-result/blob/master/src/main/kotlin/com/github/michaelbull/result/Result.kt#L35
[result-of]: https://github.com/michaelbull/kotlin-result/blob/master/src/main/kotlin/com/github/michaelbull/result/Result.kt#L17
[swalschin-rop]: https://fsharpforfunandprofit.com/rop/
[wiki]: https://github.com/michaelbull/kotlin-result/wiki
[unit-tests]: https://github.com/michaelbull/kotlin-result/tree/master/src/test/kotlin/com/github/michaelbull/result
[example]: https://github.com/michaelbull/kotlin-result/tree/master/example/src/main/kotlin/com/github/michaelbull/result/example
[swalschin-example]: https://github.com/swlaschin/Railway-Oriented-Programming-Example
[ktor]: http://ktor.io/
[customer-42]: https://github.com/michaelbull/kotlin-result/blob/master/example/src/main/kotlin/com/github/michaelbull/result/example/service/InMemoryCustomerRepository.kt#L38
[update-customer-error]: https://github.com/michaelbull/kotlin-result/blob/master/example/src/main/kotlin/com/github/michaelbull/result/example/service/CustomerService.kt#L50
[github]: https://github.com/michaelbull/kotlin-result
