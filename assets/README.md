# KTX: assets management

Utilities for management of assets and heavy resources.

### Why?

LibGDX does a good job of helping you with assets (through `AssetManager`, for example), but - as usual in case of Java
APIs - it does not use the full potential of Kotlin features. This library aims to provide Kotlin extensions and wrappers
for the existing API to make assets usage more natural in Kotlin applications.

### Guide

#### Assets

- Global `AssetManager` instance is accessible (and modifiable) through `Assets.manager` utility field. Since it is
advised to share and reuse a single `AssetManager` instance in the application and LibGDX already goes crazy with the
statics thanks to `Gdx.files` and whatnot, this `AssetManager` instance was added to reduce asset-related boilerplate.
If a utility asset loading (or accessing) method accepts an `AssetManager` instance, this global manager will be used by
default to save you the trouble of getting your `AssetManager` manually. Note that its usage is completely optional and
you can omit it entirely, while still benefiting from asset loading utility methods.
- `load` method can be used to schedule asynchronous loading of an asset. It returns an asset wrapper, which can be
used as delegate property, as well as used directly to manage the asset. Usually the asset will not be available until
`Assets.manager.finishLoading` or looped `Assets.manager.update` are called. Custom `AssetManager` instance can be passed
to loading methods if prefer not to use statics. You can use string file paths or `AssetDescriptor` instances to load
the asset. Typical usage:
```Kotlin
// Eagerly loading an asset:
val wrapper = load<Texture>("test.png")
wrapper.finishLoading()
val texture = wrapper.asset

// Delegate field:
class Test {
  val texture by load<Texture>("test.png")
  // Type of texture property is Texture.
}
```
- `asset` utility method can be used to access an already loaded asset, omitting the wrapper altogether. This is the
preferred way of accessing assets from the `AssetManager`, provided that they were already scheduled for asynchronous
loading and fully loaded. Typical usage: `val texture = asset<Texture>("test.png")`. Note that this method will fail if
asset is not loaded yet.
- `loadOnDemand` is similar to the `load` utility method, but it provides an asset wrapper that loads the asset eagerly
on first get call. It will not schedule the asset for asynchronous loading - instead, it will block current thread until
the asset is loaded on the first access. Use for lightweight assets that should be (rarely) loaded only when requested.
Typical usage:
```Kotlin
// Eagerly loading an asset:
val wrapper = loadOnDemand<Texture>("test.png")
val texture = wrapper.asset // Will load the asset.

// Delegate field:
class Test {
  val texture by loadOnDemand<Texture>("test.png")
  // Type of texture property is Texture. It will be loaded on first `texture` access.
}
```
- `isLoaded` is a utility method that allows to check if a particular asset is already loaded. For example:
`isLoaded<Texture>("test.png")`.
- `unload` is a utility method that attempts to unload an asset from the `AssetManager`. Contrary to manual
`AssetManager.unload` call, this is a graceful method that will not throw any exceptions if the asset was not even
loaded. Typical usage: `unload("test.png")`.

##### Implementation tip: type-safe assets

Create an enum with all assets of the selected type. Let's assume our application stores all images in `assets/images`
folder in `PNG` format. Given `logo.png`, `player.png` and `enemy.png` images, we would create a similar enum:
```Kotlin
enum class Images {
  logo,
  player,
  enemy;

  val path = "images/${name}.png"
  fun load() = load<Texture>(path)
  operator fun invoke() = asset<Texture>(path)
}
```
Operator `invoke()` function brings asset accessing boilerplate to mininum: `enumName()`. Thanks to wildcard imports, we
can access `logo`, `player` and `enemy` enum instances directly:
```Kotlin
import com.example.Images.*

// Scheduling logo loading:
logo.load()

// Getting Texture instance of loaded logo:
val texture = logo()

// Scheduling loading of all assets:
Images.values().forEach { it.load() }

// Accessing all textures:
val textures = Images.values().map { it() }
```

#### `Disposable`

- Null-safe `Disposable.disposeSafely()` method was added. Can be called on a nullable `Disposable?` variable. Ignores
most thrown exceptions (except for internal JVM `Error` instances, which should not be caught anyway).
- `Disposable.dispose` with an exception catch block was added. Using `asset.dispose { exception -> doSomething() }`
syntax, you can omit a rather verbose try-catch block and handle exceptions with a Kotlin lambda.
- Any `Iterable` or `Array` storing `Disposable` instances will have `dispose`, `dispose { exception -> }` and
`disposeSafely` methods that dispose stored assets ignoring any `null` elements. This is a utility for disposing
collections of assets en masse.
- All exceptions get a utility `ignore()` method that you can switch at compile time (for debugging or logging) when
needed. See `Throwable.ignore()` documentation for further details.

#### `Pool`

- `Pool` instances can be invoked like a function to provide new instances of objects. Basically, this syntax: `pool()`
has the same effect as directly calling `pool.obtain()`.
- `Pool` instances can be invoked like a one argument function to free instances of objects. Basically, this syntax:
`pool(instance)` has the same effect as directly calling `pool.free(instance)`.
- New instances of `Pool` can be easily created with Kotlin lambda syntax using `pool` method. For example, this pool
would return new instances of `Entity` once empty: `pool { Entity() }`. Since this method is inlined, you should not
worry about unnecessary extra method calls or extra objects - the `Pool` implementations are prepared at compile time.

#### `FileHandle`

- Any Kotlin string can be quickly converted to a `FileHandle` instance using `toClasspathFile`, `toInternalFile`,
`toLocalFile`, `toExternalFile` or `toAbsoluteFile`. This is basically a utility for accessing `Gdx.files.getFileHandle`
method with a pleasant Kotlin syntax.

### Alternatives

- [libgdx-utils](https://bitbucket.org/dermetfan/libgdx-utils/) feature an annotation-based asset manager implementation
which easies loading of assets (through internal reflection usage).
- [Autumn MVC](https://github.com/czyzby/gdx-lml/tree/master/mvc) is a [Spring](https://spring.io/)-inspired
model-view-controller framework built on top of LibGDX. It features its own asset management module which loads and
injects assets into annotated fields thanks to reflection.
- [Kiwi](https://github.com/czyzby/gdx-lml/tree/master/kiwi) library has some utilities for assets handling, like
graceful `Disposable` destruction methods and LibGDX collections implementing `Disposable` interface. It is aimed at
Java applications, though - **KTX** syntax feels more natural when using Kotlin.