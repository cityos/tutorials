# Using CoreCityOS with Flowthings backend

After you have your framework setup, you will need to fetch the data from some backend system. In this tutorial we’re going to look at [Flowthings](http://flowthings.io). We’re going to create simple framework that will read raw data from one parking sensor located on Flowthings, and then parse it into usable Swift objects. Complete source code is available on [Github](https://github.com/cityos/ParkingFactory).

> This tutorial does not cover [CoreCityOS](https://github.com/cityos/CoreCityOS) framework. If you are not familiar with it, see this tutorial.

We’ll need to create the `LiveDataCollection` structure first. Add new file named `ParkingDataCollection` and add following code:

```swift
import CoreCityOS

extension DataType {
    static let ParkingSpace = DataType(dataIdentifier: "Parking space")
}

public class ParkingDataCollection: LiveDataCollectionType {
    public var deviceData = DeviceData(deviceID: "test")
    public var creationDate = NSDate()
    public var allReadings = [LiveDataType]()
    
    public var parkingSpace: LiveDataType {
        return LiveData(dataType: .ParkingSpace,
            jsonKey: "16",
            unitNotation: ""
        )
    }
    
    public init() {
        self.allReadings = [
            parkingSpace
        ]
    }
}
```

This code should be familiar with you if you’ve read CoreCityOS tutorial.

After this step we are ready to fetch data from Flowthings. We fetch all data trough `Factory` classes that adopt `FactoryType` protocol.

Create new file, name it `ParkingFactory` and add following definition:

```swift
import CoreCityOS
import Flowthings
import PromiseKit

final public  class ParkingFactory: FactoryType {
    
    public static let sharedInstance = ParkingFactory()
    
		/// Flow ID
    internal let inFlowID = "f562e8c4f68056d244d594ce6"

    // Access Flowthings data through this object
    internal var api: FTAPI = {
        return FTAPI(
            accountID: “accountID”,
            tokenID: “token”
        )
    }()
```

We import frameworks we need first, and then create `ParkingFactory` class and it’s singleton object. Next up, we create inFlowID string that contains ID of the Flowthings flow where our data is located.

To access the Flowthings API, we use `FTAPI` object from Flowthings framework that we populate with `accountID` and `tokenID`.

After we’ve setup this, we add function definition to the class:

```swift
public func retrieveLatestData(completion: (data: LiveDataCollectionType?, error: ErrorType?) -> ()) {
}
```

We will use this function to asnychronusly retrieve latest live data through completion closure.

To get the lastest drop from the flow were our data is located, we can use `find` api where we can set `limit` property to 1.

```swift
var findLatestRequest = FindParams()
        findLatestRequest.limit = 1
        findLatestRequest.hints = false
        findLatestRequest.filter = "elems.12 == \"wbeg:wkej:1nrv:nfi2\""
```

After we create `FindParams` object we can call the API.

```swift
api.drop.find(inFlowID, findParams: findLatestRequest)
            .then {
                body -> () in
                do {
                    // Serialize data here and call completion when finished
                } catch {
                    // If there is error throw it
                    throw error
                }
            }.error {
                error in
                completion(data: nil, error: error)
        }
```

Calling this API will return data through `body` JSON object. We will need to serialize that object to `ParkingDataCollection` to call the completion.

To serialize the JSON data, create new file `Serializer.swift` and add following code:

```swift
class Serializer {
    
    /// Serializes drop json data
    class func serializeLiveData(jsonBody json: JSON) throws -> [LiveDataCollectionType] {
        // Check if there is valid data. Since drops should always present as array, we can check for that
        guard let array = json["body"].array else {
            throw ParkingError.InvalidData
        }
        
        let range = 0..<array.count
        
        // This instance will be returned when we serialize the data
        var parkingData = [LiveDataCollectionType]()
        
        for i in range {
            
            // Create instance of ParkingDataCollection
            var dataCollection = ParkingDataCollection()
            
            // Get timestamp from json body
            let creationDateStamp = json["body"][i]["creationDate"].double!
            
            // Create elems dictionary
            let elems = json["body"][i]["elems"].dictionary!
            
            // Read occupied state from the elems dictionary
            let occupied = elems[dataCollection.parkingSpace.jsonKey] as! Double
            
            // Create data point for the occupied value
            let occupiedDataPoint = DataPoint(value: occupied)
            
            // Set creation date
            dataCollection.creationDate = NSDate(timeIntervalSince1970: creationDateStamp / 1000.0)
            
            // Add occupied data point to the dataCollection
            dataCollection[.ParkingSpace]?.addDataPoint(occupiedDataPoint)
            
            // Append dataCollection to the global data object we will return
            parkingData.append(dataCollection)
            
        }
        
        return parkingData
    }
}
```

First, we create function definition. Function `serializeLiveData` will take JSON argument and return array of `LiveDataCollectionType` instances. Note that function can throw, so we can mark it that it can throw.

If you look up the raw JSON data that API returned, you can see that it’s located inside json[“body”] array, so we can check that there is json[“array”] object indeed. If there isn’t, we can throw error.

Next up, we can loop through that array and read all info we need. Notice how we use `DataPoint` to record the state of the parking slot.

After we record everything we need, we can return to the `ParkingFactory` class and upadate our API call.

```swift
.then {
                body -> () in
                do {
                    // Serialize data here
                    let data = try Serializer.serializeLiveData(jsonBody: body)
                    completion(data: data.first, error: nil)
                } catch {
                    // If there is error throw it
                    throw error
                }
```

This will call `Serializer` method when JSON data is recived, then it will try to serialize it. If that operation is successfull, we can call the completion. If it isn’t we throw the error.

You can scale this example in any way you want, but the principle is the same. Create factory methods and serialize JSON  responses in `Serializer`.

