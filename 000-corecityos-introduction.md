## CoreCityOS framework tutorial 

CoreCityOS framework is designed as an CityOS open-source data format standard for whole IOT stack starting from harware sensors to the user mobile devices. Framework structure which is mostly built around protocols, it can be used with any backend system on any supported platform (Linux, iOS, OS X, tvOS and watchOS). 

1. [Installation](#installation)
2. [Usage](#using-framework)
3. [Data structures](#data-structures)

## Installation 

You can build and use `CoreCityOS` framework in a number of ways:

* Carthage 
* Swift Package Manager
* Manual embeded framework installation 

**Carthage**

To install `CoreCityOS` with [Carthage](https://github.com/Carthage/Carthage) add following line to the `Cartfile` file.

```shell
github “cityos/CoreCityOS” ~> 0.0.1
```

and then run following to build the framework. 

```shell
$ carthage update
```

Depending on the platform you want to use it with, use the appropriate framework from `Carthage/Builds` folder. After building process use these instructions to add framework to your project.

> Note that you can pass `--platform` option to the `carthage update` command to specify platform that you want to use CoreCityOS with. Carthage currently supports iOS, tvOS and OSX.

**Swift Package Manager**

[Swift Package Manager](https://github.com/apple/swift-package-manager) is currently limited to Linux and OS X applications only. To use CoreCityOS with Swift Package Manager create `Package.swift` file and add following:

```swift
import PackageDescription

let package = Package (
  name: "YOUR_PROJECT_NAME",
  dependencies: [
    .Package(url: "https://github.com/cityos/CoreCityOS.git", majorVersion: 1),
  ]
)
```
Then run following to build the framework:
```bash
$ swift build
```

> Note that the [Swift Package Manager](https://swift.org/package-manager) is still in early design and development, for more infomation checkout its [GitHub Page](https://github.com/apple/swift-package-manager)

## Using framework 

After you have added framework to your project you can import it with the import statement:
```swift
import CoreCityOS
```
or you can import just the functionality you need:
```swift
import protocol CoreCityOS.LiveDataCollectionType
```

### Data structures
`CoreCityOS` is composed of several data structures and protocols utilized for easier live data access and manipulation. All of those components work together to create light yet powerfull framework. Let's examine most basic data flow:
```
Harware devices (sensors, raw data) -> Internet (processing data) -> User (recieving processed data)
```
Each device reads data from it's sensors and then in real time sends the raw data to some server over the internet (or any other type of communication). Server processes that data and sends it to the users in human readable form. Since all of these separate steps require different programming languages and techniques, it can be tough job to maintain whole stack from the hardware device to the user. More importantly, there is no defined data and communication standard for that kind of operations. That data standard is now created as a part of CityOS, and it can be used with this framework.

To understand the data standard you must first understand data structures used:

##### `DataType`
First step is to define plain types of data you want to record, for example Temperature or Humidity. To define data types you need use `DataType` structure through extension. Let’s say that you are reading air pressure and noise data. Define it like this:

```swift
extension DataType {
	static let AirPressure = DataType(dataIdentifier: “Air Pressure“)
	static let Noise = DataType(dataIdentifier: “Noise”)
}
```

Note that this only creates plain data types without any functionality. You will see how it’s used later.

##### `DataPoint`
`DataPoint` is one of the core parts of the framework that is used to create single data point in time. Each `DataPoint` is composed of some value and the `NSDate` timestamp. There are several ways to create `DataPoint`

```swift
let dataPoint = DataPoint(value: 10.0)
let dataPoint = DataPoint(value: 20.3, unixTimeStamp: 12567234.0)
```
##### `LiveDataType`

`LiveDataType` is one of the main framework protocols that is used to define how live data looks like. Protocol requires information about data type (`DataType`), unit notation (for example `db` or `Mhw`), and JSON key that is used when data is serialized from JSON to Swift objects. Internal implementation of `LiveDataType` is `LiveData`, which is a reccomended way to use the protocol, although you can create your own implementations.

This is example definition of `Noise` live data type:

```swift
var noise: LiveDataType {
	return LiveData(
		dataType: DataType.Noise,
		jsonKey: "noise",
		unitNotation: "db"
        )
}
```

##### `LiveDataCollectionType`
Each hardware device that you use is equipped with several sensors. `LiveDataCollectionType` is used to encapsulate data from these sensors using instances of `LiveDataType`. For example, let's say that some device has 2 sensors that are recording temperature and humidity data. You first create `LiveDataType` for temperature and humidity instances as mentioned before. Then you wrap it inside `LiveDataCollectionType` like this:

```swift
class MyDeviceCollection: LiveDataCollectionType {
		
	/// Device meta data
	var deviceData = DeviceData(deviceID: "test-device")

	/// Creation date
	var creationDate = NSDate()

	/// Wrap all `LiveDataType` instances in this array
	var allReadings = [LiveDataType]()
    
	// Create temperature and humidity
	var temperature: LiveDataType {
		return LiveData(
            		dataType: .Temperature,
            		jsonKey: "temp",
            		unitNotation: "C"
        	)
        }
    
	var noise: LiveDataType {
		return LiveData(
			dataType: .Humidity,
			jsonKey: “humidity”,
			unitNotation: “%”
        )
    }
    
    init() {
	// Populate the array
        self.allReadings = [
        	temperature,
        	noise
        ]
    }
}
```

Note that `allReadings` array is the key requirement for `LiveDataCollectionType` protocol. You can later loop trough it and get the data points. 

After you adopt `LiveDataCollectionType` protocol, you get some neat features like subscribing functions. Let’s say you create instance of `MyDeviceDataCollection` like:

```swift
var dataCollection = MyDeviceDataCollection()
```
If you want to add temperature `DataPoint` you can do it like this:

```swift
dataCollection[.Temperature].addDataPoint(DataPoint(value: 20.0))
```
You can also subscript with `jsonKey` property like:

```swift
let currentDataPoint = dataCollection[“humidity”].currentDataPoint
```
This enables flexible and easy way to access all live data information.
> In the hardware world these data collections are know as schemas

##### `DeviceType`
`DeviceType` is a simple protocol that is used to describe hardware device. It requires device meta data (`ID`, `shemaID`), creationDate and `LiveDataCollectionType` instance that it manages. Example implementation would be:

```swift
public class Lamp: DeviceType {
    public var deviceData: DeviceData
    public var creationDate: NSDate?
    public var location: DeviceLocation?
    public var dataCollection: LiveDataCollectionType = MyDeviceDataCollection()
    
    public init(deviceID: String) {
    	self.deviceData = DeviceData(deviceID: deviceID)
    }
}
```
After that you can create instance of that particular class and access the data
```swift
let myLamp = Lamp(deviceID: "12345-test")
myLamp.dataCollection[.Temperature]?.addDataPoint(DataPoint(value: 30.4))
```

