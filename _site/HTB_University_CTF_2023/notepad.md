# Hack The Box University CTF notepad
Windows challenge:
program takes two characters right next to each other from and input string and adds their hex values

This is compared against an array to see if the result is correct

Array key (decimal): 
156 150 189 175 147 195 148 96 162 209 194 207 156 163 166 104 148 193 215 172 150 147 147 214 168 159 210 148 167 214 143 160 163 161 163 86 158

possible input value range (decimal): 32 - 126
narrowed ranges: {48-57, 65-90, 95, 97-122, 123('{'), 125('}')}

"HTB{" is 156 150 189 ?

result: "HTB{4_d00r_cl0s35_bu7_4_w1nd0w_0p3n5!}" (see [windows_solve.py](./windows_solve.py))

### Bio rev
found the data in array
run in gdb until after the mem is decoded
find the pid and run `dd if=/proc/[pid]/fd/3 of=proc_data bs=1`
Find the function from the data that is passed into dlsys
look at the data in the function