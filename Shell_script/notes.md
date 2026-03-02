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


## loop

while: I will do xxx when xxx

until: I will do xxx when not xxx


```bash
num=1
until/while [ condition ]
do 
  execution body
done

```

```bash
for i in list
do
  execution body
done
```

## loop control

continue

break

exit

## function
```bash
func(){
execution body
}
```

Function passes the parameters in the same way the script does. This causes a lot of confusion. 

To make your code clean, use `xx=$1` in the begining of the script.

Scope: by default, bash allows you to change nonlocal vairable and make reference to it directly

To decalre some variable is local variable, you need to use `local xx`

`$?` represents the exit satus.

`return`: customized exit status, from 0 - 255.

A common workflow is, you echo something at the end of the function, then use $() for variable to pass the reference.

## Automation of script

1. `read -p` for user interaction
2. Use logic and function for sequnetial execution
3. signal


## Expect
`/usr/bin/expect <<-EOF` 


## SSH
1. Genearte private key & public key(lock)
2. copy the public key to the remote host



