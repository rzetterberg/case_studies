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
worst case it has to do 10 000 (100^2) comparisons and swaps. Comparing that to
it's replacement (heapsort), which has to in worst case do ~664 (100 *
log2(100)) comparisons and swaps. 

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

Here are the steps involved to insert a new item:

1. Start at the beginning of the array, check first item
2. If the current item is NULL, insert the given item. 
3. If not, check if the given item is older than the current item.
4. If the current item is older, insert the given item in it's place use the
current item as a given item.
5. Repeat from step 2.

Now, that might not be the best explanation of the algorithm, but here is a good
animation that show visually what happens:

![Insertion sort animation](http://upload.wikimedia.org/wikipedia/commons/0/0f/Insertion-sort-example-300px.gif)

The animation shows what the algorithm looks like when it's sorting in-place, so
imagine that the black boxes are items in the sorted array and the other ones
are the items in the original array. 
