+++
title = "Input Basics In C++ Part 3 - Multiple Inputs and Calculations"
date = "2025-11-11T18:44:42+02:00"
draft = false
type = "blog"
tags = ["programming", "posts", "articles", "C++", "input"]

[images]
    featured_image = "/img/blog/inputbasics.jpeg"
+++

![Input Validation](/img/blog/input_validation.png)
In [Part 2](InputBasicsCPP-2) we covered some basic principles about standard input in C++, we talked about the `input buffer stream`, the `iostate` and some mechanisms that the C++ programming language provides us with to get input from users. We ended with a program that takes input from users and stores them, but that had the fatal flaw of only being able to store a single number, which really doesn't make for a good calculator, does it?
So here are a few changes.

```cpp
#include <iostream>
#include <limits>
#include <string>
#include <vector>

int main() {
  std::cout << "Welcome to the mean calculator program, please enter your name: ";
  std::string username{};
  std::getline(std::cin, username);
  std::cout << username << ", please enter the numbers you want to calculate the mean for.\n";
  std::vector<double> numbers{};
  double number{};
  std::cout << username << " please enter a number to be stored: ";
  while (std::cin >> number) {
    std::cout << username << " please enter a number to be stored: ";
    numbers.push_back(number);
  }
  std::cin.clear();
  std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
  std::cout << username << ", the number you entered was: " << number;
  return 0;
}
```
![Mean Error 10](/img/blog/mean_error_10.gif)
<Br>With the current changes the program now constantly asks the user for input and only stop when the user enters something that is NOT a number, which for our case works conveniently. There is still another issue, though. At the end of the program, it says that the number entered was 0, but how could this be if we input a lot of numbers? Well, actually the program is doing what it is supposed to be doing. The value of `number` will be updated to the last input the user entered, however as soon as the input becomes something that is not a number, the `failbit` flag is set, input operations stop and `number` is set to 0, we break out of the loop to continue the execution of the program as it should. There is however one issue, if the user hits the return key without entering any value, he program seems to not do anything.

![Mean Error 11](/img/blog/mean_error_11.gif)
# The Whitespace Dilemma
<Br>Hmmm, it seems that we made a mistake, but once again the program is doing what it is supposed to do. The issue lies in how `operator>>` works for numeric types. When the users presses the return key without entering any input `operator>>` will skip any leading whitespace (spaces, tabs, `\n`). At this point the stream buffer is still empty, so essentially the program is still waiting for input. The loop has not executed because `std::cin >> number` has not completed yet, it is still waiting for input. Here is a visual example of what's going on.
```md
You type:  [Enter]
Stream:    '\n' → [operator>> skips it] → [buffer empty, waiting...]
           ↑
           Cursor waits here for numeric input
```
This design in the library allows for flexible whitespace in numeric input. For example:
```md
The following will all work the same:
42
    42
               42

42 43 44
```
![Mean Error 12](/img/blog/mean_error_12.gif)
<Br>
# Keeping it in the Loop
For this particular exercise I will leave the program as is as I don't consider it a breaking bug. The best I can do for now without off topic is just provide better instructions to the users, so that they know how to continue or quit the program, there is another issue that is more pressing and that is that none of the numbers in the vector are printed because we have not told the program to do so. I mean, yes we are putting values inside a vector, but we are not really doing anything with that vector and here is where we are introduced to another type of loop, the `Ranged-For` loop. This loop will basically tell the computer, "iterate through the container, visiting each element in sequence".
```cpp
#include <iostream>
#include <limits>
#include <string>
#include <vector>

int main()
{
    std::cout << "Welcome, please enter your name: ";
    std::string username{};
    std::getline(std::cin, username);
    std::cout << "Welcome" << username
              << "! Enter numbers (one per line) to calculate mean, min and max.\n When done, type any letter\n\n";
    std::vector<double> numbers{};
    double number{};
    std::cout << username << " please enter the first number to be stored: ";
    while (std::cin >> number)
    {
        std::cout << "You entered: " << number << '\n';
        std::cout << username << " please enter the next number to be stored: ";
        numbers.push_back(number);
    }
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    std::cout << username << ", the values you entered are:\n";

    //This is the modern way of iterating over all elements in a container
    for (const auto &num : numbers)
    {
        std::cout << " " << num << '\n';
    }
    return 0;
}
```
When we run this program, this is what will happen.
![Mean Error 13](/img/blog/mean_error_13.gif)
# It's Showtime!
At this point I believe we are ready for the next step, which is actually performing the calculations, so that the program can do what it says it can. That is, calculate the mean, the max and the min value from the vector of numbers we have.

# Calculating Max and Min
To calculate the max and the min we have two options. We either write code that will iterate over the vector, checking value by value to see which of them is the highest, then do the same again but this time to see which one is the smallest OR we can make consult the standard library to see if there is already a built in feature that can do that for us. Luckily for us, the Standard Library includes battle-tested algorithms. We'll use `std::max_element` and `std::min_element` from the `<algorithm>` header.
```cpp
#include <algorithm> //This is what we need to access std::min_element and std::max_element
#include <iostream>
#include <limits>
#include <string>
#include <vector>

int main()
{
    std::cout << "Welcome, please enter your name: ";
    std::string username{};
    std::getline(std::cin, username);
    std::cout << "Welcome" << username
              << "! Enter numbers (one per line) to calculate mean, min and max.\nWhen done, type any letter\n\n";
    std::vector<double> numbers{};
    double number{};
    std::cout << username << " please enter the first number to be stored: ";
    while (std::cin >> number)
    {
        std::cout << "You entered: " << number << '\n';
        std::cout << username << " please enter the next number to be stored: ";
        numbers.push_back(number);
    }
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    std::cout << username << ", the values you entered are:\n";
    
    if(numbers.empty())
    {
      std::cout << "No numbers entered.\n";
      return 0;
    }
    
    for (const auto &num : numbers)
    {
        std::cout << " " << num << '\n';
    }

    // We use std::max_element which returns a *position* in the vector
    auto max_pos{std::max_element(numbers.begin(), numbers.end())};
    //The * extracts the actual value at that position.
    double max_value{*max_pos};

    // We use std::min_element
    auto min_pos{std::min_element(numbers.begin(), numbers.end())};
    double min_value{*min_pos};

    // We print the values to the screen.
    std::cout << "The Max is: " << max_value << '\n';
    std::cout << "The Min is: " << min_value << '\n';

    return 0;
}
```
And this is the result after running the code.
![Mean Error 14](/img/blog/mean_error_14.gif)
<br>I understand that we added a lot of new things in here, `algorithm`, `min_value`, `max_value`. How does any of this work?
I get it, I want answers too, but this post is about input.

# Calculating the Mean
At this point calculating the mean can be done with another function called `std::accumulate` and to be able to use it we must include another header file, `<numeric>`.
```cpp
#include <algorithm> //This is what we need to access std::min_element and std::max_element
#include <iostream>
#include <limits>
#include <numeric> //This is what we need to access std::accumulate
#include <string>
#include <vector>

int main()
{
    std::cout << "Welcome, please enter your name: ";
    std::string username{};
    std::getline(std::cin, username);
    std::cout << "Welcome" << username
              << "! Enter numbers (one per line) to calculate mean, min and max.\nWhen done, type any letter\n\n";
    std::vector<double> numbers{};
    double number{};
    std::cout << username << " please enter the first number to be stored: ";
    while (std::cin >> number)
    {
        std::cout << "You entered: " << number << '\n';
        std::cout << username << " please enter the next number to be stored: ";
        numbers.push_back(number);
    }
    std::cin.clear();
    std::cin.ignore(std::numeric_limits<std::streamsize>::max(), '\n');
    std::cout << username << ", the values you entered are:\n";

    if (numbers.empty())
    {
        std::cout << "No numbers entered.\n";
        return 0;
    }

    for (const auto &num : numbers)
    {
        std::cout << " " << num << '\n';
    }

    // We use std::max_element
    auto max_pos{std::max_element(numbers.begin(), numbers.end())};
    double max_value{*max_pos};

    // We use std::min_element
    auto min_pos{std::min_element(numbers.begin(), numbers.end())};
    double min_value{*min_pos};

    // Note: We use 0.0 (not 0) as the starting value.
    // This ensures the result is a double, not an int.
    // Using 0 would truncate the sum to an integer!
    double sum{std::accumulate(numbers.begin(), numbers.end(), 0.0)};
    double mean{sum / numbers.size()};
    // We print the values to the screen.
    std::cout << "\n=== Statistics ===\n";
    std::cout << "The Max is: " << max_value << '\n';
    std::cout << "The Min is: " << min_value << '\n';
    std::cout << "The Mean is: " << mean << '\n';
    return 0;
}
```
And we get the following.
![Mean Error 15](/img/blog/mean_error_15.gif)

# What We've Accomplished
<br> In this series, we've built a complete input-driven program that:
- Handles text input with `std::getline`
- Reads multiple numbers with `std::cin >> number`
- Recovers from input errors using stream state management
- Stores data in vectors for processing
- Uses standard algorithms to perform calculations

# What's Next?
You've probably noticed we used some new syntax:
- `.begin()` and `.end()` - What are these?
- The `*` operator with `max_pos` - What does this mean?
- How do algorithms like `std::max_element` actually work?

These involve **iterators** - the mechanism that connects containers and algorithms. We'll explore this in the next post, where we'll finally understand the "plumbing" behind the Standard Library's generic algorithms.

Until then, experiment with the code! Try adding more statistics, handling edge cases, or even writing your own min/max calculation to see how the algorithm does it internally.

# References / Sources
- [C++ Vector](https://en.cppreference.com/w/cpp/container/vector.html)
- [The algorithm library](https://en.cppreference.com/w/cpp/algorithm.html)