# LucaP ERP Part 7

We're building an ERP from scratch. Last time, we introduced CRUD (Create, Read, Update, Delete), and now we'll apply it to the Chart of Accounts. However, we must solve a few problems to make this work: error handling and minimizing dependencies. For us to understand this, we'll need some basic knowledge of app architecture.

Any program has two parts: one that manipulates data and the other that gets and sends that data to the user. We call the data manipulation part a *Model*, and the user interaction part a *View*. Sometimes, a third part called a *Controller* works between those two, but for our purposes, I'm going to ignore that. 

In code, models are where we interact with tables in databases. All of the CRUD interactions happen here. In contrast, all our input fields, buttons, and reports occur in a view. The goal of this separation is *encapsulation*, a big word for making code as simple as possible to connect one part to another. While most people talk of Legos, It's more like the ports on your computer. You know what you plug in will work because it is a certain shape. The headphone jack is for sound, and the USB ports are for storage drives, mice, and keyboards. Contrast that to many laptops that have nonstandard power adapters. It's much nicer when your laptop takes a USB-C power port. Same with the model. I want it to be as independent as possible; that way, I can plug it into as many views as I like without much work. All I want is to ask the Model to add, display, update, and delete rows from the table in the view. The model does the hard work. 

This is true between views as well. Views are often in a hierarchy, again letting you reuse parts for more than one view. Here's our hierarchy so far in Luca ERP

Subviews like the toolbars could be reused if I had other views that use them. I want them to be as easy to connect as possible, so I made up two types, called enumerations, which have a limited set of values that describe what the toolbars are doing. 

One of these, **CRUDAction**, has values to indicate a table change.

```
enum CRUDAction:String,CaseIterable{

    case add = "Add"
    case update = "Update"
    case delete = "Delete"
    case search = "Search"
    case view = "View"
}    
```

The other **NavigationAction**, navigate through the table. 

```
enum NavigationAction:String,CaseIterable{
    case first = "First"
    case previous = "Previous"
    case next = "Next"
    case last = "Last"
    case noAction = "" //value for nothing happening
```


I added extra methods to these types so they have associated icons if I use them in a button. 

I have a variable for each of these two in my navigation toolbar code for the top of the view. I make some loops to go through each of these values to make buttons. The button's only purpose is to set one of those two variables to the correct value. Here's the one for the **CRUDAction**, setting a variable **displayMode**. 

```
ForEach (CRUDAction.allCases,id:\.self){ item in
    Button{
        displayMode = item
    } label: {
        item.icon
    }
    .fileNavButtonModifier(
        label:item.rawValue,
        disabled:displayMode == item)
}
```

 
I did the same for the navigation buttons. This creates a toolbar like this: 



The bottom toolbar Reads the **CRUDAction** and sets the buttons accordingly. 

```
Button{
                doAction = true
            } label:{
                Text(action.rawValue)
            }
            .buttonModifier
```

The form responds to changes in the variables set. I used two different strategies here. 
For navigation, when **navigationAction** changes, I check if it is not **.noAction**, indicating an action. I do an action as decided in a parsing table **rowAction()** when there is an action.  

```
.onChange(of:navigationAction){
            if navigationAction != .noAction{
                rowNavigation()
            }
```

For the **CRUDActions**, any change triggers a check of the parsing table for the display's appearance.

What do I mean by a parsing table? That's a set of logic that decides which command to do and then executes it. Each command may affect the view, the model, or both. 

For example, when I press the button for the first row, it parses to this code. 

```
case .first:
    selectedAccount = chartOfAccounts.firstAccount()
    message = "First Account"
```

It retrieves the first row and then sends a message via the bottom toolbar that this is the first account. 

If I tap the search button, it makes changes to the view 

```
case .search:
            selectedAccount = .blank
            formActive = false
            keyActive = true
            message = "Search for account number"
```

For a search, I blank the fields and make all but the key field read-only so that I can search by the key. 

I also have a second trigger for the actions. **doAction** connects to the button on the bottom toolbar. When true, it launches a parsing table of actions for running a CRUD operation. However, before we look at that parsing table, we have another set of issues: the model communicating with the view. 

All of our CRUD methods are in the model. Implementing them to communicate with encapsulation to any view connecting to this model presents some challenges. Let's explore this with the **findRow** code we discussed last time

```
func findRow(key:Int)->(key:Int,name:String){
	let index = findIndex(key: key)
	if !isNull(index:index) {
    return myList[index]
  }
	print("row with key \(key) not found")
	return nullRow
}
```

There are significant changes in adopting this over to the **ChartOAccounts**. First, the **print** statement was how I signified an error. But **print** is a view, not a model. I need a standard way in my code to pass this up to the model 

While there are many ways of handling this, I'm going to do the conceptually simplest. I'll pass the error back to the view when **find** executes and let the view deal with printing the error message. Like above, I have an enumeration of values that signify errors. If the row is not found, I send the code **.recordNotFound**. If found, I send **.noError** back to the view.   

When I was looking for arrays, all that stuff with **index** made sense. However, some methods get me to the row in the table directly. In Swift, I can find the row for the table in one statement with a predicate 

```
account = accounts.first(where:{row in row.accountNumber == accountNumber })
``` 

As keys are unique, the first **accountNumber** I found is the record I want. Everything else in this method is error handling. 

```
 func find(accountNumber:String) -> (account:Account,errorType:ErrorType){
        if let account = accounts.first(where:{row in row.accountNumber == accountNumber }){
            return (account,.noError)
        } else {
            return (nullAccount,.recordNotFound)
        }
    }
```

I'm returning both the account and the error. My parsing table in the view will handle that. Here's the code 

```
case .search:
    let result = chartOfAccounts.find(accountNumber:accountNumber)
    errorType = result.errorType
    if errorType == .noError{
        selectedAccount = result.account
        message = "\(accountNumber) found"
    } else {
        message = "\(accountNumber) not found"   
    }
```

I get a result from **chartOfAccounts.find(accountNumber:)**, which has the error type and the row. If there is no error, I assign the row to the selected account, which then displays the row on my view. It also sets a variable **message** 

Now, we loop back to the Bottom toolbar. I set that up so the leading half uses it for messages. There is one view inside that view that shows the message. If there is an error, it shows an error message and icon. 

With that, we completed the search function of Luca ERP. All the other functions are added the same way. Take a look at the code to see everything in place. Once all of that is working, you have a functional form. 

We've spent a lot of time on this for a very important reason: this is the template for everything that comes after it. By structuring my code as I have, I've made it easy to write anything like **LPChartofAccountsForm**, as illustrated below with **LPFormView**. I code for the green parts only. I attach a model and modify the display of the columns for the model; everything else remains the same. 

There is one more structural thing we have to do. Many forms, like the general journal, have a dependent table, often referred to as a child table, that needs to be there, too. In two weeks, we'll modify the code here to make the last template we need for the basic LucaP ERP.

the whole code

I've only shown some of the code in this newsletter. Look at the code I've posted on GitHub to see everything implemented. It has all four parts of CRUD, which I didn't code here, plus the full tables. You'll find it on [GitHub here]()  

