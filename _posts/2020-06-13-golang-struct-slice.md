---
layout: post
title: golang struct slice
tags: [go]
category: [go]
author: vikrant
comments: true
--- 

If your input data is known to you then creating struct for parsing the data could be a best choice. In this article I am parsing /etc/passwd file in which entry consists of 7 columns. 

Code snippet: 

~~~
package main
import (
  "fmt"
  "io/ioutil"
  "strings"
)

type Password struct {
  Username string
  Password string
  Uid string
  Gid string
  Description string
  HomeDir string
  Shell string
}

type Passwords struct {
  passwords []Password
}

func (passwords_list *Passwords) AddItem(item *Password)  {
    passwords_list.passwords = append(passwords_list.passwords, *item)
}

func main() {
  passwords_list := Passwords{}
  output,err := ioutil.ReadFile("/etc/passwd")
  if err == nil {
    result := strings.Split(string(output),"\n")
    for _, eachvalue := range result {
      parsed_eachline := strings.Split(eachvalue, ":")
      if len(parsed_eachline) == 7 {
        password := new(Password)
        password.Username = parsed_eachline[0]
        password.Password = parsed_eachline[1]
        password.Uid = parsed_eachline[2]
        password.Gid = parsed_eachline[3]
        password.Description = parsed_eachline[4]
        password.HomeDir = parsed_eachline[5]
        password.Shell = parsed_eachline[6]
        passwords_list.AddItem(password)
      }
    }
  }
    fmt.Printf("%v \n",passwords_list.passwords[10].Username)
}
~~~

Important parts of code snippet:

- Create var using Password struct.

~~~
password := new(Password)
~~~

- Add password struct as an element to passwords_list

~~~
passwords_list.AddItem(password)
~~~

- We have slice passwords_list whose each element is a struct. To print 10 element Username. 

~~~
fmt.Printf("%v \n",passwords_list.passwords[10].Username)
~~~

- Similarly we can do other magic with our struct of slice. 