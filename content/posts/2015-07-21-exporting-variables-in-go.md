---
categories:
- programming
date: "2015-07-21T00:00:00Z"
tags:
- go
title: Exporting variables in Go
---

This is going to be one of those classic "I'm writing this down because I know I'm going to forget and do it again" posts.

I'm currently playing around in Go, and I was wondering about using an external config file in a Go program. Heading to [Stack Overflow](http://stackoverflow.com/questions/16465705/how-to-handle-configuration-in-go) gave me several suggestions of packages and config file types, but I decided to try a basic JSON version.

(BTW, this seems likely: [Computer Programming To Be Officially Renamed "Googling Stackoverflow"](http://www.theallium.com/engineering/computer-programming-to-be-officially-renamed-googling-stackoverflow/).)

I pretty much just copied the code from the top answer, using my preferred camelCasing for the struct fields instead of what the top answer had. 

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/url"
	"os"
)

type Configuration struct {
	apiURL         string
	clientId       string
	clientSecret   string
}

func main() {
	file, _ := os.Open("conf.json")
	decoder := json.NewDecoder(file)
	configuration := Configuration{}
	err := decoder.Decode(&configuration)
	if err != nil {
		fmt.Println("error:", err)
	}

	fmt.Println(configuration.apiURL)
}
```

The problem was, whenever I ran the code, I just got blank in the console. No error, just blank. I went to look at why, and remembered that Go sets variables that start with lowercase to be private to the package, while uppercase variables are public outside of the package. This is called Exporting.

Changing the struct fields to start with uppercase, like so, made everything work.

```go
type Configuration struct {
    ApiURL          string
    ClientId        string
    ClientSecret    string
}
```

But why? This was all happening in the same package, right? Private vs. public shouldn't matter. And even if it did, shouldn't there be an error thrown when we're assigning the values?

[Stack Overflow](http://stackoverflow.com/questions/11126793/golang-json-and-dealing-with-unexported-fields) to the rescue again. I fed the encoding/json library the configuration struct and expected it to just match JSON fields to struct fields. To do that, it uses reflect to view the fields in the struct.

See the problem yet? The `encoding/json` library isn't part of this package. Therefore, can it see and write to the private fields in the `configuration` struct? Nope. And since `Decode` doesn't require a one-to-one field match between the JSON object and the struct, and Go will happily fill out ignored fields in the struct with null values, I was getting a `configuration` struct back with null values for every field. So, `fmt.Println` helpfully printed out the value of the field to console: blank.

According to one of the Stack Overflow answers above, if you want to keep the struct private, but the fields accessible for reflection, you can do it this way with a lowercase struct and uppercase fields:

```go
type configuration struct {
	ApiURL         string
	ClientId       string
	ClientSecret   string
}
```

Lessons learned. If I'm incorrect about any of this, hit me up [@shanethacker](https://twitter.com/shanethacker).

Update: [Tags in Go structs look interesting.](https://machiel.me/using-tags-in-go/) I'd bet `Decode` can use them for matching.
