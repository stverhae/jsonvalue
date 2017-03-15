![License](https://img.shields.io/dub/l/vibe-d.svg)
# JsonValue dynamic parser

JsonValue is a lightweight json parses which excells at efficient parsing of dynamic or nested json data. And it does this up to **10 times faster** than standard `encoding/json` package (depending on payload size and usage) with minimal memory footprint. See benchmarks below.

The main reason to use JsonValue is if you do not know the exact sturcture of your data beforehand (as required by `encoding/json`) or you want efficientaccess to data at certain paths, without needing the whole json tree to be parsed and copied to go structs.

The JsonParser library is build as a minimalistic wrapper on top of the `jsonparser` library (https://github.com/buger/jsonparser). Many thanks to buger for providing this awesome, blazingly fast json parser! The JsonValue library aims to make it easier and less error prone to parse complicated json structures. Instead of dealing with byte arrays it allows you to parse your json in a simpler OO style. We were also able to fix and improve some functionality which could not be changed before because of backward-compatibity.

## Example
For the given JSON our goal is to extract the user's full name, number of github followers and avatar.

```go
import "github.com/stverhae/jsonvalue"

...

data := []byte(`{
  "person": {
    "name": {
      "first": "Leonid",
      "last": "Bugaev",
      "fullName": "Leonid Bugaev"
    },
    "github": {
      "handle": "buger",
      "followers": 109
    },
    "avatars": [
      { "url": "https://avatars1.githubusercontent.com/u/14009?v=3&s=460", "type": "thumbnail" }
    ]
  },
  "company": {
    "name": "Acme"
  }
}`)
```

```go
//As you work with the library you will always deal with JsonValues, these wrap your actual data and can be used to dive deeper in the json tree and retrieve usable go types (strings, numbers, arrays, maps, etc.

//always start by callig Parse of your json data, this returns a JsonValue
json := jsonvalue.Parse(data)

// You can specify key path by providing arguments to Get function
//this returns a *JsonValue object containing the data you requested
jsonName := json.Get("person", "name", "fullName")

//you can easily check the type of data you got and retrieve it
if jsonName.IsString() {
    fmt.Println(jsonName.GetString())
} else {
    //complain when name is not a string
    fmt.Printf("Help I got an %v!", jsonName.Type)
}

// If you already know what kind of data to expect you can immediately use `GetSting`, `GetInt`, `GetFloat`, `GetBool`, etc. with the correct path
json.GetInt("person", "github", "followers")

// When you give the path to an object, you get a JsonValue (as always) which you can use to further parse the json object
company := json.Get("company") // `company` => `{"name": "Acme"}`
company.GetString("name")
//this is identical to
json.GetString("company", "name")
//also identical to (you get the idea..)
name := json.Get("company", "name").GetString()

// if at any point the parsing fails because your json is malformed or you provided an invalid path the JsonValue will contain an error
//use `JsonValue.Err()` to check this. `JsonValue` implements `Error()` so it is a valid go `error`
test := json.Get("company", "doesnotexist").Get("nono")
if test.Err() != nil {
    fmt.Println("Parsing failed: ", test) //will print the error
}
//if you use a `GetXYZ` function on a JsonValue with a parse error, it will simply return that error.
var size int64
if value, _, err := json.GetInt("company", "size"); err == nil {
  size = value
}

// You can use `ArrayEach` helper to iterate items in a jsonArray [item1, item2 .... itemN]
json.Get("person", "avatars").ArrayEach(func(avatar *JsonValue) {
	fmt.Println(avatar.GetString("url"))
})

// Or use can access fields by index!
json.GetString("person", "avatars", "[0]", "url")


// Or parse the whole array and convert it to a go array (of JsonValues)
avatars := json.Get("person", "avatars")
if avatars.IsArray() {
    for _, avatar := range avatars.ToArray() {
        fmt.Println(avatar.GetString("url"))
    }
}

// You can use `ObjectEach` helper to iterate objects { "key1":object1, "key2":object2, .... "keyN":objectN }
json.Get("person", "name").ObjectEach(func(key string, value *JsonValue) {
        fmt.Printf("Key: '%s' Value: '%s' Type: %s\n", key, value.String(), value.Type)
})

// Or use `ToMap` to concvert the whole object to a go map
for key, value := range json.Get("person", "name").ToMap {
        fmt.Printf("Key: '%s' Value: '%s' Type: %s\n", key, value.String(), value.Type)
})

// The most efficient way to extract multiple keys is `AllKeys`
paths := [][]string{
  {"person", "name", "fullName"},
  {"person", "avatars", "[0]", "url"},
  {"company", "url"},
}
for _, value := range json.AllKeys(paths...) {
    fmt.Println(value.String())
})

// For more information see docs below
```
