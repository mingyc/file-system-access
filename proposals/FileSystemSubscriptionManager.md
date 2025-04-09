# Proposal: FileSystemSubscriptionManager

## Authors

* Ming-Ying Chung (<mych@google.com>)
* Austin Sullivan (<asully@google.com>)

## Background

The [FileSystemObserver API][fso-api] allows browsing context to receive records of changes for the observed files or directories. However, changes which occur while the website has no open tabs are not visible to the website.

This document proposes integrating file system change observing with service worker registration, such that a service worker may be woken up on a change to the local file system.

[fso-api]: https://chromestatus.com/feature/4622243656630272

## Goal

Allow web applications that do not have an open tab to quickly respond to changes to registered files and directories on the local file system.

## Discussion

### Option 1. Enabling FileSystemObserver in Service Worker

By [current design][fso], an instance of FileSystemObserver should only report changes which occur while the observer is connected and the website has an open & active tab.

Enabling creations of FileSystemObserver in ServiceWorker will tie its lifetime with ServiceWorkerGlobalScope. Users might expect such observers to continue to watch files in the background. However, in browser implementation, service workers that haven’t received new events in a certain period of time, e.g. [30s in Chrome][chrome-sw], will likely be terminated.

Hence, this approach might only work for websites that either can ensure a long-running service worker or are not really interested in using service worker registration for observing the file change events after the websites are closed.

[fso]: https://docs.google.com/document/d/11o4HaslFwQxKIjJw0M9LAkkZp2ikCNwA9vvYsjA657w/edit?tab=t.0#heading=h.kktfoyrlhtnv
[chrome-sw]: https://developer.chrome.com/blog/longer-esw-lifetimes#background

### Option 2. Updating FileSystemObserver API to outlive Service Worker

Another approach is to ask what if updating FileSystemObserver to allow it to outlive without the limit of running in ServiceWorkerGlobalScope?

If such an option is implemented, there needs to be mechanisms to
Tell when the browser should stop watching file changes.
Handle file changes happen after the service worker is already terminated but the website is still open. Possibly needs a way to wake up a new service worker.

There are existing mechanisms to auto wake up new service workers. Hence the next option.

### Option 3. Utilizing Service Worker Registration

The ServiceWorkerRegistration interface represents registration of a service worker for a specific origin and scope. The browser maintains a persistent list of active ServiceWorkerRegistration even when the associated service worker is not actively running, and will wake up new service workers if a registered event happens.

## Proposed API

### Web IDL

First, define a new interface FileSystemSubscriptionManager that will be held under every ServiceWorkerRegistration, which manages subscriptions to changes to file systems:

```idl
partial interface ServiceWorkerRegistration {
  // Returns a reference to FileSystemSubscriptionManager interface, which allows
  // for subscribing to specific file changes.
  [SameObject] readonly attribute FileSystemSubscriptionManager fileSystem;
};
```

```idl
// Provides methods for managing file system subscriptions.
interface FileSystemSubscriptionManager {
  // Subscribes to changes to a FileSystemHandle with the browser with specific
  // options. Returns a Promise that resolves when the subscription completes.
  Promise<void> subscribe(FileSystemHandle handle,
                          FileSystemObserverObserverOptions options = {});
  // Unsubscribes to changes to a FileSystemHandle. Returns a Promise that
  // resolves when the unsubscription completes.
  Promise<void> unsubscribe(FileSystemHandle handle);
  // Returns a Promise that resolves with a list of FileSystemSubscription
  // representing all the current file system subscriptions with the browser.
  Promise<sequence<FileSystemSubscription>> getSubscriptions();
};
```

```idl
// Represents a subscription to changes to a FileSystemHandle.
interface FileSystemSubscription {
  required FileSystemHandle handle;
  FileSystemObserverObserverOptions? options = null;
};
```

Second, allow service workers to fire a new type of event FileSystemChangeEvent, which includes a list of FileSystemChangeRecord.

```idl
partial interface ServiceWorkerGlobalScope {
  // Fired when FileSystemSubscriptionManager observes changes.
  attribute EventHandler attribute onfilesystemchange;
};
```

```idl
// Represents a file system change event.
interface FileSystemChangeEvent : ExtendableEvent {
  constructor(DOMString type, FileSystemChangeEventInit init);
  readonly attribute sequence<FileSystemChangeRecord> records;
};

interface FileSystemChangeEventInit : ExtendableEvent {
  required sequence<FileSystemChangeRecord> records;
}
```

### Example: Observing Changes to a Directory

```javascript
// main.js
const directoryHandle = await window.showDirectoryPicker();
async function observeDirectoryChanges(directoryHandle) {
  const registration = await navigator.ServiceWorker.register("/service-worker.js");
  registration.fileSystem.subscribe(directoryHandle, {recursive: true});
}

// service-worker.js
self.addEventListener('onfilesystemchange', event => {
  for (const record of event.records) {
    assert(await directoryHandle.IsSameEntry(record.root));
    handleRecord(record);
  }
});
```
