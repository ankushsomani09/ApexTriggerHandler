# Apex Trigger Handler

![](https://img.shields.io/badge/version-1.0.0-brightgreen.svg) ![](https://img.shields.io/badge/build-passing-brightgreen.svg) ![](https://img.shields.io/badge/coverage-100%25-brightgreen.svg)

There are already many trigger handler libraries out there, but this one has some different approaches or advantanges such as state sharing, built in helper methods etc.. Just one class `Triggers.cls` with its corresponding test class `TriggersTest.cls`, and its minimal and simple.

## Features

1. Share common query results via context.state with the following handlers in the current trigger execution context.
2. Built-in helper to perform common operations with Trigger.new or Trigger.old, such as detect field changes.
3. Control flow of handler execution with context.next(), context.stop(), and context.skips.

## Usage

To create a trigger handler, you will need to create a class that implements the `Triggers.Handler` interface and its `criteria` method, and the corresponding trigger event method interfaces, such as the `Triggers.BeforeUpdate` interface and its `beforeUpdate` method.

```java
public class MyAccountHandler implements Triggers.Handler, Triggers.BeforeUpdate {
    public Boolean criteria(Triggers.Context context, Triggers.Helper helper) {
        return true;
    }

    public void beforeUpdate(Triggers.Context context, Triggers.Helper helper) {
        // do stuff
    }
}
```

### Trigger

As you have noticed, why we are creating same handlers for different trigger events? This is because handlers may need to execute in different orders for different trigger events, we need to provide developers great controls over the order of executions.

```java
trigger AccountTrigger on Account (before update, after update) {
    Triggers.prepare()
        .beforeUpdate()
            .bind(new MyAccountHandler())
            .bind(new AnotherAccountHandler()
        .afterUpdate()
            .bind(new AnotherAccountHandler()
            .bind(new MyAccountHandler())
        .execute();
}
```

### Trigger Handler

Please check the comments below for detailed explanations and tricks to customize a trigger handler.

```java
// 1. Use interfaces instead of a base class to extend a custom handler. With interface
// approach we can declare only the needed interfaces explicitly, which is much cleaner
// and clearer.
public class MyAccountHandler implements Triggers.Handler, 
                                         Triggers.BeforeUpdate, 
                                         Triggers.AfterUpdate {

    // 2. There is a "criteria" stage before any handler execution. This gives
    // developers chances to turn on and off the handlers according to
    // configurations at run time.
    public Boolean criteria(Triggers.Context context, Triggers.Helper helper) {
        return Triggers.WHEN_ALWAYS;

        // 3. There are also helper methods to check if certain fields have changes
        // return helper.isChangedAny(Account.Name, Account.Description);
        // return helper.isChangedAll(Account.Name, Account.Description);
    }

    public void beforeUpdate(Triggers.Context context, Triggers.Helper helper) {
        then(context, helper);
    }

    public void afterUpdate(Triggers.Context context, Triggers.Helper helper) {
        then(context, helper);
    }

    private void then(Triggers.Context context, Triggers.Helper helper) {
        // 4. All properties on Trigger have been exposed to context.triggerProp.
      	// Direct reference of Trigger.old and Trigger.new can be avoided,
        // instead use context.triggerProp.oldList and context.triggerProp.newList.
        if (context.triggerProp.isUpdate) {

          // 5. Use context.state to pass query or computation results down to all
          // following handlers within the current trigger context, i.e. before update.
          if (context.state.get('counter') == null) {
              context.state.put('counter', 0);
          }

          // 6. Use context.skips or Triggers.skips to prevent specific handlers from
          // execution. Please do remember restore the handler when appropriate.
          context.skips.add(ContactHandler.class);
          List<Contact> contacts = ...;
          Database.insert(contacts);
          context.skips.remove(ContactHandler.class);

          // 7-1. Call context.next() to execute the next handler. It is optional to use.
          // But this will be required by every following handlers once used.
          context.next();
          // When the next handler finishes execution, some following up
          // logics can be performed here, such as post validation.

          // 7-2. If context.stop() is called instead of context.next(), any following
          // handlers won't be executed, just like the stop in process builder.
          context.stop();
        }
    }
}
```

### More on Skips

`context.skips` references the same global static variable `Triggers.skips`. If you want to skip handlers in contexts rather than a trigger handler. Please use `Triggers.skips` instead. For example, when you want to skip a trigger handler in a batch class:

```java
global class AccountUpdateBatch implements Database.Batchable<sObject> {
    ...
    global void execute(Database.BatchableContext BC, List<sObject> scope){
        Triggers.skips.add(MyAccountHandler.class);
        // Update accounts...
        Triggers.skips.remove(MyAccountHandler.class);
    }
    ...
}
```

Or you can skip the handler during batch execution in the criteria phase:

```java
public class MyAccountHandler implements Triggers.Handler, Triggers.BeforeUpdate {
    public Boolean criteria(Triggers.Context context, Triggers.Helper helper) {
        return !System.isBatch();
    }
    ...
}
```

## APIs

### Trigger Handler Interfaces

| Interface               | Method to Implement                                          |
| ----------------------- | ------------------------------------------------------------ |
| Triggers.Handler        | `Boolean criteria(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.BeforeInsert   | `void beforeInsert(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.AfterInsert    | `void afterInsert(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.BeforeUpdate   | `void beforeUpdate(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.AfterUpdate    | `void afterUpdate(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.BeforeDelete   | `void beforeDelete(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.AfterDelete    | `void afterDelete(Triggers.Context context, Triggers.Helper helper);` |
| Triggers.BeforeUndelete | `void afterUndelete(Triggers.Context context, Triggers.Helper helper);` |

### Trigger Context

| Property/Method     | Description                                                  |
| ------------------- | ------------------------------------------------------------ |
| context.triggerProp | A read-only instance exposes every properties on `Trigger` context, i.e. <br/>   - `Trigger.new` => `context.triggerProp.newList`<br/>   - `Trigger.old` => `context.triggerProp.oldList`<br/>   - `Trigger.isUpdate` => `context.triggerProp.isUpdate`<br/>   - `Trigger.isBefore` => `context.triggerProp.isBefore` |
| context.state       | A `Map<String, Object>` provided for developers to pass any value down to other handlers. |
| context.skips       | A Set wrapper to store handler names to be skipped. You can call `context.skips.add()`, `context.skips.remove()`, `context.skips.clear()` `context.skips.contains()` etc. The passed-in handlers could be trigger handlers of different sObject triggers. |
| context.next()      | Call the next handler.                                       |
| context.stop()      | Stop execute any following handlers. A bit like the the stop in process builders. |

### Trigger Helper

| Method                  | Return Type | Description                                                  |
| ----------------------- | ----------- | ------------------------------------------------------------ |
| helper.isChanged        | Boolean     | Check if any record has a field changed during an update.    |
| helper.isChangedAny     | Boolean     | Check if any record has multiple fields changed during an update. Return true if any specified field is changed. |
| helper.isChangedAll     | Boolean     | Check if any record has multiple fields changed during an update. Return true only if all specified fields are changed. |
| helper.filterChanged    | Set\<Id\>   | Filter IDs of records have a field changed during an update. |
| helper.filterChangedAny | Set\<Id\>   | Filter IDs of records have mulantiple fields changed during an update. Return IDs if any specified field is changed. |
| helper.filterChangedAll | Set\<Id\>   | Filter IDs of records have mulantiple fields changed during an update. Return IDs only if all specified fields are changed. |

## License

MIT License