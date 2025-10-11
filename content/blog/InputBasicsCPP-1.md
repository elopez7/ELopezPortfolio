+++
title = "Input Basics In C++ Part 1"
date = "2025-09-18T18:44:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "input"]

[images]
    featured_image = "/img/blog/inputbasics.jpeg"
+++

![Input Basics](/img/blog/inputbasics.jpeg)
In your programming journey, have you ever reached a point where you feel overwhelmed with everything a language can do, leaving you unsure of what to learn first or which features are truly important? I find myself in that exact situation. With each new C++ standard, I feel like I'm falling behind. Here we are in 2025, and I'm still trying to master move semantics and how to best use smart pointers. Learning a language with a rich history like C++ can be grueling. I've noticed I often get mesmerized by what's new, shiny, and exciting, causing me to neglect the fundamentals.
That's what this post is about: practicing some of the fundamentals of the language and its standard library.
In this new series, I will explore the basics of the <iostream> library.

## The Project.
I'll create a simple unit converter to illustrate the fundamental ways of getting user input and displaying output.
The program will convert from meters to feet.
The program will only run once and won't validate incorrect input. We'll add those features in the future. But first, let's cover a few key concepts.

## Including the Input/Output Library to the Project.
![Standard Library](/img/blog/standardlibrary.jpeg)
To handle input and output operations, we must first tell the compiler that we intend to use the input/output library, a core part of the C++ Standard Library. The Standard Library is a collection of components that have been built, tested, and optimized over many years. These components exist because developers noticed that all programs, despite their differences, perform common operations like reading files, printing to the screen, and taking keyboard input. It was created so that we can all use these pre-built components without having to write them from scratch. A popular metaphor is to view each library component as a Lego set. In our case, we'll be using the `<iostream>` library. This library provides the facilities for our programs to do interesting things, such as communicating with other devices or formatting data. For this particular program, we are interested in `std::cin` for basic input and `std::cout` for basic output.
To include the library, we just need to add a single line at the top of our file.
```cpp
#include <iostream>
```
However, adding this line alone isn't enough. We also need to tell the operating system where the entry point of our program is.
The entry point is where the program's execution begins. For our program to start, we need to add a few more lines.
```cpp
#include <iostream> //Tell the compiler we want to use input/output library

//This function is the entry point of the application.
int main()
{
    return 0;
}
```
Great! We now have a working program that does absolutely nothing. Still, a working program is a good start.
Now it's time to make it interesting.

## What Are Streams?
![Streams](/img/blog/streams.jpeg)
Streams are a core concept in the `iostream` library. They represent the idea of data as a flow, typically a sequence of characters. Based on this abstraction, we can say that data "flows" into a stream during input operations and "flows" out of a stream during output operations. Think of it like a river, but instead of water, it carries a potentially infinite supply of characters (like letters and numbers). Data can flow into or out of this stream.

## Console Output
Let's continue our little exercise by making the program write something to the screen.
In this case, we will write a number.

```cpp
#include <iostream>

int main()
{
    int number{50};
    std::cout << number << std::endl;
    return 0;
}
```
In this program, we're telling the compiler: "Create a variable called number, store the value 50 in it, and then use std::cout to print that value to the screen."
The following screenshot displays the result after running the program:
![Console Output 001](/img/blog/console_output_001.png)

## Console Input
As you can see, while the program works as intended, it's not very user-friendly. Anyone running it would be lost because there's no context. What does 50 represent? Why that specific number? Why should the user care? In this simple case, maybe they shouldn't. But what if we wanted to print a different number? Do we have to close the program, edit the code, and recompile it every time? Of course not. We can allow the user to enter their own value. Let's modify the program.

```cpp
#include <iostream>

int main()
{
    int number{};
    std::cin >> number;
    std::cout << number;
    return 0;
}
```
This code tells the compiler: "Create a variable called `number`, use `std::cin` to get input from the user, store it in the `number` variable, and then use `std::cout` to print the value back to the screen."
![Console Input](/img/blog/cin.gif)
<Br>While this program also works, the lack of clarity persists. If anything, it's now worse.
If we showed this program to someone, all they would see is a blank window with a blinking cursor. They would have no idea what to do.
Out of curiosity, they might type a number and press Enter, only to see that same number printed back. What's the point? Although we've moved a step forward in our quest to understand input and output, we still need to do a little more work to make our program useful. The next step is to include another library in our program.

## The C++ String Library
![String Library](/img/blog/stringlib.jpeg)
A string in C++ is the Standard Library's way of handling and storing text. Much like the `number` variable we've been using, a string can be thought of as a variable that holds a sequence of characters, such as a word or a sentence. To use strings, we need to include the string library in our project.
```cpp
#include <string>
```
But just including the header file doesn't change anything on its own. We need to declare some string variables and put them to use.
```cpp
#include <iostream>
#include <string>

int main() {
  std::string username{};
  std::cout << "Please, tell me your name and press the enter key: ";
  std::cin >> username;
  std::cout << "Welcome " << username << "! Now please enter a number: ";
  int number{};
  std::cin >> number;
  std::cout << username << ", the number that you entered is " << number << '\n';
  return 0;
}
```
As you can see, this program is a bit more complex. In essence, we're telling the compiler: "Create a `string` variable called `username`, print a message asking for the user's name, then wait for the user to enter their name and store it in the `username` variable. After that, print another message greeting them by name and asking them to enter a number. Store that input in the `number` variable. Finally, print a message confirming the number they entered."
So, what does this accomplish? We have a program that works, but it has nothing to do with the unit converter I promised. You're right. These exercises don't perform any unit conversion. However, they bring us much closer to our goal. Now we know how to take input and produce output; we just need to apply these tools for unit conversion.

## Defining The Problem
Our task is to write a program that converts a given number of meters into feet. This obviously requires a calculation.
First, we need to understand the calculation ourselves before we can translate it into code.
The conversion formula is `1 meter = 3.28 feet`. To convert 20 meters to feet, for example, we simply multiply 20 by 3.28.
Now, let's translate this into code. We don't need to start from scratch; we can reuse the code we've already written with just a few adjustments.
```cpp
#include <iostream>
#include <string>

int main() {
  std::cout << "Welcome to this simple meters to feet converter, please tell "
               "me your name and press the enter key: ";
  std::string username{};
  std::cin >> username;
  std::cout
      << "Hello " << username
      << "! Please enter the number of meters you want to convert into feet: ";
  float meter_units{};
  std::cin >> meter_units;
  float converted_units{meter_units * 3.28f};
  std::cout << meter_units << " meters are " << converted_units << " feet\n";
  std::cout << "Thank you for using this program!\n";
  return 0;
}
```
![Meters to Feet](/img/blog/meterstofeet.gif)
<Br>Now we have a working program that does exactly what we set out to do.
There are still some issues, though. The program doesn't validate the input, so if a user enters a letter instead of a number, something unexpected will happen. Similarly, the program will behave unexpectedly if the user enters a first and last name (with a space). Furthermore, the program only runs once. We'll tackle these issues in the next part of this series.
# References / Sources

- [C++ Standard Library](https://en.cppreference.com/w/cpp/standard_library.html)
- [The iostream library](https://en.cppreference.com/w/cpp/header/iostream.html)
- [The string library](https://en.cppreference.com/w/cpp/header/string.html)