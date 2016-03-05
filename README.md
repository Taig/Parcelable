# Soap

> Scala on Android Parcelable

[![Circle CI](https://img.shields.io/circleci/project/Taig/Soap/master.svg)](https://circleci.com/gh/Taig/Soap/tree/master)
[![codecov.io](https://codecov.io/github/Taig/Soap/coverage.svg?branch=master)](https://codecov.io/github/Taig/Soap?branch=master)
[![Maven](https://img.shields.io/maven-central/v/io.taig.android/soap_2.11.svg)](http://search.maven.org/#artifactdetails%7Cio.taig.android%7Csoap_2.11%7C3.0.0%7CBETA3%7Caar)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/Taig/Soap/master/LICENSE)

Parcelable is Android's serialization tool for inter-process communication (IPC). The emphasis on performance is the prominent difference to the Java Serialization framework (which the developer is discouraged to use for this very reason). Unfortunately, Parcelable requires the developer to implement a vast portion of boilerplate code in order to work. This project combines the performance of Parcelable with the ease of Java's Serializable interface.

## Installation

````scala
libraryDependencies += "io.taig.android" %% "soap" % "3.0.0-SNAPSHOT"
````

## Overview

*Soap* works, in general, with `Bundle` and `Intent`. The library enriches the APIs by `write` and `read` methods. It is therefore always necessary `import io.taig.android.soap.implicits._`.

There are four distinct type classes that handle serialization and deserialization:

### `Writer[C, V]` & `Reader[C, V]`  

Describes how a value `V` is injected into a given container `C` (`Bundle` or `Intent`). This is used to serialize a simple value. For instance, writing an `Int` is as easy as defining:

````scala
implicit val writerBundleInt: Writer.Bundle[Int] = instance { ( bundle, key, value ) =>
    bundle.putInt( key, value )
}
````

### `Encoder[V]` & `Decoder[V]` 

Describes how a value `V` can be en- and decoded as a `Bundle`. Case classes, for instance, are represented as `Bundle`.

````scala
case class Dog( name: String, age: Int )

val bundle: Bundle = Encoder[Dog].encode( Dog( "Holly", 2 ) )
// Bundle( "name" -> "Holly", "age" -> 2 )
````

### Basic usage

*Soap* provides a Bundle alias with custom apply methods. All examples below construct the same `Bundle`.

````scala
import io.taig.android.soap.Bundle
import io.taig.android.soap.implicits._

val bundle = Bundle( 2 ).write( "key1", 42 ).write( "key2", Some( "foobar" ) )

import shapeless._
import shapeless.syntax.singleton._

val bundle = Bundle( "key1" ->> 42 :: "key2" ->> Some( "foobar" ) :: HNil )

val bundle = Bundle( 2 ).write( "key1" ->> 42 :: "key2" ->> Some( "foobar" ) :: HNil )
````

A similar API is available for `Intent`s.

````scala
import android.content.Intent
import io.taig.android.soap.implicits._

val intent = new Intent().write( "key1", 42 ).write( "key2", Some( "foobar" ) )

import shapeless._
import shapeless.syntax.singleton._

val intent = new Intent().write( "key1" ->> 42 :: "key2" ->> Some( "foobar" ) :: HNil )
````

Reading from a `Bundle` or an `Intent` does always return an `Option`.

````scala
bundle.read[Int]( "key1" ) // Some( 42 )
bundle.read[Int]( "key2" ) // None
bundle.read[String]( "key1" ) // None

intent.read[Int]( "key1" ) // Some( 42 )
````

### Activity / Intent

````scala
import android.app.Activity
import android.content.{ Context, Intent }
import io.taig.android.soap.Bundle
import io.taig.android.soap.implicits._

class MyActivity extends Activity {
    lazy val amount: Option[Int] = getIntent.read[Int]( "amount" )

    var myStateValue: Option[String] = None

    override def onCreate( state: Bundle ): Unit = {
        super.onCreate( state )

        myStateValue = Option( state ).flatMap( _.read[String]( "my-state-value" ) )
        
        // ...
    }
    
    // ...

    override def onSaveInstanceState( state: Bundle ): Unit = {
        super.onSaveInstanceState( state )

        state.write( "my-state-value", myStateValue )
    }
}

object MyActivity {
    def apply( amount: Int )( implicit c: Context ): Intent = {
        new Intent( c, classOf[MyActivity] ).write( "amount", amount )
    }
}
````

### Fragment

````scala
import android.app.Fragment
import io.taig.android.soap.Bundle
import io.taig.android.soap.implicits._
import shapeless._
import shapeless.syntax.singleton._

class MyFragment extends Fragment {
    lazy val coordinates: Option[( Int, Int )] = getArguments.read[( Int, Int )]( "coordinates" )

    lazy val orientation: String = getArguments.read[String]( "orientation" ).get


    override def onCreate(state: Bundle): Unit = {
        super.onCreate(state)
        
        val running = Option( state )
            .flatMap( _.read[Boolean]( "running" ) )
            .getOrElse( false )
        
        // ...
    }
    
    // ...

    override def onSaveInstanceState(state: Bundle): Unit = {
        super.onSaveInstanceState(state)
        
        state.write( "running", true )
    }
}

object MyFragment {
    def apply( coordinates: Option[( Int, Int )] ): MyFragment = {
        val fragment = new MyFragment
        fragment.setArguments(
            Bundle(
                "coordinates" ->> coordinates ::
                    "orientation" ->> "portrait" ::
                    HNil
            )
        )
        fragment
    }
}
````

## Adding support for custom types

All important types are supported out of the box, including everything `Parcelable`, `Array`, `Iterable`, `Map`, case classes as well as ADTs and ADT enumerations. Yet, it's still a common task to support further types which requires to provide appropiate type class instances in scope.

````scala
import cats.syntax.functor._
import cats.syntax.contravariant._
import io.taig.android.soap._
import org.threeten.bp.{ Duration, OffsetDateTime }

object mySoapInstances {
    implicit def writerDuration[C]( implicit w: Writer[C, Long] ): Writer[C, Duration] = {
        w.contramap( _.toMillis )
    }

    implicit def readerDuration[C]( implicit r: Reader[C, Long] ): Reader[C, Duration] = {
        r.map( Duration.ofMillis )
    }

    implicit def writerOffsetDateTime[C]( implicit w: Writer[C, String] ): Writer[C, OffsetDateTime] = {
        w.contramap( _.toString )
    }

    implicit def readerOffsetDateTime[C]( implicit r: Reader[C, String] ): Reader[C, OffsetDateTime] = {
        r.map( OffsetDateTime.parse )
    }
}
````

`Serializable` is not supported automatically. It is necessary to provide the `Writer` and `Reader` explicitly.

````scala
import io.taig.android.soap.Bundle
import io.taig.android.soap.Writer.writerBundleSerializable
import io.taig.android.soap.implicits._

import java.net.URL

// Please note that Soap provides a proper Reader and Writer
// instance for URL

Bundle( "url", new URL( "http://taig.io/" ) )( writerBundleSerializable )
````

## Roadmap

- `@Parcelable` annotation for case classes as a memory and performance improvement over wrapping things up in a `Bundle`
- `Codec` and `Format` type classes to easily create custom `Encoder`/`Decoder` and `Writer`/`Reader` instances

## Acknowledgements

This library is highly inspired by [travisbrown/circe][2], especially for the serialization of case classes and ADTs.

## License

MIT, see [LICENSE][1] file for more information

[1]: https://raw.githubusercontent.com/Taig/Soap/master/LICENSE
[2]: https://github.com/travisbrown/circe/
