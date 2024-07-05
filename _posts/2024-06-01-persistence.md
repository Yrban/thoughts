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

// MARK: - Migration
//    Migrate the core data from app to shared app group
//    https://menuplan.app/coding/2021/10/27/core-data-store-path-migration.html
    static func migrateCoreDataIfNecessary() {
        let oldStoreContainer = NSPersistentContainer(name: CDConstants.hotHorse)
        guard let storeDescription = oldStoreContainer.persistentStoreDescriptions.first else {
            // We can always create a new one
            PersistenceController.logger.error("Failed to retrieve a persistent store description.")
            return
        }

        // Verify if old file is there
        guard let fromStoreURL = storeDescription.url,
              FileManager.default.fileExists(atPath: fromStoreURL.path) else {
            // No need to migrate sqlite
            return
        }

        let persistence = PersistenceController()
        let oldStoreURL = NSPersistentContainer.defaultDirectoryURL().appendingPathComponent(CDConstants.hotHorseSQLite)
        let newStoreURL = AppGroup.group.containerURL.appendingPathComponent(CDConstants.hotHorseSQLite)
        let coordinator = persistence.container.persistentStoreCoordinator

        if coordinator.persistentStore(for: oldStoreURL) != nil {
            do {
                try coordinator.replacePersistentStore(
                    at: newStoreURL,
                    destinationOptions: nil,
                    withPersistentStoreFrom: oldStoreURL,
                    sourceOptions: nil,
                    type: .sqlite
                )
                try coordinator.destroyPersistentStore(at: oldStoreURL, type: .sqlite)
            } catch {
                Self.logger.warning("migrateCoreData failed with error: \(error)")
            }
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
The containerIdentifier is the CloudKit Container in iCloud. You access it through developer.apple.com. The container name will start "iCloud." and then the container name. This throws a lot of people, and is very difficult to debug.

Core Data Model Versioning:

    At some point, no matter how well you have thought through your data model, you will need to change your database. You may get a great idea for an addition to your app, or you may realize that your well thought out model missed some things. It happens, and Apple has given us the necessary tools to handle it.
    If you have not pushed your app to the store, you can change the model to your heart's content. You simply delete your app from the test device, and install the fresh model. Yes, you will delete any data that you had stored in Core Data, but that is why you are testing. Test, test and test again. Think about your model some more. If you know you want to add entities or attributes in the future, and you know exactly what they will be, add them now. If not, you can deal with them later. Once you have pushed to the App Store, things change. You now must consider versioning.
    When you need to change your model after deployment, versioning becomes your friend. While it is not strictly necessary to version your Core Data model, versioning can save you a lot of grief down the road by making compatibility, and the need for migration clear.

    DISCUSS TYPES OF MIGRATIONS
    DISCUSS LIGHTWEIGHT MIGRATIONS

Migrations with iCloud and CloudKit
    As usual, Apple has some extra rules if you are using a CloudKit container. The first, and foremost rule: You cannot delete an attribute or entity. Once they are created, they exist in perpetuity; they are zombies that will break your syncing if you kill them. You can repurpose them if you are careful.
