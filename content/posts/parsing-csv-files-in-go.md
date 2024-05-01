---
title: "Parsing CSV Files in Go"
date: 2024-04-30T22:16:22-04:00
summary: "In this post, we'll dive into processing CSV files with Go and its standard library."
tags: ["go", "csv"]
categories: ["go"]
---
## Introduction

In this post, we'll dive into processing CSV files in Go. This is particularly useful in data-driven projects. A recent example involved migrating a company off of managing and maintaining a myriad of spreadsheets and PDF reports to a data-driven application. This centralized the management of data to a single client-facing interface, facilitated the quick lookup of data without having to search and navigate through directories, files, workbooks, and sheets, preventing duplicates and bad data upon data entry, prevented incongruent changes of the same data across different machines, and leveraged the data to generate insights that drive business decisions.

Ok so now that we've touched a bit on the value of processing CSV data, let's look into the CSV format a little closer.

## Comma Separated Values

The CSV format was created to help exchange tabular data between various programs that have incompatible formats. For example, between spreadsheets and databases. A typical CSV file looks like this:

```csv
pn,description,price,qty
32-021,actuator,460.75,10
43212,washer,0.10,24
542-21,valve,21.00,2
```

Let's try to formalize a definition:

* Each record is placed in it's own line followed by a line break
* Each record may contain one or more delimited fields
* The first line may or may not contain a header
* Each line must contain the same number of fields in the file (whether this is defined in the header or implicit)

A caveat I'd like to point out is that there isn't one single standard but we'll be working with the definition laid out in  [RFC 4180](https://rfc-editor.org/rfc/rfc4180.html).

## Go standard lib encoding/csv

One of the best things about Golang is its standard library, where you can find almost anything you need. This greatly reduces the overhead cost of third-party dependencies, which brings me to [encoding/csv](https://pkg.go.dev/encoding/csv) package.

### Simple parsing

```go
package main

import (
  "encoding/CSV"
  "fmt"
  "log"
  "os"
) 

func main() {
  file, err := os.Open("inventory.csv")
  if err != nil {
    log.Fatal(err)
  } 
  
  defer func () {
    if err := file.Close(); err != nil {
      log.Fatal(err)
    }
  }()
  
  r := csv.NewReader(file)
  records, err := r.ReadAll()
  if err != nil {
    log.Fatal(err)
  }
  
  fmt.Println(records)
}
```

Output

```text
[[pn description price qty] [32-021 actuator 460.75 10] [43212 washer 0.10 24]]
```

Awesome, we're able to read data from the CSV.
Breaking the process down:

1. We first ensure we can open the `inventory.csv` file.
2. Followed by properly handling the closing of the file at the end of the function call stack. The `defer` keyword helps achieve this implicitly although we could have just as well placed the `file. Close()` at the end of the function body. It's important to always handle any possible error we might get from closing a file, hence the closure and not simply calling `defer file.Close()` which would've discarded any error returned.
3. We create a new CSV reader which is an implementation of the `io.Reader` interface. The CSV Reader implementation has options defined in the type that dictates how to interpret the CSV data but it's important to note that it expects CSV data formatted as defined by [RFC 4180](https://rfc-editor.org/rfc/rfc4180.html). So our file is loaded into this reader. We could configure it further (e.g. `r.comma = ';'` to specify that our delimiter will be a semi-colon and not a comma) if we wanted to.
4. We then read all the lines through to the end of the file into `records`.
5. Finally we print our CSV data to stdout and the file is implicitly closed.

As you can see it's really simple to work with CSV files in Go. So let's take this a step further and iterate. What if we had a file with duplicate records and we're told that we can simply discard any dupe(s)?

### Cleaning parsed data

```go
package main

import (
    "encoding/csv"
    "fmt"
    "log"
    "os"
)

func main() {
    records, err := parseCsvFile("inventory.csv")
    if err != nil {
        log.Fatal(err)
    }

    uniqnessMap := make(map[string]bool)
    var dedupedRecords [][]string
    dupeCount := 0
    for _, r := range records {
        if !uniqnessMap[r[0]] {
            dedupedRecords = append(dedupedRecords, r)
            uniqnessMap[r[0]] = true
        } else {
            dupeCount++
        }
    }

    fmt.Printf("Dupe count: %v\n", dupeCount)
    fmt.Println(dedupedRecords)
}

func parseCsvFile(path string) ([][]string, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, err
    }

    r := csv.NewReader(file)
    records, err := r.ReadAll()
    if err != nil {
        return nil, err
    }

    if err := file.Close(); err != nil {
        return nil, err
    }

    return records, nil
}

```

So here we've extracted our parsing logic into a named function, invoked it to get the CSV data, and deduped the records by. Let's break this down:

1. We first initialized a uniqueness map to help us identify duplicates.
2. Then declared a slice to track our deduped CSV data.
3. Initialized a counter to zero to provide a user with some feedback, namely how many duplicates were found. This is a minor addition that could be expanded to track the duplicates in their entirety and diff on what fields were changed between them. For the sake of simplicity we are assuming the data is identical between dupes and deduping based on unique part number.
4. Lastly we proceed to iterate over the records, marking any records we haven't seen and appending them to our deduped slice. If we have seen the record we don't append it to the deduped slice and increment the counter by one

Great, we're able to operate on some basic common tasks with our CSV data, but we're missing out an opportunity to leverage one of Go's greatest strengths. Type safety. Let's look at how we can be a little stricter with the data we're parsing.

### Safely parsing CSV data

```go
type InventoryRecord struct {
    PartNumber  string
    Description string
    Price       float64
    Quantity    int
}

func parseCsvFile(path string) ([]InventoryRecord, error) {
  ...
  records, err := r.ReadAll()
    if err != nil {
      return nil, err
    }
  inventoryRecords, err := parseInventoryRecords(records[1:])
  ...
}

func parseInventoryRecords(records[][]string) ([]InventoryRecord, error) {
    for _, record := range records[1:] {
        inventoryRecord, err := createInventoryRecord(record)
        if err != nil {
            return nil, err
        }
        parsedInventoryRecords = append(parsedInventoryRecords, inventoryRecord)
    }

    return parsedInventoryRecords, nil
}

func createInventoryRecord(record []string) (InventoryRecord, error) {
    if record[0] == "" {
        return InventoryRecord{}, fmt.Errorf("invalid record: part number is empty")
    }

    var price float64
    var qty int

    if record[2] != "" {
        price, err := strconv.ParseFloat(record[2], 64)
        if err != nil {
            return InventoryRecord{}, fmt.Errorf("failed to parse price for part number %s: %v", record[0], err)
        }
        if price < 0.01 {
            return InventoryRecord{}, fmt.Errorf("price for part number %s is less than 0.01", record[0])
        }
    }

    if record[3] != "" {
        qty, err := strconv.Atoi(record[3])
        if err != nil {
            return InventoryRecord{}, fmt.Errorf("failed to parse quantity for part number %s: %v", record[0], err)
        }
        if qty < 1 {
            return InventoryRecord{}, fmt.Errorf("quantity for part number %s is less than 1", record[0])
        }
    }

    return InventoryRecord{
        PartNumber:  record[0],
        Description: record[1],
        Price:       price,
        Quantity:    qty,
    }, nil
}
```

We've added a few things to help make our process more robust:

1. There's now an InventoryRecord struct type that will help enforce the types expected from each field.
2. We've added a new function `parseInventoryRecords()` to encapsulate the logic for the relevant lines and ignore the header.
3. In our `createInventoryRecord()` function we carry out basic data validation to ensure we're not accepting bad data.
   1. We start by checking for an empty part number. If we encounter a record with this it's most likely not a good sign and we should throw and alert the user.
   2. We then do an empty check on the price, followed by a type conversion, and finally a value check. In essence, we're checking if we have a price value that makes sense and what 'makes sense' is very much derived from the requirements so your mileage may vary depending on what is needed.
   3. We follow a similar strategy for quantity.
   4. Finally if the record we're parsing passes all our checks we can return typed record which we can then safely operate on in our program.

The big win here is that we're able to parse our CSV data safely, we know that if the program parses the CSV file without error there's a higher degree of confidence in the integrity of our data (not quite 100% but it's an improvement over our first example). This leads to a much better programming experience.

While processing CSV files synchronously is enough in most cases, there are cases where concurrency makes sense.

### Concurrency

Concurrency involves composing a program in a way that allows for multiple tasks to be executed independently in an efficient manner. In other words, we want to be more efficient with our resources by interweaving the execution of tasks rather than executing them sequentially.

Consider a scenario where we would need to process several large CSV files stored on a remote server. Without concurrency, we would typically read and process each file synchronously, waiting for one file to be read and processed before moving on to the next. Blocking the program for the duration of the time it takes to read and process all the files. When the time it takes to do this becomes a bottleneck for the rest of our program we definitely want to start to look for ways to reduce it. By introducing concurrency, we can overlap the execution of I/O-bound tasks with CPU-bound tasks and reduce the time it takes to complete these tasks.

```go
package main

import (
    "encoding/csv"
    "fmt"
    "log"
    "os"
    "sync"
)

func main() {
    files := []string{"inventory1.csv", "inventory2.csv", "inventory3.csv"}

    var wg sync.WaitGroup
    wg.Add(len(files))

    for _, file := range files {
        go func(file string) {
            defer wg.Done()

            if err := processCsvFile(file); err != nil {
        log.fatal(err)
            }
        }(file)
    }

    wg.Wait()
  fmt.Println("Finished parsing files.")
}

```

Let's see what changed to achieve this:

1. We start by initializing a slice to keep track of the file paths.
2. We then declare a wait group and give it the number of goroutines to wait for to finish before it unblocks. More specifically, a wait group is type of counter which we increase by calling `Add()`. Once this counter becomes zero, it'll release all the goroutines on `Wait()` and unblock the execution.
3. Iterating through our file paths slice we wrap our goroutines with a closure that has 2 simple tasks, process each CSV file and let the wait group know when it's done decreasing its counter. (For the sake of simplicity we're sticking to the parsing of the CSV file, but ideally, we'd have a justifiable number of expensive tasks).
4. We then block the execution of the program until all the wait group's goroutines are done.
5. Lastly we notify the user that all the files were processed.

### Going further

So we've covered quite a few ways we can work with CSV data in Go. If you're interested in seeing a data-driven application that builds on what was touched here be sure to check out my next post.
