---
layout: post
title: golang parse df output hard way
tags: [go]
category: [go]
author: vikrant
comments: true
--- 

This is completely a hypothetical scenario to bolster my golang skills. Indeed ample amount of options are available in df to get the desired output but I intentionally chose default options which I usually use. Purpose is to get the total storage provided by remote filesystems in MB.

Code snippet

~~~
package main
import (
  "fmt"
  "os/exec"
  "strings"
  "strconv"
)
func main() {
  size_mapping := map[string]string{
    "G": "1024",
    "T": "1048576",
    "P": "1073741824",
  }
  out, err := exec.Command("df", "-Ph").Output()
  if err != nil {
    fmt.Printf("%s",err)
  }
  output := string(out[:])
  splitted_output := strings.Split(output, "\n")
  var sum float64
  for _,output_value := range(splitted_output) {
    parsedline := strings.Fields(output_value)
    if len(parsedline) > 0 {
      splitted_parsed := strings.Split(parsedline[0], ":")
      if len(splitted_parsed) > 1 {
        parsed_len := len(parsedline[1])
        for key,_ := range size_mapping {
          storage_unit := parsedline[1][(parsed_len-1):parsed_len]
          if key == storage_unit {
            storage_size, _ := strconv.ParseFloat(parsedline[1][0:(parsed_len-1)],64)
            mult_factor, _ := strconv.ParseFloat(size_mapping[key],64)
            parsedline[1] = fmt.Sprintf("%f", storage_size * mult_factor)
            mb_size, _ := strconv.ParseFloat(parsedline[1],64)
            sum += mb_size
          }
        }
      }
    }
  }
   fmt.Printf("Total size for remote FS in MB is %f\n",sum)
}
~~~

Important parts of code snippet:

- Get the unit of output. Based on unit multiply it with map value. 

~~~
storage_unit := parsedline[1][(parsed_len-1):parsed_len]
~~~

- Get the size in MB and do the sum.

~~~
mb_size, _ := strconv.ParseFloat(parsedline[1],64)
~~~

- Print output in MB

~~~
fmt.Printf("Total size for remote FS in MB is %f\n",sum)
~~~