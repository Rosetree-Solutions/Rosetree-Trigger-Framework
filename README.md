# Rosetree Trigger Framework

> The Rosetree Trigger Framework is based **heavily** on [Kevin O'Hara's SFDC Trigger Framework](https://github.com/kevinohara80/sfdc-trigger-framework). Rosetree added a few additional features to Kevin's framework including a custom metadata type to bypass code as well as a way to bypass specific records in a subsequent run. Additionally, Rosetree made a few tweaks to the framework to make it more efficient and readable.  

### Overview
The [Rosetree Apex Style Guide](https://github.com/Rosetree-Solutions/Rosetree-Guides/blob/main/ApexStyleGuide.md#trigger-best-practices) outlines several best practices and standards for writing triggers and trigger handlers. These standards include having only one trigger per object, keeping the logic outside of the trigger, as well as general Apex bulkification. 

The Rosetree Trigger Framework adheres to these standards by utilizing a base class that is inherited in all Trigger Handlers. The Triggers themselves do not contain any logic as they simply instantiate the handler class. The framework itself enforces several best practices and adds features to handle common requirements of triggers. The framework is also minimal and easy to use. 
### Usage

To create a trigger handler simply create a class that extends the RTSTriggerHandler class. Then to add logic that would run during a trigger context, you simply override the contexts within a method. Below is an example for a `beforeUpdate` context.

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

Often it is desired to stop the execution of entire trigger handlers or blocks of code based on some type of logic. The framework includes several ways to bypass Apex code. 

###### Bypass Trigger Handler
If you would like to bypass a full trigger handler you can use the `RTSTriggerHandler.bypass('AccountTriggerHandler')` method. Below is an example of bypassing the Account Trigger Handler during a section of code within the Opportunity Trigger Handler. 

```` Apex
public class OpportunityTriggerHandler extends RTSTriggerHandler {

    public override void afterUpdate(){

        // Collecting a list of accounts to update in the Opportunity Handler
        List<Account> accountsToUpdate = new List<Account>();
        for (Opportunity opp : (List<Opportunity>) Trigger.new) {
            Account accountToUpdate = new Account();
            accountToUpdate.Id = opp.AccountId;
            accountToUpdate.Name = 'Test';
            accountsToUpdate.add(accountToUpdate);
        }

        /*
           Within the Opportunity Trigger Handler we want to update a list
           of Accounts, but we don't want the Account Trigger Handler to run.
        */

        RTSTriggerHandler.bypass('AccountTriggerHandler');
        update accountsToUpdate;

        /*
           Clearing the bypass so the next time the Account Trigger Handler
           is called within this Apex transaction it will not be bypassed
        */
        RTSTriggerHandler.clearBypass('AccountTriggerHandler');

    }

}

````


###### Bypass Section of Code

To Bypass a section of code, instead of a full Trigger Handler, you can pass in a descriptive string into the `.bypass()` method and then use the `isBypassed()` method before the code execution to check if the code should run.

In the below example on the AccountTriggerHandler we are checking `RTSTriggerHandler.isBypassed('AccountSync')` to see if the Account Sync logic should be bypassed.

```` Apex
public class AccountTriggerHandler extends RTSTriggerHandler {

    public override void afterUpdate(){
        initiateAccountSync((List<Account>)Trigger.new);
    }

    private static void initiateAccountSync(List<Account> accountsToSync){

        // Check if Account Sync is bypassed
        if (!RTSTriggerHandler.isBypassed('AccountSync')) {
            // Do something if the AccountSync is not bypassed
        }

    }
}

````

In the OpportunityTriggerHandler you can see where we are setting `RTSTriggerHandler.bypass('AccountSync')` to bypass that specific method/logic in the AccountTriggerHandler.

```` Apex
public class OpportunityTriggerHandler extends RTSTriggerHandler {

    public override void afterUpdate(){

        // Collecting a list of accounts to update in the Opportunity Handler
        List<Account> accountsToUpdate = new List<Account>();
        for (Opportunity opp : (List<Opportunity>) Trigger.new) {
            Account accountToUpdate = new Account();
            accountToUpdate.Id = opp.AccountId;
            accountToUpdate.Name = 'Test';
            accountsToUpdate.add(accountToUpdate);
        }

        /*
           Within the Opportunity Trigger Handler we want to update a list
           of Accounts, but we don't want the Account Sync code to run.
        */

        RTSTriggerHandler.bypass('AccountSync');
        update accountsToUpdate;

        /*
           Clearing the bypass so the next time the Account Trigger Handler
           is called within this Apex transaction it will not be bypassed
        */
        RTSTriggerHandler.clearBypass('AccountSync');

    }
}

````

###### Apex Bypass Custom Metadata Type
There is a custom metadata type called "Apex Bypass" that can be accessed from `SETUP > Custom Code > Custom Metadata Types`. To bypass a trigger handler you simply need to create a record with a DeveloperName set to the trigger handler's name **exactly** along with the Bypass__c field set to true. 

You can also use the custom metadata type to bypass a section of code that looks at the isBypassed method by creating a record with a DeveloperName of whatever String the method is checking. 


#### Max Loop Count

One way to prevent recursion using this framework is to set a max loop count for the handler using the `setMaxLoopCount()` method. Every time the trigger handler runs the count will be incremented. If the max is exceeded, an exception will be thrown. 

> It is important to note the count is incremented every time the trigger handler runs. The Before and After contexts count as two separate runs of the same trigger. Additionally, every batch of 200 also counts as separate trigger handler runs. This means if you insert a list of 1000 records the count would be 10 which is equal to 5 batches of 200, and 2 counts per batch (before and after context). Due to the limitations of this approach I prefer using the recursive static variables described in the next section to limit the same record from being called multiple times. 

```` apex

trigger OpportunityTrigger on Opportunity (
    before insert,
    before update,
    before delete,
    after insert,
    after update,
    after delete,
    after undelete
){
    OpportunityTriggerHandler handler = new OpportunityTriggerHandler();
    /*
    This would allow the trigger handler to run twice. 
    Once for the before context and once for after. 
    Keep in mind each trigger batch of 200 also counts as a separate run.
    */
    handler.setMaxLoopCount(2);
    handler.run();
}

````

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

