Give Trigger Name: ReorderNotification
Select Object: Inventory_c

  trigger ReorderNotification on Inventory__c (after update) {
    for (Inventory__c inv : Trigger.new) {
        // Ensure Stock Quantity is less than Reorder Level
        if (inv.Stock_Quantity__c < inv.Reorder_Level__c) {
            // Check if the Supplier relationship is populated
            if (inv.Supplier__c != null) {
                Supplier__c supplier = [SELECT Id, Contact_Info__c FROM Supplier__c WHERE Id = :inv.Supplier__c LIMIT 1];
                
                // Debug statement to check the supplier's email
                System.debug('Supplier ID: ' + supplier.Id);
                System.debug('Supplier Contact Info: ' + supplier.Contact_Info__c);

                if (supplier.Contact_Info__c != null) {
                    Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
                    mail.setToAddresses(new String[] { supplier.Contact_Info__c });
                    mail.setSubject('Reorder Notification');
                    mail.setPlainTextBody('Please reorder product: ' + inv.Name + 
                                          '. Current stock is: ' + inv.Stock_Quantity__c);
                    Messaging.sendEmail(new Messaging.SingleEmailMessage[] { mail });
                } else {
                    System.debug('No valid email found for supplier: ' + inv.Supplier__c);
                }
            } else {
                System.debug('Supplier relationship is null for Inventory: ' + inv.Id);
            }
        }
    }
}

 
--------------------------------------------------------------------------------------------------------------------------------------

Give Trigger Name: UpdateStockQuantity 
Select Object: Sales_Order_c                         
trigger UpdateStockQuantity on Sales_Order__c (after insert, after update) {
    // Collect product IDs (related Inventory records) from Sales Orders
    Set<Id> productIds = new Set<Id>();
    for (Sales_Order__c so : Trigger.new) {
        if (so.Product__c != null) {
            productIds.add(so.Product__c);
        }
    }

    // Query Inventory records related to the products in Sales Orders
    List<Inventory__c> inventoryList = [SELECT Id, Stock_Quantity__c FROM Inventory__c WHERE Id IN :productIds];

    // Create a Map to easily reference inventory records by their Ids
    Map<Id, Inventory__c> inventoryMap = new Map<Id, Inventory__c>(inventoryList);

    // Loop through each Sales Order to update the respective inventory
    for (Sales_Order__c so : Trigger.new) {
        if (so.Product__c != null && inventoryMap.containsKey(so.Product__c)) {
            Inventory__c inventory = inventoryMap.get(so.Product__c);

            // Log inventory and order details for debugging
            System.debug('Inventory Before Update: ' + inventory.Stock_Quantity__c);
            System.debug('Sales Order Quantity Ordered: ' + so.Quantity_Ordered__c);

            // Check if this is an update operation
            if (Trigger.isUpdate && Trigger.oldMap.containsKey(so.Id)) {
                Decimal previousQuantity = Trigger.oldMap.get(so.Id).Quantity_Ordered__c;

                // Restore the previous stock quantity (add back the previous quantity)
                if (previousQuantity != null) {
                    inventory.Stock_Quantity__c += previousQuantity;
                    System.debug('Restored Stock Quantity: ' + inventory.Stock_Quantity__c);
                }
            }

            // Reduce the stock by the new quantity (for both insert and update)
            Decimal newQuantity = so.Quantity_Ordered__c;
            if (newQuantity != null) {
                inventory.Stock_Quantity__c -= newQuantity;
                System.debug('Updated Stock Quantity: ' + inventory.Stock_Quantity__c);
            }
        } else {
            System.debug('Product not found in Inventory Map for Sales Order Product__c: ' + so.Product__c);
        }
    }

    // Update inventory records in bulk
    if (!inventoryList.isEmpty()) {
        try {
            update inventoryList;
            System.debug('Inventory records updated successfully.');
        } catch (Exception e) {
            System.debug('Error updating inventory: ' + e.getMessage());
        }
    }
}

