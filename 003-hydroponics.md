# Hydroponics iOS application tutorial

In this tutorial you will learn how to create iOS app that can present and interact with data retrieved from CityOS Hydroponics. 

Application will consist of two main features:
1. Live data view with graphs and current readings
2. Notifications view with notifications that are sent from Hydroponics

### 1. Requirements
* Xcode 7.3
* Carthage

### 2. App setup
Navigate to Xcode and create new Single View iOS application. Name it CityOS Hydroponics, set bundle identifier of your choice and set the application location.

Next up, we’ll need to setup the dependencies that will be used in our application.

Create new file in the application root folder and name it Cartfile. In this file we will specify which dependencies we need for our application. Paste following into the file:

```
github "cityos/CoreCityOS"
github "danielgindi/Charts" ~> 2.2.4
github "realm/realm-cocoa"
```

* CoreCityOS - reading and presenting data from Hydroponics system to the application
* Charts - presenting data in charts
* Realm - caching for the notifications

Run `carthage update --platform iOS` to fetch and build these frameworks from Github. 

Now we can link built frameworks with our application. Go to Xcode, click on the project and select **General** tab. After you scroll to the bottom you will see two sections *Embedded Binaries* and *Linked Frameworks and Libraries*.

Click on the little **+** button in the Embedded Binaries section. This will open up a dialog where you can select new frameworks that are embedded into the application. Click on **Add other** and navigate to *Carthage/Builds/iOS* folder inside application folder. In this folder you will find built frameworks that we specified in *Cartfile*. Select following frameworks and click Add:

```
* CoreCityOS.framework
* Realm.framework
* RealmSwift.framework
* Charts.framework
```

This will setup everything we need to start coding the application. 

### 3. CoreCityOS

To fetch data from Hydroponics and present it inside application  in usable form we will use **CoreCityOS** framework.

If you don't know what CoreCityOS is you can refer to these tutorials since that is not scope of this tutorial.

* [Introduction to CoreCityOS](http://cityos.io/tutorial/1914/CoreCityOS-Framework-Tutorial)
* [Create IOT framework with CoreCityOS](http://cityos.io/tutorial/1924/Create-framework-for-communication-with-IoT-devices)
* [Using CoreCityOS with Flowthings backend](http://cityos.io/tutorial/1915/Using-CoreCityOS-with-Flowthings-backend)

Our Hydroponics systems reads following data types:

* Temperature
* Humidity
* Light
* Float
* Soil humidity

For any application that uses CoreCityOS framework to interact with IOT devices (in this case Hydroponics) we need to adopt several protocols. 

Create new file and name it **HydroponicsDataCollection.swift**. In this file we will create the data collection that is retrieved from the Hydroponics. At the top of the file add following line:

```swift
import CoreCityOS
```

Since some data types such as temperature and humidity are already present in CoreCityOS we'll need to create those which  are not.

```swift
extension DataType {
    public static let Float = DataType(dataIdentifier: "Float")
    public static let SoilHumidity = DataType(dataIdentifier: "Soil Humidity")
}
```

Now we have the necessary setup for our data collection.

```swift
public class HydroponicsDataCollection: LiveDataCollectionType, CustomStringConvertible {
    public var deviceData: DeviceData
    public var creationDate: NSDate
    public var allReadings: [LiveDataType] = []
    
    var temperature: LiveDataType {
        return LiveData(
            dataType: .Temperature,
            jsonKey: "temp",
            unitNotation: "C"
        )
    }
    
    var humidity: LiveDataType {
        return LiveData(
            dataType: .Humidity,
            jsonKey: "humid",
            unitNotation: "%"
        )
    }
    
    var light: LiveDataType {
        return LiveData(
            dataType: .NaturalLight,
            jsonKey: "Light",
            unitNotation: "lux"
        )
    }
    
    var float: LiveDataType {
        return LiveData(
            dataType: .Float,
            jsonKey: "FlowSw1",
            unitNotation: ""
        )
    }
    
    var soilHumidity: LiveDataType {
        return LiveData(
            dataType: .SoilHumidity,
            jsonKey: "Soil",
            unitNotation: "%"
        )
    }
    
    public init(readingID: String) {
        deviceData = DeviceData(deviceID: readingID)
        creationDate = NSDate()
        allReadings = [
            temperature,
            humidity,
            light,
            float,
            soilHumidity
        ]
    }
}
```

Class definition is pretty straight-forward. We adopted `LiveDataCollectionType` protocol from CoreCityOS and defined data types that will be used using `LiveDataType` protocols.

Next up we will define actual Hydroponics device using CoreCityOS `DeviceType` protocol. 

Create new file and name it **Hydroponics.swift**. Add following to the file:

```swift
import CoreCityOS

public class Hydroponics: DeviceType, CustomStringConvertible {
    public var creationDate: NSDate?
    public var dataCollection: LiveDataCollectionType
    public var location: DeviceLocation?
    public var deviceData: DeviceData
    
    public init(deviceID: String) {
        deviceData = DeviceData(deviceID: deviceID)
        dataCollection = HydroponicsDataCollection(readingID: deviceID)
        creationDate = NSDate()
    }
}
```

Again, setup is pretty simple. We created new class named Hydroponics and adopted `DeviceType` protocol and fulfilled necessary requirements such as data collection and device meta data. 

### 4. Flowthings backend 

Hydroponics reads data from the sensors and uses wifi to send the data to Flowthings backend system. In our application we will fetch the data from the Flowthings backend using `NSURLSession` APIs from iOS. To get started download this file and add it to your application.

[**Flowthings.swift**](https://github.com/cityos/hydroponics-ios/blob/master/HydroponicsFactory/Flowthings.swift)

Now we can fetch the data from the backend. First create new file and name it **HydroponicsFactory.swift**. Class in this file will be in charge of fetching the raw data from Flowthings and returning `HydroponicsDevice` and `HydroponicsDataCollection` objects that we have created earlier.

Add following class declaration to the file:

```swift
import CoreCityOS

public typealias HydroponicsFactroryResult = (devices: [DeviceType]?, error: ErrorType?) -> Void
public typealias NotificationPayload = String

final public class HydroponicsFactory: FactoryType {
    public static let sharedInstance = HydroponicsFactory()
    
}

```
    
    
We defined some typealiases for some structures we will use several times, adopted `FactoryType` protocol from CoreCityOS and created singleton object.

Now we can write the method for retrieving data from the backend server. Add following function to the `HydroponicsFactory` class:

```swift
public func requestDataInTimeRange(completion: HydroponicsFactroryResult) {
        Flowthings.requestLatestData {
            data in
            if let error = data.error {
                completion(devices: nil, error: error)
            } else {
                if let device = data.device {
                    completion(devices: [device], error: nil)
                }
            }
        }
    }
```

This function will request and return latest data sent to the Flowthings backend. Later in the application we will run this function each 10 seconds to update to latest readings.

### 5. Application UI

To understand views and controllers that wee need to create, lets first look at the application layout:

![CityOS GRO](https://camo.githubusercontent.com/bbef76cbf53d8c5a0ff2180776fac508b1c91aa9/687474703a2f2f692e696d6775722e636f6d2f3941335a365a422e706e67)

And this is storyboard setup we want to achieve:

![Storyboard layout](http://i.imgur.com/CxgCTkj.png)

We can see that it's made up of one `UITabBarController` with two embedded `UINavigationController` controllers, one for the data readings and one for the notifications.

Data controller is simple `UIViewController` with `UISegmentedControl` at the top and the `UIScrollView` filling the rest of the view.

Notifications controller is simple `UITableViewController`.

Go to the **Main.storyboard** and delete any existing controller.  Drag one `UITabBarController` from the object library to the storyboard and check **Is Initial view controller**. 

After that drag one `UIViewController` (for data readings) and one `UITableViewController` (for notifications) to the storyboard.

Select them both and click on **Editor > Embed > Navigation Controller**. This will wrap them inside `UINavigationController`.

Now select `UITabBarController` that was added previously and `ctrl` drag to each of the navigation controllers. When you release the key select `view controllers`. This will create the layout we described earlier.

Don't worry 
