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

 Intel developed a new instruction set, registers, and dedicated hardware for vector processing with the Pentium III

```
#include <immintrin.h> //< you can include emmintrin.h if you only want sse  
__m128i Add(__m128i a, __m128i b)  
{  
return _mm_add_epi32(a, b);  
}
```

convert to assembly:
```
Add:  
movaps xmm0, xmm1 ; Move b to xmm0  
paddq xmm0, xmm2 ; Add a and b  
ret
```

`a` and `b` are 2 vector registers with 128bit (can be different on other CPU family)

the function _mm_add_epi32 adds two arrays of numbers together  
_mm_ is a prefix when we call Intel intrinsic functions  
_add_ is what we want to do (sub, mul, div…)  
_epi32 is data type that we want to use (32 bit integers).

_epi8 for 8bit integers  
_epi16 for 16 bit integers  
_ps for floats,  
_pd for doubles



