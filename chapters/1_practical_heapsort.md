#Practical heapsort

**C, Linux, Gcc, Valgrind, Performance**

The project shown in this case study was an application that tracks personal
statistics such as; how much sleep each night, how many meals consumed, how much
coffee consumed etc. 

Why I started working on this application was because I wanted to improve my
skills in these areas:

* C programming in general
* Data structures
* Algorithms
* Parsing, loading and saving binary data from/to disk
* Valgrind

What was one of the most interesting things I learned during this project was
using different algorithms and seeing how they affected performance. The biggest
performance boost I got was from switching from [Insertion Sort](http://en.wikipedia.org/wiki/Insertion_sort) to [Heap sort](http://en.wikipedia.org/wiki/Heapsort) when
sorting data points by date. 

The computer used in this case study has the following specs:

* Pentium(R) Dual-Core CPU T4500 @ 2.30GHz 
* Linux 3.0.0-15-generic
* Ubuntu 11.10

##The data

Each data point looked like this: 

```c
typedef struct Data_Point{
	size_t refs;	
	uint8_t weight;
	uint8_t slept;
	uint8_t coffee_consumed;
	uint8_t meals_consumed;
	bool slept_during_day;
	/* 
		A bunch of other fields ...
	*/
	uint8_t day;
	uint8_t month;
	uint16_t year;
} Data_Point;
```

Nothing special, just some data and a field containing how many pointers that
point to that memory. See [reference counting](http://en.wikipedia.org/wiki/Reference_counting)

Each point is contained in a static length array:

```c
typedef struct Data_Point_Array{
	size_t refs;
	size_t length;
	Data_Point **items; 
} Data_Point_Array;
``` 

A reference count, an item count and a pointer to the memory
containing the pointers that point towards each data point.

##The original algorithm

The original algorithm used to sort these points was insertion sort. It's a very
simple algorithm but it is slow. If it were to sort 100 data points, in the
worst case it has to do 10 000 (100^2) comparisons. Comparing that to
it's replacement (heapsort), which has to in worst case do ~664 (100 *
log2(100)) comparisons. 

Both these algorithms can do the sorting in-place, which means it doesn't have
to create a new array to put the results in. It just shifts the items around in
the given array. However, since I'm a novice I had implemented the insertion
sort which creates a new array to put the results in. So not only is the array
slow, but I had made a sub-optimal implementation of it. 

Here is what the first part of the code looks like: 

```c
Data_Point_Array *Data_Point_Array_insertion_sort(Data_Point_Array *array)
{
	Data_Point_Array *sorted = Data_Point_Array_create(array->length);
	Data_Point *insertee = NULL;
	size_t i = 0;

	for (; i < array->length; i++) {
		insertee = array->items[i];
		Data_Point_incref(insertee);

		insertion_sort_insert(sorted, insertee);
	}

	return sorted;
}
``` 

As you can see the array which will contain the sorted items is created with the
same length as the original one. It will only sort, not filter out any items, so
it's safe to assume that the length will be the same in the result.

Then all it does is to iterate the original array and extract each item,
increase it's reference count and then inserts the item into the sorted array.
Since each item is going to be inserted into a new array it's reference count is
increased by 1 so that when we remove the original array the memory for each
item will not be free'd.

Here is what the insert function looks like:

```c
static inline void insertion_sort_insert(
	Data_Point_Array *sorted, 
	Data_Point *insertee)
{
	Data_Point *current = NULL;
	Data_Point *tmp = NULL;
	size_t i = 0;

	for (; i < sorted->length; i++) {
		if ((current = sorted->items[i]) == NULL) {
			sorted->items[i] = insertee;	
			break;
		}	

		if (Data_Point_older(current, insertee)) {
			tmp = current;	
			sorted->items[i] = insertee;
			insertee = tmp;
		}
	}
}
```

This is the meat of the algorithm. For each item it recieves, it checks it
against items already inserted in the sorted array.

The `Data_Point_older` function checks whether the first item
is older than the second. True, if A is older than B, False if A is not older
than B and False if A is equally old as B. It is important that if the items are
equal the compare function should return False. I leave the reason as an
exercise for the reader to find out.

Here are the steps involved to insert a new item:

1. Start at the beginning of the array, check first item
2. If the current item is NULL, insert the given item. 
3. If not, check if the given item is older than the current item.
4. If the current item is older, insert the given item in it's place use the
current item as a given item.
5. Repeat from step 2.

Now, that might not be the best explanation of the algorithm, but here is a good
animation that show visually what happens:

![Insertion sort animation](https://raw.github.com/rzetterberg/case_studies/master/assets/insertion_sort_animation.gif)

The animation shows what the algorithm looks like when it's sorting in-place, so
imagine that the black boxes are items in the sorted array and the other ones
are the items in the original array. 

##Measuring the original algorithm

Now comes the interesting part, measuring the performance to get an idea of how
slow this implementation of the algorithm is. 

When doing these measurements the program was compiled with `-O3` optimization
and *7300* points were used. The data was created using `dd` to get bytes from
`/dev/urandom`.

Since the datafile is only a flat binary file were the points are stored
sequencially it is easy to generate random data with `dd`. Each point is stored
as **8 bytes**, so here is the command used to generate the test file:

```bash
dd if=/dev/urandom of=random.data bs=8 count=7300
```

Now that we have fairly large file we will use valgrind to generate profiling
data for us to look at. But first we will check the program for any memory leaks
before we do that. Valgrind can do that for us too! Just compile the program
with -O3 and -g, then run the executable with valgrind:

```bash
valgrind --leak-check=full --log-file=valgrind.log ./compiled_program
```

If there are any leaks the output can become quite long, so I like to output the
information into a file to view with my editor. Here is what my output looks
like:

```
==7339== Memcheck, a memory error detector
==7339== Copyright (C) 2002-2010, and GNU GPL'd, by Julian Seward et al.
==7339== Using Valgrind-3.6.1-Debian and LibVEX; rerun with -h for copyright
info
==7339== Command: ./compiled_program
==7339== Parent PID: 7338
==7339==
==7339==
==7339== HEAP SUMMARY:
==7339==     in use at exit: 0 bytes in 0 blocks
==7339==   total heap usage: 172 allocs, 172 frees, 4,189 bytes allocated
==7339==
==7339== All heap blocks were freed -- no leaks are possible
==7339==
==7339== For counts of detected and suppressed errors, rerun with: -v
==7339== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 11 from 6)
```

Great! No leaks found or any other problems involving memory found.

Now it's time to generate the profiling data:

```bash
valgrind --tool=callgrind --dump-instr=yes --simulate-cache=yes
--collect-jumps=yes  ./compiled_program
```

This will output a file called **compiled_program.out.XXX** where XXX is the pid
of the application executed. Now we can view this file in `kcachegrind` to get a
nice visual presentation of amount of calls to different functions in the
application. Here is what the output looks like using the insertion sort on
these 7300 data points:

![Kcachegrind insertion sort output](https://raw.github.com/rzetterberg/case_studies/master/assets/1_practical_heapsort/kcachegrind_insertion_profile.png)

As you can see the most amount of time if spend doing the comparisons of the
items. Look at the called amount. That's right, the function is run **26 641 350
times**! This took ~20 seconds for my old laptop to complete. And that's not
even the worst case, that is about half amount of calls that the worst case
would be.

I wonder what heap sort can do for us?

##The replacement algorithm

The heapsort algorithm is a bit more complex, but it is much faster. The main
idea with heapsort is that you build a heap of the data and then use the heap to
order the items sequencally. It works in-place, and the implementation I will
show you does too.

A heap is basically a special type of binary tree, but it doesn't require a
separate data structure, it is created within the array. It does this by
ordering the items in a special way which represent the tree. Here is a good
visual representation of how it does this:

![Heapsort animation](https://raw.github.com/rzetterberg/case_studies/master/assets/heapsort_animation.gif)

I won't go into detail how the heapsort algorithm works, because it's out of
scope of this article. I'll leave reading up on how heapsort works as an
exercise for the reader.

On to how I have implemented the algorithm! Here is the first part of the code
for the algorithm:

```c
void Data_Point_Array_heap_sort(Data_Point_Array *array)
{

	int32_t i = (array->length / 2) - 1;

	for (; i >= 0; i--) {
		heap_sort_sift_down(array, i, array->length - 1);
	}

	for (i = array->length - 1; i >= 1; i--) {
		Data_Point_Array_items_switch(array, 0, i);
		heap_sort_sift_down(array, 0, i - 1);
	}
}
```

Here we can see that no array is created to place the sorted result in, we just
use the existing one. First the heap is built by calling the sift down function
on half of the array starting from the middle to the start. 

When the heap is created and all items are in place, the heap is then flattened
into a sequencially sorted array.

Here is what the second part of the code looks like:

```c
#define HEAP_LEFT(I) (I * 2)
#define HEAP_RIGHT(I) (HEAP_LEFT(I) + 1)

static void heap_sort_sift_down(
	Data_Point_Array *array, 
	size_t current_root, 
	size_t bottom)
{
	size_t left = HEAP_LEFT(current_root);
	size_t right = HEAP_RIGHT(current_root);
	size_t root_candidate;

	while (left <= bottom) {
		if (left == bottom) {
			root_candidate = left;
		}else if (Data_Point_older(array->items[left], array->items[right])) {
			root_candidate = left;
		}else{
			root_candidate = right;
		}

		if (Data_Point_older(array->items[root_candidate], array->items[current_root])) {
			Data_Point_Array_items_switch(array, current_root, root_candidate);

			current_root = root_candidate;
			left = HEAP_LEFT(current_root);
			right = HEAP_RIGHT(current_root);
		}else{
			return;
		}
	}
}
```

If you have read up on heapsort you will probably have seen pseudo-code of
heapsort being implemented as a recursive algorithm. This implementation uses an
iterative implementation instead. There are two reasons:

1. For each function call, data is placed on stack memory. Eventually when the
amount of recursive calls are too many the stack memory will be overflow and the
program will crash. See [stack overflow](http://en.wikipedia.org/wiki/Stack_overflow)
2. For each recursive call there are instructions that needs to be run to set up
the function call, by using a loop instead we don't have to run those
instructions which makes the algorithm faster.

##Measuring the replacement algorithm

Assume that the same steps were performed before profiling this algorithm as
with the original algirithm. Here is what the kcachegrind output looks like for
the heapsort algorithm:

![Kcachegrind heapsort output](https://raw.github.com/rzetterberg/case_studies/master/assets/1_practical_heapsort/kcachegrind_heapsort_profile.png)

That is a big improvement! From **26 641 350 comparisons** we are not down to **179 688**.
Using this algorithm the sort took less than 1 second to run on my old laptop.
