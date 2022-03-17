# Rosetree Trigger Framework

> The Rosetree Trigger Framework is based **heavily** on [Kevin O'Hara's SFDC Trigger Framework](https://github.com/kevinohara80/sfdc-trigger-framework). Rosetree added a few additional features to Kevin's framework including a custom metadata type to bypass code as well as a way to bypass specific records in a subsequent run. Additionally, Rosetree made a few tweaks to the framework to make it more efficient and readable.  

### Overview
The [Rosetree Apex Style Guide](https://github.com/Rosetree-Solutions/Rosetree-Guides/blob/main/ApexStyleGuide.md#trigger-best-practices) outlines several best practices and standards for writing triggers and trigger handlers. These standards include only having one trigger per object, keeping the logic outside of the trigger, as well as general Apex bulkification. 

The Rosetree Trigger Framework adheres to these standards by utilizing a base class that is inherited in all Trigger Handlers. The Triggers themselves do not contain any logic as they simply instantiate the handler class. The framework itself enforces several best practices and adds features to handle common requirements of triggers. The framework is also minimal and easy to use. 
### Usage

To create a trigger handler simply create a class that extends the RTSTriggerHandler class. Then to add logic that would run during a trigger context, you simply override the contexts with in a method. Below is an update for a `beforeUpdate` context.

Note, when referencing the Trigger's static variables within the handler, SObjects are returned as opposed to specific objects like Opportunity, Account, etc. This means you must cast the objects when you reference them in the trigger handler. Notice below how we cast `Trigger.new` to `(List<Opportunity>)`.

###### Trigger Handler Example
```apex
public class OpportunityTriggerHandler extends RTSTriggerHandler {

    public override void beforeUpdate(){
        for (Opportunity opp : (List<Opportunity>) Trigger.new) {
            // Do something here
        }
    }

    // Add overrides for other contexts
}
```

To use the trigger handler within a trigger you simply need to create an instance of the handler and call the `run()` method. 
###### Trigger  Example
```apex
trigger OpportunityTrigger on Opportunity (
    before insert,
    before update,
    before delete,
    after insert,
    after update,
    after delete,
    after undelete
){
    new OpportunityTriggerHandler().run();
}
```

### Features

#### Bypass

Often it is desired to stop the execution of entire trigger handlers or certain logic within code. The framework includes several ways to bypass Apex code. 

###### Apex Bypass Custom Metadata Type
There is a custom metadata type called "Apex Bypass" that can be accessed from `SETUP > Custom Code > Custom Metadata Types`. To bypass a trigger handler you simply need to create a record with a DeveloperName set to the trigger handler's name **exactly** along with the Bypass__c field set to true. 

###### Bypass Section of Code

#### Max Loop Count

#### Recursive Static Variables
There are a couple of static variables included in the framework which can be used to prevent the same record from being executed multiple times within one trigger context. These static variables are usefule for scenarios where you don't want to set a max loop count which would limit the entire trigger context from running, but instead you want to selectively limit specified records from running. 

These Static variables are available in the Trigger Framework and will keep their values during the Apex Transaction. 
```` apex     
public static set<Id> recursiveIDCheck;
public static set<String> recursiveStringCheck;
````

An example of how these static variables could be used within a Trigger Handler.

```` apex
if (!recursiveIDCheck.contains(opp.Id)) {
    recursiveIDCheck.add(opp.Id);
    // run the logic for the record which should only run once
}
````
    
### Overridable Methods
* beforeInsert()
* beforeUpdate()
* beforeDelete()
* afterInsert()
* afterUpdate()
* afterDelete()
* afterUndelete()

### Metadata Summary
| Type | Name | Description |
| --- | --- | --- |
| Apex Class | RTSTriggerHandler.cls | This is class that drives the Trigger Framework. All Trigger Handlers should extend this base class. |
| Apex Class | RTSTriggerHandler_Test.cls | This is the test class for the RTSTriggerHandler.cls|
| Custom Metadata Type | Apex Bypass | This is a custom metadata type that can be used to declaratively bypass the execution of entire trigger handlers or sections of code. 

