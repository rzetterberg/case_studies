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

