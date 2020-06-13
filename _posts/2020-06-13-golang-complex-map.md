---
layout: post
title: golang complex map operations
tags: [go]
category: [go]
author: vikrant
comments: true
--- 

If you are from python background, map in go is similar to dictionary in python. When you are writing a complex program simple key,value pair dict is not sufficient enough. You may be using python defaultdict to create complex nested dicts. I was trying to solve similar problem with go and thought of documenting it for future reference and benefits of other. 

In both scenarios /etc/passwd file is read and fitted into map in various manner. 

Scenario 1 : Creating map whose values are slice and each element of slice is string. 

Code snippet: 

~~~
package main
import (
  "fmt"
  "io/ioutil"
  "strings"
)


func main() {
  shell_map := make(map[string][]string)
  output,err := ioutil.ReadFile("/etc/passwd")
  if err == nil {
    result := strings.Split(string(output),"\n")
    for _, eachvalue := range result {
      parsed_eachline := strings.Split(eachvalue, ":")
      if len(parsed_eachline) == 7 {
        key1 := parsed_eachline[len(parsed_eachline)-1]
        if len(key1) > 1 {
          shell_map[key1] = append(shell_map[key1],parsed_eachline[0])
        }
      }
    }
  }
  fmt.Println(shell_map)
}
~~~

Important parts of code snippet:

- Create map whole key is shell and values are slice of strings.

~~~
shell_map := make(map[string][]string)
~~~

- Get the last shell column of parsed passwd file which is a key for map.

~~~
key1 := parsed_eachline[len(parsed_eachline)-1]
~~~

- Append all users with same shell in value of map. 

~~~
shell_map[key1] = append(shell_map[key1],parsed_eachline[0])
~~~

Scenario 2 : Creating map whose values are slice and each element of slice is a map.

Code snippet: 

~~~
cat read_password_map_map.go

package main
import (
  "fmt"
  "io/ioutil"
  "strings"
  "encoding/json"
)


func main() {
  shell_map := make(map[string][]map[string]string)
  output,err := ioutil.ReadFile("/etc/passwd")
  if err == nil {
    result := strings.Split(string(output),"\n")
    for _, eachvalue := range result {
      parsed_eachline := strings.Split(eachvalue, ":")
      if len(parsed_eachline) == 7 {
        key1 := parsed_eachline[len(parsed_eachline)-1]
        if len(key1) > 1 {
          username_map := make(map[string]string)
          username_map[parsed_eachline[0]] = parsed_eachline[2]
          shell_map[key1] = append(shell_map[key1], username_map)
        }
      }
    }
  }
  //fmt.Println(shell_map["/sbin/nologin"][0])
  //for key1, value1 := range(shell_map) {
  //    for _, value2 := range(value1) {
  //      for key3, value3 := range(value2) {
  //        fmt.Println(key1,key3,value3)
  //      }
  //    }
  //}
  //fmt.Println(shell_map)
  jsonstring, err := json.Marshal(shell_map)
  fmt.Println(string(jsonstring))
}
~~~

Important parts of code snippet:

- Creating map with string key and slice value. Each element of slice is a map with key and value both string. 

~~~
shell_map := make(map[string][]map[string]string)
~~~

- Each entry of passwd file is splitted to generate a slice. 

~~~
parsed_eachline := strings.Split(eachvalue, ":")
~~~

- Create nested map which will be a element of slice. Here username is key and userid is value.

~~~
username_map := make(map[string]string)
username_map[parsed_eachline[0]] = parsed_eachline[2]
~~~

- Our parent map has shell as key and slice as a value. We are appending `{username:userid}` map in slice.  

~~~
shell_map[key1] = append(shell_map[key1], username_map)
~~~

Various printing options

- Print one parent map key with first element of slice. 

~~~
  fmt.Println(shell_map["/sbin/nologin"][0])

Resulted output

map[bin:1]

~~~

- Traversing through the map. 

~~~
  for key1, value1 := range(shell_map) {
      for _, value2 := range(value1) {
        for key3, value3 := range(value2) {
          fmt.Println(key1,key3,value3)
        }
      }
  }

Resulted output (truncated for brevity)

/bin/sync sync 5
/sbin/shutdown shutdown 6
/sbin/halt halt 7
/bin/sh tomcat 91
~~~ 

- Convert into json.

~~~
  jsonstring, err := json.Marshal(shell_map)
  fmt.Println(string(jsonstring))

Resulted output (truncated for brevity)

{"/bin/bash":[{"root":"0"},{"netadmin":"105"}]}
~~~

