---
layout: post
title: Tail Recursion In Clojure
date: 2015-03-23
---

The ability of lisp programs to manipulate themselves is beautiful. When we take into account the amount of the language that is implemeneted by manipulating itself, it becomes even more interesting. As a user of the language we also have the power to extend the language in interesting ways. I recently paired with [Colin Jones](https://github.com/trptcolin) on one such language extension.

### Immutable Iteration

One hurdle to overcome when switching from mutable to immutable languages is coming to terms with the fact that many of the statements in your programming toolbelt are no longer valid. How can it be possible to write a `while` loop when the expression that the loop is testing is immutable?

```java
void doSomething() {
  boolean isDone = false;
  while (!isDone) {
    isDone = true;
  }
}
```

The issue here is that mutating `isDone` inside of the loop is not allowed. But, if it's not possible to mutate `isDone`, then how does a program ever exit the loop? The answer in these cases is typically to reach for recursion. The same code in a tail recursive style might look like the following.

```java
void doSomething() {
  doSomething(false);
}

void doSomething(boolean isDone) {
  if (!isDone) {
    doSomething(true);
  }
}
```

Now, instead of mutating `isDone` there is a fresh value of `isDone` on each invocation of the function `doSomething`. One issue with recursing is that for more complex cases the runtime is adding a large amount of memory overhead. This is because it's adding stack frames with each recursive iteration. It is, however, possible to get the best of both worlds: immutable recursive iteration and constant memory usage.

### Tail Call Optimization

Given that _some_ recursive cases act as loops, we can optimize them by turning them into loops behind the scenes. Using macros and mutable constructs, it's possible to extend the language and give the programmer an immutable and recursive feel. The `loop`..`recur` construct achieves this, but its implementation is in the [compiler](https://github.com/clojure/clojure/blob/e03d787720a16fae37a2ec9afb3859a15e67b976/src/jvm/clojure/lang/Compiler.java#L6056). In this post we'll explore how to implement `loop`..`recur` outside of the compiler.

There are two pieces of state that we'll need to keep. First, `finished`, which will act as a sentinel. If we need to recur again (specifically if our invocation of the body ends with a `my-recur`), then we will flip finished to indicate that we are not finished. Second, we'll need state for `bindings`, which will hold the mutable bindings used in the loop. We'll use `atoms` to encapsulate the state throughout. So we might start with something like this:

```clojure
(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)])))
```

And then we can add the loop<sup><a href="footnote_1">[1]</a></sup>. We'll set `finished` to true so that it can act as a sentinel. For the time being the `while` loop will only run one iteration.

```clojure
(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)]
      (while (not @~finished)
        (reset! ~finished true)))))
```

Next, we'll need to execute the body of the loop. We can take the binding names (the even positions) of user bindings to build the function arguments and the binding values (the odd positions) as arguments to the function.

```clojure
(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)]
      (while (not @~finished)
        (reset! ~finished true)
        (apply
          (fn ~(vec (take-nth 2 user-bindings))
            ~@body
            (take-nth 2 (rest @~bindings))))))))
```

So now we've defined a really convoluted way to run the body passed in with arguments provided (which is what the `let` construct provides). We of course want to recurse in a tail call optimized manner so we'll need to add support for a `my-recur` keyword to achieve this end.

Also, we should define an example case to help visualize the end goal.

```clojure
(my-loop [a 1
          b 5]
  (if (= b 1)
    (prn "done!")
    (do
      (prn (str "a: " a " " "b: " b))
      (my-recur (inc a) (dec b)))))

; "a: 1 b: 5"
; "a: 2 b: 4"
; "a: 3 b: 3"
; "a: 4 b: 2"
; "done!"
```

The introduction of `my-recur` is the most difficult maneuver in this whole process. We'll need to do two things when we see `my-recur`: first we'll need to bind `finished` to false, since we want to continue recursing; and second we'll need to rebind `bindings` to the values supplied to `my-recur`. To aid us we'll use Zach Tellman's excellent AST walking library [riddley](https://github.com/ztellman/riddley). Riddley will provied a `walk-exprs` function, which needs three arguments. The first tells the walker whether it needs to edit a particular node or not. If the function returns false the node remains the same. If it returns true, in our case that will be whether or not the node is a `my-recur` statement, it will be rewritten. We'll start with the function that tells whether a node is an invocation of `my-recur`.

```clojure
(defn recur-statement? [expr]
  (and (seq? expr)
       (= (first expr) 'my-recur)))

(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)]
      (while (not @~finished)
        (reset! ~finished true)
        (apply
          (fn ~(vec (take-nth 2 user-bindings))
            ~@body
            (take-nth 2 (rest @~bindings))))))))
```

Next, we'll need to write a function that will rewrite `my-recur` to both reset `finished` to false and rebind `bindings` to the arguments to `my-recur`. This will be invoked to rewrite nodes for which `recur-statement?` has returned `true`.

```clojure
(defn recur-statement? [expr]
  (and (seq? expr)
       (= (first expr) 'my-recur)))

(defn recur-expression [loop-finished-sym loop-bindings-sym node]
 `(do
    (reset! ~loop-finished-sym false)
    (reset! ~loop-bindings-sym
      (vec (interleave (take-nth 2 @~loop-bindings-sym) (list ~@(rest node)))))
    nil))

(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)]
      (while (not @~finished)
        (reset! ~finished true)
        (apply
          (fn ~(vec (take-nth 2 user-bindings))
            ~@body
            (take-nth 2 (rest @~bindings))))))))
```

Now, we need to add the AST walking code into the while loop's anonymous function so that it can run over the user supplied `body`. We'll simply use the two functions `recur-statement?` and `recur-expression` to rewrite `my-recur` nodes.

```clojure
(defn recur-statement? [expr]
  (and (seq? expr)
       (= (first expr) 'my-recur)))

(defn recur-expression [loop-finished-sym loop-bindings-sym node]
 `(do
    (reset! ~loop-finished-sym false)
    (reset! ~loop-bindings-sym
      (vec (interleave (take-nth 2 @~loop-bindings-sym) (list ~@(rest node)))))
    nil))

(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)]
      (while (not @~finished)
        (reset! ~finished true)
        (apply
         (fn ~(vec (take-nth 2 user-bindings))
          ~@(riddley.walk/walk-exprs recur-statement? (partial recur-expression finished bindings) body))
         (take-nth 2 (rest @~bindings)))))))
```

Finally, we need to return the result of the last invocation. We'll introduce a third atom for holding state that will be set to the result of applying the body function. At the end of the while loop we will simply return this result.

```clojure
(defn recur-statement? [expr]
  (and (seq? expr)
       (= (first expr) 'my-recur)))

(defn recur-expression [loop-finished-sym loop-bindings-sym node]
 `(do
    (reset! ~loop-finished-sym false)
    (reset! ~loop-bindings-sym
      (vec (interleave (take-nth 2 @~loop-bindings-sym) (list ~@(rest node)))))
    nil))

(defmacro my-loop [user-bindings & body]
  (let [finished (gensym "finished")
        bindings (gensym "bindings")
        return-value (gensym "return-value")]
    `(let [~finished (atom false)
           ~bindings (atom '~user-bindings)
           ~return-value (atom nil)]
      (while (not @~finished)
        (reset! ~finished true)
        (reset! ~return-value
          (apply
           (fn ~(vec (take-nth 2 user-bindings))
            ~@(riddley.walk/walk-exprs recur-statement? (partial recur-expression finished bindings) body))
           (take-nth 2 (rest @~bindings))))
      @~return-value))))
```

### The Imitation of Immutability

While the `my-loop` macro involves a lot of state (even more state than a simple java `while` loop), we've hidden it behind the macro. In this way we've achieved both the goal of a recursive procedure and a procedure which will not add stack frames for each recursive iteration. From the perspective of users of `my-recur`, there is an imitation of immutability. In end, all immutable languages act this way, they're all running on von Neumann architectures after all.

<a name="footnote_1"></a>
1. [While is actually implemented with clojure's loop](https://github.com/clojure/clojure/blob/e03d787720a16fae37a2ec9afb3859a15e67b976/src/clj/clojure/core.clj#L6044). It is, of course, necessary to bootstrap some kind of looping mechanism from outside of the language.
