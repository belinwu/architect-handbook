# Concurrency

* Overview
* Essential Problems
  * Lost Updates
  * Inconsistent Reads
* Execution Contexts
  * Request
  * Session
  * Process
  * Thread
  * Transactions
* Isolation and Immutability
* Optimistic and Pessimistic Concurrency Control
  * Preventing Inconsistent Reads
  * Deadlocks
* [Transactions](../../databases/../../databases/transactions.md)
  * Transactional Resources
  * Lock Escalation
  * Reducing Transaction Isolation for Liveness
  * Business and System Transactions
* Patterns of Offline Concurrency Control
  * Offline Locks
  * Coarse-Grained Locks
  * Implicit Locks
* Application Server Concurrency
  * Process-per-Session
  * Pooling Process-per-Request
  * Thread-per-Request

## Overview

Whenever you have multiple processes or threads manipulating the same data, you run into concurrency problems.

> "One of the great ironies of enterprise application development is that few branches of software development use concurrency more yet worry about it less. The reason enterprise developers can get away with a naive view of concurrency is *transaction managers*" - Martin Fowler & David Rice

Transactions provide a framework that helps avoid many of the most tricky aspects of concurrency in an enterprise application. As long as you do all your data manipulation within a transaction, nothing really bad will happen to you.

> "Sadly, this doesn't mean we can ignore concurrency problems completely, for the primary reason that many interactions with a system can't be placed within a single database transaction." - Fartin Fowler & David Rice

This forces us to manage concurrency in situations where data spans transactions. The term we use is **offline concurrency**, that is, concurrency control for data that's manipulated during multiple database transactions.

Another complicated area is **application servers**, supporting multiple threads in an application server system. For this you can use server platforms that take caare of much of it for you.

## Essential Problems

> Concurrency problems occu when more than one active agent, such as a process or thread, has access to the same piece of data.

Problems will be illustrated with examples from source code control systems used by teams to coordinate changes to a code base.

Both problems cause a failure of **correctness** (or safety), and they result in incorrect behavior that would not have occurred without two people trying to work with the same data at the same time.

The **essential problem** of any concurrent programming is that it's not enough to worry about correctness; you also have to **worry about liveness**: how much concurrent activity can go on. Often people need to sacrifice some correctness to gain more liveness, depending on the seriousness and likelihood of the failures and the need for people to work on their data concurrently.

### Lost Updates

Say Martin edits a file to make some changes to the `checkConcurrency` method, a task that takes a few minutes. While he's doing this, David alters the `updateImportantParameter` method in the same file. David starts and finishes his alteration before Martin does.

This is unfortunate. When Martin read the file it didn't include David's update, so when Martin writes the file it writes over the version that David updated and Davi's update is lost forever.

### Incosistent Reads

An *inconsistent read* occurs when you read two things that are correct pieces of information but not correct at the same time.

Say Martin wishes to know how many classes are in the concurrency package, which contains two subpackages for locking and multiphase. Martin looks in the locking package and sees seven classes. At this point he gets a phone call from Roy on some abstruse question. While Martin's answering the phone, David finishes dealing with a bug in the four-phase lock code and adds two classes on the locking package and three classes to the five that were in the multiphase package.

His phone call over, Martin looks in the multiphase package to see how many classes there are and sees eight, producing a grand total of fifteen. Sadly, fifteen classes was never the right answer. The correct answer was twelve before Davi's update and seventeen afterward. Either answer would have been correct, even if not current, but fifteen was never correct.

This problem is called an inconsistent read because the data that Martin read was inconsistent.

## Execution Contexts

> Whenever processing occurs in a system, it occurs in some context and usually in more than one.

From the perspective of interacting with the outside world, two important contexts are the *request* and the *session*.

> Server software in an enterprise application sees both requests and sessions from two angles, as the server from the client and as the client to other systems. Thus, you'll often see multiple sessions: HTTP sessions from the client and databases sessins with various databases.

### Request

A *request* corresponds to a single call from the outside world which the software works on and for which it optionally sends back a response.

During a request the processing is largely in the server's court and the client is assumed to wait for a response. Some protocols allow the client to interrupt a request before it gets a response. More often a client may issue another request that may interfere with one it just sent. So a client may ask to place an order and then issue a separate request to cancel that order.

From the client's view the two requests may be obviously connected, but depending on your protocol that may not be so obvious to the server.

### Session

A *session* is a long-running interaction between a client and a server. It may consist of a single request, but more commonly it consists of a series of requests that the user regards as a consistent logical sequence.

Commonly a session will begin with a user logging in and doing various bits of work that may involve issuing queries and one or more business transactions. At the end of the session the user logs out, or he may just go away and assume that the system interprets that as logging out.

### Process

Usuall a heavyweight execution context that provides a lot of isiolation for the internal data it works on.

### Thread

Lighter-weight active agent that's set up so that multiple threads can operate in a single process, favoring a good utilization of resources.

However, threads usually share memory, and such sharing leads to concurrency problems.

Some environments allow you to control what data a thread may access, aallowing you to have **isolated threads** that don't share memory.

### Transactions (database context)

Transactions pull together several requests that the client wants treated as if they were a single request. They can occur from the application to the database (a system transaction) or from the user to an application (a business transaction).

## Isolation and Immutability

Two common solutions for concurrency problems: *isolation* and *immutability*.

With **isolation**, you partition the data so that any piece of it can only be accessed by one active agent. With it you arrange things so that the program enters an isolated zone, within which you don't have to worry about concurrency.

> "Good concurrency design is thus to find ways of creaing such zones and to ensure that as much programming as possible is done in one of them." - Martin Fowler & David Rice

You only get concurrency problems if the data you're sharing can be modified. So one way to avoid concurrency conflicts is to recognize **immutable data**. By identifying some data as immutable, or at least immutable almost all the time, we can relax our concurrency concerns for it and share it widely.

Another option is to separate applications that are only reading data, and have them use copied data sources, from which we can then relax all concurrency controls.

## Optimistic and Pessimistic Concurrency Control

What happens when we have **mutable data that we can't isolate**?

Let's suppose that Martin and David both want to edit the Customer file at the same time. With **optimistic locking** both of them can make a copy of the file and edit it freely. If David is the first to finish, he can check in his work without trouble. The concurrency control kicks in when Martin tries to commit his changes. At this point the source code control system detects a conflict between Martin's changes and David's changes. Martin's commit is rejected and it's up to him to figure out how to deal  with the situation.

With **pessimistic locking** whoever checks out the file first prevents anyone else from editing it. So if Martin is first to check out, David can't work with the file until Martin is finished with it and commits his changes.

> "Optimistic lock is about conflict detection while a pessimistic lock is about conflict prevention." - Martin Fowler & David Rice

The essence of the choice between optimistic and pessimistic locks is the frequency and severity of conflicts. If conflicts are sufficiently rare, or if the consequences are no big deal, you should usually pick optimistic locks because they give you better concurrency and are usually easier to implement. However, if the results of a conflict are painful for users, you'll need to use a pessimistic technique instead.

### Preventing Inconsistent Reads

Consider this situation. Martin edits the Customer class and adds some calls to the Order class. Meanwhile David edits the Order class and changes the interface. David compiles and checks in; Martin then compiles and checks in. Now the shared code is broken because Martin didn't realize that the Order class was altered underneath him.

> Some source code control systems will spot this inconsistent read, but others require some kind of manual discipline to enforce consistency, such as updating your files from the trunk before you check in.

**Pessimistic locks** have a well-worn way of dealing with this problem through read and write locks. To read data you need a read (or shared) lock; to write data you need a write (or exclusive) lock. Many people can have read locks on the data at one time, but if anyone has a read lock nobody can get a write lock. Conversely, once somebody has a write lock, then nobody else can have any lock.

**Optimistic locks** usually base their conflict detection on some kind of version marker on the data. This can be a timestamp or a sequential counter. To detect lost updates the system checks the version marker of your update with the version marker of the shared data. If they're the same, the system allows the update and updates the version marker.

Detecting an inconsistent read with optimistic locks is similar: In this case every bit of data that was read also needs to have its version marker compared with the shared data. Any differences indicate a conflict.

> controlling access to every bit of data that's read often causes unnecessary problems due to conflicts or waits on data thaat doesn't actually matter that much. You can reduce this burden by separating out data you've *used* from data you've merely read.

Another way is using **Temporal Reads**. These prefix each read of data with some kind of timestamp or immutable label, and the database returns the data as it was according to that time or label. The problem is that the data source needs to provide a full temporal history of changes, which takes time and space to process. You may need to provide this capability for specific areas of your domain logic.

### Deadlocks

A particular problem with **pessimistic techniques** is **deadlock**.

Say Martin starts editing the Customer file and David starts editing the Order file. David realizes that to complete his task he needs to edit the Customer file too, but Martin has a lock on it so he has to wait. Then Martin realizes he has to edit the Order file, which David has locked. They are now deadlocked, **neither can make progress until the other completes**.

It's very easy to think you have a deadlock-proof scheme and then fine some chain of events you didn't consider.

> "We prefer very simple and conservative schemes for enterprise application development. They may cause unecessary victims, but that's usually much better than the consequences of missing a deadlock scenario." - Martin Fowler and David Rice

#### Deal with deadlocks

There are various techniques to deal with deadlocks.

One is to have software that can detect a deadlock when it occurs. In this case you pick a **victim**, who has to throw away his work and his locks so the others can make progress. **Deadlock detection** is very difficult and causes pain for victims.

A similar approach is to **give every lock a time limit**. Once you hit that limit you lose your locks and your work, essentially becoming a victim. **Timeouts** are easier to implement than a deadlock detection mechanism, but if anyone holds locks for a while some people will be victimized when there actually is no deadlock present.

#### Prevent deadlocks

Other approaches try to stop deadlocks occurring at all. Deadlocks essentially occur when people who already have locks try to get more (or to upgrade from read to write locks). Thus, one way of preventing them is to **force people to acquire all their locks at once** at the beginning of their work and then **prevent them gaining more**.

You can **force an order on how everybody gets locks**. An example might be to always get locks on files in alphabetical order. This way, once David had a lock on the Order file, he can't try to get a lock on the Customer file because it's earlier in the sequence. At that point he essentially becomes a victim.

You can also make it so that, if Martin tries to acquire a lock and David already has one, Martin automatically becomes a victim. It's a drastic technique, but it's simple to implement.

> You can use multiple schemes. For example, you force everyone to get all their locks at the beginning, but add a timeout in case something goes wrong.

## [Transactions](../../databases/../../databases/transactions.md)

### Transactional Resources

Most enterprise applications run into transactions in term of databases. But there are plenty of other things that can be controlled using transactions, such as message queues, printers, and ATMs.

As a result, technical discussions of transactions use the term *transactional resource* to mean anything that's transactional, that is, **that uses transactions to control concurrency**.

To handle the greatest throughput, **modern transaction systems are designed to keep transactions as short as possible**. As a result the general advice is to **never make a transaction span multiple request**, these are generally known as **long transactions**.

For this reason, a common approach is to **start a transaction at the beginning of a request and complete it at the end**. This is known as **request transaction** model.

A variation on this is to open a transaction **as late as possible**. With a **late transaction** you may do all the reads outside it and only open it up when you do updates.

### Lock Escalation

When you use transactions, you need to be somewhat aware of what exactly is being locked.

For many database actions the transaction system locks the rows involved, which allows multiple transactions to access the same table. However, if a transaction locks a lot of rows in a table, then the database has more locks that it can handle and escalates the locking to the entire table, locking out other transactions.

This **lock escalation** can have a serious impact on concurrency, and it's particularly why **you shouldn't have some "object" table for data at the domain's _Layer Supertype_ level**. Such a table is a prime candidate for lock escalation, and locking that table shuts everybody else out of the database.

### Reducing Transaction Isolation for Liveness

> For more information on isolation levels read: [ACID - Isolation](../../databases/acid.md)

it's common to restrict the full protection of transactions so that you can get better liveness. This is particulary the case when it comes to handling isolation.

> We will be using the previous example of Martin counting files while David modifies them. There are two packages: locking and multiphase. Before David's update there are seven files in the locking package and five in the multiphase package; after his update there are nine in the locking package and eight in the multiphase package. Martin looks at the locking package and David then updates both; then Martin looks at the multiphase package.

If you have full isolation, you get **serializable transactions**. Transactions are serializable if they can be executed concurrently and you get a result that's the same as you get from some sequence of executing the transactions serially.

> Serializability guarantees that Martin gets a result that corresponds to compelting his transaction either entirely before David's transaction starts (twelve) or entirely after David's finishes (seventeen). Serializability can't guarantee which result, as in this case, but at least it guarantees a correct one.

Most transactional systems use the **SQL standard which defines four levels of isolation**.

**Serializable is the strongest level**, and each level below allows a particular kind of inconsistent read to enter the picture.

If the isolation level is serializable, the system guarantees that Martin's answer is either twelve or seventeen, both of which are correct. **Serializability can't guarantee that every run through this scenario will give the same result, but it always gets a consistent one** either the number before David's update or the number afterwards.

The next isolation level below serializable is **repeatable read**, which allows **phanthoms**. *Phantoms* occur when you add some elements to a collection and the reader sees only some of them.

> The case here is that Martin looks at the files in the locking package and sees seven. David then commits his transaction, after which Martin looks at the multiphase package and sees eight. Hence, Martin gets an incorrect result. Phanthoms occur because they are valid for some of Martin's transaction but not all of it, and they're always things that are inserted.

Then next isolation level is **read commited**, which allows **unrepeatable reads**.

> Imagine that Martin looks at the total rather than the actual files. An unrepeatable read allows him to read a total of seven for the locking package. Next David commits, then he reads a total of eights for the multiphase package. It's called an unrepeatable read because, if Martin were to reread the total for the locking package after David committed, he would get the new number of nine. His original read of seven can't be repeated after David's update.

It's easier for databases to spot unrepeatable reads than phantoms, so repeatable reads give you more correctness than read comitted but less liveness.

The lowest level of isolation is **read uncommited**, which allows **dirty reads**. At read uncommitted you can read data that another transaction hasn't actually committed yet.

> Martin might look at the locking package when David adds the first of his files but before he adds the second. As a result he sees eight files in the locking package. Another kind of error comes if David adds his files but then rolls back his transaction, in which case Martin sees files that were never really there.

To be sure of correctness you should always use the serializable isolation level. The problem is that it messes up the liveness of a system, so much that you often have to reduce serializability in order to increase throughput. You have to decide what risks you want to take and make your own trade-off of errors versus performance. You don't have to use the same isolation level for all transactions, so you should look at each transaction and decide how to balance liveness versus correctness for it.

### Business and System Transactions

RDBMS systems and application server transaction managers are well understood by application developers. However, a **system transaction** has no meaning to the user of a business system.

To an online banking system user a transaction consists of logging in, selecting an account, setting up some bill payments, and finally clicking the OK button to pay the bills. This is what we call a **business transaction**, and that it displays the same ACID properties a system transaction seems a reasonable expectation.

The obvious answer to supporting the ACID properties of a business transaction is to execute the entire business transaction within a single system transaction. Unfortunately, they often take multiple requests to complete, so this would result in a *long system transaction*.

This doesn't mean that you should never use *long transactions*, they actually avoid a lot of awkward problems. However, the application won't be scalable because they will turn the database into a major bottleneck. In addition, the refactoring from long to short transactions is both complex and not well understood.

For this reason many enterprise applications can't risk long transactions. In this case, you have to **break the business transaction down into a series of short transactions**. This means that you are left to your own devides to support the ACID properties of business transactions between system transactions, we refer to this as **offline concurrency**. Whenever the business transaction interacts with a transactional resource, the interaction will execute within a system transaction in order to maintain the integrity of that resource. However, **it's not enough to string together a series of system transactions to properly support a business transaction**. The business application must provide a bit of glue between them.

**Atomicity** and **durability** are the ACID properties most easily support for business transactions. Both are supported by running the **commit phase of he business transaction, when the user hits Save, within a system transaction**. Before the session attempts to commit all its changes to the record set, it first opens a system transaction, which guarantees that the changes will commit as a unit and will be made permanent. The  only potentially tricky part here is maintaining an accurate change set during the life of the business transaction. If the application uses a *Domain Model*, a *Unit of Work* can track changes accurately. Placing business logic in a *Transaction Script* requires a manual tracking of changes.

The tricky ACID property to enforce with business transactions is **isolation**. Failures of isolation lead to failures of **consistency**. Consistency dictates that a business transaction does not leave the record set in an invalid state. Within a single transaction the application's responsibility in support consistency is to enforce all available business rules. Across multiple transactions the application's responsibility is to ensure that one session deosn't step all over another session's changes, leaving the record set in the invalid state of having lost a user's work.

> As well as the obvious problems of clashing updates, there are the more subtle problems of inconsistent reads.

Business transactions are closely tied to sessions. In the user's view each session is a sequence of business transactions (unless they're only reading data), so we usually make the assumption that all business transactions execute in a single client session.

## Patterns of Offline Concurrency Control

Offline Concurrency Control means handling concurrency control that spans system transactions by yourself instead of using a transaction system.

> You should only use these techniques if you have to. If you can make all your business transactions ifit into a system transaction by ensuring that they fit within a single request, or can get away with long transactions by forsaking scalability, then do that and avoid a great deal of trouble.

### Offline Locks

*Optimistic Offline Lock* essentially uses optimistic concurrency control across the business transactions.

It yields the best liveness and you avoid managing locks on every object, but the limitation is that you only find out that a business transaction is going to fail when you try to commit it, and in some circumstances the pain of late discovery is too much.

> Users may have put an hour's work into entering details about a lease, and if you got lots of failures users lose faith in the system.

Your alternative is *Pessimistic Offline Lock*, with which you find out early if you're in trouble but lose out because it's harder to program and it reduces your liveness.

### Coarse-Grained Lock

It allows you to manage the concurrency of a group of objects together.

### Implicit Lock

It saves you from having to manage locks directly. Not only does this save work, it also avoids bugs when people forget.

## Application Server Concurrency

Another form of concurrency is the **process concurrency of the application server itself**. How does that server handle multiple requests concurrently and how does this affect the design of the application on the server?

### Process-per-Session

The simplest way to handle this is to use **process-per-session**, where each session runs in its own process. Its great advantage is that the state of **each process is completely isolated**. As far as memory isolation goes, it's almost equally effective to have each request start a new process or to have one process tied to the session that's idle between requests.

### Pooled Process-per-Request

The problem is that it **uses up a lot resources**, since processes are expensive. To be more efficient you can **pool the processes**, so that each one only handles a single request at one time but can handle multiple requests from different session in a sequence. This approach of **pooled process-per-request** will use many fewer processes to support a given number of session. Your isolation is almost as good, avoiding multithreading issues.

The main problem is that you have to ensure that any resources used to handle a request are released at the end of the request.

Although it's less efficient than thread-per-request, it is equally **scalable**. You also get **better robustness** (if one thread fails, it can bring down an entire process).

> Particularly with a less experienced team, the reduction of threading headaches (and the time and cost of fixing bugs) is worth the extra hardware costs.

### Thread-per-Request

You can further imrpvoe throughput by having a single process run multiple threads. With this approach, **each request is handled by a single thread within a process**. Since threads use much fewer server resources than a process, you can handle more requests with less hardware.

The problem is that **there's no isolation between treads** and any thread can touch any piece of data that it can get access to.

> Some environments provide a middle ground of allowing isolated areas of data to be assigned to a single thread.

If you use this approach, the most important thing is to **create and enter an isolated zone** where application developers can mostly ignore multithreaded issuies. **The usual way to do this is to have the thread create new objects as it starts handling the request** and to ensure that these objects aren't put anywhere (such as in a static variable) where other threads can see them.

> The problem with pooling objects is that you have to synchronize access in some way. You should compare this against the cost of object creation when using multithreading.

Creating fresh objects for each session avoids a lot of concurrency bugs and can actually improve scalability.

> While this tactic works in many cases, you should avoid static, class-based variables, global variables, or singletons, because any use of these has to be synchronized.
