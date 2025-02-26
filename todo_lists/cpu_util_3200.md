
1. [x] outline of article
2. [x] code to generate cycles from TreeMap
3. ~~[ ] CANCELLED: write code to draw graph just before loop detected (too many race conditions)~~
4. [x] draw it using graphviz
   1. [x] add colours
5. [x] sample stacktrace
6. [x] link code to article
   1. [x] simple experiment
   2. [x] printing out cycle
   3. [x] executor experiment
   3. [x] grpc experiment
7. [x] come up with a bigger example that will produce a prettier graph
8. ~~[ ] CANCELLED: animation for red-black tree deep dive~~
9.  ~~[ ] CANCELLED: solid 2 thread code interleaving that generates the loop?~~
    1.  ~~[ ] do it by hand~~
    2.  ~~[ ] generate it with logging~~
        1.  ~~[ ] wouldn't logging with synchronized make the issue less likely to happen due to locking introduced?~~
10. ~~[ ] CANCELLED: simple diagrams that demonstrate the concurrent rotation issue~~
11. [x] spring-rest/grpc realistic example
    1.  [x] get grpc protobuf generating
    2.  [x] implement service
    3.  [x] implement client-server main method
    4.  [x] write up situation in blog post
12. [x] threadpool with swallowed stack trace realistic example
13. [x] look into GRPC NPE. I thought NPEs make it to the logs. not sure what happened in my example. i'm not seeing NPE in standard out.
14. [x] high level why it happens with two threads executing and tree rotation
    pseudo code hint is the NPE
15. [ ] different languages that also have the issue (as long as you can catch NPE)
    1. [x] research
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
    2. [x] implement the ones that are possible
        1. ~~[ ] javascript~~
        2. ~~[ ] python~~
        3. ~~[ ] typescript~~
        4. [x] java
        5. [x] C#
        6. [x] C++
        7. ~~[ ] PHP~~
        8. ~~[ ] C~~
        9. [x] Go
        10. [x] Rust
        11. ~~[ ] kotlin~~
        12. [x] Ruby
16. [x] easy solution: locking/monitors
17. [x] controversial solution using lg(N) extra memory
    1.  [x] high level description
    2.  [x] drawbacks
    3.  [x] implementation
    4.  [x] re run experiment with safe TreeMap
    3.  [x] Add diff viewer somehow
18. [x] conclusion
19. [x] related work
20. ~~[ ] include source code under a detail tag in addition to a link the repo~~ jekyll wasn't liking detail tags
20. [x] include source snippets with link
    1. [x] Experiment: SimpleRepro
    1. [x] Experiment: Generate Graph
    1. [x] Real: Executor
    1. [x] Real: gRPC
    1. [x] Languages: Java
    1. [x] Languages: C#
    1. [x] Languages: Ruby
    1. [x] Languages: Go
    1. [x] Languages: C++
    1. [x] Languages: Rust
21. [x] fix link in related worked
22. [x] Fix language table
23. [ ] clean up article
    1. [x] first skim expand existing content
        1. [x] Intro
        1. [x] Experiment
        1. [x] Related Work
        1. [x] Realistic
        1. [x] How
        1. [x] Other langs
        1. [x] Easy fix
        1. [x] Controversial fix
        1. [x] Layered
        1. [x] Conclusion
    1. [x] spell check
    1. [x] proof read 1
        1. [x] Intro
        1. [x] Experiment
        1. [x] Related Work
        1. [x] Realistic
        1. [x] How
        1. [x] Other langs
        1. [x] Easy fix
        1. [x] Controversial fix
        1. [x] Layered
        1. [x] Conclusion
    1. [x] proof read 2
        1. [x] Intro
        1. [x] Experiment
        1. [x] Related Work
        1. [x] Realistic
        1. [x] How
        1. [x] Other langs
        1. [x] Easy fix
        1. [x] Controversial fix
        1. [x] Layered
        1. [x] Conclusion
24. [x] send for review
25. [x] determine commands to transfer to public repo
```
cd java-by-experiments
git remote add private ../java-by-experiments-private/.git
git fetch private
git checkout private/main -b private-main
git checkout main
git merge private-main
git branch -d private-main
git remote remove private
git push

cd josephmate.github.io
git remote add private ../josephmate.github.io-private/.git
git fetch private
git checkout private/main -b private-main
git checkout master
git merge private-main
git branch -d private-main
git remote remove private
git push
```
26. [x] push to public repos
27. [x] test links to public experiment repo
    1. [x] SimpleRepro
    1. [x] java
    2. [x] C#
    3. [x] C++
    4. [x] Go
    5. [x] Rust
    6. [x] Ruby
    7. [x] Link to ProtectedTreeMap
27. [x] report bug to jekyll about using detail tag. for now just include source snippets with link
    1. already reported: https://github.com/jekyll/jekyll/issues/9297

