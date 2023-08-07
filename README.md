# ApexTriggerDemo
Various Demos of My Salesforce Apex Triggers

//Context Variable 1 Trigger.new => a List of records this trigger has inserted or updated 

//Context Variable 2 Trigger.isBefore => returns true if the trigger is runing on the before event

//Context Variable 3 Trigger.isInsert => returns true if the trigger is called when the user has done the insert operation

//Context Variable 4 Trigger.isAfter => returns true if the trigger is called after the record is inserted or updated

//Context Variable 5 Trigger.newMap => returns a list of records that are updated or inserted with the latest values in a map format 

//Context Variable 6 Trigger.oldMap => returns a list of records that are updated or inserted with the old values in a map format 

//Context Variable 7 Trigger.old => returns a list of records that are updated or inserted with the old values

//Context Variable 8 Trigger.isUpdate => returns true if the trigger is called when the record is updated

//Context Variable 9 Trigger.isDelete => returns true if the trigger is called when the record is deleted

//Context Variable 10 Trigger.isUndelete => returns true if the trigger is called when the record is undelete


trigger AccountTrigger on Account (before insert, after insert, before update, after update, before delete, after delete, after undelete) {
    
     /*Scenario 8: when account is restored from a recycling bin, email the user who undeleted it*/
    
    //After undelete logic to be written in the block below
    If(Trigger.isAfter && Trigger.isUndelete){
      //Send email to the user who restored a deleted account from the recycling bin
      //Trigger.new is available in UNDELETE because it is like new
      //Trigger.old is not available in UNDELETE
               
        List<Messaging.SingleEmailMessage> emailObjs = new List<Messaging.SingleEmailMessage>();
        for(Account accNew: Trigger.new){
            Messaging.SingleEmailMessage emailObj = new Messaging.SingleEmailMessage();
            List<String> emailAddress = new List<String>();
            emailAddress.add(UserInfo.getUserEmail());
            emailObj.setToAddresses(emailAddress);
            emailObj.setSubject('Account has been restored/undeleted successfully'+ accNew.Name);
            emailObj.setPlainTextBody('You successfully restored this account from a recycling bin');
            emailObjs.add(emailObj);
        }
        Messaging.sendEmail(emailObjs);
    }
      
    
    
     /*Scenario 7: when account is deleted, send an email to the user who deleted it*/

    
    //After delete logic to be written in the block below
    If(Trigger.isAfter && Trigger.isDelete){
        //Send email to the user who deleted an account
        
       
        List<Messaging.SingleEmailMessage> emailObjs = new List<Messaging.SingleEmailMessage>();
        for(Account accOld: Trigger.old){
            Messaging.SingleEmailMessage emailObj = new Messaging.SingleEmailMessage();
            List<String> emailAddress = new List<String>();
            emailAddress.add(UserInfo.getUserEmail());
            emailObj.setToAddresses(emailAddress);
            emailObj.setSubject('Account has been deleted successfully'+ accOld.Name);
            emailObj.setPlainTextBody('You successfully deleted this account');
            emailObjs.add(emailObj);
        }
        Messaging.sendEmail(emailObjs);
    }
    
    
     /*Scenario 6: prohibit deletion of an active account*/
    
    
    //Before delete logic to be written in the block below
    If(Trigger.isBefore && Trigger.isDelete){
        //Trigger.new is not available in Delete operations
        //Trigger.old is available, including OldMap
        for(Account accOld: Trigger.Old){
            If(accOld.Active__c=='Yes')
                accOld.addError('You cannot delete accounts whose status is Active');
        }
    }
    
    

     /*Scenario 5: if account addresses if changed, update all associated contact records with the same address*/
    
    
    //After update logic to be written in the block below
    If(Trigger.isAfter && Trigger.isUpdate){
        Set<Id> accIDsWhichGotBillingAddressChanged = new Set<Id>();
        for(Account accRecNew: Trigger.new){
            Account accRecOld = Trigger.oldMap.get(accRecNew.Id);
            If(accRecNew.BillingStreet!= accRecOld.BillingStreet){
                accIDsWhichGotBillingAddressChanged.add(accRecNew.Id);
            }
                
        }
        List<Account> accsWithContacts = [SELECT id, name,billingcity, billingstreet, billingstate, billingcountry,(SELECT id,name FROM Contacts) FROM Account WHERE ID in: accIDsWhichGotBillingAddressChanged];
        List<Contact> contsListToUpdate = new List<Contact>();
            
        for(Account acc: accsWithContacts){
            List<Contact> contsOfTheLoopedAccount = acc.contacts;
            for(Contact con: contsOfTheLoopedAccount){
                con.MailingStreet=acc.BillingStreet;
                con.MailingCity=acc.BillingCity;
                con.MailingState=acc.BillingState;
                con.MailingCountry=acc.BillingCountry;
                contsListToUpdate.add(con);
            }
        }
    
        If(contsListToUpdate.size()>0){
            UPDATE contsListToUpdate;
        }
            
    
    }
    
    
     /*Scenario 4: prohibit account name from being changed*/
    
    //Before update logic to be written in the block below
    If(Trigger.isBefore && Trigger.isUpdate){
        
        /*System.debug('New Values');
        System.debug(Trigger.new);
        System.debug(Trigger.newMap);
        
        System.debug('Old Values');
        System.debug(Trigger.old);
        System.debug(Trigger.oldMap);*/
        
        //if you need to compare old values to new values, here's the logic below
        for(Account accRecNew: Trigger.new){
           Account accRecOld = Trigger.oldMap.get(accRecNew.Id);
            If(accRecNew.Name!= accRecOld.Name){
               accRecNew.addError ('Account name change is not allowed');

            }
        }
        
        
    }
    
    
     
    /*Scenario 3: when user creates an account, automatically create a contact whose last name matches account name*/
    
    //After insert logic to be written in the block below
    If(Trigger.isAfter && Trigger.isInsert){
        
        List<Contact> conListToInsert = new List<Contact>();
         for(Account accRec: Trigger.new){
             Contact con = new Contact();
             con.LastName = accRec.Name;
             con.AccountId = accRec.Id;
             conListToInsert.add(con);
             
    }
    
        //Check the list size, ALWAYS
        
        If(conListToInsert.size()>0)
            INSERT conListToInsert;
        
    }
    
    //Before insert logic to be written in the block below
    If(Trigger.isBefore && Trigger.isInsert){
        
        for(Account accRec: Trigger.new){
            
           
            /*Scenario2: throw an error if annual revenue is less than 1000*/
            
          
            If(accRec.AnnualRevenue < 1000)
                accRec.addError('Annual Revenue Cannot Be Less Than 1000');
                
            
            
            /*Scenario 1: match shipping and billing addresses if shipping address is not provided*/
                
            If(accRec.ShippingStreet==null)
                accRec.ShippingStreet = accRec.BillingStreet;
            If(accRec.ShippingCity==null)
                accRec.ShippingCity = accRec.BillingCity;
            If(accRec.ShippingCountry==null)
                accRec.ShippingCountry = accRec.BillingCountry;
            If(accRec.ShippingState==null)
                accRec.ShippingState = accRec.BillingState;
            If(accRec.ShippingPostalCode==null)
                accRec.ShippingPostalCode = accRec.BillingPostalCode;
            
        }
    }

}
