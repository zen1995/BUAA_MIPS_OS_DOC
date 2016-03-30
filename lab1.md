OS P1
===================

## Thinking 2.1
  1.  <code>git checkout printf.c</code>
  2.  <code>git checkout printf.c</code>
  3.  <code>git rm chached Tucao.txt</code>  

## Thinking 2.2
  1. git clone只能clone远程库的master分支，无法clone所有分支
  2. true
  3. true
  4. true

##  实验难点

###  Exercise 2.2
  1.  >填写tools/scse0_3.lds 中空缺的部分，将内核调整到正确的位置上

      程序中 '.'代表一种自增关系
  2.  验证证明这个内核的填写位置至少在p1中是不重要的。因为我参考了几个人的代码，填写位置并不完全一样，但最终都通过了测试 。。。。。

### Exercise 2.3
  1.  >完成boot/start.S 中空缺的部分。设置栈指针，跳转到main 函数。

      用个lui就达成目的了。

      最后需要 j main
      但是买又找到main 标签在哪里。
      如果没有j main 的话，就会陷入后面的loop死循环。
      然而听说把后面的loop循环删掉，也可以成功进入main函数。。。。(￣y▽￣)╭

### Exercise 2.4
  1.  >阅读相关代码，并补全lib/print.c 中lp_Print() 函数中缺失的部分来
      >实现字符输出

      这一部分首先费城让人费解的就是其中那几个参数尤其是arg 和fmt.
      不过这一部分是我认为较轻松的一个部分，只要你弄懂了fmt是什么东西，
      剩下的按照注释做就好了。非常感人的是这一部分的注释写的相当详细。

##  实验体会
  1. 做实验的时候简直是一脸懵逼。感觉无从下手，不断地去麻烦别人，然后不断地看指导书才
  慢慢懂了一点。但是做完试验后，回顾这个实验内容，每个题目的答案看起来确可以顺理成章
  地出来。我觉得这主要是因为每个实验可能需要了解的不只有一个内容，而这些的相互关系
  并不一定是拓补的，造成了刚开始无从下手的感觉。
  2.  就算是什么都不懂，也可以不必急着去抱大腿，可以尝试着写一点，尽管有可能非常不对
  ，每写一点就能感觉到稍微理解了一些。
  3.  还有就是要学会模仿，指导书前面的教学代码可以对实验起很大的帮助