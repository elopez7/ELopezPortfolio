+++
title = "Input Basics In C++ Part 3 - Validation"
date = "2025-10-11T18:44:42+02:00"
draft = true
type = "blog"
tags = ["programming", "posts", "articles", "C++", "input"]

[images]
    featured_image = "/img/blog/inputbasics.jpeg"
+++

![Input Validation](/img/blog/input_validation.png)
In [Part 2](InputBasicsCPP-2) 

```cpp
#include <iostream>  //Tell the compiler we want to include the vector header file
#include <string>
#include <vector>  //Tell the compiler we want to include the vector header file

// This function is the entry point of the application.
int main() {
  std::cout << "Welcome to the mean calculator program, please enter your name: ";
  std::string username{};
  std::getline(std::cin, username);
  std::cout << username << ", please enter the numbers you want to calculate the mean for.\n";
  std::vector<float> numbers{};
  float number{};
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
<Br>With the current changes the program now constantly asks the user for input and only stop when the user enters something that is NOT a number, which for our case works conviniently. There is still another issue, though. At the end of the program, it says that the number entered was 0, but how could this be if we input a lot of numbers? Well, actually the program is doing what it is supposed to be doing. The value of `number` will be updated to the last input the user entered, however as soon as the input becomes something that is not a number, the `failbit` flag is set, input operations stop and `number` is set to 0, we break out of the loop to continue the execution of the program as it should. None of the numbers in the vector are printed because we have not told the program to do so.

# References / Sources

- [C++ Standard Library](https://en.cppreference.com/w/cpp/standard_library.html)
- [The iostream library](https://en.cppreference.com/w/cpp/header/iostream.html)
- [The string library](https://en.cppreference.com/w/cpp/header/string.html)