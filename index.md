  1 This is going to be an old school txt blog without all the modern day flashy stuff. Mainly cause i want ppl 
  2 to be able to read it in a terminal or a browser without all the fluff. 
  3  
  4 Its also going to be a live txt file which i just push up with whatever the current changes are. So it  
  5 will evolve as I get time to work on it. 
  6  
  7 C++ Array Type 
  8  
  9 class is std::array<T, N> 
 10  
 11 Construction:  
 12  
 13 The values are allocated on the stack and are there for default constructed so the type T needs to have a default 
 14 constructor. 
 15  
 16 Default constructor ie std::array<int, 3> default initializes each object. ints etc have non determinate values 
 17                        std::array<int, 3> = {}; Zero initializes each member 
 18  
 19 Copy ctor: T and N must match. Copies each value to new array. Obviously cant just swap pointers like std::vector<> 
 20  
 21 Swap: 
 22 array<T,N>::swap(Array<T,N> & ) this is linear in time as pointers cant be swap and each value must be copied. 
 23  
 24 Size and Empty: 
 25 std::array<T,N>::size() just returns N which is how many elements the array has, since the size is fixed and  
 26 cannot be resized; 
 27 std::array<T,N>::empty() whether N == 0, ie size() = 0 and begin() would equal end() 
 28 
