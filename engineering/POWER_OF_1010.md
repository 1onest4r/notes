RULE 1:
	keep the control flow simple (No indirect or direct recursion)
	use basic loops and such

RULE 2:
	give every loop fixed upperbound so avoid stuffs like while(true)

RULE 3:
	newer ever allocate or deallocate raw memory once the main loop has
	started so load everything you might need before the loop
	and use safe data structure such as vector and unique_pointer etc

RULE 4:
	no function should be bigger than a size of a sheet of paper
	so about 60 lines and a function should be for accomplishing
	a single task if it exceeds break it down

RULE 5:
	sanity check (High assertion density) adding a little piece of
	statements to make sure that some stuffs should a certain way
	to avoid bugs when the logic gets too heavy

RULE 6:
	declare a variable in the lowest scope meaning locking down the
	variable to avoid accidental global change so e.g for loop
	we define i in the loop and so on

RULE 7:
	check each return value even if its void

RULE 8:
	limit the use of preprocessors so try to define like a normal human
	being and let the compiler check what data type is the defined vars

RULE 9:
	RESTRICT POINTER USE no function pointers are allowed, no more than
	one level of dereferencing is allowed, and may not be hidden inside
	macros so try using reference

RULE 10:
	compile with all the warnings enabled from day 0 and you should fix
	them and a code must compile with '0' errors 
	
	