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
public func requestData(completion: HydroponicsFactroryResult) {
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

Now you can name the tabs to `Data` and `Notifications` and set the icons. 

Let's move to the `Data` controllers. As we said earlier it contains one `UISegmentedController` and one `UIScrollView`.

![](http://i.imgur.com/Mfm1Qxz.png)

We can setup UI like this:

```
[UINavigationBar] (already present)

[UIView] (container view for the segmented control)
 └[UISegmentedControl]

[UIView] (main view controller view)
 └[UIScrollView]
```

Now create the layout. You can use simple constraints to pin the views to the edges of the view controller view and center constraints to center segmented control.

After that add four segments for the `UISegmentedControl`: Minutes, Hours, Days and Months.

Download [**ChartView.swift**](https://github.com/cityos/hydroponics-ios/blob/master/Hydroponics/ChartView.swift) and add it to the project. It contains some predefined logic on how to draw graphs.

One last thing we need to create is the `SensorView` with graph and data reading value that you can see in the app screenshot.
We will create it separately as single `.xib` file so that we can use it later through programmatically. 

Right click on the application folder in Xcode and click on `New file`. Select iOS > User Interface on the left and click on **View**. This is a basic .xib view. Click on Next and name it `SensorView`. Next, add new empty Swift file and name it `SensorView.swift`. In that file we will create `SensorView` class that will manage the actual SensorView.

When designing SensorView use `UIStackView` so you don't need to deal with constrains. This is a basic setup you need to achieve:

![Sensor view](http://i.imgur.com/uNw2UZz.png)

Later, we will create outlets from the `Sensor Label`, `Current Value label`, `Chart Container View` and `Time Ago Label`. `Chart Container View` is a simple empty `UIView` that will be used as a container view for the graph. 

**IMPORTANT** 
You need to set the class of the `Chart Container View` to the `ChartView`.

Now move to `SensorView.swift` and add following code:

```swift
import UIKit
import CoreCityOS

class SensorView: UIView {
    
    //MARK: Outlets
    @IBOutlet weak var sensorLabel: UILabel!
    @IBOutlet weak var chartContainerView: ChartView!
    @IBOutlet weak var currentValueLabel: UILabel!
    @IBOutlet weak var timeAgoLabel: UILabel!
    
    //MARK: Sensor data
    var data: LiveDataType! {
        didSet {
            setupWithData()
        }
    }
    
    func setupWithData() {
        if let data = data
            setupWithDataPoints(dataPoints: data.dataPoints)
            if let currentDataPoint = data.dataPoints.last {
                let valueString = NSString(format: "%.2f", currentDataPoint.value)
                self.currentValueLabel.text = valueString as String
            }
        }
    }
    
    //MARK: Functions and properties
    func setupWithDataPoints(dataPoints points: [DataPoint]) {
        chartContainerView.dataPoints = points
        chartContainerView.commonInit()
        chartContainerView.setupChart()
        chartContainerView.render()
    }
}
```

We are creating 4 outlets for the views that we defined earlier. We have `data` property that is used to present data in chart and other labels. 

>> LiveDataType is used to store data in form of data points. Refer to the CoreCityOS documentation for more details.

This is all the code that we need for the Sensor view. Now switch back to the `SensorView.xib` and set the main view class to `SensorView` and connect outlets. Now we have connected the view with it's class and we are ready to continue. 

### 6.  `SensorReadingsViewController`


Create new `SensorReadingsViewController` inside new `SensorReadingsViewController.swift` file. This controller will be in charge of retrieving data from `HydroponicsFactory` and presenting it in `SensorView` views that we created earlier. Add `import CoreCityOS` at the top of the file. 

Add following code to the class:

```swift
@IBOutlet weak var timeRangeSegmentedControl: UISegmentedControl!
@IBOutlet weak var scrollView: UIScrollView!
    
var refreshControl = UIRefreshControl()
var containerView: UIView!
```

`timeRangeSegmentedControl` is a outlet to the segmented control with time range that we created when we created storyboard setup and `scrollView` is also a outlet view that will container sensor views. In addition we have a `UIRefreshControl` that will handle refreshing on pull and `containerView` for the scroll view.

Lets setup sensor views now. Add following to to the controller:

```swift
var temperatureView: SensorView!
var humidityView: SensorView!
var lightView: SensorView!
var soilHumidityView: SensorView!


func setupSensorViews() {
    temperatureView = UIView.loadFromNibNamed("SensorView") as! SensorView
    humidityView = UIView.loadFromNibNamed("SensorView") as! SensorView
    lightView = UIView.loadFromNibNamed("SensorView") as! SensorView
    soilHumidityView = UIView.loadFromNibNamed("SensorView") as! SensorView
    
    temperatureView.translatesAutoresizingMaskIntoConstraints = false
    humidityView.translatesAutoresizingMaskIntoConstraints = false
    lightView.translatesAutoresizingMaskIntoConstraints = false
    soilHumidityView.translatesAutoresizingMaskIntoConstraints = false
    
    temperatureView.layer.cornerRadius = 6
    humidityView.layer.cornerRadius = 6
    lightView.layer.cornerRadius = 6
    soilHumidityView.layer.cornerRadius = 6
    
    temperatureView.sensorLabel.text = "Temperature"
    humidityView.sensorLabel.text = "Humidity"
    lightView.sensorLabel.text = "Light"
    soilHumidityView.sensorLabel.text = "Soil humidity"
}
```

We are creating four instances of the `SensorView` class and we initiate them from the `.xib`. Rest of the code is mostly self explanatory. Only thing you may not know about is what `.translatesAutoresizingMaskIntoConstraints = false` does. It essentially prepares views for Auto Layout. If we haven't done this, we would get tons of errors on runtime that will be hard to debug.

Onto the scroll view setup. Add following function to the controller:

```swift
func setupScrollView() {
    self.scrollView.layoutIfNeeded()
    let frame = CGRect(x: 0, y: 0, width: scrollView.frame.width, height: 992)
    containerView = UIView(frame: frame)
    
    scrollView.contentSize = containerView.bounds.size
    scrollView.addSubview(containerView)
    
    containerView.addSubview(temperatureView)
    containerView.addSubview(humidityView)
    containerView.addSubview(lightView)
    containerView.addSubview(soilHumidityView)
    
    var views = [String:AnyObject]()
    views.updateValue(temperatureView, forKey: "temp")
    views.updateValue(humidityView, forKey: "hum")
    views.updateValue(lightView, forKey: "light")
    views.updateValue(soilHumidityView, forKey: "soil")
    
    var constraints = [NSLayoutConstraint]()
    constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[temp]-|", options: [], metrics: nil, views: views))
    constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[hum]-|", options: [], metrics: nil, views: views))
    constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[light]-|", options: [], metrics: nil, views: views))
    constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat("H:|-[soil]-|", options: [], metrics: nil, views: views))
    
    constraints.appendContentsOf(NSLayoutConstraint.constraintsWithVisualFormat(
            "V:|[temp(240)]-[hum(240)]-[light(240)]-[soil(240)]",
            options: [],
            metrics: nil,
            views: views
        )
    )
    
    NSLayoutConstraint.activateConstraints(constraints)
}
```

This piece of code will setup the scroll view container view and the sensor views. It will also activate constraints created with Visual Format Language.

Let's move to the actual data fetching. We can write one simple function to fetch the data from LightFactory and present the data in sensor views.


```swift
func retrieveData() {
    HydroponicsFactory.retrieveData() {
        devices, error in
        if error == nil {
            if let device = devices?.first {
                updateViewsWithDevice(device)
            }
        }
    }
}

func updateViewsWithDevice(device: DeviceType) {
    dispatch_async(dispatch_get_main_queue(), {
        self.temperatureView.data = device.dataCollection[.Temperature]!
        self.humidityView.data = device.dataCollection[.Humidity]!
        self.lightView.data = device.dataCollection[.NaturalLight]!
        self.soilHumidityView.data = device.dataCollection[.SoilHumidity]!
    })
}
```

Now go to the `viewDidLoad()` method. In this function we will call all of the functions that we created earlier. Also we will setup a timer that will call `retrieveData()` function every 5 seconds.

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    
    setupSensorViews()
    setupScrollView()
    retrieveData()
    
    _ = NSTimer.scheduledTimerWithTimeInterval(5, target: self, selector: #selector(retrieveData(_:)), userInfo: nil, repeats: true)
}
```

Set the controller class inside the storyboard and connect the outlets. 

Now try and build the app. It should build fine and you should be presented with the latest sensor readings data. You can now add a more functionality if you need it. Core thing you should know is how to use `HydroponicsFactory` and how to use `CoreCityOS` protocols.

I hope that technologies and techniques that we used here were not too difficult to master. You can always also refer to the [official Github repository](https://github.com/cityos/hydroponics-ios) of the application and the [CoreCityOS documentation](http://cityos.github.io/CoreCityOS/master/).
