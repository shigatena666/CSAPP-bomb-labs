# Introduction

I was surprised by the bomb labs our teacher gave us to do. The aim was to get a better understanding of the assembler and reverse-engineering and I must say it worked.
For those challenges I used [Ghidra](https://github.com/NationalSecurityAgency/ghidra) together with [OnlineGDB](https://www.onlinegdb.com/online_c_compiler) until phase 6 where I started using [gdb](https://www.sourceware.org/gdb/).
The first 5 challenges were relatively easy so I didn't take notes for those. However, phase 6 and the secret phase were quite interesting, so I took some notes.

# Phase 6

Phase 6 took me a long time to understand. In fact, what helped me was to read the different write-ups made by other people. Fortunately, the bomb labs are randomly generated and none of the 6 phases contain the same answers that can be found on the internet.

With what Ghidra shows us, we understand that we are expected to enter 6 decimal numbers.
```c
read_six_numbers(param_1, local_98);
```
Apart from the function name, we can click on it. It will lead to the implementation of read_six_numbers.
```c
pcVar3 = (char *)__isoc99_sscanf(param_1, "%d %d %d %d %d %d", ...);
```

Among these 6 numbers, the program will iterate on each of them in order to verify that all of them are less or equal to 6 (<= 6).
```c
while( true ) {
if (5 < *piVar5 - 1U) {
	explode_bomb();
}
// ... more actions explained later.
piVar5 = piVar5 + 1;
```

In addition, we understand that the bomb explodes if any of the six numbers repeat.
```c
lVar3 = lVar6;
if (5 < (int)lVar6) break;
do {
	if (*piVar5 == local_98[lVar3]) {
		explode_bomb();
	}
	lVar3 = lVar3 + 1;
} while ((int)lVar3 < 6);
lVar6 = lVar6 + 1;
```

Moreover, we can see that foreach element of our array, the method executes a function where $f(x) = 7 - x$.
```c
do {
	*piVar4 = 7 - *piVar4;
	piVr4 = piVar4 + 1;
} while (local_80 != piVar4);
```

The code seems to create what appears to be a <b><u>linked list</u></b>, by creating an array that contains the  address of each nodes.
```c
do {
	iVar1 = 1;
	puVar2 = node1;
	if (1 < local_98[lVar6]) {
		do {
			puVar2 = *(undefined1 **)((long)puVar2 + 8);
			iVar1 = iVar1 + 1;
		} while (iVar1 != local_98[lVar6]);
	}
	(&local_78)[lVar6] = (int *)puVar2;
	lVar6 = lVar6 + 1;
} while (lVar6 != 6);
```

It then proceeds to assign the next node of each element of that array. The last one is, of course, set to 0 / null.
```c
*(long *)(local_78 + 2) = local_70;
*(long *)(local_70 + 8) = local_68;
*(long *)(local_68 + 8) = local_60;
*(long *)(local_60 + 8) = local_58;
*(long *)(local_58 + 8) = local_50;
*(undefined8 *)(local_50 + 8) = 0
```

For those who don't know, a linked list is a structure that contains both a value and a pointer (i.e an address) to the next element. It can be declared like this in C :
```c
struct Node
{
	int value;
	struct Node *next;
};
```

The final step of the challenge is that each element's value should be strictly greater than the next one. In other words, we can just say that each element of the array should be sorted by descending order.
```c
iVar1 = 5;
do {
	if (*local_78 < **(int **)(local_78 + 2)) {
		explode_bomb();
	}
	local_78 = *(int **)(local_78 + 2);
	iVar1 = iVar1 + -1;
} while (iVar1 != 0)
```

To sum it up, we need to :
- have x where $0 <= x <= 6$.
 - x distinct from any previous values of the array.
 - apply a formula on node indexes $7 - (index + 1)$.
 - sort all the node values to have them in descending order.

Let's try this !

To satisfy our conditions, we need first to get the node adresses, indexes and values. This will greatly help us solving the challenge.
On Ghidra, we can see the nodes in the phase_6 function. It initializes the variable with the first element of the node, aka node1.
```c
puVar2 = node1;
```

Double-click on it so that it brings you to where the node is defined on the stack.
You can now see multiple things, the node address (left), the next node address (grey'd on the right with the arrow : ? -> 00105230) and the node elements (a2 02 00 00 01...).
![[ghidra_output_6.png]]
From now, we need to retrieve the first 2 bytes on the node's byte array. **==Be careful there, the bytes need to be reversed whilst reading. It is little endian that is used.==**
We obtain the following informations :
```txt
address  => node  => int value (hex)         => index
00105220 => node1 => 0x02a2 (674 in decimal) => 0
00105230 => node2 => 0x0077 (119 in decimal) => 1
00105240 => node3 => 0x026c (620 in decimal) => 2
00105250 => node4 => 0x0301 (779 in decimal) => 3
00105260 => node5 => 0x00cd (205 in decimal) => 4
00105210 => node6 => 0x0320 (800 in decimal) => 5
```

Sort the lines by descending order on the node values
```
address  => node  => int value (sorted)      => index
00105210 => node6 => 0x0320 (800 in decimal) => 5
00105250 => node4 => 0x0301 (779 in decimal) => 3
00105220 => node1 => 0x02a2 (674 in decimal) => 0
00105240 => node3 => 0x026c (620 in decimal) => 2
00105260 => node5 => 0x00cd (205 in decimal) => 4
00105230 => node2 => 0x0077 (119 in decimal) => 1
```

Apply the formula  $f(x) = 7 - (x + 1)$  to the indexes, not the values.
```
address  => node  => int value (sorted)      => index => f(x)
00105210 => node6 => 0x0320 (800 in decimal) => 5     => 1
00105250 => node4 => 0x0301 (779 in decimal) => 3     => 3
00105220 => node1 => 0x02a2 (674 in decimal) => 0     => 6
00105240 => node3 => 0x026c (620 in decimal) => 2     => 4
00105260 => node5 => 0x00cd (205 in decimal) => 4     => 2
00105230 => node2 => 0x0077 (119 in decimal) => 1     => 5
```
The formula adds + 1 to all indexes because in C, the starting index of a list is 0. In the bomb lab, we are asked for numbers between 1 and 6 which are actually the indexes of the different nodes.
We subtract 7 because of what we saw earlier in phase_6, remember : 
```c
do {
	*piVar4 = 7 - *piVar4;
	piVr4 = piVar4 + 1;
} while (local_80 != piVar4);
```

Result :
```txt
1 3 6 4 2 5
```
From there, enter your answer. Be careful though, your answers may be different from mine and you could set off the bomb. It's better for you to understand the algorithm behind solving this lab, rather than just entering my answer and hoping it's the same for you.

# Tip (applicable to all phases)

A little tip before I finish this write-up:
When you enter your answer in the bomb lab, you can, using gdb, set a breakpoint on the explode_bomb function. This will ensure that the bomb never explodes in case of an error.
To do so, enter the following command :
```gdb
break explode_bomb
```

# Bonus (secret phase)

If you have understood how phase 6 works, then the secret phase will be even faster. 
The subtlety in this challenge lies in the structure used. Instead of a linked-list, we have a tree.

To access this challenge, you will have to look at the read_six_numbers function. 
```c
iVar2 = __isoc99_sscanf(0x1057b0,"%d %d %s",auStack128,auStack124,auStack120);
if (iVar2 == 3) {
	iVar2 = strings_not_equal(auStack120,"DrEvil");
	if (iVar2 == 0) {
		puts("Curses, you\'ve found the secret phase!");
		puts("But finding it and solving it are quite different...");
		secret_phase();
	}
}
puts("Congratulations! You\'ve defused the bomb!");
puts("Your instructor has been notified and will verify your solution.");
```

As you can see, the block will execute if a decimal, a decimal and a string are found within your input. This means that in a phase where two decimals are normally entered, you will have to add "DrEvil" to the end of your response. 
Be aware that the string may differ between challenges and yours may not be "DrEvil" ;-).

Once done, this will trigger the secret_phase. On Ghidra, double-click on the function name and let's have a look at it.
```c
void secret_phase(void)
{
	int iVar1;
	char *__nptr;
	ulong uVar2;

	// first part
	__nptr = (char *)read_line();
	uVar2 = strtol(__nptr,(char **)0x0,10);
	
	// check input
	if (1000 < (int)uVar2 - 1U) {
		explode_bomb();
	}

	// run the recursive function "fun7"
	iVar1 = fun7(n1,uVar2 & 0xffffffff);

	// last part
	if (iVar1 != 2) {
		explode_bomb();
	}
	puts("Wow! You\'ve defused the secret stage!");
	phase_defused();
	return;
}
```

This first part reads from the user a string. It will convert it to long using the strtol function.
```c
__nptr = (char *)read_line();
uVar2 = strtol(__nptr,(char **)0x0,10)
```

The part where it checks the input says that your the inputted string (that is now long), must be strictly under 1001.
```c
if (1000 < (int)uVar2 - 1U) {
	explode_bomb();
}
```

The recursive function "fun7" is now ran with the long number, alongside with doing some bitwise operations and passing what interests us, n1. 
==I could have studied it to understand what it does, but at that point, I already spent hours on the phase 6 and just decided to flag the secret phase. So, if you want a better understanding of the secret phase, please look at other ressources, as my method will only brute-force the anser without triggering the bomb.==
```c
iVar1 = fun7(n1,uVar2 & 0xffffffff);
```

A check is also made, and the result of the fun7 function must be equal to 2 in my case.
```c
if (iVar1 != 2) {
	explode_bomb();
}
```

But what is n1 ? Still in Ghidra, let's have a look at it by double-clicking.
![[ghidra_output_secret.png]]

We can see that the n1 variables points to two addresses. As stated above, this looks like a tree, where the struct in C can be defined like this :
```c
struct Node {
    struct Node *left;
    struct Node *right;
    int value;
};
```

Exactly like the phase 6, the first numbers are the value, then it is the address of the left node, and the address of the right node. They also must be read like a little endian.

We can follow the tree by clicking the grey address on the right. Get the node's name and the values, that's everything you will need. You can also scroll a little bit under n1, you will see the next nodes.
In the end, you obtain something like this :
```txt
node => value  (decimal)
n1   => 0x0024 (36)

n21  => 0x0008 (8)
n22  => 0x0032 (50)

n31  => 0x0006 (6)
n32  => 0x0016 (22)
n33  => 0x002d (45)

n41  => 0x0001 (1)
n45  => 0x0028 (40)
```

Now, hop on [Binary Search Tree](http://btv.melezinek.cz/binary-search-tree.html), enter all the decimal values you collected inside the tree and you will start visualizing it.
==Note that I didn't collect all the nodes, the answer is already in there for me. There is no way for you to know that information, but as I wrote this write-up the next day I completed the bomb lab, I already knew what the answer was.==
![[binary_tree_visualization.png]]

Now, get on gdb. Put a breakpoint on explode_bomb using the tip I gave you above and prepare to brute-force the lab ;-).
When asked for the input of the secret phase, try all the numbers you see on your binary tree and you will find the answer.
Mine was 22.

# Conclusion

I hope that you learnt something and that I helped you during those phase. Please feel free to contact me at val.unsure@pm.me if you see an error in the write-up.

Have a nice day !
