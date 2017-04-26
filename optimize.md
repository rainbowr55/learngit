###优化笔记

1. Boxing/unboxing to parse a primitive

   A boxed primitive is created from a String, just to extract the unboxed primitive value. It is more efficient to just call the static parseXXX method.
   Integer.ValueOf()改成Integer.PriseInt()
2. 循环中拼接String
3.Write to static field from instance method
