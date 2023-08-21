
1. [x] outline of article
2. [x] code to generate cycles from TreeMap
3. ~~[ ] CANCELLED: write code to draw graph just before loop detected
    (too many race conditions)~~
4. [x] draw it using graphviz
   1. [x] add colours
5. [x] sample stacktrace
6. [ ] link code to article
   1. [ ] simple experiment
   2. [ ] printing out cycle
   3. [ ] grpc experiment
7. [x] come up with a bigger example that will produce a prettier graph
8. ~~[ ] CANCELLED: animation for red-black tree deep dive~~
9.  ~~[ ] CANCELLED: solid 2 thread code interleaving that generates the loop?~~
    1.  ~~[ ] do it by hand~~
    2.  ~~[ ] generate it with logging~~
        1.  ~~[ ] wouldn't logging with synchronized make the issue less likely to happen due to locking introduced?~~
10. ~~[ ] CANCELLED: simple diagrams that demonstrate the concurrent rotation issue~~
11. [ ] spring-rest/grpc realistic example
    1.  [x] get grpc protobuf generating
    2.  [ ] implement service
    3.  [ ] implement client-server main method
    4.  [ ] write up situation in blog post
12. [ ] high level why it happens with two threads executing and tree rotation
    pseudo code hint is the NPE
13. [ ] different languages that also have the issue (as long as you can catch NPE)
    1. [ ] research
        1. [x] javascript
        2. [x] python
        3. [x] typescript
        4. [x] java
        5. [x] C#
        6. [x] C++
        7. [x] PHP
        8. ~~[ ] C~~
        9. [x] Go
        10. [x] Rust
        11. [x] kotlin
        12. [x] Ruby
    2. [ ] implement the ones that are possible
        1. ~~[ ] javascript~~
        2. ~~[ ] python~~
        3. ~~[ ] typescript~~
        4. [x] java
        5. [ ] C#
        6. [ ] C++
        7. ~~[ ] PHP~~
        8. ~~[ ] C~~
        9. [ ] Go
        10. ~~[ ] Rust~~
        11. ~~[ ] kotlin~~
        12. [ ] Ruby
    3. [ ] link to implementations
        1. ~~[ ] javascript~~
        2. ~~[ ] python~~
        3. ~~[ ] typescript~~
        4. [ ] java
        5. [ ] C#
        6. [ ] C++
        7. ~~[ ] PHP~~
        8. ~~[ ] C~~
        9. [ ] Go
        10. ~~[ ] Rust~~
        11. [ ] kotlin
        12. [ ] Ruby
14. [ ] easy solution: locking/monitors
15. [ ] controversial solution using lg(N) extra memory
    1.  [ ] high level description
    2.  [ ] drawbacks
    3.  [ ] implementation
    4.  [ ] re run experiment with safe TreeMap
16. [ ] conclusion
17. clean up article