---
published: true
title: BlocNotes
layout: post
---
##BlocNotes
    
###Summary
 BlocNotes is a note-taking app. It allows notes to be stored and deleted. BlocNotes stores the notes using Core Data. The notes can be shared with other applications and also integrates with iCloud.

###Explanation

This was the first application that I built from scratch. As part of the Bloc curriculum, we were required to pick an app to build and I selected this one as my first one. It was an ideal first application for me as it introduced me to many concepts that are essential to iOS application development such as View Controllers, Auto Layout, Core Data, Table Views, App Extensions, and iCloud integration.

###Problem

We were giving these requirements for the BlocNotes application and it was up to us to implement them:

Notes can be created, edited, and deleted.
The title of the note can be set.
The note content can be shared from other applications into BlocNotes.
The notes can be searched.
Data such as email addresses, URLs, and phone numbers are tappable links.
Notes live in the cloud and are accessible from all iOS devices.

###Solution

Notes can be created, edited, and deleted.
      I resolved by building master and detail view controllers. The master view controller lists all of the existing notes and the detail view controllers displays an individual note. The master view controller also provides the functionality to search the existing notes and allows the user to click a button to add a new note or click on a Table View row to edit a selected note. The detail view controller has an UITextView to display the note body and an UIBarButtonItem to save the note. 

I added a class CoreDataStack that is responsible for saving the note to Core Data. It has a static method called defaultStack which uses the Singleton design pattern to return an instance of CoreDataStack. This class abstracts the details of saving to Core Data and makes it easy to do a save without worrying about the details of the Core Data storage.

+ (instancetype)defaultStack {
    static CoreDataStack *defaultStack;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        defaultStack = [[self alloc] init];
    });
    
    return defaultStack;
}

The title of the note can be set.
I added an UITextField object to allow this field to be set.
 
The note content can be shared from other applications into BlocNotes.
This was accomplished by using the UIActivityViewController class which provides automated sharing. It accepts an array of items to share. The code to build this array and share them is here:

- (IBAction)shareNote:(id)sender {
    
    NSMutableArray *itemsToShare = [NSMutableArray array];
    if (self.detailItem.title.length > 0) {
        [itemsToShare addObject:self.detailItem.title];
    }
    
    if (self.detailItem.body.length > 0) {
        [itemsToShare addObject:self.detailItem.body];
    }
    
    if (itemsToShare.count > 0) {
        UIActivityViewController *activityVC = [[UIActivityViewController alloc] initWithActivityItems:itemsToShare applicationActivities:nil];
        [self presentViewController:activityVC animated:YES completion:nil];
    }
}

The notes can be searched.
I used the UISearchDisplayController class to allow the notes to be searched. The code to search the notes is here:
-(BOOL)searchDisplayController:(UISearchDisplayController *)controller shouldReloadTableForSearchString:(NSString *)searchString
{
    [self searchForText:searchString];
    [self.tableView reloadData];
    
    return YES;
}

- (void)searchForText:(NSString *)searchText
{
    if (self.managedObjectContext)
    {
        NSPredicate *predicate = nil;
        if (searchText && searchText.length > 0) {
            predicate = [NSPredicate
                         predicateWithFormat:@"body CONTAINS[cd] %@ OR title CONTAINS[cd] %@",
                         searchText, searchText];
        }
        [self.fetchRequest setPredicate:predicate];
        
        NSError *error = nil;
        self.filteredList = [self.managedObjectContext executeFetchRequest:self.fetchRequest error:&error];
        if (error)
        {
            NSLog(@"searchFetchRequest failed: %@",[error localizedDescription]);
        }
    }
}

Data such as email addresses, URLs, and phone numbers are tappable links.
I set the dataDetectorTypes property of the UITextView to UIDataDetectorTypeAll then I have a method that uses NSDataDetector to make the specified data types tappable.

Notes live in the cloud and are accessible from all iOS devices.
I modified the singleton class CoreDataStack to accomplish this. This allowed the other parts of the code to remain untouched and makes testing much easier.

I registered for iCloud notifications:
- (void)registerForiCloudNotifications:(NSPersistentStoreCoordinator *)psc {
    NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
    
    [notificationCenter addObserver:self
                           selector:@selector(storesWillChange:)
                               name:NSPersistentStoreCoordinatorStoresWillChangeNotification
                             object:psc];
    
    [notificationCenter addObserver:self
                           selector:@selector(storesDidChange:)
                               name:NSPersistentStoreCoordinatorStoresDidChangeNotification
                             object:psc];
    
    [notificationCenter addObserver:self
                           selector:@selector(persistentStoreDidImportUbiquitousContentChanges:)
                               name:NSPersistentStoreDidImportUbiquitousContentChangesNotification
                             object:psc];
}

Here’s the code to handle the notifications:
//Note: this is called when data in the cloud changes
- (void) persistentStoreDidImportUbiquitousContentChanges:(NSNotification *)changeNotification {
    
    NSLog(@"__PRETTY_FUNCTION__: %s", __PRETTY_FUNCTION__);
    NSLog(@"note.userInfo.description: %@", changeNotification.userInfo.description);
    
    NSManagedObjectContext *context = self.managedObjectContext;
    
    [context performBlock:^{
        [context mergeChangesFromContextDidSaveNotification:changeNotification];
    }];
}

- (void)storesWillChange:(NSNotification *)notification {
    NSManagedObjectContext *context = self.managedObjectContext;
    
    [context performBlockAndWait:^{
        NSError *error;
        
        //Save changes if there are any then reset the context
        if ([context hasChanges]) {
            BOOL success = [context save:&error];
            
            if (!success && error) {
                // perform error handling
                NSLog(@"%@",[error localizedDescription]);
            }
        }
        
        [context reset];
    }];
    
    /* Note: Commenting this out because although the best practice seems to be to trigger UI updates here, all of my tests work fine
             without this code here. Another reason is that the storesDidChange will be called after the store changed and in this method
             the UI will be changed
     
    // Post notification to trigger UI updates
    [[NSNotificationCenter defaultCenter] postNotificationName:@"NotesUINotification" object:self.managedObjectContext];
     */
}

- (void)storesDidChange:(NSNotification *)notification {
    // Refresh your User Interface.
    
    // Post notification to trigger UI updates
    [[NSNotificationCenter defaultCenter] postNotificationName:@"NotesUINotification" object:self.managedObjectContext];
}a

###Results

All of the solutions passed all of my tests and worked just fine. I approach to testing was to perform tests frequently during the implementation of each of the requirements and I did a full end to end test once all of the requirements were completed.


###Conclusion

I’m very glad that I started with this project. I gained a lot of knowledge during the building of this app and it gave me a solid foundation to build on as I proceed further into iOS application development. The BlocNotes app may seem simple from a UI perspective but underneath the covers there are many things being done which have been detailed in this case study. The most difficult requirement for me was to make the links tappable. This presented many challenges and it was very satisfying to overcome those challenges.

https://github.com/kennonoutlaw/bloc-notes