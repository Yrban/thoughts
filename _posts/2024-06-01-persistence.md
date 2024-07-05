---
title: "Persistence"
date: 2024-06-01
---

One of the things I embraced early was Core Data. I have read and learned a great deal about Core Data. I still regularly refer to Practical Core Data by Donny Wals. If you don't own it, you should. Persistence is the backbone for apps. What good is obtaining and manipulating data, if you are not saving the data and/or the results for the user to access in the future. Core Data works particularly well with Core Data. It was one of the very, very few frameworks where SwiftUI had a distinct advantage.

I am very excited with the advent of SwiftData. While this is unquestionably the future, it does not yet have feature parity, so once you need to do something a bit out of the box, you are back to Core Data. While it is a bit more difficult to implement (looking at you, predicates), these implementations are well documented over the years. This definitely an area where you can fake it before you have a full understanding.

Add intermediate core data

Putting your store into a shared container

Once you have a basic working model of a Core Data store, have migrated it to an App Group container, if you need, for widgets, you can also persist in the cloud. iCloud integration is relatively simple, almost trivial. If you have a default store in a default container, the coding simply a case of changing your `NSPersistentContainer` to `NSPersistentCloudKitContainer`. You should also make sure your model is set to use CloudKit. If you did not select the "Use Core Data" and "Use CloudKit" checkboxes when creating the app, you simply need to open your model, look below the Entities to Configurations, and select `Default`. With `Default` selected, select the `Data Model` inspector and select `Used with CloudKit`.

Now, we have a few more settings to update. You must have a paid developer account for this, or it will not work.
Choose the Signing & Capabilities tab in Project Settings.
Make sure that “Automatically manage signing” is selected, or you will need to update your entitlements on developer.apple.com. That is outside the scope of this article.
Specify your development team. This will be your developer account.
Click the `+ Capability` button, then scroll to iCloud and select it.

Next,
In the iCloud section select the CloudKit checkbox. This selection also adds push notifications that notify your app when remote content has changed.
Under Containers, select “Use default container” unless you have created a CloudKit container in "CloudKit" under "Additional resources"

Lastly, add another capability:
Click the `+ Capability` button, then scroll to "Background Modes" and select it.
Under Background Modes, choose "Remote Notifications"

What you have done here is added an iCloud container to sync with. This is where all of your data will go, currently into a private database for your user. We need remote notifications as this is how Apple notifies your app that something changed in the iCloud container so that your app syncs. A great deal of functionality built for you with just a couple of clicks.

Update your `PersistenceController` to look like this:
class PersistenceController {

    static let shared = PersistenceController()

    let context: NSManagedObjectContext

    static var preview: PersistenceController = {
        let result = PersistenceController(inMemory: true)
        let context = result.container.viewContext
    
        let date = Date()
        Horse.seedCoreData(context: context)
        for index in 0..<10 {
            let horse = Horse.returnNewHorse(context: context, barnName: ("Horse\(index)"))
        }
        let user = UserInfo(context: context)
        user.expiresDate_ = date.addingTimeInterval(Constant.tenMinute)
    
        do {
            try context.save()
        } catch {
            let nsError = error as NSError
            logger.warning("Unresolved error \(nsError), \(nsError.userInfo)")
        }
        return result
    }()

    let container: NSPersistentCloudKitContainer
    let coordinator: NSPersistentStoreCoordinator

    init(inMemory: Bool = false) {
        // Register Secure Value Transformers
        NSMeasurementValueTransformer.register()
    
        container = NSPersistentCloudKitContainer(name: CDConstants.hotHorse)

        let url = AppGroup.group.containerURL.appendingPathComponent(CDConstants.hotHorseSQLite)
//        Self.logger.debug("CoreData Store location: \(url)")
        let description = NSPersistentStoreDescription(url: url)
        description.setOption(true as NSNumber, forKey: NSPersistentHistoryTrackingKey)
        description.setOption(true as NSNumber, forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)
        description.cloudKitContainerOptions = NSPersistentCloudKitContainerOptions(
            containerIdentifier: CDConstants.hotHorseCK
        )

        container.persistentStoreDescriptions = [description]
        context = container.viewContext
        context.mergePolicy = NSMergeByPropertyStoreTrumpMergePolicy
        coordinator = container.persistentStoreCoordinator

        // MARK: UndoManager Support
        container.viewContext.undoManager = UndoManager()
    
        if inMemory {
            container.persistentStoreDescriptions.first!.url = URL(fileURLWithPath: "/dev/null")
        }
        container.loadPersistentStores(completionHandler: { [self] (storeDescription, error) in
            if let error = error as? NSError {
                PersistenceController.logger
                    .error(
                        "loadPersistentStores: error \(error), \(error.userInfo) for store \(storeDescription.description)"
                    )
                handlePersistentStoreError(error)
            }
        })
    
        // MARK: Cloudkit Merge
        context.automaticallyMergesChangesFromParent = true
        
        // MARK: Cloudkit Schema Initialization
#if DEBUG
        // If you are creating or updating an iCloud Schema, uncomment this code. Do NOT Leave it live after creation.
        // The .printSchema option prints the results to the console for debugging purposes.
//        do {
//            try container.initializeCloudKitSchema(options: [.printSchema])
//        } catch {
//            PersistenceController.logger.error("Schema initialization error: \(error)")
//        }
#endif
    }

    func undo() {
        let context = container.viewContext
        if context.hasChanges {
            context.undo()
        }
    }

    private func handlePersistentStoreError(_ error: NSError) {
        let url = AppGroup.group.containerURL.appendingPathComponent(CDConstants.hotHorseSQLite)

        do {
            // Attempt to destroy the existing persistent store
            try container.persistentStoreCoordinator.destroyPersistentStore(at: url, type: .sqlite, options: nil)
            
        } catch {
            // If we fail to recreate the store, handle the error accordingly.
            Self.logger.warning("PersistenceController handlePersistentStoreError: Failed to delete old store with error: \(error)")
        }
        
        do {
            // Attempt to add a new persistent store
            _ = try container.persistentStoreCoordinator.addPersistentStore(type: .sqlite, configuration: "Default", at: url, options: nil)
            
        } catch {
            // If we fail to recreate the store, handle the error accordingly.
            Self.logger.warning("PersistenceController handlePersistentStoreError: Failed to create new store with error: \(error)")
        }
    }
    
    private static let logger = Logger(
        subsystem: Bundle.main.bundleIdentifier!,
        category: String(describing: PersistenceController.self)
    )
}


I want to point out this code:
        description.cloudKitContainerOptions = NSPersistentCloudKitContainerOptions(
            containerIdentifier: CDConstants.hotHorseCK
        )
The container 
