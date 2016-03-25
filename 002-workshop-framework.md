# Creating iOS framework for communication with real IoT devices 

## Steps
- Introduction
- Example framework
- Requirements
- Creating the framework
- Integrating CoreCityOS framework
- Defining weather station device
- Defining weather station data collection
- Pulling data from server
- Creating `WeatherStationFactory` class

### Introduction

There are a lot of interesting IoT devices around us and each day people around the world design and create some new. Of course, making the hardware IoT device is just beginning part of the story. The real challenge with the IoT devices is how to present useful data to the device users, and this tutorial aims to teach you how it's done using some exciting CityOS technologies. 

Code with final version of the framework is available on [Github](https://github.com/cityos/WeatherStationFactory).

### Example framework

In order to show the best possible practices and Swift code examples, we are going to create iOS framework that will communicate with IoT device that acts as weather station. This example is chosen because weather station device can be created by anyone with simple Arduino. 

For our example, weather station device is device that reads data from several sensors:

* Temperature
* Humidity
* PM10 (air pollution)

We will define this data in our iOS framework and later expose it to the iOS application.

We are going to use `CoreCityOS` as a base framework for our framework so it does all heavy lifting for us. If you are not familiar with `CoreCityOS` framework, I suggest reading [CoreCityOS tutorial](http://cityos.io/tutorial/1914/CoreCityOS-Framework-Tutorial) that explains in detail how `CoreCityOS` works with IoT devices.

Also, you can refer to the documentation website of the `CoreCityOS` framework that is located [here](http://cityos.github.io/CoreCityOS)

Installation of the CoreCityOS framework will be done using [Carthage](https://github.com/Carthage/Carthage) package manager, which will be explained in the next step.

### Requirements

Requirements for the framework we will be creating are:

* Xcode 7.0+
* [Carthage](https://github.com/Carthage/Carthage)

If you don't have Carthage installed you can refer to the [official installation instructions](https://github.com/Carthage/Carthage#installing-carthage), or if you have **Brew** OS X package manager you can install it by running:

```bash
$ brew install carthage
```

### Creating the framework

To create iOS framework targeting iOS 8+, go ahead and open Xcode. In the welcome window, select **Create a new Xcode project**. In the left pane under *iOS* section, select **Framework & Library** and select **Cocoa Touch Framework** and click **Next**

![Create new framework dialog](http://i.imgur.com/CmyqLvw.png)

In the next dialog set following:

* *Product name*: WeatherStationFactory
* *Organization name*: CityOS
* *Organization identifier*: com.cityos

Set language to Swift and uncheck *Include unit tests* option.

![New framework dialog](http://i.imgur.com/UngfBkk.png)

Click on **Next**, select location of the framework and click **Next**. This will create framework and add all appropriate files to file system.

### Integrating the CoreCityOS framework

After the initial framework setup, it's time to integrate CoreCityOS to our framework. As already mentioned, we are going to use [Carthage package manager](https://github.com/Carthage/Carthage).

Go to the project root folder, and create new file named **Cartfile**. [Cartfile](https://github.com/Carthage/Carthage/blob/master/Documentation/Artifacts.md#cartfile) is plain text file that Carthage uses to fetch frameworks you need. All you need to do is input Github link to the framework and you're done. You can use any text editor that you prefer to write contents of the file. 

Write following to the contents of the file:

```
# CoreCityOS framework
github "cityos/CoreCityOS" ~> 0.0.3
```

![Carfile opened in TextEdit](http://i.imgur.com/3Cyu2m7.png)

After saving the file navigate to the root folder where **Cartfile** is stored using the Terminal app.

```bash
$ cd ~/WeatherStationFactory
```

and run

```bash
$ carthage update
```

This command will fetch and build **CoreCityOS** framework, so that we can use the binaries in our framework. After successful building, CoreCityOS binary will be placed in `Cartage/Builds/iOS` folder under the name `CoreCityOS.framework`. This is the file we will need in our framework.

Drag the file `CoreCityOS.framework` to your Xcode project. Select **Copy items if needed** checkbox and click on **Finish** button. It should look like this:

![Xcode files](http://i.imgur.com/4Kf3rrT.png)

### Defining weather station device

After successful integration of the CoreCityOS framework, we can use it to define our weather station devices and weather station data.

Create new Swift file inside your project in Xcode and name it `WeatherStation.swift`. In this file we are going to define how our Weather Station device will look like.

We will use `DeviceType` protocol from the CoreCityOS framework that defines single IoT device. Reason why CoreCityOS is built around protocols is that you can just adopt protocol for any type of device and you get a ton of functionally through protocol extensions for free.

Add import statement first:

```swift
import CoreCityOS
```

Next up we will create new class `WeatherStation` that adopts `DeviceType` protocol.

```swift
public class WeatherStation: DeviceType {
    public var deviceData: DeviceData
    public var creationDate: NSDate?
    public var location: DeviceLocation?
    public var dataCollection: LiveDataCollectionType
    
    public init(deviceID: String, location: DeviceLocation? = nil) {
        self.deviceData = DeviceData(deviceID: deviceID)
        self.location = location
			// We will initialize dataCollection later
    }
}
```

Let's go through all properties. `DeviceType` protocol requires several properties:

* `deviceData` - this structure contains meta data about the device like device ID and etc.
* `creationDate` - `NSDate` property which contains info about device creation date. It's typically read from the device itself.
* `location` - device physical location
* `dataCollection` - defines what data this device reads. We will write it in the next step.

Also, we added init method with `deviceID` and optional `location` parameter. At this point, compiler will complain that we didn't initialized all of the properties (`dataCollection`), but don't worry about it. We are going to create and explain what data collection in the next step.

To recap, we have created `WeatherStation` device definition that is based on our hardware device. In the next step, we will define which data our weather station device can read.

### Defining weather station data collection

After we have created our device implementation with `DeviceType` framework it's time to define what data does our physical device records. We already defined the sensors that our device uses:

* Temperature
* Humidity
* PM10 (air pollution)

Now, only thing we need to do is to define them in code. Once again we will use some protocols from `CoreCityOS` framework so that we don't need to create everything from scratch.

Create new file in your project and name it `WeatherStationDataCollection.swift`. In this file we will create our data collection definition and add it to our `WeatherStation` class that we've created earlier. To do this, we'll use `LiveDataCollectionType` protocol.

In nutshell, `LiveDataCollectionType` is a protocol that wraps the data that is read by sensors in our device. Each data type (temperature, humidity and etc. ) is represented by `LiveDataType` protocol. To really understand this let us write following code:

Add import statement at the top of the file:

```swift
import CoreCityOS
```

Create new class named `WeatherStationDataCollection` and set it to adopt protocol `LiveDataCollectionType`

```swift
public class WeatherStationDataCollection: LiveDataCollectionType {

}
```

If you look at the definition of the protocol, you will see that it requires several properties:

* `deviceData` - Info about the device, same one that is used in `WeatherStation` class we've created earlier
* `creationDate` - When the data from the sensors is read
* `allReadings` - Array of `LiveDataType` instances. In our case that will be array that contains definitions of temperature, humidity and PM10.

Add following to class definition:

```swift
public var creationDate: NSDate
public var deviceData: DeviceData
public var allReadings: [LiveDataType] = []
```

You can see that `allReadings` array is empty at this point, but we will populate it in `init` method

Next up is definition of our data. Add following to the class definition:

```swift
public var temperature: LiveDataType {
	return LiveData(
	    dataType: .Temperature,
	    jsonKey: "temperature",
	    unitNotation: "C"
	)
}

public var humidity: LiveDataType {
	return LiveData(
	    dataType: .Humidity,
	    jsonKey: "humidity",
	    unitNotation: "%"
	)
}

public var pm10: LiveDataType {
	return LiveData(
	    dataType: .ParticleMatter10,
	    jsonKey: "pm",
	    unitNotation: "ug"
	)
}
```

> If you are not sure what the `dataType` property is, you can check out [official documentation](http://cityos.github.io/CoreCityOS/master/Structs/DataType.html). Essentially it is an `enum` definition of sensor data types. `CoreCityOS` framework comes with some predefined types, like the types that we use above.

If you look at the code, you can see that it's pretty straight forward. We're creating `LiveDataType` computed properties by using `LiveData` conforming structure from the `CoreCityOS` framework. We supply each one with the data type, JSON key we'll use later and unit notation. Now, all we need to do is to populate `allReadings` property.

Add init method at the end of class definition:

```swift
public init(deviceID: String) {
    creationDate = NSDate()
    deviceData = DeviceData(deviceID: deviceID)
    allReadings = [
        temperature,
        humidity,
        pm10
    ]
}
```

Our `WeatherStationDataCollection` is now ready to be linked with `WeatherStation` class. Go to the end of the `init` method inside `WeatherStation` class, and add following code.

```swift
public init(deviceID: String, location: DeviceLocation? = nil) {
		// ...
		// ...
		dataCollection = WeatherStationDataCollection(deviceID: deviceID)
}
```

Framework should build successfully now.
We are now done with defining our weather station device and data that it reads from sensors. Last part of the puzzle is fetching the data from your device API and serializing it to classes we've created

### Pulling data from server

In order to request data from the server we need to have communication with server and a way to request the data and get JSON response. Most common way is to use any Swift API for that. Some people may prefer `Alamofire` and `SwifyJSON` stack and other people may go to the Foundation implementation with classes `NSURLSession` and `NSJSONSerialization`. Since that is the case, for the purpose of this tutorial we will download this small file named [`Backend.swift`](https://raw.githubusercontent.com/cityos/WeatherStationFactory/master/WeatherStationFactory/Backend.swift) that contains logic that enables fetching data from the server. Reason we are doing this is because you may all kinds of APIs and we can't cover them all, so we're going to use data from example server that our device uses.

Go ahead and download [`Backend.swift`](https://raw.githubusercontent.com/cityos/WeatherStationFactory/master/WeatherStationFactory/Backend.swift) and add it to your project.

Inside this file, there is a single class named `Backend` with one method:

```swift
func requestLatestData(completion: (data: Dictionary<String, AnyObject>?, error: ErrorType?) -> ())
```

This function will return `Dictionary` object with our data in format:

```swift
(
	"temperature" : 25.9,
	"humidity" : 56,
	"pm10" : 360,
	"id" : "ghwo-251"
)
```

> Note that this is the data that we get when we use `print` function for the returned dictionary object

All of the values from the dictionary are self explanatory (`id` is ID of the device that sent the data). Now we have our data inside `Dictionary` object and all that's left to do is to connect this data with our classes `WeatherStation` and `WeatherStationDataCollection`.

### Creating `WeatherStationFactory` class

To correctly use the data that is returned from the `Backend` class, we will create new class and name if `WeatherStationFactory`.

Go ahead and create new file named `WeatherStationFactory`. At the top of the file add import statement

```swift
import CoreCityOS
```

For this class, we are going to use `FactoryType` protocol from `CoreCityOS` framework. This protocol defines where you can get data is retrieved from the server. Reason for this is that any user of the framework knows that any structure conforming protocol `FactoryType` can be used to retrieve data. You don't need to use `FactoryType`, but it is a standard.

Protocol `FactoryType` has only one requirement and that is the singleton object named `sharedInstance`

> Singleton is an object which is instantiated exactly once.
