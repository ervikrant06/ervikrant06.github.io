---
layout: post
title: How to parse json using golang
tags: [go]
category: [go]
author: vikrant
comments: true
--- 

During lockdown I started learning golang. In this series I will be posting articles useful for system admins starting with go. I don't want to go into very basic stuff related to go which is covered in on-line courses. I will try to keep the examples interesting enough to cover two/three topics. Today topic is about parsing json with golang. 

Below is the code snippet along with json which we want to parse. Our json is an array consists of user information of throttled users hammering storage. Since I know about json format hence struct is created to parse json. Template is used to prepare the message which we can send to user for notificatio purpose. 

Explanation of important part in code is provided later. 

~~~
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"os"
	"text/template"
)

type Throttle struct {
	Id              int     `json:"id"`
	Storage_type    string  `json:"storage_type"`
	Team            string  `json:"team"`
	Throttle_active string  `json:"throttle_active"`
	Throttle_date   string  `json:"throttle_date"`
	Throttle_factor float32 `json:"throttle_factor"`
	Throttle_time   float32 `json:"throttle_time"`
	Throughput      float32 `json:"throughput"`
	Username        string  `json:"username"`
}

func main() {
	const message = `User {{.Username}} from {{.Team}} hammering {{.Storage_type}} storage with throughput {{.Throughput}} throttled with {{.Throttle_factor}} for {{.Throttle_time}}`
	fmt.Println("Starting the application")
	b := `[{
		"id": 103,
		"storage_type": "gpfs_ssd",
		"team": "teama",
		"throttle_active": "no",
		"throttle_date": "2020-05-30T16:02:49.711961",
		"throttle_factor": 60.0,
		"throttle_time": 600,
		"throughput": 31,
		"username": "test1"
	}, {
		"id": 104,
		"storage_type": "gpfs_hdd",
		"team": "teamb",
		"throttle_active": "no",
		"throttle_date": "2020-05-30T16:02:49.711961",
		"throttle_factor": 60.0,
		"throttle_time": 600,
		"throughput": 30,
		"username": "test2"
	}]`
	var result []Throttle
	err := json.Unmarshal([]byte(b), &result)
	t := template.Must(template.New("message").Parse(message))
	if err != nil {
		fmt.Printf("something wrong %s\n", err)
	} else {
		for _, value := range result {
			err := t.Execute(os.Stdout, value)
			if err != nil {
				log.Println("executing template:", err)
			}
			fmt.Println()
		}
	}
}
~~~

Important parts of code snippet:

- `result` slice based on `Throttle` struct is created. 

~~~
var result []Throttle
~~~

- json is unmarshalled. 

~~~
err := json.Unmarshal([]byte(b), &result)
~~~

- Template for preparing the notification message

~~~
t := template.Must(template.New("message").Parse(message))
~~~

Resulted output: 

~~~
PS C:\Users\DELL\Documents\VIKRANT\GO_SCRIPTS> go run .\go_20.go
Starting the application
User test1 from mosaic hammering gpfs_ssd storage with throughput 31 throttled with 60 for 600
User test2 from mosaic hammering gpfs_hdd storage with throughput 30 throttled with 60 for 600
~~~