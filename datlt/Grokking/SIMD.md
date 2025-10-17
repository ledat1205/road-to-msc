The Expansion of SIMD is ‘Single Instruction Multiple Data’ as its name describes. It allows us to process multiple data with a single instruction.  
by ‘Process’ I mean performing operations like adding, subtracting, multiplying, dividing, and also logical operations such as ‘and’, ‘or’, and ‘xor’.

problem describe in program:
```
int Add(int a, int b)  
{  
int c = a + b;  
return c;  
}
```

convert to assembly:
```
Add:  
add eax, ebx  
ret
```



```
void Add(int a[4], int b[4], int c[4])  
{  
c[0] = a[0] + b[0];  
c[1] = a[1] + b[1];  
c[2] = a[2] + b[2];  
c[3] = a[3] + b[3];  
}
```

convert to assembly:
```
Add:  
mov eax, dword ptr [rdx]  
add eax, dword ptr [rcx]  
mov dword ptr [r8], eax  
mov eax, dword ptr [rdx + 4]  
add eax, dword ptr [rcx + 4]  
mov dword ptr [r8 + 4], eax  
mov eax, dword ptr [rdx + 8]  
add eax, dword ptr [rcx + 8]  
mov dword ptr [r8 + 8], eax  
mov eax, dword ptr [rdx + 12]  
add eax, dword ptr [rcx + 12]  
mov dword ptr [r8 + 12], eax  
ret
```

