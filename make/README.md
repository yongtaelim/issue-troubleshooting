# make && Makefile

# Troubleshooting
```
<action>
Makefile안에서 PATH 설정
export PATH=${PWD}/bin:$(PATH)

<error>
*** Recursive variable 'PATH' references itself (eventually).  Stop.

<cause>
잘못된 설정...

<solution>
PATH=$(shell printenv PATH):${PWD}/bin
```