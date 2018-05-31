Good Practices for Designing Record Types
=========================================

This guide aims at helping you leverage the best of GRDB [records] and [associations].

Since GRDB sits right between low-level libraries like SQLite itself or [FMDB], and high-level ORM like [Core Data] or [Realm], you may face questions when designing the model layer of your application.

Let's say we are designing a library application, which lets the user crawl through books and their authors.

If we were to design types that implement this library in memory, we may write something along:

```swift
class Author {
    var name: String
    var books: [Book]
}

class Book {
    var title: String
    unowned var author: Author
}
```

And we would then extend those types with the GRDB [record protocols] that provide handy fetching and persistence methods.

This approach looks pretty at first sight. Unfortunately, it embeds a few flaws that hinder the development of our application. We will explain why. And we will then drift away from this initial design, keep the library application focused on its tasks, and preserve a clean dispatching of responsabilities in our application types.


## A few Design Principles and their Consequences

The general **[Separation of Concerns]** principle states that a program that is split into several sections focused on well-defined *concerns* is easier to develop and maintain. Basically, it is the [Divide and Conquer] strategy applied to computer programs, in order to keep them under strict control of the developer.

You can see this principle live in popular and well-known application architecture patterns such as MVC or MVVM. In our library application, we'd like concerns such as thread-safety, data consistency, or cache management to be addressed in proper places. We'd also like that independent features of our application, such as crawling the library, or focusing on a given book, as well as future features, can be developed independently and without undesired overlapping or coupling.

More practically, the **[Single Responsibility Principle]** prefers that our application functionalities are embedded in dedicated types, fully responsible for their tasks. Easy to write, this principle is surprisingly difficult to apply in practice. After all, all those tiny responsibilities have to be orchestrated and composed together: it is not always easy to draw the line between them.

Finally, we'd like to leverage the **[Single Source of Truth]** principle, that states that every piece of information is stored exactly once, with unambiguous location. After all, we are dealing with a local SQLite database, not a cluster of distributed databases spread all over the world, and this can simplify our coding practices a lot.

Let's see how our initial code violates each one of those principles:

```swift
// Not recommended
class Author: /* Record protocols */ {
    var name: String
    var books: [Book]
}

class Book: /* Record protocols */ {
    var title: String
    unowned var author: Author
}
```

To be clear, problems only emerge because of the adjunction of record protocols on the Author and Book classes. Without them, the design is perfectly fine.

- **Separation of Concerns** is violated.
    
    Let's say that a screen of our applications lists all library authors, from Shinzō Abe to Émile Zola, so that the user can crawl and search his favorite writers. According to the definition of the `Author` type, with its `books` property, we can't have an author in memory without its books. We'd thus have to make a choice:
    
    1. Obey the `Author.books` property, and fetch books in memory even if we don't use them.
    2. Spare memory and I/O accesses, by deciding that the books property sometimes contains an author's books, and sometimes is empty, *for reasons*.
    3. Redesign the Author type so that it is able to express that it comes with, or without its books. Why not also define *two* types, one for authors with books, and one for authors without books. At this point, we have acknowledged that our initial design is flawed.
    
    Another screen of our application displays a book sheet, so that the user can decide what novel he will bring with him during his next vacation. According to the definition of our `Book` type with its `author` property, we can't have a book in memory without its author, and all sibling books. Again, we'd have to make a choice, between spilling memory and I/O accesses because of a strict fulfilment of our model types, or cheating our types by loading partial data, or changing our types so that they accommodate our application needs.
    
    Those two examples show how the early definition of our types prevent distinct parts of our application to perform efficiently. This is because **Separation of Concerns** is violated: it is not the model layer's business to mandate how application data should be loaded in memory.
    
    Now you may wonder: but most ORM libraries allow me to load authors and their books at will. They simply perform lazy loading in order to prevent memory spilling and useless database accesses. Doesn't GRDB do that?
    
    No, GRDB does not perform lazy loading. Lazy loading either requires record to be *managed* (as in [Core Data] and [Realm]), or that all data processing happens in a *single function* (think of an HTTP request handled with a web-oriented ORM like [Active Record] and [Django]). The underlying issue is *data consistency*: you always want your memory objects to accurately represent your application data, without any glitch, ever. This involves the subtle database concept of [isolation]. In a GUI application, this can't be achieved without a very complex record management, and non-trivial pain points for the application developer. GRDB has removed lazy loading from the list of desirable features, on purpose. See the "Solving Problems" chapter of [Why Adopt GRDB?](WhyAdoptGRDB.md#solving-problems) for more information.

- **Single Responsibility Principle** is having a hard time.
    
    To see how, let's first look at an innocuous piece of code:
    
    ```swift
    // Insert an author and a book in the database:
    let melville = Author(name: "Hermann Melville")
    let mobyDick = Book(author: melville, title: "Moby-Dick")
    melville.books.append(mobyDick)
    try dbQueue.write { db in
        try mobyDick.insert(db) // insert book
        try melville.insert(db) // insert author
    }
    ```
    
    It has a bug: you can't insert the book in the database before the author, because a well-behaved relational database won't accept to store a book before its author. You thus change the order of insertions:
    
    ```swift
    try dbQueue.write { db in
        try melville.insert(db) // insert author
        try mobyDick.insert(db) // insert book
    }
    ```
    
    It works, but now you realize that this way of coding is fragile, since it is so heavily dependent on proper ordering of operations. You thus decide that the only way to make your program robust is to make an author responsible for inserting its books. You provide a custom implementation for `Author.insert`, and you write:
    
    ```swift
    try dbQueue.write { db in
        try melville.insert(db) // inserts author, and its books
    }
    ```
    
    Fine. Now you apply the principle of single responsability, and conclude that *an author is responsible for managing its books in the database*. Why so? Because you don't want ever to wonder whether database book management should be performed by authors or books. You don't want one part of the application to rely on books, and another part to rely on authors. When a new fellow coworker asks you "who is saving books?", you don't want to answer "it depends."
    
    And this doesn't work:
    
    ```swift
    // Moby-Dick is exhausted
    melville.books.remove(mobyDick)
    try dbQueue.write { db in
        try melville.save(db) // ???
    }
    ```
    
    Since melville has no longer any reference to mobyDick, it can't handle its deletion. For this code to work, we'd have to modify the Author type so that it keeps its deleted books somewhere.
    
    This other technique does not work either, because it would be impractical: you'd always need a database connection in order to change an author's books:
    
    ```swift
    // Moby-Dick is exhausted
    try dbQueue.write { db in
        try melville.delete(book: mobyDick)
    }
    ```
    
    On top of that, it would be difficult to change a book's author, in case of mistake: which author would be responsible for the change? Of course, the only working code, given our initial design, is below. So who is responsible of books in the database, actually? *It depends*:
    
    ```swift
    // Moby-Dick is exhausted
    melville.books.remove(mobyDick)
    try dbQueue.write { db in
        try mobyDick.delete(db)
    }
    ```
    
    Again, you may wonder: several ORM libraries allow inserting, modifying, and deleting records while keeping related records in sync. Doesn't GRDB allow that? Indeed, Core Data does that perfectly. Realm does too, to some extent. Both *manage* records, a path that GRDB has not chosen. Active Record does also manage relations, but synchronization is not fully automatic. Anyway, this *can not* be achieved with a plain `Author.books` Array property: we'll have to find another way.
    
    We'll see later how a more robust splitting of responsibilities will solve this issue.
    

- **Single Source of Truth** is at stake.

    When you ask yourself the question: "what are an author's books", where is the answer? Is it in the database, or is it in the `books` property of an Author instance? Considering that `Author.books` is a mutable array, and that Author is a class whose instances can remain in the memory long after they have been fetched, long after the database has been modified by other components of the application, the answer is not always trivial. And our current design makes it easy to process obsolete values.
    
    The answer to this question, in other ORM libraries, is called "auto-updating records". They make sure, whenever you grab a record object, that it always contains the latest values from the database. Except that it is not exactly true as soon as your application is multi-threaded: each thread of your application has a different view of the database, with as many opportunities for inconsistency and unexpected values as your application tasks hop from one thread to another. GRDB has removed auto-updating records from the list of desirable features, on purpose. See the "Solving Problems" chapter of [Why Adopt GRDB?](WhyAdoptGRDB.md#solving-problems) for more information.



[records]: ../README.md#records
[associations]: AssociationsBasics.md
[FMDB]: https://github.com/ccgus/fmdb
[Core Data]: https://developer.apple.com/documentation/coredata
[Realm]: https://realm.io
[Active Record]: http://guides.rubyonrails.org/active_record_basics.html
[Django]: https://docs.djangoproject.com/en/2.0/topics/db/
[record protocols]: ../README.md#record-protocols-overview
[Separation of Concerns]: https://en.wikipedia.org/wiki/Separation_of_concerns
[Single Responsibility Principle]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[Single Source of Truth]: https://en.wikipedia.org/wiki/Single_source_of_truth
[Divide and Conquer]: https://en.wikipedia.org/wiki/Divide_and_rule
[Why Adopt GRDB?]: WhyAdoptGRDB.md
[isolation]: https://en.wikipedia.org/wiki/Isolation_(database_systems)