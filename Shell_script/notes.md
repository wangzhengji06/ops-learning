## Shebang

#!/bin/bash -> force the shell to be this


bin/bash file.sh

./file.sh

source file.sh # load the environment variable to the current shell environment


## Variable

echo $varname / env

### Local Environment
Only works in the current shell environment

A=b # No special character allowed

A='b' # Taken literally

A="b" # Allow reference to other variable

A=`command` # 

A=$(command)


### GLobal Environment
First define a local variable, the export it

Or directly export xx=xx

How to undo? unset 


### Domain

System -> User -> SHELL -> File -> Code

System: /etc/profile /etc/profile.d/xx.sh  /etc/bashrc

User: ~/.bashrc  ~/.bashprofile


### Inner Variable

$0 -> current shell script name

$n -> paramaeter

$# -> parameters count

$? -> success or not


## String manipulation

${string:0:5}

${string:0-5:5}


## Checking variable

$variable_name

"$variable_name"

${variable_name}

"${variable_name}"

Only the third way and forth way is recommended.


## Passing parameter

1. /bin/bash file.sh arg1 arg2  -> $1 $2
2. read -p "Please enter variable x: " x


## Expression
$[]

let a=6*5

Calculation: expr 2 + 3

echo 'scale=2; 6 * 8 / 5' | bc

## Expression part 2

test /  [] -> check whether !? = 0 or 1

&& || ! -> Command mode and Option mode.

Str:

== / !=

-z / -n "" is empty / is not empty 

File: 

-d directory?

-f file?

-x executable?

Number:

-eq -gt -ge  -ne


## MultiConditional

[ condition1 -a condition2 ]

[ condition1 -o condition2 ] 

[[ condition1 && condition2 ]]

[[ condition1 || condition2 ]]

Also only [[]] support regex


## If logic

if then fi

if then else fi

if then elif then elif then ... else fi

case "variable_name" in
  "xx")
    do operation 1;;
  "yy")
    do operation 2;;
esac
