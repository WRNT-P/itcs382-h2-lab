# Lab Report, Designing the Data Management Layer with H2

Repository URL: https://github.com/WRNT-P/itcs382-h2-lab

## Tools

| Item | Version or name |
| --- | --- |
| JDK | Temurin 25.0.4.1 |
| Maven | 3.10.0 |
| IDE | Visual Studio Code 1.145.2 |
| AI tool and exact model, if you used one | Claude Opus 5.5 |

## Part 0. From a store in memory to H2

Commit of the starter, before any change: abceff7

0.3 Your prediction for `staff`, written before the second run:

I think `staff` will show nothing and all 3 lockers will be free, because the parcels are only kept while the program is running.

Output of the second run, for `staff` and `quit`:

```text
> staff
Free lockers: 3
> quit
```

State in one sentence where you think the parcels of the first run were kept.

They were kept in a Map inside InMemoryParcelRepository, which is only in memory, so they were gone once the program ended.

0.5 The line of Main that you would change to keep the parcels somewhere else:

```java
ParcelRepository repository = new InMemoryParcelRepository();
```

0.6 Commit of the switch to H2: ca16f64

0.7 Output of `SELECT * FROM PARCEL;` after you stored two parcels:

```text
sql> SELECT * FROM PARCEL;
CODE  | STUDENT_ID | PARCEL_SIZE | STATUS | LOCKER_NUMBER
P1001 | S1001      | SMALL       | STORED | 1
P1002 | S1002      | LARGE       | STORED | 3
(2 rows, 14 ms)
```

0.8 Explain why ParcelPoint could not start while the H2 Shell was open, and identify the class that wrote the message `Cannot create the PARCEL table`.

The H2 Shell still had the database open. H2 is embedded, so whichever program opens it first locks parcelpoint.mv.db, and a second program can't get in (the error says `The file is locked`). ParcelPoint tries to connect in the H2ParcelRepository constructor to create the table, so it failed right at startup. The message `Cannot create the PARCEL table` comes from H2ParcelRepository, not from H2. Its constructor catches the SQLException and throws an IllegalStateException with that text, and the stack trace points to `H2ParcelRepository.<init>`.

0.9 Explain why `staff` shows no parcel while `find` shows each one, and why all three lockers look free.

`find` uses findByCode, which actually runs a SELECT on the PARCEL table, so it finds the parcels. `staff` uses findAll, but in the starter findAll just returns `List.of()` without asking the database, so nothing is listed. The lockers look free for the same reason. When the program starts, the ParcelService constructor calls `findAll()` to put the waiting parcels back into their lockers, and since it gets an empty list, every locker stays empty.

0.11 Your findAll method:

```java
    @Override
    public List<Parcel> findAll() {
        String sql = "SELECT * FROM PARCEL ORDER BY CODE";
        List<Parcel> parcels = new ArrayList<>();
        try (Connection connection = connect();
             PreparedStatement statement = connection.prepareStatement(sql)) {
            try (ResultSet row = statement.executeQuery()) {
                while (row.next()) {
                    parcels.add(toParcel(row));
                }
            }
        } catch (SQLException e) {
            throw new IllegalStateException("Cannot read the parcels", e);
        }
        return parcels;
    }
```

Commit that adds findAll: 0be8e97

0.12 Output of `SELECT * FROM PARCEL;` after the pickup:

```text
sql> SELECT * FROM PARCEL;
CODE  | STUDENT_ID | PARCEL_SIZE | STATUS    | LOCKER_NUMBER
P1001 | S1001      | SMALL       | PICKED_UP | null
P1002 | S1002      | LARGE       | STORED    | 3
(2 rows, 13 ms)
```

0.13 Optional. Describe what `find P9999'OR'1'='1` showed while the SQL was joined from strings, and what it showed after you put the placeholder back.

With the SQL built from strings, `find P9999'OR'1'='1` printed `P1001, S1001, SMALL, PICKED_UP` even though no parcel has that code. My quotes ended the string early and added `OR '1'='1'`, which is true for every row, so every row came back and findByCode showed the first one. After putting the `?` back, the same input gave `No parcel with code P9999'OR'1'='1`, because H2 now treats the whole input as one value to compare with CODE.

## Part A. Model the design

A1 Complete the flowchart with one node for each package and one arrow for each dependency, from the package that uses to the package that is used. Mark the package that accesses the data store.

```mermaid
flowchart TB
    parcelpoint
    cli
    service
    domain
    storage["storage, accesses the data store"]
    parcelpoint --> cli
    parcelpoint --> service
    parcelpoint --> storage
    cli --> service
    cli --> domain
    service --> domain
    service --> storage
    storage --> domain
```

State whether cli depends on storage, and explain why.

No. ParcelCommandLine only imports from domain and service, never from storage. It gets parcels by calling ParcelService, and only ParcelService talks to the repository. So we can swap the data store without touching cli.

A2 Complete the class diagram of the data management layer with ParcelService, the interface ParcelRepository with its three operations, and the two classes that implement it.

```mermaid
classDiagram
    class ParcelService {
        -ParcelRepository repository
        +ParcelService(ParcelRepository repository)
        +store(Parcel parcel) Optional~Locker~
        +pickUp(String code, int days) Optional~Double~
        +find(String code) Optional~Parcel~
        +allParcels() List~Parcel~
        +countFreeLockers() long
    }
    class ParcelRepository {
        <<interface>>
        +save(Parcel parcel) void
        +findByCode(String code) Optional~Parcel~
        +findAll() List~Parcel~
    }
    class InMemoryParcelRepository {
        -Map parcels
        +save(Parcel parcel) void
        +findByCode(String code) Optional~Parcel~
        +findAll() List~Parcel~
    }
    class H2ParcelRepository {
        -String url
        +H2ParcelRepository(String path)
        +save(Parcel parcel) void
        +findByCode(String code) Optional~Parcel~
        +findAll() List~Parcel~
        -connect() Connection
        -toParcel(ResultSet row) Parcel
    }
    ParcelService "1" --> "1" ParcelRepository : repository
    ParcelRepository <|.. InMemoryParcelRepository
    ParcelRepository <|.. H2ParcelRepository
```

A3 Draw a sequence diagram for the command `store P1003 S1003 MEDIUM`, from the staff member to the repository. Show each message as an operation of the class that receives it.

```mermaid
sequenceDiagram
    actor Staff
    participant cli as :ParcelCommandLine
    participant parcel as :Parcel
    participant service as :ParcelService
    participant locker as :Locker
    participant repository as :ParcelRepository
    Staff->>cli: store P1003 S1003 MEDIUM
    cli->>parcel: Parcel("P1003", "S1003", MEDIUM)
    cli->>service: store(parcel)
    loop for each Locker, until one fits
        service->>locker: fits(parcel)
        locker-->>service: true or false
    end
    service->>locker: setParcel(parcel)
    service->>locker: getNumber()
    locker-->>service: 2
    service->>parcel: storeIn(2)
    service->>repository: save(parcel)
    service-->>cli: Optional of locker 2
    cli-->>Staff: Parcel P1003 is in locker 2
```

Explain why the last lifeline is the interface ParcelRepository rather than H2ParcelRepository.

ParcelService only knows its field as a ParcelRepository and never mentions H2ParcelRepository, so `save` is sent to the interface. Which class is actually behind it is decided in Main. That way the diagram stays correct whether we run with the in-memory store or H2, and it doesn't show a dependency that ParcelService doesn't really have.

## Part B. Design questions

1. Identify the files that you changed to move from the store in memory to H2, and explain why no other class had to change. Use the terms of this course.

   I changed Main.java (two lines) and H2ParcelRepository.java (findAll). No other class changed because ParcelService depends on the ParcelRepository interface, not a concrete class (Dependency Inversion Principle), and only Main picks which class to use.

2. The PARCEL table stores the size of a parcel as SMALL, MEDIUM or LARGE rather than as 1, 2 or 3. Identify the type of connascence between ParcelSize and the table in each case, and explain which one is weaker.

   SMALL, MEDIUM, LARGE is connascence of name, while 1, 2, 3 is connascence of convention (both sides must agree 1 means SMALL), or of position if the numbers follow the enum order. Name is weaker, because the value is readable and a wrong name fails loudly, while reordering the enum would silently change what every number means.

3. Project Phase 2 asks for a data store that needs no separate server. Explain why an embedded database such as H2 meets this rule, and describe the limit of an embedded database that Step 0.8 showed.

   H2 runs inside the program as a library and keeps its data in a file, so there is no server to install or start. The limit is that only one program can open it at a time, so in Step 0.8 ParcelPoint could not start while the H2 Shell had the file locked.

4. State the data store that you would choose for your Project Phase 2 team, files or an embedded database, the requirement that drives the choice, and the package that will access it.

   I would choose files, because the requirement is that data stays after the program ends with no separate server, and files do that with no extra library while staying easy to open and check by hand. If the data grows and we need it organised for long-term use, I would move to H2, since querying tables is easier than parsing files. Either way only the storage package would access the data store, behind a repository interface, so the switch would only touch that package and Main.

## AI usage declaration

| AI tool and exact model | Date | Purpose | Files or answers it produced or changed | How you checked its output |
| --- | --- | --- | --- | --- |
| Claude Opus 5.5 | 11-12 Nov 2026 | Walked me through Part 0 step by step and explained each step | Git setup commands, the two line change in Main.java, findAll in H2ParcelRepository.java, the break and repair of findByCode in Step 0.13 | I built and ran the program after every change and compared the output with the lab sheet. After findAll, `staff` listed both parcels, and I checked the table in the H2 Shell |
| Claude Opus 5.5 | 11-12 Nov 2026 | Helped write the report answers and the Part A diagrams | Answers 0.3, 0.5, 0.8, 0.9, 0.13, A1 to A3 and Part B 1 to 4 (4 is my own choice, which it wrote up) | I checked the answers against the code and the error messages I got, matched the package arrows to the import lines, and checked that the diagrams render on GitHub |
