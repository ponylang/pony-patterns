# Limiting String Allocations

## Problem

Your code is performance sensitive and need to make your `String` concatenation
code as fast as possible.

## Solution

Replace any usage of `String.add` with `String.append`. Going from code like:

```pony
let output = file_name + ":" + file_linenum + ":" + file_linepos + ": " + msg
```

to `String.append` where we pre-allocate the memory needed to hold our final
string.

```pony
let output = recover String(file_name.size()
  + file_linenum.size()
  + file_linepos.size()
  + msg.size()
  + 4) end

output.append(file_name)
output.append(":")
output.append(file_linenum)
output.append(":")
output.append(file_linepos)
output.append(": ")
output.append(msg)
output
```

## Discussion

If you want to make your Pony code go fast (and who doesn't want to go fast?),
there are two simple steps you can take that will get  you a lot of reward:
reducing the number of objects you create and reducing the number of memory
allocations. Our solution does both. 

While String.add can be very convenient, it's a bit of a dog performance wise.

```pony
let output = file_name + ":" + file_linenum + ":" + file_linepos + ": " + msg
```

will create a new `String` and allocate memory for it on each `+`. In the case 
of our example, that's six objects that get created and six different memory
allocations. Our solution addresses both these issues. First, 

```pony
let output = recover String(file_name.size()
  + file_linenum.size()
  + file_linepos.size()
  + msg.size()
  + 4) end
```

allocates all the memory it is going to need in one go; cutting our memory
total allocations by five. Then by using `append`

```
output.append(file_name)
output.append(":")
output.append(file_linenum)
output.append(":")
output.append(file_linepos)
output.append(": ")
output.append(msg)
output
```

we don't create any additional objects. 

All told, by switching from `String.add` to `String.append`, drop down to a
single memory allocation and a single object being created. Replacing a single 
use of `+` with `append` isn't going to get you much, however, if the code you
are replacing is called a lot, it's going to be a huge win. In the case of our
solution above, we took the code from the Pony `logger` package. Given how often
logging methods get called, switching from `+` to `append` has a huge impact.


