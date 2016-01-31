## CoreCityOS framework tutorial 

CoreCityOS framework is designed as an CityOS open-source data format standard in the new IOT world. Because of the framework structure which is mostly built around protocols, it can be used with any backend system on any supported platform (Linux, iOS, OS X, tvOS and watchOS). 

1. Installation
2. Protocol oriented approach 
3. Data structures
4. Usage
5. Creating data factory 
6. Serializer concept 
7. Example application

### Installation 

You can use CoreCityOS in a number of ways as noted in the README file of the repository

* Swift Package Manager
* Carthage 
* Manual embeded framework installation 

**Carthage**

To install CoreCityOS with Carthage package manager add following line to the Cartfile file.

```shell
github “cityos/CoreCityOS” ~> 0.0.1
```

and then run following to build the framework. 

```shell
$ carthage update
```

Depending on the platform you want to use it with, use the appropriate framework from `Carthage/Builds` folder. After building process use these instructions to add framework to your project.

**Swift Package Manager**

Swift Package Manager is currently limited to Linux and OS X applications only. To use CoreCityOS with Swift Package Manager create `Package.swift` file and add following:

```swift
import PackageDescription

let package = Package (
  name: "CityOS",
  dependencies: [
    .Package(url: "https://github.com/cityos/CoreCityOS.git", majorVersion: 1),
  ]
)
```
### Using framework 

After you have added framework to your project you can import it with the import statement

```swift
import CoreCityOS
```

### Data structures
When dealing with constant flow of live data, it’s important to define structure of that data, and more importanly, define how that data is accesed.

#### DataType structure
First step is to define plain types of data you want to record, for example Temperature or Humidity. To define data types you need use `DataType` structure through extension. Let’s say that you are reading air pressure and noise data. Define it like this:

```swift
extension DataType {
	static let AirPressure = DataType(dataIdentifier: “Air Pressure“)
	static let Noise = DataType(dataIdentifier: “Noise”)
}
```

Note that this only creates plain data types without any functionality. You will see how it’s used later.

#### LiveDataType protocol 

`LiveDataType` is one of the main framework protocols that is used to define how live data looks like. Protocol requires information about data type (`DataType`), unit notation (for example db or Mhw), and json key that is used when data is serialized from JSON to core objects. Internal implementation of  `LiveDataType` is `LiveData`, although you can create your own implementation.

This is example definition of Noise live data type:

```swift
var noise: LiveDataType {
        return LiveData(
            dataType: DataType.Noise,
            jsonKey: "noise",
            unitNotation: "db"
        )
    }
```
