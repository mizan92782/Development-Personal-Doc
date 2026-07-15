First, a small correction: it's **pseudocode**, not **sudo code**.

**Pseudocode** is a simple, human-readable way to describe an algorithm without following the syntax of any specific programming language. Interviewers use it to evaluate your **problem-solving ability**, not whether you remember the syntax of Java, Python, or C++.

---

# What is Pseudocode?

Pseudocode is a mixture of:

* English language
* Programming logic
* Mathematical notation

It is **not executable**.

For example, suppose the problem is:

> Find the sum of two numbers.

Instead of writing Java:

```java
Scanner sc = new Scanner(System.in);
int a = sc.nextInt();
int b = sc.nextInt();
System.out.println(a+b);
```

You simply write:

```text
START

Input first number
Input second number

sum ← first number + second number

Display sum

END
```

Notice:

* No semicolon
* No braces
* No Java syntax
* Easy to read

---

# Why do interviewers ask for pseudocode?

They want to know whether you can:

* Understand a problem
* Think logically
* Break a problem into steps
* Design an algorithm

They usually don't care about language syntax during this stage.

---

# Basic Rules of Pseudocode

There is **no official universal standard**, but good pseudocode follows these rules.

## Rule 1: Use simple English

Bad

```text
Instantiate integer variable.
```

Good

```text
Create variable count
```

---

## Rule 2: One action per line

Good

```text
Input number
Calculate square
Display result
```

Bad

```text
Input number and calculate square and display result
```

---

## Rule 3: Indent blocks

Example

```text
IF age >= 18
    Display "Eligible"
ELSE
    Display "Not Eligible"
END IF
```

Indentation makes the logic clear.

---

## Rule 4: Capitalize keywords

Most interviewers expect keywords like

```text
START
END

IF
ELSE
END IF

FOR
END FOR

WHILE
END WHILE

FUNCTION
RETURN
```

---

# Common Keywords

## Input

```text
INPUT number
```

or

```text
READ number
```

---

## Output

```text
DISPLAY result
```

or

```text
PRINT result
```

---

## Assignment

Instead of

```text
=
```

many people use

```text
←
```

Example

```text
sum ← a + b
```

or

```text
SET sum = a + b
```

Both are acceptable.

---

# Structure of Every Pseudocode

Almost every algorithm follows this pattern.

```text
START

Input

Processing

Output

END
```

---

# Example 1

## Add Two Numbers

Problem

Input two numbers.

Output their sum.

Pseudocode

```text
START

INPUT A
INPUT B

SUM ← A + B

DISPLAY SUM

END
```

---

# Example 2

Find Bigger Number

```text
START

INPUT A
INPUT B

IF A > B
    DISPLAY A
ELSE
    DISPLAY B
END IF

END
```

---

# Example 3

Even or Odd

```text
START

INPUT NUMBER

IF NUMBER mod 2 = 0
    DISPLAY "Even"
ELSE
    DISPLAY "Odd"
END IF

END
```

---

# Example 4

Positive or Negative

```text
START

INPUT NUMBER

IF NUMBER > 0
    DISPLAY "Positive"

ELSE IF NUMBER < 0
    DISPLAY "Negative"

ELSE
    DISPLAY "Zero"

END IF

END
```

---

# Loops

## While Loop

Print 1 to 5

```text
START

COUNT ← 1

WHILE COUNT <= 5

    DISPLAY COUNT

    COUNT ← COUNT + 1

END WHILE

END
```

---

## For Loop

```text
START

FOR I ← 1 TO 10

    DISPLAY I

END FOR

END
```

---

# Arrays

Suppose

```
Array = [5,3,8,1]
```

Print all elements.

```text
START

INPUT ARRAY

FOR I ← 0 TO LENGTH(ARRAY)-1

    DISPLAY ARRAY[I]

END FOR

END
```

---

# Find Maximum Element

```text
START

INPUT ARRAY

MAX ← ARRAY[0]

FOR I ← 1 TO LENGTH(ARRAY)-1

    IF ARRAY[I] > MAX

        MAX ← ARRAY[I]

    END IF

END FOR

DISPLAY MAX

END
```

---

# Functions

Example

```text
FUNCTION Add(A, B)

    RETURN A + B

END FUNCTION


START

INPUT X
INPUT Y

RESULT ← Add(X, Y)

DISPLAY RESULT

END
```

---

# Searching Example (Linear Search)

Find 25 in an array.

```text
START

INPUT ARRAY
INPUT TARGET

FOUND ← FALSE

FOR EACH ITEM IN ARRAY

    IF ITEM = TARGET

        FOUND ← TRUE
        BREAK

    END IF

END FOR

IF FOUND

    DISPLAY "Found"

ELSE

    DISPLAY "Not Found"

END IF

END
```

---

# Binary Search

```text
START

INPUT SORTED ARRAY
INPUT TARGET

LOW ← 0
HIGH ← LENGTH(ARRAY)-1

WHILE LOW <= HIGH

    MID ← (LOW + HIGH) / 2

    IF ARRAY[MID] = TARGET

        DISPLAY "Found"
        STOP

    ELSE IF ARRAY[MID] < TARGET

        LOW ← MID + 1

    ELSE

        HIGH ← MID - 1

    END IF

END WHILE

DISPLAY "Not Found"

END
```

---

# Interview Format

Suppose the interviewer asks:

> Reverse a string.

A good answer is:

```text
FUNCTION Reverse(STRING)

    RESULT ← empty string

    FOR I ← LENGTH(STRING)-1 DOWN TO 0

        RESULT ← RESULT + STRING[I]

    END FOR

    RETURN RESULT

END FUNCTION
```

---

# How to Write Pseudocode Step by Step

Whenever you get a problem, follow these steps.

### Step 1: Understand the problem

Ask yourself:

* What is the input?
* What is the output?
* What are the constraints?

---

### Step 2: Write inputs

```text
INPUT N
```

---

### Step 3: Think about processing

Example

```text
TOTAL ← TOTAL + NUMBER
```

---

### Step 4: Use decisions if needed

```text
IF

ELSE

END IF
```

---

### Step 5: Use loops if needed

```text
FOR

WHILE
```

---

### Step 6: Display the answer

```text
DISPLAY RESULT
```

---

### Step 7: Finish

```text
END
```

---

# Example: Sum of an Array

Problem

Find the total sum.

Pseudocode

```text
START

INPUT ARRAY

SUM ← 0

FOR EACH NUMBER IN ARRAY

    SUM ← SUM + NUMBER

END FOR

DISPLAY SUM

END
```

---

# Example: Count Even Numbers

```text
START

INPUT ARRAY

COUNT ← 0

FOR EACH NUMBER IN ARRAY

    IF NUMBER mod 2 = 0

        COUNT ← COUNT + 1

    END IF

END FOR

DISPLAY COUNT

END
```

---

# Common Mistakes

❌ Mixing programming language syntax:

```text
int a;
scanf();
System.out.println();
```

Avoid language-specific code.

❌ Forgetting indentation:

```text
IF X > 0
DISPLAY X
ELSE
DISPLAY 0
```

Instead:

```text
IF X > 0
    DISPLAY X
ELSE
    DISPLAY 0
END IF
```

❌ Writing every implementation detail:

```text
Create Scanner object
Allocate memory
```

Keep the focus on the algorithm.

---

# A Simple Template for Interviews

```text
START

INPUT ...

Initialize variables

Process data

IF needed
    ...
END IF

FOR or WHILE if needed
    ...
END LOOP

DISPLAY RESULT

END
```

## Practice Problems

To become comfortable writing pseudocode, practice these in order:

1. Add two numbers.
2. Find the largest of three numbers.
3. Check if a number is even or odd.
4. Calculate the factorial of a number.
5. Check if a string is a palindrome.
6. Reverse a string.
7. Find the maximum element in an array.
8. Find the second-largest element.
9. Linear search.
10. Binary search.
11. Bubble sort.
12. Selection sort.
13. Merge two sorted arrays.
14. Detect duplicates in an array.
15. Check if two strings are anagrams.

These problems cover the types of logic commonly tested in software engineering interviews.

If you're preparing for software engineering interviews, I can also teach you **interview-style pseudocode for common Data Structures and Algorithms (arrays, linked lists, stacks, queues, trees, graphs, recursion, dynamic programming, etc.)**, with the level of detail expected by companies like Google, Microsoft, Amazon, and other top tech firms.
