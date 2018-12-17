  #Cpp Info
  
  This is going to be an old school txt blog without all the modern day flashy stuff. Mainly cause i want ppl 
  to be able to read it in a terminal or a browser without all the fluff. 
    
  Its also going to be a live txt file which i just push up with whatever the current changes are. So it  
  will evolve as I get time to work on it. 
   
  ##C++ Array Type 
   
  class is std::array<T, N> 
  
 Construction:  
   
 The values are allocated on the stack and are there for default constructed so the type T needs to have a default 
 constructor. 
  
 Default constructor ie std::array<int, 3> default initializes each object. ints etc have non determinate values 
                        std::array<int, 3> = {}; Zero initializes each member 
   
 Copy ctor: T and N must match. Copies each value to new array. Obviously cant just swap pointers like std::vector<> 
 Swap: 
 array<T,N>::swap(Array<T,N> & ) this is linear in time as pointers cant be swap and each value must be copied. 
  
 Size and Empty: 
 std::array<T,N>::size() just returns N which is how many elements the array has, since the size is fixed and  
 cannot be resized; 
 std::array<T,N>::empty() whether N == 0, ie size() = 0 and begin() would equal end() 
 
