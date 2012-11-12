# unsession


`comp` is a Clojure function that takes 2 or more functions and returns a function that is a composition of those functions. The output of one function is used as the input to the next function in the chain

`(def f (comp (partial * 2) inc))`

Now, write two functions. The first takes an integer and returns vector that contains that integer times 2 and times 5.

`(defn f [x] (vector (* 2 x) (* 5 x)))`

The second takes an integer and returns a vector that contains that integer added to 2 to it and added to 5.

`(defn g [x] (vector (+ 2 x) (+ 5 x)))`

The type signatures of these functions can be written as:

`int -> [int]`

What happens when you try to `comp` these functions?

`(def h (comp g f))`

So write a function that will correctly compose functions of this type. Actually, let's be a little more general and write a function that will take the output of these functions and correctly feed it another one. The return value should be a vector of integers. So the type of this function will look like:

`[int] -> (int -> [int]) -> [int]`

We'll name this function `bind` for reasons to be explained later. One caveat, you cannot use Clojure's `mapcat` or `for` statements.

`(defn bind [mv F]
     (apply concat
            (map f mv)))`

Then see how that function works:

`(bind [4 2] f)`

`(bind [4 2] g)`

`(bind (f 3) g)`

`(bind (f 2) f)`

Notice:

`(bind (vector 4) f)`

Which is a long way to write:

 `(f 4)`

So:

`(= (bind (vector 4) f) (f 4))`

Another thing to notice is that:

`(= (bind (vector 4) vector) (vector 4))`

And one more thing. What is the result of:

`(bind (bind (vector 4) f) g)`

And what is the result of:

`(bind (vector 4) (fn [x] (bind (f x) g)))`

So the way `bind` and `vector` work, they satisfy these 3 equivalences:

`(= (bind (vector 4) f)            (f 4))`
`(= (bind (vector 4) vector)       (vector 4))`
`(= (bind (bind (vector 4) f) g)   (bind (vector 4) (fn [x] (bind (f x) g))))`

So if you have two functions like `vector` and `bind` that satisfy those three equivalences, you have a monad. And those equivalences are the tree monad laws. Sometimes when working with monads, functions like `vector` are renamed to `result` or `unit`. The idea being that it is a function that accepts any value and puts it into, or wraps it with, a monadic value. So `[4]` is a monadic value that is a result of `(vector 4)`. Also, realize that while we've only been working with integers inside our monadic values, values of any type could be contained inside them. One more point to see is that `bind` always returns a monadic value. In this case, a vector of integers.

So lets look at another of Clojure's container types. The hash set.

`(hash-set 4)`

Write two functions similar to the ones above that take an integer and return a hash-set of integers.

`(defn f [x] (hash-set (* 2 x) (* 5 x)))`
`(defn g [x] (hash-set (+ 2 x) (+ 5 x)))`

Now write a corresponding `bind` function. Here's a hint, it's the exact same as the `bind` function for `vector` with a single change.

`(require '[clojure.set :refer :all])
 (defn bind [mv f]
      (apply union
             (map f mv)))`

You can play around with `hash-set`, `bind`, `f` and `g` to see how they all work. Also, try out the three monad laws and see if `hash-set` and this `bind` satisfy them.

Pay particular attention to this expression as compared to the same one for 'vector'.

`(bind (f 2) f)`

Now let's take a look at another monad. Suppose we made the rule that our monadic values were vectors that could only contain at most 1 integer. Now write a function that takes an integer and if that integer is even, returns a vector of that integer incremented by one. But if it's odd, returns an empty vector.

`(defn f [x]
     (if (even? x)
        (vector (inc x))
        (vector)))`

Try using the `bind` function from before and you get a wrong answer. Why?

`(bind (vector 5) f)`
`(f 5)`

So let's write a valid `bind` function. We're also going to totally deconstruct all the various steps because of something we'll run into later.

`(defn bind [mv f]
     (let [v (first mv)
           new-mv (f v)
           new-v (first new-mv)]
        (if (nil? new-v)
           (vector)
           (vector new-v))))`

Here are the steps we take in this bind.

* unwrap the monadic value passed in to get the value inside it
* call 'f' with that value.
* unwrap the monadic value f returns
* create an appropriate monadic value as the result of `bind`

Now what other types of containers do we have a available in Clojure?

How about functions? Think about `constantly`

`(def f (constantly 5))`
`(f)`

Just so we have somethin to work with, write a function that takes an integer and returns a function of no arguments that returns the integer times 2 when it is called.

`(defn f [x] (fn [] (* 2 x)))`
`((f 5))`

How might we write a `bind` function for `constantly`?

* start be realizing that `bind` will need to return a function of no arguments
* then, realize that the monadic value passed in is a function, so to 'unwrap' it, it has to be called
* take the unwrapped value and call 'f' with it
* unwrap the monadic value 'f' returns by calling it

`(defn bind [mv f]
     (fn []
        (let [v (mv)
              new-mv (f v)]
          (new-mv))))`

I call this the 'Useless Monad' because it doesn't do anything useful. But it does show us how write a bind function when our monadic values are functions. Proving the monadic laws for this monad is a little tricky since you can't just do an '='s operation on functions. But you can still execute the functions and see that the return values are the same.

So, we see how to use functions as monadic values when the have no arguments. What might we accomplish if we could pass arguments to these monadic values? Here's one variant of a `result` function.

`(defn result [x]
    (fn [state]
       [x state]))`

Nothing too complicated, it just takes a value 'x' and returns a function which takes a value 'state' and returns a vector containing 'x' and 'state'. Notice that 'state' can be any type of value, just like 'x' can.

Now let's write a function that takes an integer and returns a function that increments the integer, while also adding some information the state that is passed in.

`(defn f [x]
    (fn [state]
       [(inc x) (conj state :inc)]))`

`((f 5) [])`

And another that doubles the integer.

`(defn g [x]
    (fn [state]
       [(* 2 x) (conj state :doubled)]))`

`((g 4) [])`

So how would we write a `bind` for such a `result` function?

* start by realizing that `bind` will need to return a function which takes a 'state' argument
* then, realize that the monadic value passed in is a function, so to 'unwrap' it, it has to be called with a state value
* the result is going to be a vector with an unwrapped value and a state, so destructure that
* take the unwrapped value and call 'f' with it
* unwrap the monadic value 'f' returns by calling it with the unwrapped state value

`(defn bind [mv f]
    (fn [state]
       (let [[v new-state] (mv state)
             new-mv (f v)]
          (new-mv new-state))))`

`(def h (bind (f 5) g))`
`(h [])`
`(h #{})`

`(def h (bind (f 4) f))`
`(h [])`
`(h #{})`

That is the state monad, or a variant of it.

Let's go back to the vector monad.

`(defn bind [mv f]
    (mapcat f mv))`

What if we have two vectors of ints and we want a vector all possible pairs of integers from them? We could use `bind` and `vector` to do this.

`(def odds [1 3 5 7])`
`(def evens [2 4 6 8])`
`(bind odds (fn [x]
                (bind evens (fn [y]
                                (vector [x y])))))`

The interesting thing to see here is that if we rename `vector` to `result`, this expression would work for any monad which had a `bind` and `result` defined. We could capture this in a macro.

`(defmacro m-do [result
                 [sym1 mv1
                  sym2 mv2]
                 expression]
    `(bind ~mv1 (fn [~sym1]
                    (bind ~mv2 (fn [~sym2]
                                   (~result ~expression))))))`

`(m-do vector
     [x odds
      y evens]
    [x y])`

Which is exactly the same as

`(for [x odds
       y evens]
     [x y])`

So `bind` is used to bind the values inside monadic values to symbols so they can be used in expressions.

Here's one more monad.

`(def result identity)`
`(defn bind [mv f] (f mv))`

`(m-do result
       [x 19
        y (+ x 5)]
     [x y])`

Which is exactly the same as.

`(let [x 19
       y (+ x 5)]
    [x y])`

Thus ends the discussion.
