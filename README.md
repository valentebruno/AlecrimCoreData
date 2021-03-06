![AlecrimCoreData](https://raw.githubusercontent.com/Alecrim/AlecrimCoreData/master/AlecrimCoreData.png)

[![Language: Swift](https://img.shields.io/badge/lang-Swift-orange.svg?style=flat)](https://developer.apple.com/swift/)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg?style=flat)](https://raw.githubusercontent.com/Alecrim/AlecrimCoreData/develop/LICENSE)
[![CocoaPods](https://img.shields.io/cocoapods/v/AlecrimCoreData.svg?style=flat)](http://cocoapods.org)
[![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)
[![Forks](https://img.shields.io/github/forks/Alecrim/AlecrimCoreData.svg?style=flat)](https://github.com/Alecrim/AlecrimCoreData/network)
[![Stars](https://img.shields.io/github/stars/Alecrim/AlecrimCoreData.svg?style=flat)](https://github.com/Alecrim/AlecrimCoreData/stargazers)

AlecrimCoreData is a framework to easily access Core Data objects in Swift.

## Getting Started

### Data Context

To use AlecrimCoreData you will need to create a inherited class from `AlecrimCoreData.Context` and declare a property or method for each entity in your data context like the example below:

```swift
import AlecrimCoreData

let dataContext = DataContext()!

class DataContext: Context {
	var people:      Table<Person>     { return Table<Person>(context: self) }
	var departments: Table<Department> { return Table<Department>(context: self) }
}
```

It's important that properties (or methods) always return a _new_ instance of a `AlecrimCoreData.Table` class.

### Entities

It's assumed that all entity classes was already created and added to the project. In the above section example there are two entities: `Person` and `Department`.

### Optional Code Generation Tool

You can write managed object classes by hand or generate them using Xcode. Now you can also use ACDGen. ;-)

ACDGen app is a Core Data entity class generator made with AlecrimCoreData in mind. It is completely optional, but since it can also generate attribute class members for use in closure parameters, the experience using AlecrimCoreData is greatly improved. You can download it for free from http://opensource.alecrim.com/AlecrimCoreData/ACDGen.zip.

## Usage

### Fetching

#### Basic Fetching

Say you have an Entity called Person, related to a Department (as seen in various Apple Core Data documentation). To get all of the Person entities as an array, use the following methods:

```swift
for person in dataContext.people {
	println(person.firstName)
}
```

You can also skip some results:

```swift
let people = dataContext.people.skip(3)
```

Or take only some results:

```swift
let people = dataContext.people.skip(3).take(7)
```

Or, to return the results sorted by a property:

```swift
let peopleSorted = dataContext.people.orderBy({ $0.lastName })
```

Or, to return the results sorted by multiple properties:

```swift
let peopleSorted = dataContext.people.orderBy({ $0.lastName }).thenBy({ $0.firstName })

// OR

let peopleSorted = dataContext.people.sortBy("lastName,firstName")
```

Or, to return the results sorted by multiple properties with different attributes:

```swift
let peopleSorted = dataContext.people.orderByDescending({ $0.lastName }).thenByAscending({ $0.firstName })

// OR

let peopleSorted = dataContext.people.sortBy("lastName:0,firstName:1")

// OR

let peopleSorted = dataContext.people.sortBy("lastName:0:[cd],firstName:1:[cd]")
```

If you have a unique way of retrieving a single object from your data store (such as via an identifier), you can use the following code:

```swift
if let person = dataContext.people.first({ $0.identifier == 123 }) {
	println(person.name)
}
```

#### Advanced Fetching

If you want to be more specific with your search, you can use filter predicates:

```swift
let itemsPerPage = 10  

for pageNumber in 0..<5 {
	println("Page: \(pageNumber)")
	
	let peopleInCurrentPage = dataContext.people
	    .filter({ $0.department << [dept1, dept2] })
	    .orderBy({ $0.firstName })
	    .thenBy({ $0.lastName })
	    .skip(pageNumber * itemsPerPage)
	    .take(itemsPerPage)
	
	for person in peopleInCurrentPage {
	    println("\(person.firstName) \(person.lastName) - \(person.department.name)")
	}
}
```

#### Asynchronous Fetching

You can also fetch entities asynchronously and get the results later on main thread:

```swift
let progress = dataContext.people.fetchAsync { fetchedEntities, error in
    if let entities = fetchedEntities {
        // ...
    }
}
```

#### Returning an Array

The data is actually fetched from Persistent Store only when `toArray()` is explicitly or implicitly called. So you can combine and chain other methods before this.

```swift
let peopleArray = dataContext.people.toArray()

// OR

let peopleArray = dataContext.people.sortBy("firstName,lastName").toArray()

// OR

let theSmiths = dataContext.people.filter({ $0.lastName == "Smith" }).orderBy({ $0.firstName })
let count = theSmiths.count()
let array = theSmiths.toArray()

// OR

for person in dataContext.people.sortBy("firstName,lastName") {
	// .toArray() is called implicitly when enumerating
}
```

#### Converting to other class types

Call the `to...` method in the end of chain.

```swift
let fetchRequest = dataContext.people.toFetchRequest()
let arrayController = dataContext.people.toArrayController() // OS X only
let fetchedResultsController = dataContext.people.toFetchedResultsController() // iOS only
```

#### Find the number of entities

You can also perform a count of the entities in your Persistent Store:

```swift
let count = dataContext.people.filter({ $0.lastName == "Smith" }).count()
```

### Creating new Entities

When you need to create a new instance of an Entity, use:

```swift
let person = dataContext.people.createEntity()
```

You can also create or get first existing entity matching the criteria. If the entity does not exist, a new one is created and the specified attribute is assigned from the searched value automatically.

```swift
let person = dataContext.people.firstOrCreated({ $ 0.identifier == 123 })
```

### Deleting Entities

To delete a single entity:

```swift
if let person = dataContext.people.first({ $0.identifier == 123 }) {
	dataContext.people.deleteEntity(person)
}
```

## Saving

You can save the data context in the end, after all changes were made.

```swift
let person = dataContext.people.firstOrCreated({ $0.identifier == 9 })
person.firstName = "Christopher"
person.lastName = "Eccleston"
person.additionalInfo = "The best Doctor ever!"

// get success and error
let (success, error) = dataContext.save()

if success {
	// ...
}
else {
	println(error)
}
```

### Threading

You can fetch and save entities in background calling a global function that creates a new data context instance for this:

```swift
// assuming that this department is saved and exists...
let department = dataContext.departments.first({ $0.identifier == 100 })!

// the closure below will run in a background context queue
performInBackground(dataContext) { backgroundDataContext in
	if let person = backgroundDataContext.people.first({ $0.identifier == 321 }) {
	    // must bring to backgroundDataContext
	    person.department = department.inContext(backgroundDataContext)! 
	    person.otherData = "Other Data"
	}
	
	backgroundDataContext.save()
}
```

## Advanced Configuration

You can use `ContextOptions` class for a custom configuration.

### iCloud Core Data sync

Example configuration when using iCloud Core Data sync.

```swift
import Foundation
import AlecrimCoreData

class DataContext: AlecrimCoreData.Context {

	var people:      Table<PersonEntity>     { return Table<PersonEntity>(context: self) }
	var departments: Table<DepartmentEntity> { return Table<DepartmentEntity>(context: self) }

	// MARK - custom init

	init?() {
		let contextOptions = ContextOptions(stackType: .SQLite)

        // only needed if model is not in main bundle
		contextOptions.modelBundle = NSBundle(forClass: DataContext.self)

        // only needed if entity class names are different from entity names
		contextOptions.entityClassNameSufix = "Entity"

        // enable iCloud Core Data sync
		contextOptions.ubiquityEnabled = true

        // only needed if the identifier is different from default identifier
		contextOptions.ubiquitousContainerIdentifier = "iCloud.com.company.MyApp"

		// call super
		super.init(contextOptions: contextOptions)
	}

}
```

### Ensembles

Example configuration when using [Ensembles](http://www.ensembles.io).

```swift
import Foundation
import AlecrimCoreData
import Ensembles

class DataContext: AlecrimCoreData.Context {

	var people:      Table<PersonEntity>     { return Table<PersonEntity>(context: self) }
	var departments: Table<DepartmentEntity> { return Table<DepartmentEntity>(context: self) }

	// MARK: - ensembles support

	var cloudFileSystem: CDEICloudFileSystem! = nil
	var ensemble: CDEPersistentStoreEnsemble! = nil
	var ensembleDelegate: EnsembleDelegate! = nil

	var obs1: AnyObject! = nil
	var obs2: AnyObject! = nil

	// MARK - custom init

	init?() {
		let contextOptions = ContextOptions(stackType: .SQLite)

        // only needed if model is not in main bundle
        contextOptions.modelBundle = NSBundle(forClass: DataContext.self)

        // only needed if entity class names are different from entity names
        contextOptions.entityClassNameSufix = "Entity"

        // call super
		super.init(contextOptions: contextOptions)

		//
		self.cloudFileSystem = CDEICloudFileSystem(ubiquityContainerIdentifier: "iCloud.com.company.MyApp")
		self.ensemble = CDEPersistentStoreEnsemble(
			ensembleIdentifier: "EnsembleStore",
			persistentStoreURL: contextOptions.persistentStoreURL,
			managedObjectModelURL: contextOptions.managedObjectModelURL,
			cloudFileSystem: self.cloudFileSystem
		)

		self.ensembleDelegate = EnsembleDelegate(managedObjectContext: self.managedObjectContext)
		ensemble.delegate = self.ensembleDelegate

        //
		self.obs1 = NSNotificationCenter.defaultCenter().addObserverForName(CDEMonitoredManagedObjectContextDidSaveNotification, object: nil, queue: nil) { [unowned self] notification in
			self.sync()
		}

		self.obs2 = NSNotificationCenter.defaultCenter().addObserverForName(CDEICloudFileSystemDidDownloadFilesNotification, object: nil, queue: nil) { [unowned self] notification in
			self.sync()
		}

		//
		self.sync()
	}

	deinit {
		NSNotificationCenter.defaultCenter().removeObserver(self.obs1)
		NSNotificationCenter.defaultCenter().removeObserver(self.obs2)
	}

	func sync() {
		if self.ensemble.leeched {
			self.ensemble.mergeWithCompletion { error in
				if let error = error {
					println(error)
				}
			}
		}
		else {
			self.ensemble.leechPersistentStoreWithCompletion { error in
				if let error = error {
					println(error)
				}
			}
		}
	}
}

class EnsembleDelegate: NSObject, CDEPersistentStoreEnsembleDelegate  {

	let managedObjectContext: NSManagedObjectContext

	init(managedObjectContext: NSManagedObjectContext) {
		self.managedObjectContext = managedObjectContext
	}

	@objc func persistentStoreEnsemble(ensemble: CDEPersistentStoreEnsemble, didSaveMergeChangesWithNotification notification: NSNotification) {
		var currentContext: NSManagedObjectContext? = self.managedObjectContext

		while let c = currentContext {
			c.performBlockAndWait {
				c.mergeChangesFromContextDidSaveNotification(notification)
			}

			currentContext = currentContext?.parentContext
    	}
	}

	@objc func persistentStoreEnsemble(ensemble: CDEPersistentStoreEnsemble, globalIdentifiersForManagedObjects objects: [AnyObject]) -> [AnyObject] {
		return (objects as NSArray).valueForKeyPath("uniqueIdentifier") as! [AnyObject]
	}

}

```


## Using

### Minimum Requirements

- Xcode 6.3
- iOS 8.0 / OS X 10.10

### Installation

#### CocoaPods

[CocoaPods](http://cocoapods.org) is a dependency manager for Cocoa projects.

CocoaPods 0.36 adds supports for Swift and embedded frameworks. You can install it with the following command:

```bash
$ gem install cocoapods
```

To integrate AlecrimCoreData into your Xcode project using CocoaPods, specify it in your `Podfile`:

```ruby
source 'https://github.com/CocoaPods/Specs.git'
platform :ios, '8.0'
use_frameworks!

pod 'AlecrimCoreData', '~> 3.0-beta.4'
```

Then, run the following command:

```bash
$ pod install
```

#### Carthage

Carthage is a decentralized dependency manager that automates the process of adding frameworks to your Cocoa application.

You can install Carthage with [Homebrew](http://brew.sh/) using the following command:

```bash
$ brew update
$ brew install carthage
```

To integrate AlecrimCoreData into your Xcode project using Carthage, specify it in your `Cartfile`:

```ogdl
github "Alecrim/AlecrimCoreData" >= 2.0
```

#### Manually

You can add AlecrimCoreData as a git submodule, drag the `AlecrimCoreData.xcodeproj` file into your Xcode project and add the framework product as an embedded binary in your application target.

### Branches and Contribution

- master - The production branch. Clone or fork this repository for the latest copy.
- develop - The active development branch. [Pull requests](https://help.github.com/articles/creating-a-pull-request) should be directed to this branch.

If you want to contribute, please feel free to fork the repository and send pull requests with your fixes, suggestions and additions. :-)

### Inspired By

- [MagicalRecord](https://github.com/magicalpanda/MagicalRecord)
- [QueryKit](https://github.com/QueryKit/QueryKit)


### Version History

- 3.0 - Swift framework; added attributes support and many other improvements
- 2.1 - Swift framework; added CocoaPods and Carthage support
- 2.0 - Swift framework; first public release as open source
- 1.1 - Objective-C framework; private Alecrim team use
- 1.0 - Objective-C framework; private Alecrim team use

---

## Contact

- [Vanderlei Martinelli](https://github.com/vmartinelli)

## License

AlecrimCoreData is released under an MIT license. See LICENSE for more information.
